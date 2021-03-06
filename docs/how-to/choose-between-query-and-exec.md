# How to choose between QUERY and EXEC

zeeSQL comes with two commands that seem similar:

[`ZEESQL.QUERY`](../references.md#zeesql-query) and [`ZEESQL.EXEC`](../references.md#zeesql-exec).

These two commands have the same syntax, but different semantic.

[`ZEESQL.QUERY`](../references.md#zeesql-query) is for **read-only** operations like `SELECT`.

[`ZEESQL.EXEC`](../references.md#zeesql-exec) is for updating the status of the database using `DELETE` or `INSERT` or `UPDATE` statements. However, `ZEESQL.EXEC` supports also `SELECT`, mostly for keeping things simple for everybody.

Still, especially in a production environment, `ZEESQL.QUERY` should be preferred whenever you are executing a read-only operation.

Redis support AOF and primary/secondary replication. Every time a command that modifies the status of the database is invoked in Redis, the same command is sent to the AOF file \(if enable\) and to the replicas.

We know that `ZEESQL.QUERY` will never modify the status of the database, hence all the operations executed with it, are not replicated, nor to the AOF file, nor to the replicas.

This is not true for `ZEESQL.EXEC`. Since we don't know if the operation in `ZEESQL.EXEC` will modify or not the database, the command needs to be replicated. This implies that the replicas will repeat the command against their own internal status.

It is a waste of resources to send a `SELECT` with the `ZEESQL.EXEC` command because it will be replicated by your replicas and by the AOF file, while not changing the structure of the database.

`ZEESQL.QUERY`, being a read-only Redis command, can be executed also by the replicas.

If you have a very demanding application with a lot of load, you may have to use read-only replicas. To read-only replicas, you cannot send Redis command that might modify the internal status like `ZEESQL.EXEC`.

Against read-only replicas, you can only send read-only commands, like `ZEESQL.QUERY`.

While coding up your application it is a great idea to take the time and use `ZEESQL.QUERY` when possible instead of blindly using `ZEESQL.EXEC`.

`ZEESQL.QUERY` will raise an error when you try to use it with something that is not a read-only query, so you will catch possible misuses of `ZEESQL.QUERY` extremely fast.

```text
127.0.0.1:6379> ZEESQL.CREATE_DB DB
1) 1) "OK"
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "CREATE TABLE foo(a INT, b INT);"
1) 1) "DONE"
2) 1) (integer) 0
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "INSERT INTO foo VALUES(1,2),(2,3);"
1) 1) "DONE"
2) 1) (integer) 2
127.0.0.1:6379> ZEESQL.QUERY DB COMMAND "INSERT INTO foo VALUES(1,2),(2,3);"
(error) Statement is not read only but it may modify the database, use `EXEC` instead.
127.0.0.1:6379> ZEESQL.QUERY DB COMMAND "SELECT * FROM foo;"
1) 1) "RESULT"
2) 1) "a"
   2) "b"
3) 1) "INT"
   2) "INT"
4) 1) (integer) 1
   2) (integer) 2
5) 1) (integer) 2
   2) (integer) 3
```

## About zeeSQL

zeeSQL is a Redis Module that provides SQL capabilities to Redis. It allows the creation and management of several SQL databases, each one independent from the other. Moreover, zeeSQL provides out-of-the-box [secondary indexes](../secondary-indexes.md) capabilities, allowing fast and easy search by value in Redis.

