#PostgreSQL Replication
PostgreSQL replication is the process of copying data from one database server (primary) to
another (replica) in real-time or near real-time. 
This allows for high availability, load balancing, and disaster recovery.

## Get our Primary (Master) PostgreSQL up and running
Let's start by running our primary PostgreSQL in docker

Few things to note here:
- We start our instance with a different name to identify it as the first instance with the `--name postgres-1` flag and `2` for the second instance
- Set unique data volumes for data between instances
- Set unique config files for each instance

Start with instance 1 (master):

```bash
docker run -it --rm --name postgres-1 \
  -e POSTGRES_USER=postgresadmin \
  -e POSTGRES_PASSWORD=admin123 \
  -e POSTGRES_DB=postgresdb \
  -e PGDATA=/data \
  -v ${PWD}/postgres-1/pgdata:/data \
  -v ${PWD}/postgres-1/config:/config \
  -v ${PWD}/postgres-1/archive:/mnt/server/archive \
  -p 5000:5432 \
  postgres:15.0 -c config_file=/config/postgresql.conf
```

## Create Replication User
In order to take a backup we will use a new PostgreSQL user account which has the permissions to do replication.
Let's create this user account by logging into `postgres-1`:

```bash 
docker exec -it postgres-1 bash

# create a new user
createuser -U postgresadmin -P -c 5 --replication replicationUser

exit
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

## Take a base backup
To take a database backup, we'll be using the [pg_basebackup](https://www.postgresql.org/docs/current/app-pgbasebackup.html) utility.

The utility is in the PostgreSQL docker image, so let's run it without running a database as all we need is the `pg_basebackup` utility.
Note that we also mount our blank data directory as we will make a new backup in there:

```bash
docker run -it --rm \
  -v ${PWD}/postgres-2/pgdata:/data \
  --entrypoint /bin/bash \
  postgres:15.0
```
Take the backup by logging into `postgres-1` with our `replicationUser` and writing the backup to `/data`.

```bash
pg_basebackup -h postgres-1 -p 5432 -U replicationUser -D /data/ -Fp -Xs -R
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
```bash

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
