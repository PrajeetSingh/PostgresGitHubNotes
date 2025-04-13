# Cascading Replication - Step by Step Complete Setup

To accomplish this, we need to setup a Primary Server, an upstream Standby Server and one or more Cascading Servers.

Below is detailed Step-By-Step guide to do it.

### 1. Prepare the Primary Server

* Install PostgreSQL on the Primary Server
```sh
sudo apt update
sudo apt install postgresql-13
sudo pg_ctlcluster 13 main status
```

* Add below parameters to postgresql.conf file

```sh
listen_addresses = '*'
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
wal_keep_size = '10G'
```

Additional parameters
```sh
hot_standby = on
archive_mode = on
archive_command = 'test ! -f /var/lib/postgresql/archivelog/%f && cp %p /var/lib/postgresql/archivelog/%f'
restore_command = 'cp /var/lib/pgsql/archivelog/%f %p'
```

```sh
mkdir /var/lib/postgresql/archivelog
```

* Add the follow to pg_hba.conf to allow replication connections to the Standby server IP address

```sh
host replication all 192.168.1.0/24 md5
```

* Restart the PostgreSQL to apply changes
```sh
pg_ctlcluster 13 main restart
sudo pg_ctlcluster 13 main status
# OR
sudo systemctl restart postgresql-13
sudo systemctl status postgresql-13
```

### 2. Create a Replication Role

* Login to Primary Server and create a Replication Role
```sql
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'your_secret_password';
```

### 3. Setup the Upstream Standby Server

* Install PostgreSQL on Standby Server
```sh
sudo apt update
sudo apt install postgresql-13
sudo pg_ctlcluster 13 main status
```

* Stop the PostgreSQL and Empty the Data Directory
```sql
show data_directory;
show config_file;
show hba_file;
```
```sh
cd /var/lib/postgresql/13/main/
rm -rf /var/lib/postgresql/13/main/*
```

* Take a base backup from the Primary Server to Data Directory

Login to the Standby Server and run below command to get the base backup of the Primary server
```sh
pg_basebackup -h <primary_ip> -D /var/lib/postgresql/13/main/ -U replicator -c fast -Fp (or -Ft -z) -Xs -P -R
```
 OR, you can take base backup locally and then rsync that to the Standby Server
 ```sh
rsync -a basebackup/ postgres@172.31.85.180:/var/lib/postgresql/13/main/
 ```

* Edit postgresql.conf on Standby Server

Ensure that these parameters are already added to postgresql.conf or postgresql.auto.conf, otherwise add these.
```sh
hot_standby = on
primary_conninfo = 'host=<primary_ip> port=5432 user=replicator password=your_password'
```

* Create the standby.signal file in Data Directory of Standby Server
```sh
touch /var/lib/postgresql/13/main/standby.signal
```

* Startup PostgreSQL service on Standby Server and check the data
```sh
pg_ctlcluster 13 main start
pg_ctlcluster 13 main status
pg_lsclusters
\l
\dt+
# OR
sudo systemctl restart postgresql-13
sudo systemctl status postgresql-13
```

* Verify that Replication is working fine between Primary and Standby servers
```sql
-- On Primary
\x
select * From pg_stat_replication;

-- On Standby
select pg_is_in_recovery();
\x
select * from pg_stat_wal_receiver;
```