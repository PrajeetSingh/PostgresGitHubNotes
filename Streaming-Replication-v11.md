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

## Steps to Configure Streaming Replication

### On the Primary Server

1. Create a Replication User

Create a dedicated user for replication with the REPLICATION role. Choose a strong password.

```sql
CREATE USER rep_user WITH REPLICATION ENCRYPTED PASSWORD 'mysecretpassword';
\q
```

2. Configure postgresql.conf

Edit the postgresql.conf file (usually located in /var/lib/postgresql/data or /etc/postgresql/<version>/main/). Find and modify or add the following parameters.

```text
# Connections
listen_addresses = '*' # Allow connections from all interfaces (or specify specific IPs like 'localhost,192.168.1.101')
max_connections = 100  # Adjust as needed, ensure enough for clients + replication

# Write-Ahead Log (WAL)
wal_level = replica    # Required for streaming replication (or 'hot_standby' in older versions)
archive_mode = on      # Enables WAL archiving (essential for point-in-time recovery and allowing standby to catch up if it falls behind)
archive_command = 'cp %p /path/to/wal_archive/%f' # Set a command to archive WALs. This is crucial for standbys to catch up if they fall behind. You might use `rsync` to a network share, or a dedicated archiving solution. For basic setup, a local directory is fine for testing. Replace `/path/to/wal_archive` with an actual directory and ensure `postgres` user has write permissions.
max_wal_senders = 5    # Number of concurrent connections for WAL sending (at least 1 per standby + some for pg_basebackup)
wal_keep_size = 1024   # (Or wal_keep_segments in older versions) Amount of WAL to retain in `pg_wal` for standbys to catch up. This is in MB. Set it high enough to cover typical network outages. If standbys fall behind this, they will need to retrieve WALs from the archive.
hot_standby = on       # Allows read-only queries on the standby server
```
