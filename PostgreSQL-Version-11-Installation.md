Ubuntu version (24.04.2 LTS, Noble Numbat) instructions for adding the PostgreSQL APT repository.

Here are the updated steps to install and configure PostgreSQL 11 on your Ubuntu 24.04.2 LTS (Noble Numbat) VM on AWS:

**Prerequisites:**

* An Ubuntu VM on AWS with sudo privileges.  
* Basic understanding of the Linux command line.  
* Internet connectivity on your VM to download packages.

**Steps to Install and Configure PostgreSQL 11 (for Ubuntu 24.04 LTS Noble Numbat):**

**1\. Update System Packages:**

It's always a good practice to update your system's package list before installing new software.

Bash

sudo apt update  
sudo apt upgrade \-y

**2\. Add the PostgreSQL APT Repository:**

The official PostgreSQL APT repository provides the latest stable versions of PostgreSQL.

* **Install prerequisites for the repository:**  
  Bash  
  sudo apt install \-y curl ca-certificates gnupg

* **Import the PostgreSQL signing key:**  
  Bash  
  curl \-fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg \--dearmor \-o /usr/share/keyrings/postgresql.gpg

* **Add the repository to your sources list (using 'noble' for 24.04 LTS):**  
  Bash  
  echo "deb \[signed-by=/usr/share/keyrings/postgresql.gpg\] http://apt.postgresql.org/pub/repos/apt/ noble-pgdg main" | sudo tee /etc/apt/sources.list.d/pgdg.list

  **Explanation:** We've replaced jammy with noble as that's the codename for Ubuntu 24.04 LTS.  
* **Update your package lists again to include the new repository:**  
  Bash  
  sudo apt update

**3\. Install PostgreSQL 11:**

Now you can install the specific version of PostgreSQL.

Bash

sudo apt install \-y postgresql-11 postgresql-client-11 postgresql-contrib-11

This command installs:

* postgresql-11: The core PostgreSQL database server.  
* postgresql-client-11: Command-line tools for interacting with PostgreSQL.  
* postgresql-contrib-11: Additional utilities and extensions.

**4\. Verify Installation and Service Status:**

PostgreSQL should start automatically after installation. You can check its status:

Bash

sudo systemctl status postgresql

You should see output indicating that the service is active (exited). This is normal for PostgreSQL's systemd service; it indicates the service manager has successfully started the cluster.

You can also check the installed version:

Bash

psql \--version

This should show psql (PostgreSQL) 11.x.

**5\. Access PostgreSQL and Set up Initial User/Password:**

By default, PostgreSQL creates a Linux user named postgres that is associated with the default PostgreSQL superuser role.

* **Switch to the postgres user:**  
  Bash  
  sudo \-i \-u postgres

* **Access the PostgreSQL prompt:**  
  Bash  
  psql

  You should now be at the postgres=\# prompt.  
* **Set a password for the postgres database user:**  
  SQL  
  ALTER USER postgres WITH PASSWORD 'your\_strong\_password';

  Replace 'your\_strong\_password' with a secure password. Remember this password\!  
* **Exit the PostgreSQL prompt:**  
  SQL  
  \\q

* **Exit the postgres Linux user session:**  
  Bash  
  exit

**6\. Configure Remote Access (Optional but common for AWS VMs):**

By default, PostgreSQL only listens for connections from localhost. If you want to connect from other machines (e.g., from your local machine using a GUI client or another application server), you need to configure it.

* Edit postgresql.conf:  
  This file controls core server settings. The path will typically be /etc/postgresql/11/main/postgresql.conf.  
  Bash  
  sudo nano /etc/postgresql/11/main/postgresql.conf

  Find the line \#listen\_addresses \= 'localhost' and change it to:  
  listen\_addresses \= '\*'

  This tells PostgreSQL to listen on all available network interfaces. Save and exit (Ctrl+O, Enter, Ctrl+X in nano).  
* Edit pg\_hba.conf:  
  This file controls client authentication. The path will be /etc/postgresql/11/main/pg\_hba.conf.  
  Bash  
  sudo nano /etc/postgresql/11/main/pg\_hba.conf

  You'll see lines like:  
  \# TYPE  DATABASE        USER            ADDRESS                 METHOD

  \# "local" is for Unix domain socket connections only  
  local   all             all                                     peer  
  \# IPv4 local connections:  
  host    all             all             127.0.0.1/32            scram-sha-256  
  \# IPv6 local connections:  
  host    all             all             ::1/128                 scram-sha-256

  To allow connections from other IP addresses, you'll need to add a new line. For example, to allow connections from any IP address using md5 password authentication (less secure than scram-sha-256, but sometimes required for older clients or simpler setups):  
  \# TYPE  DATABASE        USER            ADDRESS                 METHOD  
  host    all             all             0.0.0.0/0               md5

  **Important Security Note:** 0.0.0.0/0 allows connections from *any* IP address. In a production environment, it's highly recommended to restrict this to specific IP ranges (e.g., your office IP, your application server's IP, or your VPC's CIDR block). For example, host all all 192.168.1.0/24 md5 would allow connections from the 192.168.1.x subnet.  
  Also, change the existing scram-sha-256 methods to md5 if you want to use MD5 password authentication for local connections as well (this is often simpler for testing, but scram-sha-256 is more secure).  
  \# IPv4 local connections:  
  host    all             all             127.0.0.1/32            md5  
  \# IPv6 local connections:  
  host    all             all             ::1/128                 md5

  Save and exit.  
* **Restart PostgreSQL to apply changes:**  
  Bash  
  sudo systemctl restart postgresql

**7\. Configure AWS Security Group:**

For your AWS VM, you **must** open port 5432 (PostgreSQL's default port) in the security group associated with your EC2 instance.

* Go to your EC2 instance in the AWS console.  
* Under "Security," click on the security group linked to your instance.  
* Go to the "Inbound Rules" tab and click "Edit inbound rules."  
* Add a new rule:  
  * **Type:** Custom TCP  
  * **Port range:** 5432  
  * **Source:**  
    * My IP (if connecting from your current public IP)  
    * Custom and specify a CIDR block (e.g., 0.0.0.0/0 for anywhere, but be cautious with this in production, or a specific IP range like your\_application\_server\_ip/32).  
* Save the rules.

**8\. Create a New Database and User (Recommended):**

It's best practice not to use the postgres superuser for your applications. Create a dedicated user and database.

* **Switch back to the postgres Linux user:**  
  Bash  
  sudo \-i \-u postgres

* **Access the PostgreSQL prompt:**  
  Bash  
  psql

* **Create a new user (role) with a password:**  
  SQL  
  CREATE USER myuser WITH PASSWORD 'my\_secure\_password';

  Replace myuser and my\_secure\_password with your desired username and a strong password.  
* **Create a new database and set its owner:**  
  SQL  
  CREATE DATABASE mydatabase OWNER myuser;

  Replace mydatabase with your desired database name.  
* **Grant all privileges on the database to the new user:**  
  SQL  
  GRANT ALL PRIVILEGES ON DATABASE mydatabase TO myuser;

* **Exit the PostgreSQL prompt and postgres user session:**  
  SQL  
  \\q  
  exit

**9\. Test Connectivity:**

You can now test connectivity from your local machine or another server.

* **From your local machine (if psql client is installed):**  
  Bash  
  psql \-h your\_aws\_vm\_public\_ip \-U myuser \-d mydatabase

  You will be prompted for myuser's password. If successful, you'll enter the mydatabase prompt.  
* **From the Ubuntu VM itself (as your regular sudo user):**  
  Bash  
  psql \-U myuser \-d mydatabase

  Again, enter the password when prompted.

You have successfully installed and configured PostgreSQL 11 on your Ubuntu 24.04 LTS (Noble Numbat) VM in AWS\!

**Sources**  
1\. [https://www.alibabacloud.com/blog/setting-up-a-postgresql-database-on-ubuntu\_594910](https://www.alibabacloud.com/blog/setting-up-a-postgresql-database-on-ubuntu_594910)  
2\.  
 [https://forum.onlyoffice.com/t/onlyoffice-some-endpoint-result-in-502-when-hosting/5014](https://forum.onlyoffice.com/t/onlyoffice-some-endpoint-result-in-502-when-hosting/5014)