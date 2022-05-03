# Linux Installation Project #
##### Syed Abdullah Hassan 21801072 #####

### Project: pgBackRest Backup Server #

* Install a pgBackRest server to manage PostgreSQL backups.
* Install a PostgreSQL 14 on a seperate (second) GNU/Linux server.
* Configure the server to backup that PostgreSQL server every day. Full backup once a month, incremental backups for the rest of the days.
Should be able to restore to a specific backup 6 months ago. 
* Create a specific user on PostgreSQL with limited privileges to
perform the backup. 
* The applications should have their own system user/group and the
daemon should be running using that user/group.
* The server should deny all incoming connections that aren't essential
for its services to run (except for SSH).



#### 1. Installing the Posgres Server #### 

1. Downloaded and installed the Ubuntu Server 
2. While installing, it asked for packages I want to install with it and I chose Posgresql with it.
3. Set the username to abdullah and password to 123
4. Went to the ssh config file in `etc/ssh/ssh_config` and enabled `password authentication yes`
5. logged into the server using ssh from my mac `ssh abdullah@139.179.211.162` (by first briding the connection and then finding the ip address) 

Creating a new postgres user admin for the database `sudo -u postgres createuser --interactive` and created a user `admin` with password `123`.
 making a new database by
 
 1. `su - postgres` to change to the postgres user
 2. `createdb -0 admin projectData` to create the database with the user admin.
 
 Try to access the DB from outside localhost
 1. `cd /etc/postgresql/14/main/ `
 2. `nano postgresql.conf` and change `#listen_addresses = 'localhost'` to  `listen_addresses = '*'` so that it doesnt block traffic from other server
 3. Go to the `pg_hba.conf` file and change `host    all             all             127.0.0.1/32 ` to `host    all             all             0.0.0.0/0` so that it gives access to the remote server.
 4. Allow firewall acces by using `sudo ufw allow 5432/tcp` which will stop the firewall from dropping the packets on port 5432 which is used by posgres.
 5. `systemctl restart postgresql` to restart the service 
 6. `psql -h 139.179.211.162 -d admin -U projectData` to access the Database from another machine. I also used TablePlus to connect to the db and it connected. 


 self note: \s command history is psql


 ## Backup Server Installation ##

 Installed ububtu server with user `server` and password `123`
 
 Went to the ssh config file in `etc/ssh/ssh_config` and enabled `password authentication yes`

 logged into the server using ssh from my mac `ssh server@139.179.211.162` (by first briding the connection and then finding the ip address) 

## Install pgBackRest

#### Before that need to understand types of backups: ####

* Full Backup: pgBackRest copies the entire contents of the database cluster to the backup. The first backup of the database cluster is always a Full Backup. pgBackRest is always able to restore a full backup directly. The full backup does not depend on any files outside of the full backup for consistency.

* Differential Backup: pgBackRest copies only those database cluster files that have changed since the last full backup. pgBackRest restores a differential backup by copying all of the files in the chosen differential backup and the appropriate unchanged files from the previous full backup. 


* Incremental Backup: pgBackRest copies only those database cluster files that have changed since the last backup (which can be another incremental backup, a differential backup, or a full backup). As an incremental backup only includes those files changed since the prior backup, they are generally much smaller than full or differential backups.



We need the servers to communicate with each other through SSH so I generated ssh keys for both using `ssh-keygen -t rsa -b 4096 -N ""` on both VMs.
This created the public-private key pair in `/var/lib/postgresql/.ssh` on both machines with the names `id_rsa` for private key and `id_rsa.pub` for the public key. 
Now we need to share the public keys from one machine to the other. So I tried logging into the postgres user using ssh but then I remembered that it requires a password. So I tried changing the password using `passwd postgres` but that requires old password so I just copied the password hash in `etc/shadow file` so then I tried `123` and it works. Now I can access ssh using the `postgres` user.

To allow SSH for the postgres user, I went to the `etc/ssh/sshd_config` and added `AllowUsers root postgres` (and the third user which is abdullah in db and server in backup) to both the servers. 
I restarted the ssh service `systemctl restart sshd`. After that I accessed the postgres user on the database from the postgres user from backup and the opposite. Both side of the connections are working. 

Next I copied the public keys from `/var/lib/postgresql/.ssh/` and pasted them into the `/var/lib/postgresql/.ssh/authorized_keys/` folder of both. I used scp to do the transfer of public keys using: `scp id_rsa.pub abdullah@139.179.211.162:~` and then `cp id_rsa.pub /var/lib/postgresql/.ssh/authorized_keys/` after going to the users home directory as root. I did the same thing with both the servers so now the keys are in the right place.

* To setup the backup, I change the content of `/etc/pgbackrest.conf` on Database server and added the following:
~~~
[my_cluster]
pg1-path=/var/lib/postgresql/12/main

[global]
repo1-host=pg-backup
repo1-host-user=postgres

[global:archive-push]
compress-level=3
~~~

Then went to `/etc/postgresql/14/main/postgresql.conf` and changed these variables:
```
archive_command = 'pgbackrest --stanza=my_cluster archive-push %p'
archive_mode = on
listen_addresses = '*'
max_wal_senders = 3
wal_level = replica
```

Then Restarted the service `sudo systemctl restart postgresql`

Created some dummy data on the server. 

```
sudo -u postgres psql

CREATE DATABASE projectData2;

\c projectData2

CREATE TABLE students ( name VARCHAR(24), year INT(10), department VARCHAR(15));

INSERT INTO students VALUES ('abdullah', 2022, 'CTIS');
INSERT INTO students VALUES ('goku', 1998, 'DBZ');
INSERT INTO students VALUES ('naruto', 2003, 'Naruto');
INSERT INTO students VALUES ('Ash', 1892, 'Pokemon');

select * from students;
```
Output:
```
   name   | year | department 
----------+------+------------
 abdullah | 2022 | CTIS
 goku     | 1998 | DBZ
 naruto   | 2003 | Naruto
 Ash      | 1892 | Pokemon
```
Now I have database named `projectData2`, table `students`.


Now to configure the backup, I will set the following configurations in the `/etc/pgbackrest.conf` on the backup server. 

```
[my_cluster]
pg1-host=posgresqserver
pg1-host-user=postgres
pg1-path=/var/lib/postgresql/12/main                                                                                                               

[global]                                                                                                                                  
process-max=2
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2
repo1-retention-diff=1
repo1-host-user=postgres
start-fast=y

```

I tried `sudo -u postgres pgbackrest --stanza=my_cluster --log-level-console=info stanza-create` to create the first backup. This command should work on both the machines. But it failed.

I had some things I was hoping that wouldn't cause a problem but my guess was that they're causing this error. 

1. postgres user was not in the sudoer file so I changed that using `	
sudo usermod -a -G sudo postgres` 

2. Just realized that I had setup the SSH incorrectly. I removed the folder `authorized keys` in both the machines and ran `ssh-copy-id postgres@139.179.211.162` and `ssh-copy-id postgres@139.179.211.246` so that both machines login to each other without a password using the keys.

3. Hostname not working, when I do ssh with ip address it works, but doesnt work when I use the hostname of `backupserver` for example. Also the ip addresses are configured by DHCP so there is a problem that if the devices change the network, their ips will change and all the configuration will get ruined.

Right now Backup Server has ip: `139.179.211.246`
posgresqserver has ip: `139.179.211.162`

I should change these ips to being static but what if they're not resolved. Another option is to make the connctions NAT and then give them static IPs, since the environment will not change. 

So Making an `internal network` from Virtualbox would be the best choice for such a thing so that it can communicate with each other and ip addresses don't change. 










