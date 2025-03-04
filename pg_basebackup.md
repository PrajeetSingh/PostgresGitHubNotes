# Backing up a database cluster using pg_basebackup

## Overview 

* Only the backups taken using pg_basebackup can be used for point-in-time recovery, not pg_dump or pg_dumpall.

* pg_basebackup can also be performed from the Standby and not just the Master. 

* pg_basebackup cannot run in Parallel mode and does not support Incremental or Differential backups.

## Getting ready

* In order to run a backup using pg_basebackup, it requires an empty directory.

* It is wise to have a Dated Directory created to store the backup for each day.

### Privileges required to run pg_basebackup

* User needs the REPLICATION role to be assigned to it and it must be created on Master DB.

* As the pg_basebackup runs usin gthe Replication Protocol, the user must be allowed to conect from th server from where the backup is running.

* Replication user's entry must be present in pg_hba.conf.

## Steps to run pg_basebackup

1. Create an empty target directory to the perform the backup
```sh
sudo mkdir -p ~/backup_dir/`date +"%Y%m%d"`
sudo chown -R postgres:postgres ~/backup_dir/`date +"%Y%m%d"`
```

2. Create the Replication user
```sql
CREATE UESR backup_user WITH REPLICATION PASSWORD 'MySecretPassword';
```

3. Add appropriate entries in pg_hba.conf file to allow replication connections and perform a Reload
```sh
vi pg_hba.conf
host    replication backup_user <backup_server_ip>/32   md5

psql -c "select pg_reload_conf()"
```

4. Run backup using pg_basebackup
```sh
pg_basebackup -h <master_node_ip/hostname> -U <backup_user> -p <port> -D <database_directory> -c <checkpoing_mode> -F<format> -P -X<wal_method> -l <backup_label>

# Example
pg_basebackup -h 192.168.10.20 -U backup_user -p 5432 -D ~/backup_dir/`date +"%Y%m%d"` -c fast -Ft -z -P -Xs -l backup_label

# -Ft means create a tar file otherwise we can use -Fp. 
# -z is to zip the tar file
# -Xs is to stream the WAL files along with backup when backup is running. It keeps the backup consistent.
# If we are taking this backup to create Replica, then we can include -R too. It creates the required entries for the parameters in postgresql.auto.conf.
```

**Note:** 
* Backup can be taken using superuser too but it is not recommended. 
* pg_basebackup automatically creates the backup directory but good if we care it ourselves.
* The user needs to be allowed to make Replication Connections from the Backup server.


# Restoring a Backup taken using pg_basebackup

Backup taken using pg_basebackup can be used for both performing a Point-In-Time recovery and setup of Replication.

* In order to restore a backup taken using pg_basebackup, there should be an empty directory that has available storage equivalent to the original data directory.
* The backup should be extracted to the Target Data Directory as a Postgres User.

## Steps to Restore backup

1. Create an empty directory
```sh
sudo mkdir -p ~/pgdata
```

2. Extract the backup (base.tar.gz) file to the target directory
```sh
tar xzf ~/backup_dir/base.tar/gz -C ~/pgdata
```

**NOTE** 
If the database server contains one or more tablespaces, then the individual tablespaces should also be extracted to different directories.

A *tablespace_map* file exists in the Data Directory that is extracted using *base.tar-gz*. So, each of the tablespaces must be extracted to the directories mentioned in the *tablespace_map* file. If the locations need to be modified, then modify the tablespace_map file manually and add the new locations for each tablespace.

3. Extract the WAL segments/files generated during the backup to the *pg_wal* directory
```sh
tar xzf pg_wal.tar.gz -C ~/pgdata/pg_wal
```

4. Start PostgreSQL using the Data Directory restored
```sh 
pg_ctl -D ~/pgdata start
```