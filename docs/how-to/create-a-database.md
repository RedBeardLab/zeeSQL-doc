# How to create a new database in zeeSQL

To start using zeeSQL you need to create a database.

In zeeSQL you can have as many databases as necessary.

Each database you create in zeeSQL is an SQLite database, it can either be an in-memory database or a file-backed database.

Each database is associate with a Redis key.

Delete the key, and the database is deleted. If the database is backed by a file, then the file is closed, but it is not deleted.

To create a database, you can use the command [`ZEESQL.CREATE_DB`][createdb]

```
127.0.0.1:6379> ZEESQL.CREATE_DB DB
1) 1) "OK"
```

In this way a new in-memory database is created, it is associated with the `DB` key in Redis.

In-memory databases will always be created empty in zeeSQL.

To create a database-backed by a file, you need to pass the `PATH` flag.

```
127.0.0.1:6379> ZEESQL.CREATE_DB FILE_DB PATH file.sqlite
1) 1) "OK"
```

The `FILE_DB` database, will be associate with an SQLite database stored in disk, in the file `file.sqlite`.

Please note that databases stored in disk have different performances than the databases stored in memory. Usually slower.

## Importing an external database

Sometimes you may want to import some data from some other source.

zeeSQL allows you to load any SQLite database.

Suppose you already have an SQLite database that contains your users, and that you saved it in `users.sqlite`.

To import that database inside zeeSQL you only need to:

```
127.0.0.1:6379> ZEESQL.CREATE_DB USERS PATH users.sqlite
1) 1) "OK"
127.0.0.1:6379> ZEESQL.EXEC USERS COMMAND "select * from users"
1) 1) "RESULT"
2) 1) "id"
   2) "username"
   3) "score"
3) 1) "INT"
   2) "TEXT"
   3) "INT"
4) 1) (integer) 100
   2) "joh"
   3) (integer) 3
5) 1) (integer) 101
   2) "mary"
   3) (integer) 7
6) 1) (integer) 102
   2) "brand"
   3) (integer) 4
```

## About zeeSQL

zeeSQL is a Redis Module that provides SQL capabilities to Redis.
It allows the creation and management of several SQL databases, each one independent from the other.
Moreover, zeeSQL provides out-of-the-box [secondary indexes](../secondary-indexes.md) capabilities, allowing fast and easy search by value in Redis.

[createdb]: ../references.md#zeesql-create_db


