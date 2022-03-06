[Generate A Random Table](#generate-a-random-table)

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
