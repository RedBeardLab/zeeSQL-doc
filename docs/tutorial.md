# Tutorial for zeeSQL

## Setting up the environment

If you want to start exploring zeeSQL, this tutorial will walk you through the most important commands to know and how to use them.

The simplest way to follow this tutorial is to launch an instance of `zeeSQL` using docker.

```
docker run --name zeesql --rm -d library/redis
```

The next thing you will need to follow the tutorial is the `redis-cli`.

You can connect to the same docker container and start the `redis-cli` with:

```
docker exec -it zeesql redis-cli
```

At this point you are inside the `redis-cli`, ready to interact with `Redis` and `zeeSQL`.


## Creating a database in zeeSQL

The very first step when working with `zeeSQL` it is to create a new database.

The database can be created with a single command.

```
127.0.0.1:6379> ZEESQL.CREATE_DB DB
1) 1) "OK"
```

This command will create a new database and it will associate it with the redis key `DB`.

Any time we want to interact with this database, we will pass `DB` to the `zeeSQL` commands.

It is possible to create more than one database, you can create as many as you like, since they are very lightweigjt.

## Sending commands to the database

After creating a database, we want to interact with it. It is possible to interact with the database sending it commands.

```
127.0.0.1:6379> REDISQL.V2.EXEC DB COMMAND 'select 1;' NO_HEADER
1) 1) "RESULT"
2) 1) (integer) 1
```

We have successfully send our first command to `zeeSQL` and get our first `RESULT`, the integer 1.

## Modify the database structure

A database without tables is not very useful.
We will now create a table to store information about users.

In the table we want to store the username, its score and the user email.
The username and the email will be text fields, while the score will be an integer field.

```
127.0.0.1:6379> REDISQL.V2.EXEC DB COMMAND 'CREATE TABLE users(username TEXT, score INT, email TEXT);'
1) 1) "DONE"
2) 1) (integer) 0
```

The operation was successfully and 0 rows have been modified.

However, we now have a table were we can store information about the user.

## Add data to the database

We can now starting to add users to our table.

```
127.0.0.1:6379> REDISQL.V2.EXEC DB COMMAND 'INSERT INTO users VALUES("jsmith", 3, "jon.smith@gmail.com");'
1) 1) "DONE"
2) 1) (integer) 1
127.0.0.1:6379> REDISQL.V2.EXEC DB COMMAND 'INSERT INTO users VALUES("EvelineInArgentina", 3, "eve.frank@yahoo.com");'
1) 1) "DONE"
2) 1) (integer) 1
127.0.0.1:6379> REDISQL.V2.EXEC DB COMMAND 'INSERT INTO users VALUES("DuffyAlone", 12, "mr.duffy@proton.com");'
1) 1) "DONE"
2) 1) (integer) 1
127.0.0.1:6379> REDISQL.V2.EXEC DB COMMAND 'INSERT INTO users VALUES("far", 6, "farrington@nv.ru");'
1) 1) "DONE"
2) 1) (integer) 1
```

Each insert was successful and each one added one more row to the database.

## Query the database

After having added data to the database, we want to query those data back.

We can ask for the score of the user `jsmith`

```
127.0.0.1:6379> REDISQL.V2.EXEC DB COMMAND 'SELECT score FROM users WHERE username = "jsmith"'
1) 1) "RESULT"
2) 1) "score"
3) 1) "INT"
4) 1) (integer) 3
```

In this case the score is an integer and it is of value 3.


Or to know what users have a score greater than 5

```
127.0.0.1:6379> REDISQL.V2.EXEC DB COMMAND 'SELECT username FROM users WHERE score > 5'
1) 1) "RESULT"
2) 1) "username"
3) 1) "TEXT"
4) 1) "DuffyAlone"
5) 1) "far"
```

In this other case the usernames are of type TEXT and are: `DuffyAlone` and `far`.

Since we are not modifying the database, instead of EXECUTING a command with `EXEC` we can just QUERY.

```
127.0.0.1:6379> REDISQL.V2.QUERY DB COMMAND 'SELECT username FROM users WHERE score > 5'
1) 1) "RESULT"
2) 1) "username"
3) 1) "TEXT"
4) 1) "DuffyAlone"
5) 1) "far"
```

`QUERY` does not work when trying to modify the database.

```
127.0.0.1:6379> REDISQL.V2.QUERY DB COMMAND 'INSERT INTO users VALUES("far", 6, "farrington@nv.ru");'
(error) Statement is not read only but it may modify the database, use `EXEC_STATEMENT` instead.
```

## Modify the data

Our users, keep using our platform, are increasing their score. We can increase the score with an update.

```
127.0.0.1:6379> REDISQL.V2.EXEC DB COMMAND 'UPDATE users SET score = score + 1 WHERE username = "jsmith";'
1) 1) "DONE"
2) 1) (integer) 1
127.0.0.1:6379> REDISQL.V2.QUERY DB COMMAND 'SELECT score FROM users WHERE username = "jsmith"'
1) 1) "RESULT"
2) 1) "score"
3) 1) "INT"
4) 1) (integer) 4
```

In this example, we first increase the score of the user `jsmith` of one and then we query the same score.

## Delete the data

Eventually our user will leave the platform, in this case we can delete them.

```
127.0.0.1:6379> REDISQL.V2.EXEC DB COMMAND 'DELETE FROM users WHERE username = "far";'
1) 1) "DONE"
2) 1) (integer) 1
```

# DONE


```
redis-cli -h demo.redisql.com
```

If you connect to the demo machine, please remember that it is a shared machine and other users can be using it at the same time. Also note that we clean up the machine every hour.

<iframe width="560" height="315" src="https://www.youtube.com/embed/YRusC-AIq_g" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe><Paste>

## Create your database

The first fundamental concept in zeeSQL is the concept of database.

One database is a SQLite database that is associated with a Redis key, creating a new database also means to spawn a new thread that will be the one that actually does the work leaving the main Redis thread available to answer standard Redis queries.

To create a new zeeSQL database just run:

```
REDISQL.CREATE_DB TUTORIAL

OK
```

This will create a new in-memory database associated with the name `TUTORIAL`.

(If you receive an error at this stage, most likely someone else is following the tutorial, no worries, instead of `TUTORIAL` simply use another name).

## First interaction

Now that we have create a new database, we can interact with it.

The simplest command to use is the `REDISQL.EXEC` command that will execute one query against our newly created database, try:

```
REDISQL.EXEC TUTORIAL "SELECT 1;"

1) 1) (integer) 1
```

This command should simply return 1.

What we can do, is to create a simple SQL table.

```
REDISQL.EXEC TUTORIAL "CREATE TABLE first_table(a integer, b string);"

1) DONE
2) (integer) 0
```

Congrats, you have just created a new table.

## Insert values

After we create our first table, we can insert some value into it.

```
REDISQL.EXEC TUTORIAL "INSERT INTO first_table VALUES(1, 'the first row'), (2, 'the second row');"

1) DONE
2) (integer) 2
```

This command will return 2, the number of rows inserted.

## Query values

Now we can try to select some values from our table, as an example:

```
REDISQL.EXEC TUTORIAL "SELECT * FROM first_table;"

1) 1) (integer) 1
   2) "the first row"
2) 1) (integer) 2
   2) "the second row"
```

Another way to select values is to use the `QUERY` command, this command works only if the query you send is a read-only query.

```
REDISQL.QUERY TUTORIAL "SELECT * FROM first_table WHERE a = 1;"

1) 1) (integer) 1
   2) "the first row"
```

## Statements

Writing this commands all over again get tedious, moreover, it is inefficient since the SQL engine needs to parse and compile the SQL query every time. The solution, in this case, is the use of statements.

```
REDISQL.CREATE_STATEMENT TUTORIAL filter_on_a "SELECT * FROM first_table WHERE a = ?1;"


OK
```

The statements can both query (as above) or insert rows:

```
REDISQL.CREATE_STATEMENT TUTORIAL new_row "INSERT INTO first_table VALUES(?1, 'insert_from_statement');"

OK
```

Statements can be invoked using:

```
REDISQL.EXEC_STATEMENT TUTORIAL new_row 5

1) DONE
2) (integer) 1
```

Or if they are read-only statements:

```
REDISQL.QUERY_STATEMENT TUTORIAL filter_on_a 5

1) 1) (integer) 5
   2) "insert_from_statement"
```

## Push to streams

Sometimes you may want to store the result of your queries in Redis itself, to consume it little by little, or for caching. In such cases, the solution is to push the results into a Redis stream.

```
REDISQL.QUERY.INTO tutorial_stream TUTORIAL "SELECT * FROM first_table;"

1) 1) "tutorial_stream"
   2) "1569174107783-0"
   3) "1569174107783-2"
   4) (integer) 3
```

This will push the result of the query into the stream `tutorial_stream` and it will return:
1. The name of the stream
2. The first ID
3. The last ID
4. The total number of elements added

In order to retrieve the values from the Redis stream, you can use the standard Redis API:

```
XRANGE tutorial_stream - +

1) 1) "1569174107783-0"
   2) 1) "int:a"
      2) "1"
      3) "text:b"
      4) "the first row"
2) 1) "1569174107783-1"
   2) 1) "int:a"
      2) "2"
      3) "text:b"
      4) "the second row"
3) 1) "1569174107783-2"
   2) 1) "int:a"
      2) "5"
      3) "text:b"
      4) "insert_from_statement"
```

# End

I hope that this tutorial was helpful :)

If you have any question, feel free to contact me simone@redbeardlab.com or on github: [RedBeardLab/rediSQL](https://github.com/RedBeardLab/rediSQL)
