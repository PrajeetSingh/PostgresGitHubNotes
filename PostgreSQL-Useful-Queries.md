# PostgreSQL Useful Queries

### Number of active sessions
```sql
SELECT count(*) FROM pg_stat_activity WHERE state = 'active';
```

### Number of Inactive sessions
```sql
SELECT count(*) FROM pg_stat_activity WHERE state = 'idle';
```

### Long running queries ( session query running time )
```sql
SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes' AND state = 'active';
```

### Number of locks
```sql
SELECT pid, usename, pg_blocking_pids(pid) AS blocked_by, query AS blocked_query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

### Waiting sessions
```sql
SELECT count(*) FROM pg_stat_activity WHERE state = 'idle in transaction';
```

### SQL queries being run
```sql
SELECT pid AS process_id, query AS active_query
FROM pg_stat_activity
WHERE state = 'active';
```

### CPU and memory occupied by each session
```sql
Not sure yet how to get it	
```

### Number of connections
```sql
SELECT count(*) FROM pg_stat_activity;
```

### Primary / Standby delay
```sql
SELECT extract(epoch FROM now() - pg_last_xact_replay_timestamp());
```

### Any errors in logs.
```sql
I think better check log files
```

### Database size
```sql
SELECT pg_size_pretty(pg_database_size('your_database_name'));
```

### Tablespace size
```sql
SELECT pg_size_pretty(pg_tablespace_size('pg_default'));
```

### Object (table/index) size
```sql
SELECT pg_size_pretty(pg_relation_size('my_table'));
```

### Last vacuum / autovacuum
```sql
SELECT relname, last_vacuum, last_autovacuum, last_analyze, last_autoanalyze
FROM pg_stat_user_tables;
```

### Last analyze / autoanalyzed
```sql
SELECT relname, last_vacuum, last_autovacuum, last_analyze, last_autoanalyze
FROM pg_stat_user_tables;
```

### Number of checkpoints
```sql
I think you can get it from logs
```

### Number of wal files generated
```sql
SELECT COUNT(*) FROM pg_ls_waldir();
```