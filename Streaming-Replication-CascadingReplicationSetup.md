# Cascading Replication - Step by Step Complete Setup

To accomplish this, we need to setup a Primary Server, an upstream Standby Server and one or more Cascading Servers.

Below is detailed Step-By-Step guide to do it.

### 1. Prepare the Primary Server

* Install PostgreSQL on the Primary Server
```sh
sudo apt update
sudo apt install postgresql-13
sudo pg_lsclusters
```

* Add below parameters to postgresql.conf file

```sh
vi /etc/postgresql/13/main/postgresql.conf
# or check with "show config_file;"

listen_addresses = '*'
wal_level = replica
max_wal_senders = 10
wal_keep_size = '10GB'
hot_standby = on
archive_mode = on
archive_command = 'test ! -f /var/lib/postgresql/archivelog/%f && cp %p /var/lib/postgresql/archivelog/%f'
restore_command = 'cp /var/lib/pgsql/archivelog/%f %p'
```

```sh
su - postgres
mkdir /var/lib/postgresql/archivelog
```

* Add the IP addresses of Primary and Cascading servers to pg_hba.conf to allow replication connections to the Upstream Standby server

```sh
vi /etc/postgresql/13/main/pg_hba.conf
# or check with "show hba_file;"

host replication all 192.168.1.0/24 md5
```

* Restart the PostgreSQL to apply changes
```sh
pg_ctlcluster 13 main restart
pg_lsclusters
# OR
sudo systemctl restart postgresql-13
sudo systemctl status postgresql-13
```

* Verify parameters
```sql
select name, setting from pg_settings where name in ('listen_addresses','wal_level','max_wal_senders','wal_keep_size','hot_standby','archive_mode','archive_command','restore_command');
-- wal_keep_segments instead of wal_keep_size if version is below 13
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
sudo pg_lsclusters
```

* Stop the PostgreSQL and Empty the Data Directory
```sql
show data_directory;
show config_file;
show hba_file;
```
```sh
sudo pg_ctlcluster 13 main stop
pg_lsclusters
cd /var/lib/postgresql/13/main/
ls
rm -rf /var/lib/postgresql/13/main/*
ls -lrth
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

Ensure that these parameters are already added to postgresql.conf or postgresql.auto.conf, otherwise add these and restart Postgresql service.
```sh
cat postgresql.auto.conf

vi /etc/postgresql/13/main/postgresql.conf

listen_addresses = '*'
wal_level = replica
max_wal_senders = 10
wal_keep_size = '10GB'
hot_standby = on
archive_mode = on
archive_command = 'test ! -f /var/lib/postgresql/archivelog/%f && cp %p /var/lib/postgresql/archivelog/%f'
restore_command = 'cp /var/lib/pgsql/archivelog/%f %p'

su - postgres
mkdir /var/lib/postgresql/archivelog
```
```sql
select name, setting from pg_settings where name in ('listen_addresses','wal_level','max_wal_senders','wal_keep_size','hot_standby','archive_mode','archive_command','restore_command');
```

* Create the standby.signal file in Data Directory of Standby Server if it doesn't exist
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

### 4. Configure Cascading Standby Servers

* Install postgresql on Cascading Server and stop the service
```sh
sudo apt update
sudo apt install postgresql-13
pg_lsclusters
pg_ctlcluster 13 main stop
```

* Stop the PostgreSQL and Empty the Data Directory
```sql
show data_directory;
show config_file;
show hba_file;
```
```sh
sudo pg_ctlcluster 13 main stop
pg_lsclusters
cd /var/lib/postgresql/13/main/
ls
rm -rf /var/lib/postgresql/13/main/*
ls -lrth
```

* Add below parameters to postgresql.conf file

```sh
vi /etc/postgresql/13/main/postgresql.conf
# or check with "show config_file;"

listen_addresses = '*'
wal_level = replica
max_wal_senders = 10
wal_keep_size = '10GB'
hot_standby = on
archive_mode = on
archive_command = 'test ! -f /var/lib/postgresql/archivelog/%f && cp %p /var/lib/postgresql/archivelog/%f'
restore_command = 'cp /var/lib/pgsql/archivelog/%f %p'
```

```sh
su - postgres
mkdir /var/lib/postgresql/archivelog
```

* Take a base backup from the Upstream Standby Server
```sh
pg_basebackup -h <primary_ip> -D /var/lib/postgresql/13/main/ -U replicator -c fast -Fp (or -Ft -z) -Xs -P -R

# rsync it if command was run on Primary server
rsync -a basebackup/ postgres@<cascading server ip>:/var/lib/postgresql/13/main/
```

* Verify Standby parameters on Cascading Standby Server
```sh 
select name, setting from pg_settings where name in ('listen_addresses','wal_level','max_wal_senders','wal_keep_size','hot_standby','archive_mode','archive_command','restore_command');
```

* Create the standby.signal file in Data Directory of Cascading Standby Server if it doesn't exist
```sh
touch /var/lib/postgresql/13/main/standby.signal
```

* Verify primary_conninfo entry in postgresql.auto.conf.

There should be only one entry of primary_conninfo in postgresql.auto.conf i.e., Upstream Standby Server. If there is entry of Primary Server too, then remove/comment it.

* Start Cascading Standby Server and Verify Replication
```sh
pg_ctlcluster 13 main start
pg_lsclusters
psql
\l
\dt+
# OR
sudo systemctl restart postgresql-13
sudo systemctl status postgresql-13
```

* Verify Replication on Cascading Standby Server
```sql
select pg_is_in_recovery();
\x
select * from pg_stat_wal_receiver;
```

* Verify Replication on both Primary and Upstream Servers
```sql
-- Run this on both Primary and Upstream servers.
-- Primary will confirm connection/replication to Upstream and Upstream will confirm connection/replication to Cascasding server. 
\x
select * from pg_stat_replication;

-- Run below on Upstream Standby server. It'll confirm replication from Primary server.
\x
select * from pg_stat_wal_receiver;
```

* Run this on Primary and verify it on both Upstream and Cascading Standbys
```sql
create database mydb1;
\c mydb1
\dt
create table pj_table1(id serial primary key);
insert into pj_table1 values (default);
insert into pj_table1 values (default);
insert into pj_table1 values (default);
insert into pj_table1 values (default);
select * from pj_table1;
```

Good to read: https://www.mydbops.com/blog/setting-up-cascading-replication-in-postgresql