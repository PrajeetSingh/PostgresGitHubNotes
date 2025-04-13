# Cascading Replication - Step by Step Complete Setup

To accomplish this, we need to setup a Primary Server, an upstream Standby Server and one or more Cascading Servers.

Below is detailed Step-By-Step guide to do it.

## 1. Prepare the Primary Server

* Install PostgreSQL on the Primary Server

* Add below parameters to postgresql.conf file

```sh
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
```

* Add the follow to pg_hba.conf to allow replication connections to the Standby server IP address

```sh
host replication all 0.0.0.0/0 mdf
```

* Restart the PostgreSQL to apply changes
```sh
pg_ctlcluster 13 main restart
# OR
sudo systemctl restart postgresql-13
```