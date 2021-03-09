# How to copy a database

zeeSQL can manage multiple databases at the same time.

Sometimes, it can be convenient to copy the content of one database into another one.

Copying databases can help if you want to transfer a database backed by a disk into memory, or the other way, transferring an in-memory database to disk.
Having databases on disk is make it simple to backup them, and reading databases from disk makes it simple to import databases from different sources.
On the other hand, having databases in memory makes them extremely fast.

For some use cases, you may want to have a set of databases that all share the same structure.
For instance, if each of your users is associated with a database, instead of creating all the tables needed every time new users signup, you could just clone a template database.

Copying databases can also help in spreading read load against an immutable database.
Instead of reading from a single database, you can copy the database into few other databases and spread the load.

To copy a database, you can use the [`ZEESQL.COPY`](../references.md) database.

The command takes two databases, after the `FROM` and the `TO` flags.
The database after the `FROM` flag is the source database.
The one after the `TO` flag is the destination database.

The destination database is completely wiped out, and replaced by the content of the source database.

The source database is left intact.


```
127.0.0.1:6379> ZEESQL.CREATE_DB DB01
1) 1) "OK"
127.0.0.1:6379> ZEESQL.EXEC DB01 COMMAND "create table foo(a, b);"
2) 1) (integer) 0
127.0.0.1:6379> ZEESQL.EXEC DB01 COMMAND "create table bar(a, b);"
1) 1) "DONE"
2) 1) (integer) 0
127.0.0.1:6379> ZEESQL.EXEC DB01 COMMAND "insert into foo values(1,2),(2,3); insert into bar values(1,2),(2,3);"
1) 1) "DONE"
2) 1) (integer) 4
1) 1) "OK"
127.0.0.1:6379> ZEESQL.EXEC DB02 COMMAND "select * from foo, bar where foo.a = bar.a"
(error) no such table: foo
127.0.0.1:6379> ZEESQL.COPY FROM DB01 TO DB02
1) 1) "OK"
127.0.0.1:6379> ZEESQL.EXEC DB02 COMMAND "select * from foo, bar where foo.a = bar.a"
1) 1) "RESULT"
2) 1) "a"
   2) "b"
   3) "a"
   4) "b"
3) 1) "INT"
   2) "INT"
   3) "INT"
   4) "INT"
4) 1) (integer) 1
   2) (integer) 2
   3) (integer) 1
   4) (integer) 2
5) 1) (integer) 2
   2) (integer) 3
   3) (integer) 2
   4) (integer) 3
```

In the example, we create and populate two tables in the `DB01`.

Then we create the `DB02` and we try to immediately read from it, `zeeSQL` of course returns an error since the `DB02` database is empty.

After copying the content of the `DB01` database into the `DB02` database, the same query success.

## About zeeSQL

zeeSQL is a Redis Module that provides SQL capabilities to Redis. It allows the creation and management of several SQL databases, each one independent from the other. Moreover, zeeSQL provides out-of-the-box [secondary indexes](../secondary-indexes.md) capabilities, allowing fast and easy search by value in Redis.


