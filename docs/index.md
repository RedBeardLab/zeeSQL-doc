# index

## Tutorial for zeeSQL

### Setting up the environment

If you want to start exploring zeeSQL, this tutorial will walk you through the most important commands to know and how to use them.

The simplest way to follow this tutorial is to launch an instance of `zeeSQL` using docker.

```text
docker run --name zeesql --rm -d redbeardlab/zeesql
```

The next thing you will need to follow the tutorial is the `redis-cli`.

You can connect to the same docker container and start the `redis-cli` with:

```text
docker exec -it zeesql redis-cli
```

At this point you are inside the `redis-cli`, ready to interact with `Redis` and `zeeSQL`.

### Creating a database in zeeSQL

The very first step when working with `zeeSQL` it is to create a new database.

The database can be created with a single command.

```text
127.0.0.1:6379> ZEESQL.CREATE_DB DB
1) 1) "OK"
```

This command will create a new database, and it will associate it with the Redis key `DB`.

Any time we want to interact with this database, we will pass `DB` to the `zeeSQL` commands.

It is possible to create more than one database, you can create as many as you like, since they are very lightweight.

### Sending commands to the database

After creating a database, we want to interact with it. It is possible to interact with the database sending it commands.

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND 'select 1;' NO_HEADER
1) 1) "RESULT"
2) 1) (integer) 1
```

We have successfully sent our first command to `zeeSQL` and get our first `RESULT`, the integer 1.

### Modify the database structure

A database without tables is not very useful. We will now create a table to store information about users.

In the table we want to store the username, its score and the user email. The username and the email will be text fields, while the score will be an integer field.

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND 'CREATE TABLE users(username TEXT, score INT, email TEXT);'
1) 1) "DONE"
2) 1) (integer) 0
```

The operation was successfully and 0 rows have been modified.

However, we now have a table where we can store information about the user.

### Add data to the database

We can now start to add users to our table.

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND 'INSERT INTO users VALUES("jsmith", 3, "jon.smith@gmail.com");'
1) 1) "DONE"
2) 1) (integer) 1
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND 'INSERT INTO users VALUES("EvelineInArgentina", 3, "eve.frank@yahoo.com");'
1) 1) "DONE"
2) 1) (integer) 1
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND 'INSERT INTO users VALUES("DuffyAlone", 12, "mr.duffy@proton.com");'
1) 1) "DONE"
2) 1) (integer) 1
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND 'INSERT INTO users VALUES("far", 6, "farrington@nv.ru");'
1) 1) "DONE"
2) 1) (integer) 1
```

Each insert was successful and each one added one more row to the database.

### Query the database

After having added data to the database, we want to query those data back.

We can ask for the score of the user `jsmith`

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND 'SELECT score FROM users WHERE username = "jsmith"'
1) 1) "RESULT"
2) 1) "score"
3) 1) "INT"
4) 1) (integer) 3
```

In this case the score is an integer, and it is of value 3.

Or to know what users have a score greater than 5

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND 'SELECT username FROM users WHERE score > 5'
1) 1) "RESULT"
2) 1) "username"
3) 1) "TEXT"
4) 1) "DuffyAlone"
5) 1) "far"
```

In this other case the usernames are of type TEXT and are: `DuffyAlone` and `far`.

Since we are not modifying the database, instead of EXECUTING a command with `EXEC` we can just QUERY.

```text
127.0.0.1:6379> ZEESQL.QUERY DB COMMAND 'SELECT username FROM users WHERE score > 5'
1) 1) "RESULT"
2) 1) "username"
3) 1) "TEXT"
4) 1) "DuffyAlone"
5) 1) "far"
```

`QUERY` does not work when trying to modify the database.

```text
127.0.0.1:6379> ZEESQL.QUERY DB COMMAND 'INSERT INTO users VALUES("far", 6, "farrington@nv.ru");'
(error) Statement is not read only but it may modify the database, use `EXEC_STATEMENT` instead.
```

### Modify the data

Our users, keep using our platform, are increasing their score. We can increase the score with an update.

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND 'UPDATE users SET score = score + 1 WHERE username = "jsmith";'
1) 1) "DONE"
2) 1) (integer) 1
127.0.0.1:6379> ZEESQL.QUERY DB COMMAND 'SELECT score FROM users WHERE username = "jsmith"'
1) 1) "RESULT"
2) 1) "score"
3) 1) "INT"
4) 1) (integer) 4
```

In this example, we first increase the score of the user `jsmith` of one, and then we query the same score.

### Delete the data

Eventually our user will leave the platform, in this case we can delete them.

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND 'DELETE FROM users WHERE username = "far";'
1) 1) "DONE"
2) 1) (integer) 1
```

### Use arguments for your queries

Up to now, we send only SQL queries that contains all the parameters. It is also possible to send a query with placeholders followed by arguments. This is useful to avoid SQL injections attacks and to avoid string constructions at runtime.

In our example we can increment the score of a player by a specific amount.

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND 'UPDATE users SET score = score + ?2 WHERE username = ?1;' ARGS jsmith 3
1) 1) "DONE"
2) 1) (integer) 1
127.0.0.1:6379> ZEESQL.QUERY DB COMMAND 'SELECT score FROM users WHERE username = ?1' ARGS jsmith
1) 1) "RESULT"
2) 1) "score"
3) 1) "INT"
4) 1) (integer) 7
```

The first argument is `?1` \(not `?0`\) and the second argument is `?2`.

## Using secondary indexes \(or search by values\) in Redis

Now that we have understood how to deal with standard zeeSQL databases and how to query them, we can move forward.

Now we will introduce zeeSQL secondary indexes, or how to search and project Redis `hash` data.

In Redis, it is common to store values as hash. Each hash is univocally identify by a key, and it has one of more field. Each field has a value associated with.

Redis already provide fast access to elements by their key. But it is not possible to search keys from their values.

With zeeSQL we can solve this problem, but automatically push the hashes keys and values to a specific table.

We will work with a simple telemetric system. We have different sensors, each sensor send a timestamp, a temperature value and a humidity value.

### Saving the data into Redis

As first step let's see how we model the data in raw Redis, using Redis Hashes.

```text
127.0.0.1:6379> HMSET sensor:001:1612733809 timestamp 1612733809 sensor 1 temperature 23 humidity 56
OK
127.0.0.1:6379> HMSET sensor:001:1612633809 timestamp 1612633809 sensor 1 temperature 18 humidity 21
OK
127.0.0.1:6379> HMSET sensor:001:1612633819 timestamp 1612633819 sensor 1 temperature 20 humidity 23
OK
127.0.0.1:6379> HMSET sensor:002:1612733809 timestamp 1612733809 sensor 2 temperature 32 humidity 11
OK
127.0.0.1:6379> HMSET sensor:002:1612633809 timestamp 1612633809 sensor 2 temperature 21 humidity 16
OK
127.0.0.1:6379> HMSET sensor:002:1612633819 timestamp 1612633819 sensor 2 temperature 23 humidity 12
OK
```

The key is in the form `sensor:$sendor_id:$timestamp` and the other fields contains the telemetries' data.

### Creating an index

We now want to store the information in the sensor in an SQL table, for easier access.

```text
127.0.0.1:6379> ZEESQL.INDEX DB NEW TABLE sensors PREFIX sensor:* SCHEMA timestamp INT sensor INT temperature INT humidity INT
OK
```

The command creates a new secondary index associate with the table `sensors`. The index will be concerned only for the hashes which key start with the prefix `sensor:` \(`*` being a catch-all\). The schema used by the index will have 4 rows, each of them will be an INTEGER, and the name of those columns are respectively `timestamp`. `sensor`, `temperature` and `humidity`.

The creation of an index, imply the creation of the table in the database. If the table already exists, it is assumed to contain the correct columns.

### Reading data from the index

As soon as the index is created, the Redis keys space is scanned and the hashes are added to the table. Which means that we can immediately query the table.

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND 'select * from sensors;'
1) 1) "RESULT"
2) 1) "key"
   2) "timestamp"
   3) "sensor"
   4) "temperature"
   5) "humidity"
3) 1) "TEXT"
   2) "INT"
   3) "INT"
   4) "INT"
   5) "INT"
4) 1) "sensor:001:1612633809"
   2) (integer) 1612633809
   3) (integer) 1
   4) (integer) 18
   5) (integer) 21
5) 1) "sensor:002:1612733809"
   2) (integer) 1612733809
   3) (integer) 2
   4) (integer) 32
   5) (integer) 11
6) 1) "sensor:001:1612733809"
   2) (integer) 1612733809
   3) (integer) 1
   4) (integer) 23
   5) (integer) 56
7) 1) "sensor:002:1612633819"
   2) (integer) 1612633819
   3) (integer) 2
   4) (integer) 23
   5) (integer) 12
8) 1) "sensor:002:1612633809"
   2) (integer) 1612633809
   3) (integer) 2
   4) (integer) 21
   5) (integer) 16
9) 1) "sensor:001:1612633819"
   2) (integer) 1612633819
   3) (integer) 1
   4) (integer) 20
   5) (integer) 23
```

### Modify the table of the index

It is possible to add, remove and update values to the index table manually. Do not do that, the data between Redis and zeeSQL will go out of sync.

### Adding hashes

The index continuously listens to the HASH commands of Redis and keeps the table in sync.

```text
127.0.0.1:6379> HMSET sensor:003:1612633819 timestamp 1612633819 sensor 3 temperature 50 humidity 8
OK
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND 'select * from sensors where sensor = 3;'
1) 1) "RESULT"
2) 1) "key"
   2) "timestamp"
   3) "sensor"
   4) "temperature"
   5) "humidity"
3) 1) "TEXT"
   2) "INT"
   3) "INT"
   4) "INT"
   5) "INT"
4) 1) "sensor:003:1612633819"
   2) (integer) 1612633819
   3) (integer) 3
   4) (integer) 50
   5) (integer) 8
```

In this example we added the sensor with ID 3, after the secondary index was already in place.

### Removing hashes

Similarly, if a hash is deleted, the correct row is deleted from the table.

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND 'select key from sensors where sensor = 2;'
1) 1) "RESULT"
2) 1) "key"
3) 1) "TEXT"
4) 1) "sensor:002:1612733809"
5) 1) "sensor:002:1612633819"
6) 1) "sensor:002:1612633809"
127.0.0.1:6379> DEL sensor:002:1612633809
(integer) 1
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND 'select key from sensors where sensor = 2;'
1) 1) "RESULT"
2) 1) "key"
3) 1) "TEXT"
4) 1) "sensor:002:1612733809"
5) 1) "sensor:002:1612633819"
```

As you can see we delete a hash, and the correct row was deleted also from the table.

### Updating hashes

As with deletion, updating a hash is also reflected on the index table.

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND 'select key, temperature from sensors where sensor = 3;'
1) 1) "RESULT"
2) 1) "key"
   2) "temperature"
3) 1) "TEXT"
   2) "INT"
4) 1) "sensor:003:1612633819"
   2) (integer) 50
127.0.0.1:6379> HINCRBY sensor:003:1612633819 temperature 33
(integer) 83
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND 'select key, temperature from sensors where sensor = 3;'
1) 1) "RESULT"
2) 1) "key"
   2) "temperature"
3) 1) "TEXT"
   2) "INT"
4) 1) "sensor:003:1612633819"
   2) (integer) 83
```

In this case we update a hash, and also the table was updated.

### Querying the secondary index table

The secondary index table, it is just a plain SQL table. zeeSQL does not add any index on the table. User is free to add the SQL indexes it desired to the secondary index table.

Without any index, each query will go through a full table scan.

## End

I hope that this tutorial was helpful :\)

If you have any question, feel free to contact me simone@redbeardlab.com or on github: [RedBeardLab/zeeSQL-doc](https://github.com/RedBeardLab/zeeSQL-doc)

