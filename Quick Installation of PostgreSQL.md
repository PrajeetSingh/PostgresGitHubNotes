## Quick installation of PostgreSQL version 13
This is quick installation of Primary and Replica servers with PostgreSQL version 13.

```sh
# Login as root 
sudo su -

# Run updates on o/s
apt update && sudo apt -y full-upgrade

# Change Hostname 
hostnamectl set-hostname pgmaster
exit

# Go to https://www.postgresql.org/download/linux/ubuntu/ and download dependencies and install them
apt install curl gpg gnupg2 software-properties-common apt-transport-https lsb-release ca-certificates
apt install curl ca-certificates
install -d /usr/share/postgresql-common/pgdg
curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc
sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'

# Run update again
apt update

# Install PostgreSQL version Server and Client
apt -y install postgresql-13 postgresql-client-13

# Check installed Postgres  packages
dpkg -l | grep -i postgres

# Check status of installed PostgreSQL cluster
pg_lsclusters
pg_ctlcluster 13 main status

# Change password of postgres o/s user
passwd postgres

# Login to postgres and check installed databases
su - postgres
psql
\l
```