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
4. Went to the ssh config file in `etc/ssh/sshd_config` and enabled `password authentication yes`
5. logged into the server using ssh from my mac `ssh abdullah@139.179.211.162` (by first briding the connection and then finding the ip address) 

Creating a new postgres user admin for the database `sudo -u postgres createuser --interactive` and created a user `admin` with password `123`.
 making a new database by
 
 1. `su - postgres` to change to the postgres user
 2. `createdb -0 admin projectData` to create the database with the user admin.
 
 Try to access the DB from outside localhost
 1. `cd /etc/postgresql/14/main/ `
 2. `nano postgresql.conf` and change `#listen_addresses = 'localhost'` to  `listen_addresses = '*'` so that it doesnt block traffic from other server
 3. Go to the `pg_hba.conf` file and change `host    all             all             127.0.0.1/32 ` to `host    all             all             0.0.0.0/0` so that it gives access to the remote server.
 4. Allow firewall acces by using `sudo ufw allow 5432/tcp` which will stop the firewall from dropping the packets on port 5432 which is used by posgres (right now firewall is disabled).
 5. `systemctl restart postgresql` to restart the service 
 6. `psql -h 139.179.211.162 -d admin -U projectData` to access the Database from another machine. I also used TablePlus to connect to the db and it connected. 


 self note: \s command history is psql


 ## Backup Server Installation ##

 Installed ububtu server with user `server` and password `123`
 
 Went to the ssh config file in `etc/ssh/sshd_config` and enabled `password authentication yes`

 logged into the server using ssh from my mac `ssh server@139.179.211.162` (by first briding the connection and then finding the ip address) 

## Install pgBackRest

#### Types of backups: ####

Definations:
1. Full Backup: pgBackRest copies the entire contents of the database cluster to the backup. The first backup of the database cluster is always a Full Backup. pgBackRest is always able to restore a full backup directly. The full backup does not depend on any files outside of the full backup for consistency.

2. Differential Backup: pgBackRest copies only those database cluster files that have changed since the last full backup. pgBackRest restores a differential backup by copying all of the files in the chosen differential backup and the appropriate unchanged files from the previous full backup. 


3. Incremental Backup: pgBackRest copies only those database cluster files that have changed since the last backup (which can be another incremental backup, a differential backup, or a full backup). As an incremental backup only includes those files changed since the prior backup, they are generally much smaller than full or differential backups.


* A stanza is the configuration for a PostgreSQL database cluster that defines where it is located, how it will be backed up, archiving options, etc.


We need the servers to communicate with each other through SSH so I generated ssh keys for both using `ssh-keygen -t rsa -b 4096 -N ""` on both VMs.
This created the public-private key pair in `/var/lib/postgresql/.ssh` on both machines with the names `id_rsa` for private key and `id_rsa.pub` for the public key. 
Now we need to share the public keys from one machine to the other. So I tried logging into the postgres user using ssh but then I remembered that it requires a password. So I tried changing the password using `passwd postgres` but that requires old password so I just copied the password hash in `etc/shadow file` so then I tried `123` and it works. Now I can access ssh using the `postgres` user.

To allow SSH for the postgres user, I went to the `etc/ssh/sshd_config` and added `AllowUsers root postgres` (and the third user which is abdullah in db and server in backup) to both the servers. 
I restarted the ssh service `systemctl restart sshd`. After that I accessed the postgres user on the database from the postgres user from backup and the opposite. Both side of the connections are working. 

Next I copied the public keys from `/var/lib/postgresql/.ssh/` and pasted them into the `/var/lib/postgresql/.ssh/authorized_keys/` folder of both. I used scp to do the transfer of public keys using: `scp id_rsa.pub abdullah@139.179.211.162:~` and then `cp id_rsa.pub /var/lib/postgresql/.ssh/authorized_keys/` after going to the users home directory as root. I did the same thing with both the servers so now the keys are in the right place.

* To setup the backup, I change the content of `/etc/pgbackrest.conf` on Database server and added the following:
~~~
[my_cluster]
pg1-path=/var/lib/postgresql/14/main

[global]
repo1-host=backupserver
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
Alternate is to go to psql and write the commands
```
alter system set archive_command = 'pgbackrest --stanza=my_cluster archive-push %p';
alter system set archive_mode = 'on';
 alter system set wal_level = 'replica';
```
etc...

then you can check by going into psql 
```
show wal_level;
show archive_mode;
show archive_command;
show hot_standby; 
```
If they are the same as what we changed. It's good to go. 

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
[global]                                                                                                                                  
process-max=2
repo1-path=/var/lib/pgbackrest
repo1-retention-full=6
repo1-retention-incr=30
repo1-host-user=postgres
start-fast=y

[my_cluster]
pg1-host=posgresqserver
pg1-host-user=postgres
pg1-path=/var/lib/postgresql/14/main    

```

`repo1-retention-full=6`  means there will be 6 full backups retained. Each is after a month that means we can have full backups of 6 months back.

`repo1-retention-incr=30` means it will retain 30 incremental backups. Since there is one everyday, we will have of everyday until there is a new full backup. 


I tried `sudo -u postgres pgbackrest --stanza=my_cluster --log-level-console=info stanza-create` to create the Stanza. This command should work on both the machines. But it failed.


I had some things I was hoping that wouldn't cause a problem but my guess was that they're causing this error. 

1. postgres user was not in the sudoer file so I changed that using 
`sudo usermod -a -G sudo postgres` 

2. Just realized that I had setup the SSH incorrectly. I removed the folder `authorized keys` in both the machines and ran `ssh-copy-id postgres@139.179.211.162` and `ssh-copy-id postgres@139.179.211.246` so that both machines login to each other without a password using the keys.

3. Hostname not working, when I do ssh with ip address it works, but doesnt work when I use the hostname of `backupserver` for example. Also the ip addresses are configured by DHCP so there is a problem that if the devices change the network, their ips will change and all the configuration will get ruined.

### Configuring Hostnames ###
Went and edited the `/etc/hosts` file and added

 `139.179.211.246` `backupserver` 

 `139.179.211.162` `posgresqserver`

Right now:

`backupserver` has ip: `139.179.211.246`

`posgresqserver` has ip: `139.179.211.162` (need to make them static as well)


Tried ssh using the hostnames:
`ssh postgres@posgresqserver`, it logged in without the password from backupserver
`ssh postgres@backupserver`, it logged in without the password from backupserver but this time I got an error. There is a problem with accessing backupserver from server. 

- So I tried `ssh -v postgres@139.179.211.246` which led to successful ssh connection (without password ) to the backup server. I checked the etc/hosts file again and it's also correctly configured. Then I tried seeing using nslookup. It shows the correct ip address `nslookup backupserver`
- I tried `ping backupserver`, it also works. But when I do ssh with the hostname, it refuses. I will configure the appication with the ip address instead then.

 I double checked and it was a typo in writing ip address in the `etc/host` file. I accidently wrote `139.179.221.246` instead of `139.179.211.246` and that caused me 5 hours of debugging -_-


### Static IPs Configuration ###
Making IP addresses static so that they do not change when run on a seperate machine. The settings of the VM is still bridge so that they can communicate but we can't let the ip address change or the servers won't be able to communicate with each other.

Side note: This version of Ubuntu does not have `/network/interfaces` file so need to do this configuration using `nano /etc/netplan/00-installer-config.yaml`


Static Address Configuration of Backup Server
```
network:
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 139.179.211.246/24
      gateway4: 139.179.211.1
      nameservers:
          addresses: [8.8.8.8, 1.1.1.1]
  version: 2
```

Static Address Configuration of DB Server

```
network:
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 139.179.211.162/24
      gateway4: 139.179.211.1
      nameservers:
          addresses: [8.8.8.8, 1.1.1.1]
  version: 2
```

Then `sudo netplan apply` to apply the changes.

Now the ip addresses won't change and cause problems.
* This might cause problems if the machines connect to other wifi routers whose gateway has a different address, but regardless they will always communicate with each other so this configuration works.



After configuring all this I ran the following commands on the backup server:

`pgbackrest --stanza=my_cluster --log-level-console=info stanza-create`

Output:

```
INFO: stanza-create command begin 2.37: --exec-id=8038-f7a9150c --log-level-console=info --pg1-host=posgresqserver --pg1-host-user=postgres --pg1-path=/var/lib/postgresql/14/main --repo1-path=/var/lib/pgbackrest --stanza=my_cluster
2022-05-03 18:05:37.846 P00   INFO: stanza-create for stanza 'my_cluster' on repo1
2022-05-03 18:05:37.956 P00   INFO: stanza-create command end: completed successfully (1341ms)
```


Then I try ` sudo -u postgres pgbackrest --stanza=my_cluster --log-level-console=info check` on both server and backup to check if everything is configured correctly and working.

The Result is: 

```
2022-05-03 22:38:26.872 P00   INFO: check command begin 2.37: --exec-id=2650-4cdf2f72 --log-level-console=info --pg1-path=/var/lib/postgresql/14/main --repo1-host=backupserver --repo1-host-user=postgres --stanza=my_cluster
2022-05-03 22:38:27.477 P00   INFO: check repo1 configuration (primary)
2022-05-03 22:38:28.306 P00   INFO: check repo1 archive for WAL (primary)
2022-05-03 22:38:28.712 P00   INFO: WAL segment 00000001000000000000000F successfully archived to '/var/lib/pgbackrest/archive/my_cluster/14-1/0000000100000000/00000001000000000000000F-c483f004858e4b4df481e6cde9c01a0225adeb2f.gz' on repo1
2022-05-03 22:38:28.812 P00   INFO: check command end: completed successfully (1943ms)
```

That means everything is okay.

Next I try to make a full backup by running `pgbackrest --stanza=my_cluster --log-level-console=info backup` (need to be logged in as the postgres user) 

or `sudo -u postgres pgbackrest --stanza=my_cluster --log-level-console=info backup` if from other users.


Result:

```
2022-05-03 22:45:29.002 P00   INFO: backup command begin 2.37: --exec-id=9048-5187e4b5 --log-level-console=info --pg1-host=posgresqserver --pg1-host-user=postgres --pg1-path=/var/lib/postgresql/14/main --process-max=2 --repo1-path=/var/lib/pgbackrest --repo1-retention-diff=1 --repo1-retention-full=2 --stanza=my_cluster --start-fast
WARN: no prior backup exists, incr backup has been changed to full
2022-05-03 22:45:30.085 P00   INFO: execute non-exclusive pg_start_backup(): backup begins after the requested immediate checkpoint completes
2022-05-03 22:45:30.593 P00   INFO: backup start archive = 000000010000000000000012, lsn = 0/12000028
2022-05-03 22:45:30.593 P00   INFO: check archive for prior segment 000000010000000000000011
2022-05-03 22:45:36.216 P00   INFO: execute non-exclusive pg_stop_backup() and wait for all WAL segments to archive
2022-05-03 22:45:36.420 P00   INFO: backup stop archive = 000000010000000000000012, lsn = 0/12000138
2022-05-03 22:45:36.425 P00   INFO: check archive for segment(s) 000000010000000000000012:000000010000000000000012
2022-05-03 22:45:36.551 P00   INFO: new backup label = 20220503-224529F
2022-05-03 22:45:36.602 P00   INFO: full backup size = 41.6MB, file total = 1538
2022-05-03 22:45:36.602 P00   INFO: backup command end: completed successfully (7603ms)
2022-05-03 22:45:36.604 P00   INFO: expire command begin 2.37: --exec-id=9048-5187e4b5 --log-level-console=info --repo1-path=/var/lib/pgbackrest --repo1-retention-diff=1 --repo1-retention-full=2 --stanza=my_cluster
2022-05-03 22:45:36.612 P00   INFO: expire command end: completed successfully (8ms)
```

The full backup was a success. That means it is working. Just need to setup cron jobs for it now. 

Tried `sudo -u postgres pgbackrest --type=diff --stanza=my_cluster --log-level-console=info backup` 

and Result:

```
2022-05-04 00:04:51.600 P00   INFO: backup command begin 2.37: --exec-id=9285-9589fda3 --log-level-console=info --pg1-host=posgresqserver --pg1-host-user=postgres --pg1-path=/var/lib/postgresql/14/main --process-max=2 --repo1-path=/var/lib/pgbackrest --repo1-retention-diff=1 --repo1-retention-full=2 --stanza=my_cluster --start-fast --type=diff
2022-05-04 00:04:52.741 P00   INFO: last backup label = 20220503-224529F, version = 2.37
2022-05-04 00:04:52.741 P00   INFO: execute non-exclusive pg_start_backup(): backup begins after the requested immediate checkpoint completes
2022-05-04 00:04:53.248 P00   INFO: backup start archive = 000000010000000000000014, lsn = 0/14000028
2022-05-04 00:04:53.248 P00   INFO: check archive for prior segment 000000010000000000000013
2022-05-04 00:04:55.692 P00   INFO: execute non-exclusive pg_stop_backup() and wait for all WAL segments to archive
2022-05-04 00:04:55.896 P00   INFO: backup stop archive = 000000010000000000000014, lsn = 0/14000138
2022-05-04 00:04:55.900 P00   INFO: check archive for segment(s) 000000010000000000000014:000000010000000000000014
2022-05-04 00:04:56.033 P00   INFO: new backup label = 20220503-224529F_20220504-000452D
2022-05-04 00:04:56.089 P00   INFO: diff backup size = 8.3KB, file total = 1538
2022-05-04 00:04:56.089 P00   INFO: backup command end: completed successfully (4492ms)
2022-05-04 00:04:56.089 P00   INFO: expire command begin 2.37: --exec-id=9285-9589fda3 --log-level-console=info --repo1-path=/var/lib/pgbackrest --repo1-retention-diff=1 --repo1-retention-full=2 --stanza=my_cluster
2022-05-04 00:04:56.097 P00   INFO: expire command end: completed successfully (8ms)
postgres@backupserver:~$ sudo -u postgres pgbackrest --type=diff --stanza=my_cluster --log-level-console=info backup


```

Tried `sudo -u postgres pgbackrest --type=incr --stanza=my_cluster --log-level-console=info backup`

and the Incremental backup was also a success.

Next `crontab -e` to open the cron file to set the scheduled backups.
* it is recommended to use full paths in the cron file. Athough it was working without it as well. 


``` 
0 1  1   *   *     /usr/bin/pgbackrest --type=full --stanza=my_cluster backup #every month
0 2  *   *   *   /usr/bin/pgbackrest --type=incr --stanza=my_cluster backup #every day
```


Need to make directories for logs of pgbackrest 
so `sudo mkdir -p /var/log/pgbackrest`
`sudo chown postgres:postgres /var/log/pgbackrest` to give ownership to postgres user

 `sudo chmod 640 /etc/pgbackrest.conf` changing permissions of the main config file as recommended by the documentation.


The path where the backup repo is located is `/var/lib/pgbackrest` on the backup server.

Just to double check the if pgbackrest is working:

 `pgbackrest check --stanza=my_cluster --log-level-console=info`

returns:

```
2022-05-04 14:16:30.152 P00   INFO: check command begin 2.37: --exec-id=1411-16d5facf --log-level-console=info --pg1-host=posgresqserver --pg1-host-user=postgres --pg1-path=/var/lib/postgresql/14/main --repo1-path=/var/lib/pgbackrest --stanza=my_cluster
2022-05-04 14:16:31.152 P00   INFO: check repo1 configuration (primary)
2022-05-04 14:16:31.271 P00   INFO: check repo1 archive for WAL (primary)
2022-05-04 14:16:31.974 P00   INFO: WAL segment 00000001000000000000001D successfully archived to '/var/lib/pgbackrest/archive/my_cluster/14-1/0000000100000000/00000001000000000000001D-dbdbf830a265a09f5fa288b19288e73789c0570c.gz' on repo1
2022-05-04 14:16:32.080 P00   INFO: check command end: completed successfully (1932ms)
```

Working fine.



The Backup and it's scheduler is in place.

## Restore ##

To restore your DB from the backups, you need to stop the postgresql first by `sudo systemctl stop postgresql.service`

` pgbackrest info` on the db server to see if there are backups that exist. Then `pgbackrest restore --stanza=my_cluster --log-level-console=info --delta` to restore the databse from backup.

`sudo systemctl start postgresql.service` to run it again. And you will have a restored database from the backup server.



## Firewall ##

### Database Server ###

I configured the firewall so that no other request is accepted.

```
sudo su
ufw default allow outgoing
ufw default deny incoming
ufw allow ssh
ufw allow 5432/tcp # port 5432 is default for Postgresql
ufw enable
ufw status
```
`ufw status` is used to check if the firewall is enabled or not.

Result:

```
Status: active

To                         Action      From
--                         ------      ----
5432/tcp                   ALLOW       Anywhere                  
22/tcp                     ALLOW       Anywhere                  
5432/tcp (v6)              ALLOW       Anywhere (v6)             
22/tcp (v6)                ALLOW       Anywhere (v6)             

```
So no one can access the server on other ports now.

Next I tried the same on the backup server. I accidently added the port 5432 but we only have backup not the postgresql server so I deleted it using `ufw status numbered` to get the number and then `ufw delete 3` to delete the rule from the firewall.

I reboot both machines to see if the firewall is working on reboot. I checked using sudo ufw status and the firewall is working perfectly. 

Just to check if the firewall is not interfaring with the backup, I ran ` sudo -u postgres pgbackrest --stanza=my_cluster --log-level-console=info check`
then 
 `sudo -u postgres pgbackrest --type=incr --stanza=my_cluster --log-level-console=info backup`

Both executed perfectly. 



Just realized that I had to do this as another user with minimun privileges. So Instead of doing all this for another user, I made another user called admin and gave it full privileges. Then I removed all the privileges from postgres exept read. 


```  
ALTER user admin WITH SUPERUSER;
ALTER USER postgres set default_transaction_read_only = on;

```

This made the postgres user read only and it can carry out backups as I configured previously. 

#### I know that was not the best practice but I had very less time left before deadline ####

Then I also found that I only had to change the name of the user in the config file of the backup server, but I wasn't sure if that will be enough to change the main user making the backups from the server. 

I accessed the db from external client and tried inserting and viewing data etc. Everythng is working on that end. But Postgres user will not be able to edit things. Which might not be the best decision to make while designing a system.



*************
If I had enough time. I would reset the db user postgres and then make a new one called backup using `sudo -u postgres createuser --interactive` name `backup` with no password and would be a superuser. 
Then I would run `ALTER USER postgres set default_transaction_read_only = on;`
That would only give it read access. Then I would edit the `/etc/pgbackrest.conf` 

```

[global]
process-max=2
repo1-path=/var/lib/pgbackrest
repo1-retention-full=6
repo1-retention-diff=30
repo1-host-user=backup
start-fast=y

[my_cluster]
pg1-host=posgresqserver
pg1-host-user=postgres
pg1-path=/var/lib/postgresql/14/main                                           >


```
This will enable the backup server to access db from the `backup` user.


I was having problems settings the postgres user to be able to make changes. I found how to change it back. 
```
psql
set default_transaction_read_only=off;
```
Result:
```
SET
```

I was so happy. I had 30 mins to implemend this. 

Still gave the same error so I tried
```
ALTER USER postgres set default_transaction_read_only = off;
```
It worked this time.

Then

```
sudo -u postgres createuser --interactive
```
name: backup
superuser: yes


```
ALTER USER backup set default_transaction_read_only = on;
```
The user can only read now

then `repo1-host-user=backup` set in `/etc/pgbackrest.conf` on the backup server.
reboot both machines.


I tried the backup check command:  ` sudo -u postgres pgbackrest --stanza=my_cluster --log-level-console=info check`

Success

Tried Backup: `sudo -u postgres pgbackrest --type=incr --stanza=my_cluster --log-level-console=info backup`

completed successfully

Everything on this project is working now. 

User `backup` is being used for backups for the database.
