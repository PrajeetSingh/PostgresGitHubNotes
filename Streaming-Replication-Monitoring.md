# Monitoring Streaming Replication

### On Standby : To check whether standby is in recovery or not
```sql
select pg_is_in_recovery();
```

### On Master: Query to monitor replication
```sql
select * from pg_stat_replication;
```

### On Standby: Query to monitor incoming replication
```sql
select * from pg_stat_wal_receiver;
```

### Query to find current wal lsn on Primary
```sql
select * from pg_current_wal_lsn();
```

### On Standby: Query to check what was the last wal_lsn received:
```sql
select * from pg_last_wal_receive_lsn();
```

### On Standby: Query to find last replayed lsn
```sql
select pg_last_wal_replay_lsn();
```

### Query to find the difference between lsn (Primary and Standby), output will be in bytes
```sql
select pg_wal_lsn_diff('0/23000073','0/3342343');
```

### Fnd the difference between Primary and Standby in MB
```sql
select round(pg_wal_lsn_diff('0/23000073','0/3342343')/pow(1024,2.0) ,2) difference_in_mb;
```

### On Master: Query to find the physical name of wal file using lsn 
```sql
select pg_walfile_name('0/23000073');
```

### On Master: Query to find replication slots
```sql
select * from pg_replication_slots;
```

### Query to find replay lag, receiving lag
```sql
select pid,application_name,pg_wal_lsn_diff(pg_current_wal_lsn(),sent_lsn) sending_lag,
pg_wal_lsn_diff(sent_lsn,flush_lsn) receiving_lag,
pg_wal_lsn_diff(flush_lsn,replay_lsn) replaying_lag,
pg_wal_lsn_diff(pg_current_wal_lsn(),replay_lsn) total_lag,
from pg_stat_replication;
```
