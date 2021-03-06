# Why you should migrate from RediSQL to zeeSQL

zeeSQL is the successor of RediSQL. zeeSQL is RediSQL V2.

It is based on largely the same codebase but improved in several ways with novel features.

## Secondary indexes, or search by value

Secondaries indexes allow searching Redis keys by value. If you are interested in searching Redis keys by value or manipulate Redis hashes without maintains secondary data structures, secondary indexes are a perfect way to do it.

Secondary indexes have been introduced in RediSQL V2, also know as zeeSQL.

Secondary indexes themselves are a more than valid motivation to switch to zeeSQL instead of keeping using RediSQL.

## Better API

The APIs of zeeSQL are just better.

They are more flexible, they allow extensions, and they provide more information to the user.

The zeeSQL APIs were improved keeping what worked well with RediSQL and fixing what could have been improved.

### Return type and column name

The new APIs of zeeSQL, by default, return both the column name and the column type.

This allows writing wrappers or client library that automatically creates the correct type reading from Redis.

In the example below, you can see what RediSQL returned, versus what zeeSQL returns.

```text
127.0.0.1:6379> REDISQL.V1.CREATE_DB DB
OK
127.0.0.1:6379> REDISQL.V1.EXEC DB "create table foo(a, b, c);"
1) DONE
2) (integer) 0
127.0.0.1:6379> REDISQL.V1.EXEC DB "insert into foo values(1 , 'aaa', 2)"
1) DONE
2) (integer) 1
127.0.0.1:6379> REDISQL.V1.EXEC DB "select * from foo;"
1) 1) (integer) 1
   2) "aaa"
   3) (integer) 2
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "select * from foo;"
1) 1) "RESULT"
2) 1) "a"
   2) "b"
   3) "c"
3) 1) "INT"
   2) "TEXT"
   3) "INT"
4) 1) (integer) 1
   2) "aaa"
   3) (integer) 2
```

The result from zeeSQL is more structured. It contains the columns' name, then their type, and finally the rows.

The result from RediSQL returns directly the rows without providing information about the column name and their type.

### Structured returns

Values from zeeSQL and RediSQL can either be:

1. OK
2. DONE
3. Some result set

RediSQL returns either:

1. A simple string with the value "OK"
2. An array with first string the "DONE" string
3. A nested array, that has as first elements of the first subarray, the string "RESULT"

This is clearly suboptimal and it was fixed in zeeSQL.

zeeSQL returns always a nested array.

The first element, of the first subarray, identify the type of the result. It can be either the string "OK", or the string "DONE" or the string "RESULT".

But zeeSQL returns **always** a nested array.

Let's compare the same workflow with RediSQL and with zeeSQL.

```text
127.0.0.1:6379> REDISQL.V1.CREATE_DB DB
OK
127.0.0.1:6379> REDISQL.V1.EXEC DB "create table foo(a, b, c);"
1) DONE
2) (integer) 0
127.0.0.1:6379> REDISQL.V1.EXEC DB "insert into foo values(1 , 'aaa', 2)"
1) DONE
2) (integer) 1
127.0.0.1:6379> REDISQL.V1.EXEC DB "select * from foo;"
1) 1) (integer) 1
   2) "aaa"
   3) (integer) 2
```

In this example, RediSQL returned three different types of results.

```text
127.0.0.1:6379> ZEESQL.CREATE_DB DB1
1) 1) "OK"
127.0.0.1:6379> ZEESQL.EXEC DB1 COMMAND "create table foo(a);"
1) 1) "DONE"
2) 1) (integer) 0
127.0.0.1:6379> ZEESQL.EXEC DB1 COMMAND "insert into foo values(1),(2);"
1) 1) "DONE"
2) 1) (integer) 2
127.0.0.1:6379> ZEESQL.EXEC DB1 COMMAND "select * from foo;"
1) 1) "RESULT"
2) 1) "a"
3) 1) "INT"
4) 1) (integer) 1
5) 1) (integer) 2
```

In this other example, zeeSQL returned always a single type of result, a nested array.

### The JSON flag

The [JSON flag](references.md#json-flag) is a new feature of zeeSQL, not available in RediSQL.

Instead of returning a nested subarray, that in some programming language are difficult to manage, we can return a single JSON string that represents the same result set.

This makes working with zeeSQL much simpler in both dynamic and statically typed languages.

The string returned in a valid JSON string.

```text
127.0.0.1:6379> ZEESQL.CREATE_DB DB1
1) 1) "OK"
127.0.0.1:6379> ZEESQL.EXEC DB1 COMMAND "create table foo(a);" JSON
"{\"result\":\"done\",\"modified_rows\":0}"
127.0.0.1:6379> ZEESQL.EXEC DB1 COMMAND "insert into foo values(1),(2);" JSON
"{\"result\":\"done\",\"modified_rows\":2}"
127.0.0.1:6379> ZEESQL.EXEC DB1 COMMAND "select * from foo;" JSON
"{\"rows\":[{\"a\":1},{\"a\":2}],\"number_of_rows\":2,\"columns\":{\"a\":\"INT\"}}"
```

### Arguments to the EXEC and QUERY command

In RediSQL there is no way to pass arguments to simple queries.

You can either submit a full query, or create a statement, and submit the statement with arguments.

[In zeeSQL is possible to submit queries with arguments.](references.md#args-arguments) Especially if the arguments are coming from the users you should always submit them as arguments.

In RediSQL you had no choice, but to create a statement, and bind the arguments. In zeeSQL you can bind them to a simple query, without the need of creating a statement.

```text
127.0.0.1:6379> ZEESQL.CREATE_DB DB
1) 1) "OK"
127.0.0.1:6379> ZEESQL.EXEC DB1 COMMAND "create table foo(a);"
1) 1) "DONE"
2) 1) (integer) 0
127.0.0.1:6379> ZEESQL.EXEC DB1 COMMAND "insert into foo values(?1 + 3),(?2 + ?1);" ARGS 5 2
1) 1) "DONE"
2) 1) (integer) 2
127.0.0.1:6379> ZEESQL.EXEC DB1 COMMAND "select * from foo;"
1) 1) "RESULT"
2) 1) "a"
3) 1) "INT"
4) 1) (integer) 8
5) 1) (integer) 7
```

## Maintainance

zeeSQL is maintained, RediSQL is not maintained anymore.

Code updates, security fixes, and SQLite engine upgrades will happen only in zeeSQL.

No changes at all are expected to RediSQL.

## Backward compatibility

Besides all these upgrades, zeeSQL maintains backward compatibility with RediSQL.

All the RediSQL commands are prefixed by the `REDISQL.` string.

In zeeSQL you can find exactly the same commands, with exactly the same semantic, but a different prefix: `REDISQL.V1.`.

So if in RediSQL you used the `REDISQL.EXEC` command, with zeeSQL you can use the exact same command with `REDISQL.V1.EXEC`.

