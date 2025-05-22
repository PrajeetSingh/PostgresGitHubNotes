# Upgrade PostgreSQL Primary and Replica databases from version 11 to version 13 on Debian 11, using Hard Link method

Upgrading large PostgreSQL replicated cluster from version 11 to 13 on Debian 11 using the hard link method with `pg_upgradecluster` is an good choice for speed, especially with large databases. This method avoids copying the entire data directory, significantly reducing the upgrade time.

## Key Requirement for Hard Link Method

The old PostgreSQL 11 data directory (`/var/lib/postgresql/11/main`) and the new PostgreSQL 13 data directory (`/var/lib/postgresql/13/main`) must reside on the same filesystem (i.e., the same disk partition). If they are on different filesystems, the hard link method will fail, and you'll have to use the default copy method.

## Assumptions

* You have a working PostgreSQL 11 streaming replication setup.
* You are running Debian 11 on both servers.
* PostgreSQL 11 data directory: `/var/lib/postgresql/11/main`.
* PostgreSQL 11 config directory: `/etc/postgresql/postgresql/11/main`.
* PostgreSQL 13 data directory: `/var/lib/postgresql/13/main (will be created during installation/upgrade)`.
* PostgreSQL 13 config directory: `/etc/postgresql/postgresql/13/main`.
* The system user running PostgreSQL is `postgres`.

## High-Level Strategy

### 1. Upgrade Standby (Replica) First

* Stop PostgreSQL 11 on the standby.
* Install PostgreSQL 13 packages.
* Use `pg_upgradecluster --method=link` to upgrade the standby's data directory.
* **Crucially**: After the upgrade, you will still take a fresh `pg_basebackup` from the still-running PostgreSQL 11 primary to the newly upgraded PG13 standby data directory. This is because `pg_upgradecluster` turns the standby into a standalone server, and for replication, you need a fresh base backup from the actual primary.
* Start the upgraded standby, which will replicate from the PG11 primary.

### 2. Upgrade Primary

* Stop PostgreSQL 11 on the primary (brief downtime).
* Install PostgreSQL 13 packages.
* Use `pg_upgradecluster --method=link` to upgrade the primary's data directory.
* Configure and start PostgreSQL 13 primary.

### 3. Re-synchronize Standby with New Primary

* Take a fresh `pg_basebackup` from the newly upgraded PostgreSQL 13 primary to the (already upgraded to PG13) standby.
* Configure and start the standby to replicate from the new PostgreSQL 13 primary.

---

## Preparation (On Both Primary and Standby)

### 1. Verify Filesystem Location

Ensure both the old and new data directories are on the same filesystem. You can check this using `df -h /var/lib/postgresql/11/main` and noting the "Mounted on" column. It should be the same partition where `/var/lib/postgresql/13/main` will reside.

### 2. Review System Resources

Ensure you have enough disk space for WAL files and new system catalogs (hard links save on the main data copy, but not new files).
Check available RAM and CPU.

### 3. Stop Applications

Temporarily stop all applications that connect to your PostgreSQL databases to prevent new writes and ensure a clean upgrade.

### 4. Full Backup of Existing Data

This is the most critical step. Before starting any upgrade, take a full backup of your PostgreSQL 11 primary cluster. This can be done using pg_dumpall or by stopping PostgreSQL 11 and creating a file-system level backup of the data directory. O/S snapshot can be a good option too.

```BASH
# On Primary Server
sudo -u postgres pg_dumpall > /path/to/your/backup/all_databases_pg11.sql
# OR (for file-system backup, requires stopping PG11)
sudo systemctl stop postgresql@11-main
sudo cp -rp /var/lib/postgresql/11/main /path/to/your/backup/pg11_main_backup_$(date +%Y%m%d%H%M%S)
sudo systemctl start postgresql@11-main
```

## I. Upgrade Standby Server (standby.example.com)

### 1. Stop PostgreSQL 11 Service on Standby

Ensure the PostgreSQL 11 service is completely stopped.

```Bash
sudo systemctl stop postgresql@11-main
sudo systemctl disable postgresql@11-main # Prevent it from starting on reboot
# or
sudo pg_ctlcluster 11 main stop
```

### 2. Install PostgreSQL 13 Packages

Install PostgreSQL 13 on the standby server. This will install the binaries and create a new default cluster for PG13, usually at `/var/lib/postgresql/13/main`.

```Bash
sudo apt update
sudo apt install postgresql-13
```

The `postgresql@13-main` service will likely start automatically. Stop it immediately as we need to upgrade the existing data.

```Bash
sudo systemctl stop postgresql@13-main
```

### 3. Perform the Cluster Upgrade on Standby (using hard link method)

Use `pg_upgradecluster` with the `--method=link` option to migrate your PostgreSQL 11 cluster to version 13.

```Bash
sudo pg_upgradecluster 11 main --method=link
```

* This command will stop the old (version 11) cluster, create a new (version 13) cluster, and then use `pg_upgrade` with hard links to transfer the data.
* **Important:** Unlike the copy method, the old 11 cluster's data directory (`/var/lib/postgresql/11/main`) will not be renamed to main.old. The new 13 cluster will directly link to the files in the 11 directory. If the upgrade fails, the 11 cluster data is still there, but you cannot simply delete `/var/lib/postgresql/13/main` without affecting the 11 cluster. `pg_upgradecluster` handles cleanup automatically on success.

### 4. Prepare PostgreSQL 13 for Replication (Important!)

The `pg_upgradecluster` command creates a standalone PostgreSQL 13 cluster from your old PG11 data. For it to become a standby again, you need to re-initialize it from the primary.

* Remove the data from the newly upgraded `/var/lib/postgresql/13/main` directory. This step is essential because `pg_upgradecluster` created a full, standalone cluster from the old standby's data, which was already behind the primary. To re-establish streaming replication correctly, you need a fresh base backup from the current primary.

```Bash
sudo rm -rf /var/lib/postgresql/13/main/*
```

* Take a fresh base backup from the Primary (PG11) to the Standby (PG13 data dir)

Run this command on the standby, targeting the new PG13 data directory:

```Bash
sudo -u postgres pg_basebackup -h 192.168.1.100 -U rep_user -D /var/lib/postgresql/13/main -F p -X stream -c fast -R -P
```

  * `-h 192.168.1.100:` IP of your PostgreSQL 11 primary.
  * `-U rep_user:` The replication user.
  * `-D /var/lib/postgresql/13/main:` The new, empty PostgreSQL 13 data directory on the standby.
  * `-R:` *Crucially*, for PostgreSQL 13, this creates the `standby.signal` file and configures primary_conninfo in postgresql.auto.conf within the new data directory. You no longer use `recovery.conf` in PG13.

### 5. Start PostgreSQL 13 Service on Standby

The standby is now configured and should start replicating from the PG11 primary.

```Bash
sudo systemctl start postgresql@13-main
```

### 6. Verify Replication (Temporary)

On the Primary (PG11):

```SQL
SELECT client_addr, state, sync_state FROM pg_stat_replication;
```

You should see the `standby.example.com` (or its IP) connecting.

On the Standby (PG13):

```SQL
SELECT pg_is_in_recovery();
-- This should return t (true).
\x
SELECT * FROM pg_stat_wal_receiver;
```

## II. Upgrade Primary Server (primary.example.com)

Now that the standby is upgraded and temporarily replicating from the PG11 primary, we can upgrade the primary.

### 1. Stop PostgreSQL 11 Service on Primary

Stop the PostgreSQL 11 service. This will cause a brief downtime.

```Bash
sudo systemctl stop postgresql@11-main
sudo systemctl disable postgresql@11-main # Prevent it from starting on reboot
```

### 2. Install PostgreSQL 13 Packages on Primary

Install PostgreSQL 13 on the primary server.

```Bash
sudo apt update
sudo apt install postgresql-13
```

Stop the automatically started PG13 service

```Bash
sudo systemctl stop postgresql@13-main
```

### 3. Perform the Cluster Upgrade on Primary (using hard link method)

Use `pg_upgradecluster` with the `--method=link` option to migrate the PostgreSQL 11 cluster to version 13.

```Bash
sudo pg_upgradecluster 11 main --method=link
```

* Again, the old 11 cluster's data directory will not be renamed. The new 13 cluster will use hard links to the existing files.
* If successful, `pg_upgradecluster` will generate a script to remove the old cluster files, which you'll run later.

### 4. Configure PostgreSQL 13 Primary

Edit the `postgresql.conf` for the new PostgreSQL 13 cluster. It's usually located at `/etc/postgresql/13/main/postgresql.conf`.

```Bash
sudo nano /etc/postgresql/13/main/postgresql.conf
```

Ensure the following parameters are set for a primary server (they might already be set correctly by `pg_upgradecluster` or the default PG13 config, but verify):

```Ini, TOML
# CONNECTIONS AND AUTHENTICATION
listen_addresses = '*'
max_connections = 100

# WRITE AHEAD LOG (WAL)
wal_level = replica
archive_mode = on
archive_command = 'cp %p /var/lib/postgresql/wal_archive/%f' # Update this path if you changed it.
                                                              # Ensure directory exists and is writable by postgres user.
                                                              # For production, use shared storage or rsync to standbys.
max_wal_senders = 10 # Increase if you plan to have more standbys
wal_keep_size = 2048 # In MB for PG13+ (was wal_keep_segments in PG11, units are 16MB)
hot_standby = on     # Needed for standby to work correctly
```

Also, update `pg_hba.conf` (`/etc/postgresql/13/main/pg_hba.conf`) to allow replication connections from the standby (which will soon be connecting as a PG13 replica).

```Bash
sudo nano /etc/postgresql/13/main/pg_hba.conf
```

Add or verify this line:

```Ini, TOML
host    replication     rep_user        192.168.1.101/32        md5
```

### 5. Start PostgreSQL 13 Service on Primary

```Bash
sudo systemctl start postgresql@13-main
# or
sudo pg_ctlcluster 13 main start
sudo pg_ctlcluster 13 main status
```

### 6. Verify Primary Operation

Connect to the new PostgreSQL 13 primary and ensure your applications can connect and data is accessible.

```Bash
sudo -u postgres psql
```

## III. Re-synchronize Standby with New Primary (standby.example.com)

The standby is currently replicating from the old PG11 primary (which is now stopped and upgraded). It needs to be re-initialized to replicate from the new PG13 primary.

### 1. Stop PostgreSQL 13 Service on Standby

Stop the currently running PostgreSQL 13 service on the standby.

```Bash
sudo systemctl stop postgresql@13-main
# or
sudo pg_ctlcluster 13 main stop
```

### 2. Clear Old Standby Data (PG13)

Remove the data from the `/var/lib/postgresql/13/main` directory on the standby. This data was synchronized from the old PG11 primary and is now outdated.

```Bash
sudo rm -rf /var/lib/postgresql/13/main/*
```

### 3. Take a Fresh Base Backup from New Primary (PG13)

Run ```pg_basebackup``` again on the standby, but this time, connect to the new PostgreSQL 13 primary.

```Bash
sudo -u postgres pg_basebackup -h 192.168.1.100 -U rep_user -D /var/lib/postgresql/13/main -F p -X stream -c fast -R -P
```

* `-h 192.168.1.100:` IP of your PostgreSQL 13 primary.
* `-D /var/lib/postgresql/13/main:` The PostgreSQL 13 data directory on the standby.
* `-R:` This will ensure standby.signal and primary_conninfo (in postgresql.auto.conf) are correctly set for PG13 replication.

### 4. Start PostgreSQL 13 Service on Standby

The standby is now configured and should start replicating from the new PG13 primary.

```Bash
sudo systemctl start postgresql@13-main
# or
sudo pg_ctlcluster 13 main start
```

### 5. Verify Final Replication

On the Primary (PG13):

```Bash
sudo -u postgres psql -c "SELECT client_addr, state, sync_state FROM pg_stat_replication;"
```

You should see the standby.example.com (or its IP) connecting, and state should be streaming.

On the Standby (PG13):

```SQL
SELECT pg_is_in_recovery();
-- This should return t (true).
\x
SELECT * FROM pg_stat_wal_receiver;
```

Test by inserting data on the primary and verifying it appears on the standby.
