# Upgrade PostgreSQL Primary and Replica databases from version 11 to version 13 on Debian 11, using Hard Link method

Upgrading large PostgreSQL replicated cluster from version 11 to 13 on Debian 11 using the hard link method with `pg_upgradecluster` is an good choice for speed, especially with large databases. This method avoids copying the entire data directory, significantly reducing the upgrade time.

## Key Requirement for Hard Link Method

The old PostgreSQL 11 data directory (/var/lib/postgresql/11/main) and the new PostgreSQL 13 data directory (/var/lib/postgresql/13/main) must reside on the same filesystem (i.e., the same disk partition). If they are on different filesystems, the hard link method will fail, and you'll have to use the default copy method.

## Assumptions

* You have a working PostgreSQL 11 streaming replication setup.
* You are running Debian 11 on both servers.
* PostgreSQL 11 data directory: `/var/lib/postgresql/11/main`.
* PostgreSQL 11 config directory: `/etc/postgresql/postgresql/11/main`.
* PostgreSQL 13 data directory: `/var/lib/postgresql/13/main (will be created during installation/upgrade)`.
* PostgreSQL 13 config directory: `/etc/postgresql/postgresql/13/main`.
* The system user running PostgreSQL is `postgres`.

## High-Level Strategy

### 1. Upgrade Standby (Replica) First:

* Stop PostgreSQL 11 on the standby.
* Install PostgreSQL 13 packages.
* Use pg_upgradecluster --method=link to upgrade the standby's data directory.
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

