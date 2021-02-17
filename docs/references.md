# References

This document explains all the API that zeeSQL provide to the users.

This document refers to the latest API, but we commit to backward-compatible API.

For each command, it exposes first the name and then the syntax, and finally a brief explanation of what is going on inside the code.

Where is possible, it provides also an estimate of the complexity but since we are talking about databases not all queries have the same time and spatial complexity.

Finally, if it is appropriate the document also provides several references to external material that the interested reader can use to understand better the dynamics of every command.

## ZEESQL.CREATE\_DB

```text
ZEESQL.CREATE_DB db_key [PATH path]
```

This command creates a new DB and associates it with the key.

The path argument is optional and, if provided is the file that SQLite will use. It can be an existing SQLite file or it can be a not existing file.

If the file exists and if it is a regular SQLite file that database will be used. If the file does not exist a new file will be created.

If the path is not provided it will open an in-memory database. Not providing a path is equivalent to provide the special string `:memory:` as the path argument.

After opening the database it inserts metadata into it and then starts a thread loop.

**Complexity**: O\(1\), it means constant, it does not necessarily mean _fast_. However, is fast enough for any use case facing human users \(eg create a new database for every user logging into a website.\)

**Examples**:

```text
127.0.0.1:6379> ZEESQL.CREATE_DB DB 
1) 1) "OK"
```

This command created an in-memory database. Persistency is managed by Redis with AOF or RDB following the setting of your Redis instance.

```text
127.0.0.1:6379> ZEESQL.CREATE_DB on_disk_db PATH /tmp/foo.sqlite
1) 1) "OK"
```

This command created a database that uses a file as storage support, in this case `/tmp/foo.sqlite`.

If the file does not exist, it is created.

If the file is already an SQLite database, it gets used immediately with all the data already loaded.

If the file is not an SQLite database, an error is raised.

**See also**:

1. [SQLite `sqlite3_open_v2`](https://sqlite.org/c3ref/open.html)

## DEL

```text
DEL db_key [key ...]
```

This command is a generic command from Redis.

It eliminates keys from Redis itself, as well if the key is a RediSQL database create with [`ZEESQL.CREATE_DB`](references.md#redisqlcreate_db) it will eliminate the SQLite database, stop the thread loop and clean up everything left.

If the database is backed by a file the file will be closed, but it won't be deleted.

**Complexity**: DEL is O\(N\) on the number of keys, if you are only eliminating the key associated with the SQLite database will be constant, O\(1\).

**Examples**:

```text
127.0.0.1:6379> ZEESQL.CREATE_DB DB 
1) 1) "OK"
127.0.0.1:6379> DEL DB
(integer) 1
```

**See also**:

1. [SQLite `sqlite3_close`](https://sqlite.org/c3ref/close.html)
2. [Redis `DEL`](https://redis.io/commands/del)

## ZEESQL.EXEC

```text
ZEESQL.EXEC db_key 
    ( (COMMAND "command") | (STATEMENT statement) ) 
    [NOW] 
    [READ_ONLY] 
    [INTO stream] 
    [NO_HEADER] 
    [JSON] 
    [ARGS arg1 arg2 ... argn]
```

The EXEC command is the main command of zeeSQL. It allows interaction with the database stored in the `db_key`.

It takes as input the database `db_key`, either a `COMMAND` or a `STATEMENT`, an optional series of flags, and an optional variadic number of arguments.

You need to supply **EITHER** a command \(using the `COMMAND` flag\) or a statement \(using the `STATEMENT` flag\), but you need one of them.

### Command vs Statement

The command is a valid SQL string. The command SQL string can contain arguments in the form `?n`. Those arguments will be matched to the one provided at the end of the command. The first argument is bound against `?1`, NOT against `?0`. Arguments that are not provided will be bound to `NULL`.

The statement can be created with the `ZEESQL.STATEMENT` command. The arguments follow the same logic of the COMMAND variants.

**Examples**:

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "create table foo(a INT, b string);"
1) 1) "DONE"
2) 1) (integer) 0
```

In this example, we create a new table.

```text
127.0.0.1:6379> ZEESQL.STATEMENT DB NEW insert "insert into foo values(?1, ?2);"
1) 1) "OK"
127.0.0.1:6379> ZEESQL.EXEC DB STATEMENT insert ARGS 1 one
1) 1) "DONE"
2) 1) (integer) 1
127.0.0.1:6379> ZEESQL.EXEC DB STATEMENT insert ARGS 2 two
1) 1) "DONE"
2) 1) (integer) 1
```

Then we create a new STATEMENT to insert values, and we execute the statement twice.

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "select * from foo;"
1) 1) "RESULT"
2) 1) "a"
   2) "b"
3) 1) "INT"
   2) "TEXT"
4) 1) (integer) 1
   2) "one"
5) 1) (integer) 2
   2) "two"
```

We queried the values just inserted in the table executing another command.

### NOW flag

By default `zeeSQL` offload the SQL computation to a secondary thread. This free the main Redis thread and keep the Redis instance reactive.

The `NOW` flags force `zeeSQL` to run the SQL computation in the main Redis thread.

### READ\_ONLY flag

The `READ_ONLY` flags communicate to `zeeSQL` that the execution will not modify the database.

If the execution might modify the database, and the `READ_ONLY` flag is passed, `zeeSQL` will return an error.

Passing the `READ_ONLY` flags allow `zeeSQL` to not replicate the command. Possibly saving computing resources of the replicas.

**Examples**:

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "select * from foo where a > ?1;" READ_ONLY ARGS 1
1) 1) "RESULT"
2) 1) "a"
   2) "b"
3) 1) "INT"
   2) "TEXT"
4) 1) (integer) 2
   2) "two"
```

Here we correctly used the `READ_ONLY` flag to communicate with `zeeSQL` that the command will not modify the database.

`zeeSQL` correctly executes the query.

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "insert into foo values(3, 'three')" READ_ONLY
(error) Statement is not read only but it may modify the database, use `EXEC` instead.
```

In this other case, we ask `zeeSQL` to modify the database while using the `READ_ONLY` flag.

`zeeSQL` correctly refuses to modify the database and returns an error.

### NO\_HEADER flag

By default `zeeSQL` returns information about the result set. It returns the name of the columns and their type.

This information might not be useful nor desirable.

With the `NO_HEADER` flag, only the result itself is returned.

**Examples**:

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "select a as number_as_integer, b as number_as_string from foo;"
1) 1) "RESULT"
2) 1) "number_as_integer"
   2) "number_as_string"
3) 1) "INT"
   2) "TEXT"
4) 1) (integer) 1
   2) "one"
5) 1) (integer) 2
   2) "two"
```

This is the default result from `zeeSQL`. It reports the name of the columns \(in this case `number_as_integer` and `number_as_string` and their type `INT` and `TEXT`\).

If we are not intereted in such information:

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "select a as number_as_integer, b as number_as_string from foo;" NO_HEADER
1) 1) "RESULT"
2) 1) (integer) 1
   2) "one"
3) 1) (integer) 2
   2) "two"
```

the `NO_HEADER` flags will omit it for us.

### JSON flag

By default `zeeSQL` returns its result as an array of array. This makes parsing the result a little complex in some programming languages.

The JSON flags instruct `zeeSQL` to return a single JSON string as result.

**Examples**:

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "select a as number_as_integer, b as number_as_string  from foo;" JSON
"{\"rows\":[{\"number_as_integer\":1,\"number_as_string\":\"one\"},{\"number_as_integer\":2,\"number_as_string\":\"two\"}],\"number_of_rows\":2,\"columns\":{\"number_as_integer\":\"INT\",\"number_as_strin
g\":\"TEXT\"}}"
```

The JSON is valid JSON even if compressed:

```text
$ redis-cli ZEESQL.EXEC DB COMMAND "select a as number_as_integer, b as number_as_string  from foo;" JSON | jq
{
  "rows": [
    {
      "number_as_integer": 1,
      "number_as_string": "one"
    },
    {
      "number_as_integer": 2,
      "number_as_string": "two"
    }
  ],
  "number_of_rows": 2,
  "columns": {
    "number_as_integer": "INT",
    "number_as_string": "TEXT"
  }
}
```

Of course, this can be combined with the `NO_HEADER` flag.

```text
$ redis-cli ZEESQL.EXEC DB COMMAND "select a as number_as_integer, b as number_as_string  from foo;" JSON NO_HEADER | jq
{
  "rows": [
    {
      "number_as_integer": 1,
      "number_as_string": "one"
    },
    {
      "number_as_integer": 2,
      "number_as_string": "two"
    }
  ]
}
```

### INTO stream

`zeeSQL` can push the result of a computation in a Redis Stream.

This is desirable if you want to:

1. consume the result at a later time,
2. or cache the result, 
3. or if the result is rather big and you don't want to send all of it over the network.

The `INTO stream` option will inform `zeeSQL` to push the result of the computation into the Redis STREAM called `stream`.

The `INTO stream` option is available only if the query is marked as `READ_ONLY`.

The command executes [`XADD`](https://redis.io/commands/xadd) to the stream.

If the stream does not exist a new one is created.

If the stream already exists the rows are simply appended.

The command itself is eager, hence it computes the whole result, append it into the stream, and then it returns. Once the command returns, the whole result set is already in the Redis stream.

The return value of the command depends on the result of the query:

If the result of the query is empty, it simply returns `["DONE", 0]`.

If at least one row is returned by the query the command returns:

1.the name of the stream where it appended the resulting rows, which is always the one passed as input 2. the first ID added to the stream 3. the last ID added to the stream 4. and the total number of entries added to the stream.

Using a standard Redis Stream all the standard consideration applies.

1. The stream is not deleted by zeeSQL, hence it can be used for caching, on the other hand too many streams will use memory.
2. The stream uses a standard Redis key, in a cluster environment you should be sure that the database that is executing the query and the stream that will accommodate the results are on the same cluster node. 

This can be accomplished easily by forcing the stream name to hash to the same cluster node of the database, it is sufficient to use a `stream_name` composed as such `{db_key}:what:ever:here`. Redis will hash only the part between the `{` and `}` to compute the cluster node. 3. The result can be consumed using the standard [Redis streams commands](https://redis.io/commands#stream), two good starting points are [`XREAD`](https://redis.io/commands/xread) and [`XRANGE`](https://redis.io/commands/xrange).

The key of the stream elements are the tuple `(type, column name)` separated by a colon `:`.

**Examples**:

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "select * from foo;" INTO foo_stream
(error) Asked a STREAM, but the query is not `READ_ONLY` (flag not set), this is not supported.
```

At first, we tried to push the result of a query that is not marked as `READ_ONLY` and this correctly fails.

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "select * from foo where a > 100;" READ_ONLY INTO foo_stream
1) 1) "DONE"
2) 1) (integer) 0
```

Above we execute a query that returns an empty result.

`zeeSQL` simply communicates to us that the result is empty.

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "select * from foo;" READ_ONLY INTO foo_stream                                                                                       1) 1) "RESULT"
2) 1) "foo_stream"
   2) "1612797707753-0"
   3) "1612797707753-1"
   4) (integer) 2
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "select * from foo;" READ_ONLY INTO foo_stream JSON
"{\"rows\":[{\"stream\":\"foo_stream\",\"first_id\":\"1612797722505-0\",\"last_id\":\"1612797722505-1\",\"size\":2}]}"
```

Then we push into the Redis stream `foo_stream` the result of the query `select * from foo`.

We do it twice, once using the standard return and the second time using the JSON return.

We pushed twice, a query that returned two results, so we expect 4 elements in the stream.

```text
127.0.0.1:6379> XLEN foo_stream
(integer) 4
```

We can read the elements from the stream using the standard Redis stream interface. In this case, we are going to read only the first two elements.

```text
127.0.0.1:6379> XRANGE foo_stream - + COUNT 2
1) 1) "1612797707753-0"
   2) 1) "int:a"
      2) "1"
      3) "text:b"
      4) "one"
2) 1) "1612797707753-1"
   2) 1) "int:a"
      2) "2"
      3) "text:b"
      4) "two"
```

**Complexity**: Besides the complexity of the query, the `INTO stream` options add the complexity of adding each row to the Redis stream, which is `O(n)` where `n` is the amount of row returned by the query.

**See also**:

1. [Redis Streams Intro](https://redis.io/topics/streams-intro)
2. [Redis Streams Commands](https://redis.io/commands#stream)
3. [`XADD`](https://redis.io/commands/xadd)
4. [`XREAD`](https://redis.io/commands/xread)
5. [`XRANGE`](https://redis.io/commands/xrange)

### ARGS arguments

The ARGS arguments are used to pass arguments to the statement or command.

They are variadic, and you can pass as many as you need.

In the SQL, the first argument will be bound to `?1`, the second to `?2`, and so on. Please note that the first argument is NOT bound to `?0`.

If an argument is not bound, for instance, you pass a query with `?4` but only provide 3 arguments, then, that argument is bound to `NULL`.

Redis works using a text protocol, all the arguments are encoded as text, hence the module is forced to use the procedure `sqlite3_bind_text`, however, SQLite is smart enough to recognize numbers and treat them correctly. Numbers will be treated as numbers and text will be treated as text.

**See also**:

1. [SQLite `sqlite3_prepare_v2`](https://sqlite.org/c3ref/prepare.html)
2. [SQLite `statement` aka `sqlite3_stmt`](https://sqlite.org/c3ref/stmt.html)
3. [SQLite `sqlite3_step`](https://sqlite.org/c3ref/step.html)
4. [SQLite `PRAGMA`s](https://sqlite.org/pragma.html)
5. [Redis Blocking Command](https://redis.io/topics/modules-blocking-ops)

## ZEESQL.QUERY

```text
ZEESQL.QUERY db_key 
    ( (COMMAND "command") | (STATEMENT statement) ) 
    [NOW] 
    [INTO stream] 
    [NO_HEADER] 
    [JSON] 
    [ARGS arg1 arg2 ... argn]
```

This command behaves similarly to [`ZEESQL.EXEC`](references.md#redisqlexec) but it imposes an additional constraint on the statement it executes.

It only executes the statement if it is a read-only operation, otherwise, it returns an error.

A read-only operation is defined by the result of calling [`sqlite3_stmt_readonly`](https://www.sqlite.org/c3ref/stmt_readonly.html) on the compiled statement.

The statement is executed if and only if [`sqlite3_stmt_readonly`](https://www.sqlite.org/c3ref/stmt_readonly.html) returns true.

This command is exactly like `ZEESQL.EXEC ... READ_ONLY` however it can be executed against Redis replicas.

**Complexity**: Similar to [`ZEESQL.EXEC`](references.md#redisqlexec), however, if a statement is not read-only it is aborted immediately and it does return an appropriate error.

**See also**:

1. [SQLite `sqlite3_prepare_v2`](https://sqlite.org/c3ref/prepare.html)
2. [SQLite `statement` aka `sqlite3_stmt`](https://sqlite.org/c3ref/stmt.html)
3. [SQLite `sqlite3_step`](https://sqlite.org/c3ref/step.html)
4. [SQLite `PRAGMA`s](https://sqlite.org/pragma.html)
5. [Redis Blocking Command](https://redis.io/topics/modules-blocking-ops) 
6. [`ZEESQL.EXEC`](references.md#redisqlexec)
7. [SQLite `sqlite3_stmt_readonly`](https://www.sqlite.org/c3ref/stmt_readonly.html)
8. [`ZEESQL.QUERY_STATEMENT`](references.md#redisqlquery_statement) 

## ZEESQL.STATEMENT

```text
ZEESQL.STATEMENT db_key
    (
        (NEW stmt "query" [CAN_UPDATE]) | 
        (DELETE stmt) | 
        (UPDATE stmt "query" [CAN_CREATE]) |
        (SHOW stmt) |
        LIST
    )
    [NOW]
```

This command manages `zeeSQL` statements.

A statement is a pre-compiled SQL query, if you are going to execute your query over and over again, it is a good idea to make it into a statement. Under the hood it is a [sqlite statement](https://sqlite.org/c3ref/stmt.html).

Statements can be used in the `ZEESQL.EXEC db_key STATEMENT stmt` command and in the `ZEESQL.QUERY db_key STATEMENT stmt` command.

Five different actions can be invoked with the STATEMENT command.

1. Create a new statement with the `NEW` option.
2. Delete a statement with the `DELETE` option.
3. Update a statement with the `UPDATE` option.
4. Show the SQL behind a statement with the `SHOW` option.
5. List all the statements with the `LIST` option.

The `STATEMENT` command includes the `NOW` flag. The `NOW` flag forces `zeeSQL` to execute the action in the main Redis thread. In standard operations mode, it should not be used.

### NEW

The `NEW` option takes as input the name to associate with the statement and an SQL query to compile.

The command compiles the SQL query into a pre-compiled statement, and associate it with the name.

The `CAN_UPDATE` flag to the `NEW` command, instruct `zeeSQL` to behave as an `UPDATE` if the statement name is already allocated to an old statement. Otherwise, without the `CAN_UPDATE` flag, if the statement name is already used by a different statement, the command fails with an error.

### DELETE

The `DELETE` option deletes a statement.

### UPDATE

The `UPDATE` option updates a statement, associating the old name with the statement compiled from the SQL query.

If the name does not exists, `UPDATE` fails, unless the `CAN_CREATE` flag is provided. In such a case `UPDATE` behave like `NEW`.

### SHOW

The `SHOW` option returns the SQL query behind one statement.

### LIST

The `LIST` option returns all the statements and their SQL queries.

Both `SHOW` and `LIST` will report:

1. The name of the statement
2. The SQL query associate with the statement
3. The number of parameters the statement expects
4. If the statement is read only or not

**Complexity**: Operation on the statements happens in constant time O\(1\). Listing the statements happens in O\(n\) with `n` number of statements present in the database.

**Examples**:

At first we create a new statement,

```text
127.0.0.1:6379> ZEESQL.STATEMENT DB NEW select_1 "SELECT 1;"
1) 1) "OK"
```

We can then list, the statement:

```text
127.0.0.1:6379> ZEESQL.STATEMENT DB LIST
1) 1) "RESULT"
2) 1) "identifier"
   2) "SQL"
   3) "parameters_count"
   4) "read_only"
3) 1) "TEXT"
   2) "TEXT"
   3) "INT"
   4) "INT"
4) 1) "select_1"
   2) "SELECT 1;"
   3) (integer) 0
   4) (integer) 1
```

The statement can be updated, but we need to use the `UPDATE` command or the `CAN_UPDATE` flag.

```text
127.0.0.1:6379> ZEESQL.STATEMENT DB NEW select_1 "SELECT '1';"
(error) The statement is already present in the database, try with UPDATE_STATEMENT
127.0.0.1:6379> ZEESQL.STATEMENT DB NEW select_1 "SELECT '1';" CAN_UPDATE
1) 1) "OK"
127.0.0.1:6379> ZEESQL.STATEMENT DB UPDATE select_1 "SELECT '1';"
1) 1) "OK"
```

We change `select_1` to return a string and not an integer.

```text
127.0.0.1:6379> ZEESQL.STATEMENT DB UPDATE select_plus_one "SELECT ?1 + 1;" CAN_CREATE
1) 1) "OK"
```

We create another statement, this time we create the statement with the `UPDATE` command and the `CAN_CREATE` flag.

```text
127.0.0.1:6379> ZEESQL.STATEMENT DB LIST
1) 1) "RESULT"
2) 1) "identifier"
   2) "SQL"
   3) "parameters_count"
   4) "read_only"
3) 1) "TEXT"
   2) "TEXT"
   3) "INT"
   4) "INT"
4) 1) "select_1"
   2) "SELECT '1';"
   3) (integer) 0
   4) (integer) 1
5) 1) "select_plus_one"
   2) "SELECT ?1 + 1;"
   3) (integer) 1
   4) (integer) 1
127.0.0.1:6379> ZEESQL.EXEC DB STATEMENT select_plus_one NO_HEADER ARGS 5
1) 1) "RESULT"
2) 1) (integer) 6
```

**See also**:

1. [SQLite `sqlite3_prepare_v2`](https://sqlite.org/c3ref/prepare.html)
2. [SQLite `statement` aka `sqlite3_stmt`](https://sqlite.org/c3ref/stmt.html)
3. [SQLite bindings, `sqlite3_bind_text`](https://sqlite.org/c3ref/bind_blob.html)
4. [Redis Blocking Command](https://redis.io/topics/modules-blocking-ops)

## ZEESQL.COPY

```text
ZEESQL.COPY 
    FROM db_key_source 
    TO db_key_destination 
    [NOW]
```

The command copies the source database into the destination database.

The content of the destination databases is completely ignored and lost.

It is not important if the databases are stored in memory or backed by disk, the `COPY` command will work nevertheless.

This command is useful to:

1. Create backups of databases
2. Load data from slow, disk-based, databases into a fast in-memory one
3. To persist data from an in-memory database into a disk-based database
4. Initialize a database with a predefined status

Usually, the destination database is an empty database just created, while the source one is a database where we have been working for a while.

This command use the [backup API](https://www.sqlite.org/backup.html) of sqlite.

**Complexity**: The complexity is linear on the number of pages \(dimension\) of the source database, beware it can be "slow" if the source database is big, during the copy the `source` database is busy and it cannot serve other queries.

**Example**:

```text
127.0.0.1:6379> ZEESQL.CREATE_DB DB01
1) 1) "OK"
127.0.0.1:6379> ZEESQL.EXEC DB01 COMMAND "create table foo(a, b);"
1) 1) "DONE"
2) 1) (integer) 0
127.0.0.1:6379> ZEESQL.EXEC DB01 COMMAND "insert into foo values(1,2),(3,4);"
1) 1) "DONE"
2) 1) (integer) 2
127.0.0.1:6379> ZEESQL.CREATE_DB DB02
1) 1) "OK"
127.0.0.1:6379> ZEESQL.EXEC DB02 COMMAND "select * from foo"
(error) no such table: foo
127.0.0.1:6379> ZEESQL.COPY FROM DB01 TO DB02
1) 1) "OK"
127.0.0.1:6379> ZEESQL.EXEC DB02 COMMAND "select * from foo"
1) 1) "RESULT"
2) 1) "a"
   2) "b"
3) 1) "INT"
   2) "INT"
4) 1) (integer) 1
   2) (integer) 2
5) 1) (integer) 3
   2) (integer) 4
```

In the example we create a database, we create a table, and then we pushed few rows into the table.

We then create another database, but this time we didn't create the table, neither pushed any row.

As expected, trying to query the second database returned an error.

After copying the content of the first database into the second, the second database has become a perfect copy of the first one.

**See also**:

1. [Backup API](https://www.sqlite.org/backup.html)

## ZEESQL.INDEX

The index command accepts 3 different options:

1. NEW
2. LIST
3. DELETE

We will illustrate them separately.

### ZEESQL.INDEX NEW

```text
ZEESQL.INDEX db_key 
    NEW
    TABLE table_name
    [PREFIX prefix]
    SCHEMA column_name column_type [column_name column_type ...]
```

Creates a new secondary index table for the Redis hashes.

The secondary index will refer to the table `table_name` and will use the columns indicated in the `SCHEMA` parameter.

`column_type` can be whatever is accepted by SQLite as column name, suggestions are `TEXT`, `INT`, `FLOAT` or `BLOB`.

If a prefix is provided, only the Redis hashes that start with that prefix are indexed in the table. If the prefix is omitted, the `*` prefix \(catch-all\) is assumed.

If the table does not exists when the index is created, `zeeSQL` creates it.

If the table already exists, `zeeSQL` takes no action. However, if the table exists but contains the wrong columns, `zeeSQL` may find it impossible to insert the hashes into the secondary index.

It is possible to create multiple indexes, with the same table, same schema, but different prefix.

`zeeSQL` store in the secondary index table, the main Redis hash key, as primary key.

The table created by `zeeSQL` behaves like any other table, so it can be queried, modified, and indexed, using the standard `ZEESQL.EXEC` interface.

No steps are taken to avoid manual deletions or updates of the secondary index table by the user.

Secondary indexes are univocally identified by the combination of the table in which they write and the prefix.

**Example**:

In this example, we can see how to create a secondary index, and how the secondary index automatically keeps the values between Redis and zeeSQL in synchronism.

```text
127.0.0.1:6379> ZEESQL.INDEX DB NEW TABLE users prefix user:* SCHEMA username STRING score INT
OK
127.0.0.1:6379> HMSET user:1001 username aaa score 0
OK
127.0.0.1:6379> HMSET user:1002 username bbb score 0
OK
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "select * from users"
1) 1) "RESULT"
2) 1) "key"
   2) "username"
   3) "score"
3) 1) "TEXT"
   2) "TEXT"
   3) "INT"
4) 1) "user:1001"
   2) "aaa"
   3) (integer) 0
5) 1) "user:1002"
   2) "bbb"
   3) (integer) 0
127.0.0.1:6379> HINCRBY user:1002 score 3
(integer) 3
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "select * from users"
1) 1) "RESULT"
2) 1) "key"
   2) "username"
   3) "score"
3) 1) "TEXT"
   2) "TEXT"
   3) "INT"
4) 1) "user:1001"
   2) "aaa"
   3) (integer) 0
5) 1) "user:1002"
   2) "bbb"
   3) (integer) 3
```

### ZEESQL.INDEX LIST

The list option shows the active secondary index.

The secondary indexes are identified by the table in which they write and by the prefix they use to filter the keys.

**Example**:

```text
127.0.0.1:6379> ZEESQL.INDEX DB LIST
1) 1) "RESULT"
2) 1) "users"
   2) "user:*"
127.0.0.1:6379> ZEESQL.INDEX DB NEW TABLE games prefix games:* SCHEMA first_player STRING second_player STRING score_player_1 INT score_player_2 INT
OK
127.0.0.1:6379> ZEESQL.INDEX DB LIST
1) 1) "RESULT"
2) 1) "users"
   2) "user:*"
3) 1) "games"
   2) "games:*"
```

### ZEESQL.INDEX DELETE

```text
ZEESQL.INDEX db_key 
    DELETE
    TABLE table_name
    [PREFIX prefix]
```

The `DELETE` option removes a secondary index.

Hashes that match the prefix are not inserted anymore in the table after the index is removed.

The `DELETE` option takes as input the table and the prefix that identifies the secondary index to remove.

If the prefix is omitted the `*` \(catch-all\) prefix is assumed.

**Example**:

```text
127.0.0.1:6379> ZEESQL.INDEX DB NEW TABLE users PREFIX user:* SCHEMA username STRING score INT
OK
127.0.0.1:6379> HMSET user:1001 username first_user score 12
OK
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "select * from users" NO_HEADER
1) 1) "RESULT"
2) 1) "user:1001"
   2) "first_user"
   3) (integer) 12
127.0.0.1:6379> ZEESQL.INDEX DB DELETE TABLE users PREFIX user:*
1) 1) "DONE"
2) 1) (integer) 1
127.0.0.1:6379> HMSET user:1002 username second_user score 5
OK
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "select * from users" NO_HEADER
1) 1) "RESULT"
2) 1) "user:1001"
   2) "first_user"
   3) (integer) 12
```

In the example, we first create an index.

Then we add a Redis hash that matches the secondary index prefix, so it is added to the secondary index table.

Then, delete the index.

And we can confirm that new users are not added any more to the secondary index table.

## ZEESQL.LICENSE

```text
ZEESQL.LICENSE
    ( SET "license" ) | SHOW
```

The `SET` option will set the license to use for `zeeSQL`.

The license is first checked against our backend server, and if the license is correct, it will return `OK` otherwise it will return an error.

It is required an internet connection to set the license.

The `SHOW` option will show the license that is actually in use in your `zeeSQL` process.

