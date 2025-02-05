## Running a simple postgres database
```bash
 docker run --name some-postgres -e POSTGRES_PASSWORD=mysecretpassword -p 5432:5432 -d postgres

```
#### The authentication credentials for the PostgreSQL instance will be:
- Username: `postgres` (default PostgreSQL superuser
- Password: `mysecretpassword` (set via the `POSTGRES_PASSWORD` environment variable)
- Port: `5432`

## Persisting Data

To persist data to PostgreSQL, we simply mount a docker volume. </br>
This is the way to persist container data. </br>
PostgreSQL stores its data by default under `/var/lib/postgresql/data` 
Also take note we are running a specific version of PostgreSQL now:

```bash
docker run -it --rm --name postgres \
  -e POSTGRES_PASSWORD=admin123 \
  -p 5432:5432 \
  -v ${PWD}/pgdata:/var/lib/postgresql/data \
  postgres:15.0
```
#### enter the container

```bash
docker exec -it postgres bash
```
# login to postgres
psql -h localhost -U postgres

#### create a table
```bash
CREATE TABLE customers (firstname text,lastname text, customer_id serial);

#add record
INSERT INTO customers (firstname, lastname) VALUES ( 'Bob', 'Smith');

#show table
\dt

# get records
SELECT * FROM customers;

# quit 
\q
```
Now we can see our data persisted by killing and removing the container:
