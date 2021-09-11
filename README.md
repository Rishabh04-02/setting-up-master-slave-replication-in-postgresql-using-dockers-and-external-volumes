# Setting up Master Slave Replication in PostgreSQL using Dockers and external volumes:

## Understanding replication in PostgreSQL 12

Streaming replication in PostgreSQL works on log shipping. Every transaction in postgres is written to a transaction log called WAL (write-ahead log) to achieve durability. A slave uses these WAL segments to continuously replicate changes from its master.

There exists three mandatory processes – wal sender , wal receiver and startup process, these play a major role in achieving streaming replication in postgres.

A wal sender process runs on a master, whereas the wal receiver and startup processes runs on its slave. When you start the replication, a wal receiver process sends the LSN (Log Sequence Number) up until when the WAL data has been replayed on a slave, to the master. And then the wal sender process on master sends the WAL data until the latest LSN starting from the LSN sent by the wal receiver, to the slave. Wal receiver writes the WAL data sent by wal sender to WAL segments. It is the startup process on slave that replays the data written to WAL segment. And then the streaming replication begins.

Note: Log Sequence Number, or LSN, is a pointer to a location in the WAL.

## Firewall -
UFW or Uncomplicated Firewall is an application to manage the iptables based firewall on Ubuntu. UFW is the default firewall configuration tool for Ubuntu Linux and provides a user-friendly way to configure the firewall.

```
apt-get install -y ufw
```

Add new services to the UFW firewall: add SSH and PostgreSQL services with commands below.

```
ufw allow ssh
ufw allow postgresql
```

Enable the UFW firewall and check the status.

```
ufw enable
ufw status
```

UFW firewall has been installed and the PostgreSQL service has been added.

## Setting up Docker using external volume

### Install docker
You can install docker from your default package manager or using some other service like [**Snapcraft**](https://snapcraft.io/) e.g. ``snap install docker``

### Setup Docker engine
#### Pull postgress in docker
```
docker pull postgres
```

#### Create docker
```
docker run --name DOCKER_NAME -e POSTGRES_PASSWORD=PASSWORD -d -p 0.0.0.0:5432:5432 -v /mnt/EXTERNAL_VOLUME_NAME/postgres:/var/lib/postgresql/data  postgres
```

### Check for running dockers
```
docker ps
```

### View all available dockers
```
docker ps -a
```

### Enter into Docker shel
```
docker exec -it DOCKER_NAME /bin/bash
```

## Master -
Create a role dedicated to the replication -
Create the user in master using whichever slave should connect for streaming the WALs. This user must have REPLICATION ROLE.

```
CREATE USER replica REPLICATION LOGIN ENCRYPTED PASSWORD 'STRONG_PASSWORD_HERE';
```

Now check the new user with 'du' query below, and you will see the replica user with replication privileges.

```
\du
```

### Edit postgresql.conf -
Note - the postgresql.conf would be present in the following location in case of external volume ``/mnt/EXTERNAL_VOLUME_NAME/postgres/postgresql.conf``

The following parameters on the master are considered as mandatory when setting up streaming replication.
* **archive_mode** : Must be set to ON to enable archiving of WALs.
* **wal_level** : Must be at least set to hot_standby  until version 9.5 or replica  in the later versions.
* **max_wal_senders** : Must be set to 3 if you are starting with one slave. For every slave, you may add 2 wal senders.
* **wal_keep_segments** : Set the WAL retention in pg_xlog (until PostgreSQL 9.x) and pg_wal (from PostgreSQL 10). Every WAL requires 16MB of space unless you have explicitly modified the WAL segment size. You may start with 100 or more depending on the space and the amount of WAL that could be generated during a backup.
* **archive_command** : This parameter takes a shell command or external programs. It can be a simple copy command to copy the WAL segments to another location or a script that has the logic to archive the WALs to S3 or a remote backup server.
* **listen_addresses** : Set it to * or the range of IP Addresses that need to be whitelisted to connect to your master PostgreSQL server. Your slave IP should be whitelisted too, else, the slave cannot connect to the master to replicate/replay WALs.
* **hot_standby** : Must be set to ON on standby/replica and has no effect on the master. However, when you setup your replication, parameters set on the master are automatically copied. This parameter is important to enable READS on slave. Otherwise, you cannot run your SELECT queries against slave.

The above parameters can be set on the master using these commands followed by a restart:

```
wal_level = replica
max_wal_senders = 3 # max number of walsender processes
wal_keep_segments = 64 # in logfile segments, 16MB each; 0 disables
listen_addresses = '*'
# or listen_address = ‘IP_OF_SERVER’
archive_mode = on
archive_command = 'cp %p /var/lib/postgresql/12/main/archive/%f'
synchronous_commit = local
synchronous_standby_names = 'pgslave001'
```

In the postgresql.conf file, the archive mode is enabled, so we need to create a new directory for the archive. Create a new archive directory, change the permission and change the owner to the postgres user.

```
mkdir -p /var/lib/postgresql/12/main/archive/
chmod 700 /var/lib/postgresql/12/main/archive/
chown -R postgres:postgres /var/lib/postgresql/12/main/archive/
```

Create the archive dir in the external storage -
Mount the location on docker configuration file (Docker has permission to read write and execute everything, we are running it through root)
```
/mnt/external_volume:/var/lib/postgresql/12/main/archive/
```

### Edit pg_hba.conf -

Add an entry to pg_hba.conf of the master to allow replication connections from the slave. The default location of pg_hba.conf is the data directory. However, you may modify the location of this file in the file  postgresql.conf. In Ubuntu/Debian, pg_hba.conf may be located in the same directory as the postgresql.conf file by default. You can get the location of postgresql.conf in Ubuntu/Debian by calling an OS command => pg_lsclusters.

```
# Localhost
host    replication     replica          127.0.0.1/32             md5
# PostgreSQL Master IP address
host    replication     replica          10.0.15.10/32            md5
# PostgreSQL SLave IP address
host    replication     replica          10.0.15.11/32            md5
```

Save and exit, then restart PostgreSQL.

```
systemctl restart postgresql
```

PostgreSQL is running under the IP address 10.0.15.10, check it with netstat command.
```
netstat -plntu
```

## Slave -

  YET TO BE UPDATED

## Storing the archive files -
* How to recreate database from the archive files?

## References -
1. [Setting up Master Slave Replication in PostgreSQL (upto version 11) using Dockers and external volumes](https://github.com/Rishabh04-02/Master-Slave-Replication-in-PSQL)
2. [Setting up Streaming Replication Postgresql](https://www.percona.com/blog/2018/09/07/setting-up-streaming-replication-postgresql/)
3. [Postgresql Raveland blog](https://blog.raveland.org/post/postgresql_sr/)
4. [How to set up master slave replication for postgresql](https://www.howtoforge.com/tutorial/how-to-set-up-master-slave-replication-for-postgresql-96-on-ubuntu-1604/)
5. [How to Set Up Streaming Replication in PostgreSQL 12
](https://www.percona.com/blog/2019/10/11/how-to-set-up-streaming-replication-in-postgresql-12/)

## Required Modifications
1. Edit postgresql.conf section
2. Add the process for docker configuration
3. Take  master and slave sample IP
4. Grammatical corrections.
