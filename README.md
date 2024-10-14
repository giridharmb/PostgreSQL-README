[Generate A Random Table](#generate-a-random-table)

[Compare JSON Values](#compare-json-values)

[Get Table Size](#get-table-size)

[Fetch Running Queries Older Than 5 Minutes](#fetch-running-queries-older-than-5-minutes)

[Clear PGSQL Locks](#clear-pgsql-locks)

[PGSQL Settings](#pgsql-settings)

[Foreign Data Wrapper](#foreign-data-wrapper)

[Monitor PostgreSQL Connections](#monitor-postgresql-connections)

<hr/>

#### [Generate A Random Table](#generate-a-random-table)

- Login Into GCP Console
- Create PG SQL Instance
- Create a Database (Ex: `test-db`)
- Add a user with some random username (Ex: `testuser`) and random password (Ex: `testpassword`)

Populate `~/.pgpass` with below line

```
my-pgsql-host.company.com:5432:test-db:testuser:testpassword
```

Login into the DB and create a table

```
psql -h my-pgsql-host.company.com test-db testuser
```

Create Table `t_random`

```sql
CREATE TABLE t_random(
   random_num INT NOT NULL,
   random_float DOUBLE PRECISION NOT NULL,
   md5 TEXT NOT NULL
);
```

If You Want To Drop (Existing) Table

```sql
drop table t_random;
```

See Table Details (For `t_random`)

```sql
test-db=> \d
          List of relations
 Schema |   Name   | Type  |  Owner
--------+----------+-------+----------
 public | t_random | table | testuser
(1 row)

test-db=> \d t_random;
                     Table "public.t_random"
    Column    |       Type       | Collation | Nullable | Default
--------------+------------------+-----------+----------+---------
 random_num   | integer          |           | not null |
 random_float | double precision |           | not null |
 md5          | text             |           | not null |
```

Insert Random Data Into Table `t_random` (Below Example Will Insert 100 Rows)

```sql
INSERT INTO t_random (random_num, random_float, md5) 
    SELECT 
    floor(random()* (999-100 + 1) + 100), 
    random(), 
    md5(random()::text) 
 from generate_series(1,100);
```

In The Above Example

```
floor(random()* (999-100 + 1) + 100) -> Will Generate Ranom Number Between 100 and 999
random()                             -> Will Generate Float Number
md5(random()::text)                  -> Will Generate MD5 (Hash) Of a Random Float
```

After Populating Some Rows, Let Us Fetch 10 Rows To See The Table Rows/Columns


```sql
test-db=> select * from t_random limit 10;
 random_num |    random_float     |               md5
------------+---------------------+----------------------------------
        642 | 0.04866303424917717 | 4148034f7ff59deddce0507cf5df6136
        299 | 0.42088215715087784 | 50cf40c1cd51fa0c9bcda04d19118637
        452 |  0.5040329323515316 | d83b110f8c24dc876feb39e1d32fc9a9
        558 |   0.680895277194864 | bd5e6593d1133df5abbe2dcd7ed41b22
        436 |  0.9993317095718943 | b0e288ee1d4d5ea7c50995091222526c
        143 |  0.4972750511722168 | 1ecf469484a66c646f33f1ad26d2c120
        987 |  0.7588865512592768 | 04705f29ec44fede34940cd954acf20c
        337 |  0.9723930089149349 | fe0f5b228256a75beccfe7156af05616
        321 |   0.637456665167111 | 5522e09f9a97abd695bf821ce1eca750
        896 |  0.9731822082784944 | 34726f12c5bae77be4b3bb0293ccb38a
(10 rows)
```

#### [Compare JSON Values](#compare-json-values)

```sql
postgres=> create table table1 ( key text NOT NULL , value jsonb NOT NULL );

CREATE TABLE

postgres=> \d
         List of relations
 Schema |  Name  | Type  |  Owner
--------+--------+-------+----------
 public | table1 | table | postgres
(1 row)

postgres=> \d table1

              Table "public.table1"
 Column | Type  | Collation | Nullable | Default
--------+-------+-----------+----------+---------
 key    | text  |           | not null |
 value  | jsonb |           | not null |

postgres=> insert into table1 values ( 'k1' , '{"x":11, "y":22}');
INSERT 0 1

postgres=> select * from table1;

 key |       value
-----+--------------------
 k1  | {"x": 11, "y": 22}
(1 row)

postgres=> insert into table1 values ( 'k2' , '{"y":22, "x":11}');
INSERT 0 1

postgres=> select * from table1;
 key |       value
-----+--------------------
 k1  | {"x": 11, "y": 22}
 k2  | {"x": 11, "y": 22}
(2 rows)

postgres=> select value from table1 where key = 'k1';

       value
--------------------
 {"x": 11, "y": 22}
(1 row)

postgres=> select (select value from table1 where key = 'k1') = (select value from table1 where key = 'k2');
 ?column?
----------
 t
(1 row)
```

```sql
postgres=> insert into table1 values ( 'k3' , '{"x":11, "y":22, "x1":[1,2] , "y1":[3,4]}');
INSERT 0 1

postgres=> insert into table1 values ( 'k4' , '{"y":22, "x":11, "x1":[1,2] , "y1":[3,4]}');
INSERT 0 1

postgres=> select * from table1;

 key |                     value
-----+------------------------------------------------
 k1  | {"x": 11, "y": 22}
 k2  | {"x": 11, "y": 22}
 k3  | {"x": 11, "y": 22, "x1": [1, 2], "y1": [3, 4]}
 k4  | {"x": 11, "y": 22, "x1": [1, 2], "y1": [3, 4]}
(4 rows)

postgres=> select (select value from table1 where key = 'k3') = (select value from table1 where key = 'k4');
 ?column?
----------
 t
(1 row)

postgres=> insert into table1 values ( 'k5' , '{"y":22, "x":11, "x1":[1,2] , "y1":[3,4], "z" : "1234" }');
INSERT 0 1

postgres=> select * from table1;
 key |                            value
-----+-------------------------------------------------------------
 k1  | {"x": 11, "y": 22}
 k2  | {"x": 11, "y": 22}
 k3  | {"x": 11, "y": 22, "x1": [1, 2], "y1": [3, 4]}
 k4  | {"x": 11, "y": 22, "x1": [1, 2], "y1": [3, 4]}
 k5  | {"x": 11, "y": 22, "z": "1234", "x1": [1, 2], "y1": [3, 4]}
(5 rows)


postgres=> select (select value from table1 where key = 'k3') = (select value from table1 where key = 'k4');
 ?column?
----------
 t
(1 row)

postgres=> select (select value from table1 where key = 'k3') = (select value from table1 where key = 'k5');
 ?column?
----------
 f
(1 row)
```

#### [Get Table Size](#get-table-size)

```sql
SELECT
    relname AS "relation",
    pg_size_pretty (
        pg_total_relation_size (C .oid)
    ) AS "total_size"
FROM
    pg_class C
LEFT JOIN pg_namespace N ON (N.oid = C .relnamespace)
    WHERE
        nspname NOT IN (
            'pg_catalog',
            'information_schema'
        )
    AND C .relkind <> 'i'
    AND nspname !~ '^pg_toast'
    ORDER BY
        pg_total_relation_size (C .oid) DESC;
```

#### [Fetch Running Queries Older Than 5 Minutes](#fetch-running-queries-older-than-5-minutes)

```sql
SELECT
    pid,
    now() - pg_stat_activity.query_start AS duration,
    query,
    state
FROM pg_stat_activity
    WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes';
```

#### [Clear PGSQL Locks](#clear-pgsql-locks)

```sql
SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE pid <> pg_backend_pid();
```

#### [PGSQL Settings](#pgsql-settings)

```sql
postgres=# select * from pg_settings where name = 'idle_session_timeout' \gx

-[ RECORD 1 ]---+-------------------------------------------------------------------------------
name            | idle_session_timeout
setting         | 0
unit            | ms
category        | Client Connection Defaults / Statement Behavior
short_desc      | Sets the maximum allowed idle time between queries, when not in a transaction.
extra_desc      | A value of 0 turns off the timeout.
context         | user
vartype         | integer
source          | default
min_val         | 0
max_val         | 2147483647
enumvals        |
boot_val        | 0
reset_val       | 0
sourcefile      |
sourceline      |
pending_restart | f

select pg_reload_conf();

postgres=# SHOW data_directory;

       data_directory
-----------------------------
 /var/lib/postgresql/14/main
(1 row)

sudo -u postgres psql -c "SELECT version();"
```

#### Stored Procedure

```sql
create or replace procedure public.refresh_table_matviews()
LANGUAGE plpgsql as
$$ begin
    REFRESH MATERIALIZED VIEW CONCURRENTLY public.table1;
    REFRESH MATERIALIZED VIEW CONCURRENTLY public.table2;
    commit;
end $$;
```

#### Stored Procedure Definition

```sql
\df+ refresh_table_matviews
```

#### CRON Job

```sql
CREATE EXTENSION pg_cron;

GRANT USAGE ON SCHEMA cron TO postgres;

select * from cron.job;

SELECT * FROM cron.job_run_details;

select cron.schedule('job_01', '*/10 * * * *','CALL public.refresh_table_matviews();');

SELECT cron.unschedule(7); -- unschedule cron-job with jobid=7
```

#### [Foreign Data Wrapper](#foreign-data-wrapper)

```sql
-- First, you need to install and enable the postgres_fdw extension on the PostgreSQL database 
-- where you want to create the foreign tables.

CREATE EXTENSION IF NOT EXISTS postgres_fdw;

-- Next, create a foreign server object that defines the connection information for the remote PostgreSQL server.

CREATE SERVER remote_server
FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (host 'remote_host', port '5432', dbname 'remote_db');

-- Create a user mapping that links a local PostgreSQL role to a remote PostgreSQL role, 
-- specifying the credentials to use for the remote connection.

CREATE USER MAPPING FOR local_user
SERVER remote_server
OPTIONS (user 'remote_user', password '<REDACTED>');

-- You can import the schema from the remote database, which will create foreign tables in your 
-- local database corresponding to tables in the remote database.

IMPORT FOREIGN SCHEMA public
FROM SERVER remote_server
INTO local_schema;

-- Optional:
-- Alternatively, you can manually create foreign tables that map to specific tables in the remote database.

CREATE FOREIGN TABLE local_table_name (
    id INTEGER,
    name TEXT
)
SERVER remote_server
OPTIONS (schema_name 'remote_schema', table_name 'remote_table_name');

-- You can now query the foreign table just like any other local table.

SELECT * FROM local_table_name;
```

#### [Monitor PostgreSQL Connections](#monitor-postgresql-connections)

```sql
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    client_port,
    state,
    query,
    backend_start
FROM
    pg_stat_activity
ORDER BY
    backend_start;
```

```sql
SELECT count(*), client_addr
FROM pg_stat_activity
GROUP BY client_addr
ORDER BY count DESC;
```

```bash
sudo netstat -natp | grep postgres
```

```bash
sudo ss -tnp | grep postgres
```

```sql
SHOW max_connections;
```

```sql
SELECT count(*) FROM pg_stat_activity;
```

```bash
tail -f /var/log/postgresql/postgresql-<version>-main.log
```