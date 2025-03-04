# Quick Commands

\du		-- list users

\l		-- list databases

\dt		-- list tables 

\dn		-- list schemas 

\q		-- quit

\c		-- change connection to different user and databases

\conninfo	-- You are connected to database "postgres" as user "postgres" via socket in "/var/run/postgresql" at port "5432".


psql -U postgres -d postgres -h ipaddress -p port 		-- login with ipaddress and port

```sql
select txid_current();				
```
```sql
# Is same like kill -6
select pg_cancel_backend('os_pid');

# Is same like kill -9
select pg_terminate_backend('os_pid');
```

```sql
# current settings 
select * from pg_settings;

# current user
select current_user;

# current database 
select current_database();

# generate numbers from 1-10
select generate_Series(1,10);

# Current version
select version();

select * from pg_stat_all_tables where relname = 'emp';
```
 
analyze <table_name>;

vacuum verbose <table_name>;										<-- Vacuum table 

vacuum <table_name>;

select name, setting from pg_settings where name like 'log%';		<-- To get settings in database

select pg_relation_filepath('emp');									<-- To get the data file name of table.

select * from pg_available_extentions;								<-- To see the available extentions 





