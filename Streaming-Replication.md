## Streaming Replication Setup
**Step 1)** Create a replication user
```sql
	create user replicator with replication password 'password';
	-- or
	create user replicator with replication;
```
**Step 2)** Below parameter are already set as expected by default
```sql
	show wal_level; (replica)
	show max_wal_senders; (10)
	show hot_standby; (on)
```
**Step 3)** Continuous archiving is disabled by default. We need to enable it.
```sh
	su - postgres
	mkdir -p /var/lib/postgresql/archivelog	
	show data_directory;
	show config_file;
	show hba_file;
	vi /var/lib/pgsql/12/data/postgresql.conf
	listen_adrdesses = '*' or 'IP_address'
	archive_mode = on
	archive_command = 'test ! -f /var/lib/pgsql/archivelog/%f && cp %p /var/lib/pgsql/archivelog/%f'
			(This means, check if archived log file already exists in /var/lib/pgsql/archivelog/, if not then copy it.)
	vi /etc/postgresql/13/main/pg_hba.conf
#	Type	database	user		address		method
	host	replication	replicator	192.168.1.0/24	md5

	# Restart Postgres Cluster, as postgresql.conf is updated. If only pg_hba.conf is updated, then only reload is enough.	
	systemctl restart postgresql-12.service or pg_ctl -D <data_directory_path> restart
	or
	sudo pg_ctlcluster 13 main restart
	sudo pg_ctlcluster 13 main status
	pg_lsclusters
```
**Step 4)** Copy backup of Primary to Standby using pg_basebackup
```sh
	pg_basebackup -h <primary_ip> -U replicator -p 5432 -D /var/lib/postgresql/basebackup -Fp -Xs -P -R -c fast -C -S myslot1
	or
	pg_basebackup -h localhost -U replicator -p 5432 -D /var/lib/postgresql/basebackup -c fast -C -S myslot1 -Fp -Xs -P -R

	# -C -S myslot1 if we are creating a new slot, just -S myslot1 if slot already exists.
```
**Step 5)** rsync the backup to data directory of Standby
```sh
	rsync -a basebackup/ postgres@172.31.85.180:/var/lib/postgresql/13/main/
#	copied directory should also have standby.signal file too on the Standby server
```
**Step 6)** Startup Postgres service on Standby server
	you'll find that db and data is there in Standby now but it'll be Read Only
```sql
	sudo pg_ctlcluster 13 main start
	sudo pg_ctlcluster 13 main status
	pg_lsclusters
	\l
	\dt+
```
**Step 7)** Check postgresql.auto.conf on Replica Server
```sh
	cat postgresql.auto.conf	
```
**Step 8)** Test Replication
	It is good to use tools like pgwatch2 to monitor Replication but we can do it manually too.
```sh
#	On Primary:
	psql
	\x
	select * from pg_stat_replication;
#	On Standby:
	psql
	\x
	select * From pg_stat_wal_receiver;
	select pg_is_in_recovery();
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
systemctl restart postgresql-12.service or pg_ctl -D <data_directory_path> restart

Configuring the Replica Server
1. If Replica already exists, then Ensure that Postgres instance is stopped and the data directory is empty

2. Copy the backup of pg_basebackup to the Data Directory of Standby server
	pg_base_backup -h <primary_ip>/localhost -U replicator -p 5432 -D basebackup -c -Fp -Xs -P -R -C
	-C means, recyle the Wal File only when Replica has fully consumed it.

3. Check postgresql.auto.conf on replica, it contains connection information to Primary
	cat postgresql.auto.conf

4. Start the Replica
	pg_ctl -D <data_directory_path> start	

Verify Replication
	On Primary:
	psql
	\x
	select * from pg_stat_replication;
	On Standby:
	psql
	\x
	select * From pg_stat_wal_receiver;
	select pg_is_in_recovery();