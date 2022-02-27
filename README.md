#### PGSQL SQL Commands

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