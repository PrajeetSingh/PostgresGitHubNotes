# Replication Slots in Synchronous Replication

### Create a Physical Replication Slot.
```sql
select pg_create_physical_replication_slot('MySlot1');
```

### Allocate Replication Slot to Replica
```sh
vi postgresql.conf
primary_slot_name='MySlot1'
# Now restart the Replica Cluster
```

### Monitor a Replication Slot
```sql
select * from pg_replication_slots;
```

### Delete a Replication Slot
```sql
select pg_drop_replication_slot('MySlot1');
-- Ensure to remove Orphan Replication Slots, otherwise they take their toll on performance.
```