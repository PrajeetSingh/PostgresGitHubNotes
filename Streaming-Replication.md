# Setting up Streaming Replication (PostgreSQL 13)
Streaming Replication (physical replication) is a byte by byte replication. It involves a continuous application of WAL records from Primary to Standby.

For every Replica/Standby, there exists a WAL Sender process on the Primary server and a WAL Receiver process on the Standby server in Streaming replication.

Starting from PostgreSQL 13, we don't use *recovery.conf* file unlike version 11 and earlier. Instead, the parameters that were initially added to the *recovery.conf* can now be set in *postgresql.conf* or *postgresql.auto.conf*.


## Steps to Setup Streaming Replication

**Step 1)** Create a replication user
```sql
CREATE USER replicator WITH REPLICATION PASSWROD 'MySecretPassword';
-- or
CREATE USER replicator WITH REPLICATION;
\du
```

**Step 2)** Below parameters are already set as expected by default
```sql
show wal_level; (replica)
show max_wal_senders; (10)
show hot_standby; (on)
```

**Step 3)** Continuous archiving is disabled by default. We need to enable it.
```sh
su - postgres
mkdir -p /var/lib/postgresql/archivelog	

psql
show data_directory;
show config_file;
show hba_file;
	
vi /var/lib/pgsql/12/data/postgresql.conf
listen_adrdesses = '*' or 'IP_address'
archive_mode = on
archive_command = 'test ! -f /var/lib/pgsql/archivelog/%f && cp %p /var/lib/pgsql/archivelog/%f'
(This means, check if archived log file already exists in /var/lib/pgsql/archivelog/, if not then copy it.)
```

**Step 4)** Verify and Setup other parameters if you need to on Primary cluster
```sql
select name, setting from pg_settings where name in ('listen_addresses','archive_mode','archive_command','wal_keep_segments','restore_command');
-- Instead of wal_keep_segments, we may need to setup wal_keep_size too. We can set it to 10 GB.
-- restore_command parameer is for automatic recovery of Replica using archived location.
-- Or you can setup these parameters
alter system set listen_addresses to '*';
alter system set restore_command to 'cp /var/lib/pgsql/archivelog/%f %p';
alter system set wal_keep_segments to '100';
alter system set wal_keep_size to '10G';
```

**Step 5)** Setup backup user in pg_hba.conf
```sh
vi /etc/postgresql/13/main/pg_hba.conf
#	Type	database	user		address		method
host	replication	replicator	192.168.1.0/24	md5

# Restart Postgres Cluster, as postgresql.conf is updated. If only pg_hba.conf is updated, then only reload is enough.	
systemctl restart postgresql-12.service or pg_ctl -D <data_directory_path> restart
or
sudo pg_ctlcluster 13 main restart
sudo pg_ctlcluster 13 main status
or
sudo pg_ctlcluster 13 main reload

pg_lsclusters
```

**Step 6)** Copy backup of Primary to Standby using pg_basebackup
```sh
# If running from Standby server
pg_basebackup -h <primary_ip> -U replicator -p 5432 -D /var/lib/postgresql/basebackup -c fast -Fp (or -Ft -z) -C -S myslot1 -Xs -P -R
# or
# If running locally on Primary server
pg_basebackup -h localhost -U replicator -p 5432 -D /var/lib/postgresql/basebackup -c fast (or --checkpoint=fast) -C -S myslot1 -Fp -Xs -P -R

# -C -S myslot1 if we are creating a new slot, just -S myslot1 if slot already exists.
```

**Step 7)** rsync the backup to data directory of Standby
```sh
# Remove all files in /var/lib/postgresql/13/main/ directory on Standby server first.
rsync -a basebackup/ postgres@172.31.85.180:/var/lib/postgresql/13/main/
#	copied directory should also have standby.signal file too on the Standby server
```

**Step 8)** Check postgresql.auto.conf on Replica Server
```sh
cat postgresql.auto.conf	
```

**Step 9)** Add/Verify the *primary_conninfo* setting in postgresql.conf or postgresql.auto.conf 

**Step 10)** Verify if *standby.signal* file exists
```sh
touch $PGDATA/standby.signal
```

**Step 11)** Startup Postgres service on Standby server
	you'll find that db and data is there in Standby now but it'll be Read Only
```sql
sudo pg_ctlcluster 13 main start
sudo pg_ctlcluster 13 main status
pg_lsclusters
\l
\dt+
```

**Step 12)** Test Replication
	It is good to use tools like pgwatch2 to monitor Replication but we can do it manually too.
```sql
--	On Primary:
-- psql
\x
select * from pg_stat_replication;
--	On Standby:
-- psql
\x
select * From pg_stat_wal_receiver;
select pg_is_in_recovery();
```

# Switchover/Failover

**On Primary**
```sql
-- Check if there is any Replication Lag.
select usename, client_addr, write_lag, replay_lag from pg_stat_replication;
select pg_switch_wal();
select usename, client_addr, write_lag, replay_lag from pg_stat_replication;
```

```sh
# Stop Primary Cluster
pg_lsclusters
pg_ctlcluster 13 main status
pg_ctlcluster 13 main stop
pg_lsclusters
```

**On Standby**
```sh
-- Promote Standby Cluster
pg_lsclusters
pg_ctlcluster 13 main status
pg_ctlcluster 13 main promote
pg_ctlcluster 13 main start
pg_ctlcluster 13 main status
```

```sql
select pg_is_in_recovery();
select * from pg_stat_wal_receiver;
```

**Recap:**

There are 4 things we need to do on Primary side to configure Replication
1. Create a replication user 
create user replicator with replication password 'password';
2. Change postgresql.conf to enable networking
listen_addresses = '*'
3. Allow remote access in pg_hba.conf
Type 	Database 	User 			Address 	Method
host	all			replication		<sandby_ip>	md5 or trust
4. Restart the Primary Server
systemctl restart postgresql-13.service or pg_ctl -D <data_directory_path> restart or pg_ctlcluster 13 main restart

Configuring the Replica Server
1. If Replica already exists, then Ensure that Postgres instance is stopped and the data directory is empty

2. Copy the backup of pg_basebackup to the Data Directory of Standby server

	pg_base_backup -h <primary_ip>/localhost -U replicator -p 5432 -D basebackup -c -Fp -Xs -P -R -C

	-C is to create the slot, it allows recyle of the Wal File only when Replica has fully consumed it.

3. Check postgresql.auto.conf on replica, it contains connection information to Primary
	cat postgresql.auto.conf

4. Start the Replica
```sh
pg_ctl -D <data_directory_path> start	
or
pg_ctlcluster 13 main start
```	

**Verify Replication**

On Primary:

```sql
psql
\x
select * from pg_stat_replication;
```
On Standby:

```sql
psql
\x
select * From pg_stat_wal_receiver;
select pg_is_in_recovery();
```