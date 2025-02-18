**Step 1)** Create a replication user
	create user replicator with replication password 'password';

**Step 2)** Below parameter are already set as expected by default
	show wal_level; (replica)
	show max_wal_senders; (10)
	show hot_standby; (on)

**Step 3)** Continuous archiving is disabled by default. We need to enable it.
	mkdir -p /var/lib/pgsql/archivelog	
	show config_file;
	vi /var/lib/pgsql/12/data/postgresql.conf
	archive_mode = on
	archive_command = 'test ! -f /var/lib/pgsql/archivelog/%f && cp %p /var/lib/pgsql/archivelog/%f'
			(This means, check if archived log file already exists in /var/lib/pgsql/archivelog/, if not then copy it.)
	vi pg_hba.conf
	Type	database	user		address		method
	host	replication	replicator	192.168.1.0/24	md5
	systemctl restart postgresql-12.service

**Step 4)** Copy backup of Primary to Standby using pg_basebackup
	pg_base_backup -h <primary_ip> -U replicator -p 5432 -D basebackup -Fp -Xs -P -R 

**Step 5)** rsync the backup to data directory of Standby
	rsync -a basebackup/ postgres@192.168.1.198:/var/lib/pgsql/12/data/
	copied directory should also have standby.signal file too on the Standby server

**Step 6)** Startup Postgres service on Standby server
	you'll find that db and data is there in Standby now but it'll be Read Only
	\l
	\dt+