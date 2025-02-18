# PostgreSQL Replication
PostgreSQL replication is the process of copying data from one database server (primary) to
another (replica) in real-time or near real-time. 
This allows for high availability, load balancing, and disaster recovery.

## 1. Get our Primary (Master) PostgreSQL up and running
Let's start by running our primary PostgreSQL in docker

Few things to note here:
- We start our instance with a different name to identify it as the first instance with the `--name master` flag and `2` for the second instance
- Set unique data volumes for data between instances
- Set unique config files for each instance

## Make sure you have config files and directory
if not download
```bash
sudo apt install -y unzip && mkdir -p "${PWD}/postgres/master/" && wget -O "${PWD}/postgres/master/master.zip" https://github.com/Swe-HimelRana/Notebook/releases/download/postgres-replica/master.zip && unzip -d "${PWD}/postgres/master/" "${PWD}/postgres/master/master.zip" > /dev/null 2>&1 && rm "${PWD}/postgres/master/master.zip" && mv "${PWD}/postgres/master/master" "${PWD}/postgres/master/config"
```
## Make sure archive directory available and postgres user will have access in archive directory
```bash
mkdir -p "${PWD}/postgres/master/archive" && sudo chown -R 999:999 "${PWD}/postgres/master/archive"
```

## Start with instance 1 (master):
```bash
docker run --name master \
  -e POSTGRES_PASSWORD=himosoft123 \
  -e POSTGRES_DB=postgresdb \
  -e PGDATA=/data \
  -v ${PWD}/postgres/master/data:/data \
  -v ${PWD}/postgres/master/config:/config \
  -v ${PWD}/postgres/master/archive:/mnt/server/archive \
  -p 5000:5432 \
  postgres:15.0 -c config_file=/config/postgresql.conf
```
Default postgresql username: `postgres`

Check timezone: 
```bash
docker exec -it master psql -U postgres -c "SHOW timezone;"
```
Check pg_hba file location

```bash
docker exec -it master psql -U postgres -c "SHOW hba_file;"
```
Check shared_buffer
```bash
docker exec -it master psql -U postgres -c "SHOW shared_buffers;"
```
Check datestyle

```bash
docker exec -it master psql -U postgres -c "SHOW datestyle;"
```

## Create Replication User
In order to take a backup we will use a new PostgreSQL user account which has the permissions to do replication.
Let's create this user account by logging into `postgres-1`:

```bash 
docker exec -it master psql -U postgres -c "CREATE ROLE replicationuser WITH REPLICATION LOGIN PASSWORD 'NewStrongPassword' CONNECTION LIMIT 5;"
```
verify replicationUser created or not 

```bash
docker exec -it master psql -U postgres -c "SELECT rolname, rolsuper, rolreplication, rolconnlimit FROM pg_roles WHERE rolname = 'replicationuser';"
```


## Enable Write-Ahead Log and Replication
There is quite a lot to read about PostgreSQL when it comes to high availability.

The first thing we want to take a look at is [WAL](https://www.postgresql.org/docs/current/wal-intro.html)
Basically PostgreSQL has a mechanism of writing transaction logs to file and does not accept the transaction until its been written to the transaction log and flushed to disk.

This ensures that if there is a crash in the system, that the database can be recovered from the transaction log.
Hence it is "writing ahead".

More documentation for configuration [wal_level](https://www.postgresql.org/docs/current/runtime-config-wal.html) and [max_wal_senders](https://www.postgresql.org/docs/current/runtime-config-replication.html)

```bash
wal_level = replica
max_wal_senders = 3
```
# 2. Get our Secondary (replica) PostgreSQL up and running
## Make sure you have config files and directory
if not download
```bash
sudo apt install -y unzip && mkdir -p "${PWD}/postgres/replica/" && wget -O "${PWD}/postgres/replica/replica.zip" https://github.com/Swe-HimelRana/Notebook/releases/download/postgres-replica/replica.zip && unzip -d "${PWD}/postgres/replica/" "${PWD}/postgres/replica/replica.zip" > /dev/null 2>&1 && rm "${PWD}/postgres/replica/replica.zip" && mv "${PWD}/postgres/replica/replica" "${PWD}/postgres/replica/config"
```
## Take a base backup
To take a database backup, we'll be using the [pg_basebackup](https://www.postgresql.org/docs/current/app-pgbasebackup.html) utility.

The utility is in the PostgreSQL docker image, so let's run it without running a database as all we need is the `pg_basebackup` utility.
Note that we also mount our blank data directory as we will make a new backup in there:

```bash
docker run -it --rm \
  -v ${PWD}/postgres/master/pgdata:/data \
  --entrypoint /bin/bash \
  postgres:15.0
```
Take the backup by logging into `master` with our `replicationuser` and writing the backup to `/data`.

```bash
pg_basebackup -h your_master_server_ip -p 5432 -U replicationuser -D /data/ -Fp -Xs -R
```

Now we should see PostgreSQL data ready for our second instance in `${PWD}/postgres-2/pgdata`

## Start standby (replica) instance

```bash
docker run -it --rm --name postgres-2 \
  -e POSTGRES_USER=postgresadmin \
  -e POSTGRES_PASSWORD=admin123 \
  -e POSTGRES_DB=postgresdb \
  -e PGDATA=/data \
  -v ${PWD}/postgres-2/pgdata:/data \
  -v ${PWD}/postgres-2/config:/config \
  -v ${PWD}/postgres-2/archive:/mnt/server/archive \
  -p 5001:5432 \
  postgres:15.0 -c config_file=/config/postgresql.conf
```

## Test the replication

Let's test our replication by creating a new table in `postgres-1`
On our primary(master) instance (vps), lets do that:

```bash
# login to postgres
psql --username=postgresadmin postgresdb

#create a table
CREATE TABLE customers (firstname text, customer_id serial, date_created timestamp);

#show the table
\dt
```

Now lets log into our `postgres-2` instance and view the table:

```bash
docker exec -it postgres-2 bash

# login to postgres
psql --username=postgresadmin postgresdb

#show the tables
\dt
```
