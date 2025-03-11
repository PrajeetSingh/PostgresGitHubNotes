# Failover in Streaming Replication

We can manually failover PostgreSQL from Master/Primary to Replica/Standby using one of below mentioned methods:

* pg_ctlcluster 13 main promote
* ```sql select pg_promote(); ```
* Create a Trigger File with the file name and path specified by the `promote_trigger_file` parameter

**On Primary**
```sql
-- Check if there is any Replication Lag.
select usename, client_addr, write_lag, replay_lag from pg_stat_replication;
select pg_switch_wal();
select usename, client_addr, write_lag, replay_lag from pg_stat_replication;

-- We can also check lsn values on Primary and Standby
-- On Primary
select pg_current_wal_lsn();

-- On Standby
select pg_last_wal_receive_lsn();
```

```sh
# Stop Primary Cluster
pg_lsclusters
pg_ctlcluster 13 main status
pg_ctlcluster 13 main stop
pg_lsclusters
```

**On Standby**
```sh
-- Promote Standby Cluster
pg_lsclusters
pg_ctlcluster 13 main status
pg_ctlcluster 13 main promote
# or
select pg_promote();
pg_ctlcluster 13 main start
pg_ctlcluster 13 main status
```

```sql
-- Value of pg_is_in_recovery should be false and standby.signal file should be missing now.
select pg_is_in_recovery();
select * from pg_stat_wal_receiver;
```

*REMEMBER*: You may need to create Replication Slots again after Failover.
```sql
select * from pg_replication_slots;
```