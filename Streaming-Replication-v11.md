# Streaming Replication - PostgreSQL version 11

## Understanding Key Concepts

* **WAL (Write-Ahead Log)**: PostgreSQL uses WAL to ensure data durability. All changes to the database are first written to WAL files. Streaming replication works by sending these WAL records from the primary to the standby(s).
* **Primary Server:** The server that accepts read and write operations.
* **Standby Server(s):** Read-only copies of the primary server that continuously receive and replay WAL records.
* **Streaming Replication:** The process of continuously transferring WAL records from the primary to the standby(s) in real-time.
* **pg_basebackup:** A utility used to take a consistent base backup of a running PostgreSQL cluster, which serves as the initial data for a new standby server.
* **primary_conninfo:** A parameter on the standby server that defines the connection string to the primary server.
* **standby.signal (PostgreSQL 12+):** An empty file in the standby's data directory that signals the server to start in standby mode. (Prior to PostgreSQL 12, this was handled by recovery.conf).
* **Replication User:** A dedicated PostgreSQL user with REPLICATION privileges used by the standby to connect to the primary.
* **pg_hba.conf:** PostgreSQL's client authentication configuration file. You'll need to add entries to allow replication connections.

### Important Note for PostgreSQL 11 vs. 12+:
A significant change in PostgreSQL 12 and later is the deprecation of recovery.conf. In PostgreSQL 11 and earlier, you must create a recovery.conf file in the standby's data directory. For PostgreSQL 12+, this file is replaced by the standby.signal file and replication settings moved to postgresql.auto.conf (often automatically handled by pg_basebackup -R).

* PostgreSQL Data Directory (Debian 11 default): /var/lib/postgresql/11/main
* PostgreSQL Configuration Directory (Debian 11 default): /etc/postgresql/11/main

## Steps to Configure Streaming Replication

###I. On the Primary Server

**1. Create a Replication User**

Create a dedicated user for replication with the REPLICATION role. Choose a strong password.

```sql
CREATE USER rep_user WITH REPLICATION ENCRYPTED PASSWORD 'mysecretpassword';
\q
```

**2. Configure postgresql.conf**

Edit the postgresql.conf file (show config_file;). Find and modify or add the following parameters.

```TOML
# Connections
listen_addresses = '*' # Allow connections from all interfaces (or specify specific IPs like 'localhost,192.168.1.101')
max_connections = 100  # Adjust as needed, ensure enough for clients + replication

# Write-Ahead Log (WAL)
wal_level = replica    # Required for streaming replication (or 'hot_standby' in older versions)
archive_mode = on      # Enables WAL archiving (essential for point-in-time recovery and allowing standby to catch up if it falls behind)
archive_command = 'cp %p /path/to/wal_archive/%f' # Set a command to archive WALs. This is crucial for standbys to catch up if they fall behind. You might use `rsync` to a network share, or a dedicated archiving solution. For basic setup, a local directory is fine for testing. Replace `/path/to/wal_archive` with an actual directory and ensure `postgres` user has write permissions.
max_wal_senders = 5    # Number of concurrent connections for WAL sending (at least 1 per standby + some for pg_basebackup)
wal_keep_size = 1024   # (Or wal_keep_segments in older versions) Amount of WAL to retain in `pg_wal` for standbys to catch up. This is in MB. Set it high enough to cover typical network outages. If standbys fall behind this, they will need to retrieve WALs from the archive.
wal_keep_segments = 256      # (Units are 16MB WAL segments). Amount of WAL to retain in pg_wal for standbys to catch up.
                             # Set it high enough to cover typical network outages. If standbys fall behind this,
                             # they will need to retrieve WALs from the archive.
hot_standby = on       # Allows read-only queries on the standby server
```

*Note*: Using rsync for archiving to a remote server (needs SSH key-based authentication set up for the postgres user between servers): archive_command = 'rsync -a %p standby.example.com:/path/to/wal_archive/%f'

```sh
sudo mkdir -p /var/lib/postgresql/wal_archive
sudo chown postgres:postgres /var/lib/postgresql/wal_archive
```

**3. Configure pg_hba.conf**

Edit the pg_hba.conf file (show hba_file;). Add an entry to allow the replication user from the standby server's IP address to connect for replication.

```text
# TYPE  DATABASE        USER            ADDRESS                 METHOD
host    replication     rep_user        192.168.1.101/32        md5
```

*replication*: A special database keyword for replication connections.
*rep_user*: The replication user you created.
*192.168.1.101/32*: The IP address of your standby server (use /32 for a single host). You can use a subnet (e.g., 192.168.1.0/24) if you have multiple standbys in that range.
*md5*: Specifies password authentication.

**4. Reload/Restart PostgreSQL**
Apply the changes by reloading or restarting the PostgreSQL service. A reload is usually sufficient for postgresql.conf changes, but a restart is safer to ensure all changes take effect.

```sh
sudo systemctl reload postgresql@11-main # or postgresql-<version>
# or
sudo systemctl restart postgresql@11-main
# or
pg_ctlcluster 11 main reload
# or
pg_ctlcluster 11 main restart
```

###II. On the Standby Server

**1. Stop PostgreSQL (if running):**

If PostgreSQL is already initialized and running on the standby, stop it.

```sh
sudo systemctl stop postgresql@11-main
# or
pg_ctlcluster 11 main stop
```

**2. Clear Existing Data Directory**
This step will delete all data in the standby's PostgreSQL data directory. Ensure you're targeting the correct directory and that it's empty or can be safely overwritten.

```sh
sudo rm -rf /var/lib/postgresql/11/main/*
```

**3. Take a Base Backup from the Primary**
Run this command on the standby server as the postgres user. This connects to the primary, takes a snapshot of its data, and copies it to the standby's data directory.

```sh
sudo -u postgres pg_basebackup -h 192.168.1.100 -U rep_user -D /var/lib/postgresql/11/main -F p -X stream -c fast -P -v
```
```text
You will be prompted for the rep_user password.

-h 192.168.1.100: Hostname or IP address of the primary server.
-U rep_user: The replication user created on the primary.
-D /var/lib/postgresql/11/main: The destination directory for the base backup on the standby.
-F p: Specifies "plain" format (direct copy of the data directory).
-X stream: Streams WAL data during the backup. This is crucial for a consistent backup that can be used directly for streaming replication.
-c fast: Performs a fast checkpoint on the primary before starting the backup.
-P: Shows progress.
-v: Verbose output.
```
**Important for PG11**: Unlike PostgreSQL 12+, pg_basebackup in PG11 does not automatically create recovery.conf with the -R option. You need to create it manually in the next step.

**4. Create recovery.conf file**
On PostgreSQL 11, you must create a recovery.conf file inside the standby's data directory (/var/lib/postgresql/11/main) to tell it to act as a standby and connect to the primary.

```sh
sudo vi /var/lib/postgresql/11/main/recovery.conf
```
Add the following content
```TOML
standby_mode = 'on'
primary_conninfo = 'host=192.168.1.100 port=5432 user=rep_user password=your_secure_password application_name=standby.example.com'
restore_command = 'cp /var/lib/postgresql/wal_archive/%f %p' # Only needed if using a separate WAL archive on the primary.
                                                              # If you're relying solely on streaming, this can be omitted,
                                                              # but it's good for robustness if the standby falls behind or for PITR.
                                                              # Ensure the standby can access the archive path (e.g., via NFS mount or rsync).
recovery_target_timeline = 'latest'
```