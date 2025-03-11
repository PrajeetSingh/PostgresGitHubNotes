# Cascading Replication


## Prerequisites

* Have Primary and Replicas running 
* Have third server with PostgreSQL server installed with same version and Primary and Replica and empty Data Directory.

**Step 1:** Create the second replica server

To create the second replica server, you need to first take a base backup of the first replica server. This backup will be used to initialize the second replica server.
To take a base backup, run the following command on the second replica server:

```sh
pg_basebackup -h first_replica_server_ip_address -U replicator -c fast -P -R -X stream -D /path/to/second_replica/data/directory

# Example:
pg_basebackup -h 172.31.85.180 -U replicator -c fast -P -R -Xs -D /var/lib/postgresql/13/main
```

Replace `first_replica_server_ip_address` with the IP address of the first replica server and `/path/to/second_replica/data/directory` with the path to the data directory on the second replica server.


**Step 2:** Configure cascading replication
To configure cascading replication, you need to modify the recovery.conf / postgresql.auto.conf file on the second replica server to point to the first replica server instead of the primary server.

```sh
standby_mode = on
primary_conninfo = 'host=first_replica_server_ip_address port=5432 user=replicator password=password'
```
Replace first_replica_server_ip_address with the IP address of the first replica server and password with the password for the replication user.

Repeat Step 4 to add more replica servers.