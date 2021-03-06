# zeeSQL and secondary indexes, how to search Redis key by value

Redis, at his heart, is a key-value store. It makes it simple, easy, and fast to search for values given their keys.

Often time, however, is necessary to look for keys that values respect some properties.

For instance, find all the keys for which their value is greater than 5. Or the keys that have a value set to a specific string like "admin".

The standard solution for this problem in Redis is to keep a set of secondary indexes. To keep everything in sync, those secondary indexes need to be updated along with the primary keys. Keeping the indexes in sync is an activity that the developer who uses Redis need to take care of, and it brings a sizable increment in complexity.

zeeSQL aims to solve this problem.

## Redis Hashes

Redis provides different data structures, one of them is Redis Hashes.

Redis Hashes map between string fields and string values. From the Redis documentation:

```text
HMSET user:1000 username antirez password P1pp0 age 34
HGETALL user:1000
HSET user:1000 password 12345
HGETALL user:1000
```

In this case to the Redis key `user:1000` we associate a Redis Hash.

The Redis Hash has 3 fields, `username` with value `antirez`, `password` with value `P1pp0`, and `age` with value 34.

Redis Hashes are the only data structure that can be indexed using zeeSQL.

At the moment, zeeSQL we can index only Redis Hashes.

## Creating the Index

Secondary indexes in zeeSQL are standard SQL tables.

The user needs to provide the schema of the table, and zeeSQL will automatically keep the table in sync with the Redis Hashes.

The user can also provide a prefix so that only some Redis Hashes, the ones that match the prefix, are stored in the secondary index.

A secondary table can be created using the [`ZEESQL.INDEX` command](references.md#zeesql-index-new).

```text
> ZEESQL.INDEX DB NEW TABLE $table_name [PREFIX prefix] SCHEMA column_name column_type [column_name column_type]
```

This command will perform two actions.

At first, it will try to create a table called `$table_name`. If the table already exists, this step is skipped. If the table does not exists, the new table is created with the columns indicated after the `SCHEMA` keyword.

After the table is created, we register a callback.

The callback, listen to all the events that happen to the keys that start with the prefix, if the prefix is omitted, the callback listen to events for all the keys.

The callback is invoked passing as arguments the type of event and against which key the event was fired. From there, the callback has all the information it needs to keep the secondary index table in sync.

Every time that one key, matching the prefix is modified, zeeSQL updates the secondary index table.

## Secondary index table structure

The structure of the secondary index table is very simple.

There is a primary key, which is always a string, which value is the Redis key itself. In the Redis Hash above, the primary key will be the value `user:1000`

Next to the primary key, there are all the columns defined in the schema, with their respective types.

## It is JUST A TABLE

The secondary index table is a standard SQLite table.

There is nothing special about it, besides being managed by zeeSQL itself and not by the user.

You can, of course, query it, in whichever way you find more appropriate.

However, you could also modify it, even though it is strongly discouraged.

Being a standard SQLite table, it is possible to define indexes also on your secondary index table. This will allow even faster lookups.

Moreover, it is also possible to define triggers.

## Fire and forget

The commands to modify the secondary index tables, are fire and forget.

The works seamlessly on a standard table without constraints. However, if you start to add constraints and triggers to the secondary index table, it will be your responsibility to keep the database in a consistent state.

Unfortunately, zeeSQL cannot provide any feedback, if an update or insertion failed.

The use cases when this could be a problem, are extremely advanced.

## An Example

In our Redis we can store users for a simple online game. Of those users we store a simple ID, the name, and the score.

We want to search for all the user who scores is greater than 5.

We start by creating a zeeSQL database.

```text
> ZEESQL.CREATE_DB DB
1) 1) "OK"
```

Now we can start populating our users.

```text
127.0.0.1:6379> HMSET user:100 id 100 name foo score 3
OK
127.0.0.1:6379> HMSET user:103 id 103 name bar score 5
OK
127.0.0.1:6379> HMSET user:105 id 105 name baz score 4
```

We have created 3 users, each with its own name, id, and score.

At this point, we can create a secondary index.

```text
127.0.0.1:6379> ZEESQL.INDEX DB NEW PREFIX user:* TABLE users SCHEMA id INT name STRING score INT
OK
```

Now we have created the secondary index table.

We can visualize what table was created by querying the `sqlite_master` special table.

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "select * from sqlite_master;"
1) 1) "RESULT"
2) 1) "type"
   2) "name"
   3) "tbl_name"
   4) "rootpage"
   5) "sql"
3) 1) "TEXT"
   2) "TEXT"
   3) "TEXT"
   4) "INT"
   5) "TEXT"
5) 1) "table"
   2) "users"
   3) "users"
   4) (integer) 3
   5) "CREATE TABLE users(key PRIMARY KEY, id INT, name STRING, score INT)"
```

Exactly what we would expect, the `key` column as primary key, where we will store the key of the Redis Hash, and then the schema we asked for.

Since the secondary index was created after some Redis Hashes were already inside Redis, the table is already populated.

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "select * from users;"
1) 1) "RESULT"
2) 1) "key"
   2) "id"
   3) "name"
   4) "score"
3) 1) "TEXT"
   2) "INT"
   3) "TEXT"
   4) "INT"
4) 1) "user:105"
   2) (integer) 105
   3) "baz"
   4) (integer) 4
5) 1) "user:103"
   2) (integer) 103
   3) "bar"
   4) (integer) 5
6) 1) "user:100"
   2) (integer) 100
   3) "foo"
   4) (integer) 3
```

If now we add a new user, the new user will be automatically added to the secondary index table.

```text
127.0.0.1:6379> HMSET user:109 id 109 name joe score 2
OK
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "select * from users;"
1) 1) "RESULT"
2) 1) "key"
   2) "id"
   3) "name"
   4) "score"
3) 1) "TEXT"
   2) "INT"
   3) "TEXT"
   4) "INT"
4) 1) "user:105"
   2) (integer) 105
   3) "baz"
   4) (integer) 4
5) 1) "user:103"
   2) (integer) 103
   3) "bar"
   4) (integer) 5
6) 1) "user:100"
   2) (integer) 100
   3) "foo"
   4) (integer) 3
7) 1) "user:109"
   2) (integer) 109
   3) "joe"
   4) (integer) 2
```

Similarly, if a user is updated, the table will reflect the new status of the Redis Hash.

```text
127.0.0.1:6379> HSET user:109 score 5
(integer) 0
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "select * from users where id = 109;"
1) 1) "RESULT"
2) 1) "key"
   2) "id"
   3) "name"
   4) "score"
3) 1) "TEXT"
   2) "INT"
   3) "TEXT"
   4) "INT"
4) 1) "user:109"
   2) (integer) 109
   3) "joe"
   4) (integer) 5
```

Similarly, a Redis Hash deleted, will be removed from the secondary index table.

```text
127.0.0.1:6379> DEL user:105 user:103
(integer) 2
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "select * from users;"
1) 1) "RESULT"
2) 1) "key"
   2) "id"
   3) "name"
   4) "score"
3) 1) "TEXT"
   2) "INT"
   3) "TEXT"
   4) "INT"
4) 1) "user:100"
   2) (integer) 100
   3) "foo"
   4) (integer) 3
5) 1) "user:109"
   2) (integer) 109
   3) "joe"
   4) (integer) 5
```

## More advanced example

The first example was very straightforward. But we can use zeeSQL for something more.

For instance, maybe we want to give a rank to our users.

Users with a score between 0 and 5 will be "novice" and users with a score above it will be expert.

An easy way to achieve this would be to create a view on top of the `users` tables.

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "create view ranked_users AS select id, name, score, case  when score < 5 then 'novice' else 'expert' end as rank from users;"
1) 1) "DONE"
2) 1) (integer) 0
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "select * from ranked_users;"
1) 1) "RESULT"
2) 1) "id"
   2) "name"
   3) "score"
   4) "rank"
3) 1) "INT"
   2) "TEXT"
   3) "INT"
   4) "TEXT"
4) 1) (integer) 100
   2) "foo"
   3) (integer) 3
   4) "novice"
5) 1) (integer) 109
   2) "joe"
   3) (integer) 5
   4) "expert"
```

If the user with ID 100, gains a few more points, he will become an expert as well.

Using views on top of secondary indexes, we only need to care about the user score, not about the rank. Using plain Redis we would need to keep track also of the rank ourselves.

```text
127.0.0.1:6379> HSET user:100 score 7
(integer) 0
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "select * from ranked_users;"
1) 1) "RESULT"
2) 1) "id"
   2) "name"
   3) "score"
   4) "rank"
3) 1) "INT"
   2) "TEXT"
   3) "INT"
   4) "TEXT"
4) 1) (integer) 100
   2) "foo"
   3) (integer) 7
   4) "expert"
5) 1) (integer) 109
   2) "joe"
   3) (integer) 5
   4) "expert"
```

If our game gains a lot of users some queries could become slow.

On top of the secondary index table, it is possible to add SQLite indexes.

For instance, we might want to know how many users have a particular score. If there are a lot of users, this query might be slow.

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "explain query plan select * from users where score = 3;"
1) 1) "RESULT"
2) 1) "id"
   2) "parent"
   3) "notused"
   4) "detail"
3) 1) "INT"
   2) "INT"
   3) "INT"
   4) "TEXT"
4) 1) (integer) 2
   2) (integer) 0
   3) (integer) 0
   4) "SCAN TABLE users"
```

This query uses a full table scan.

We can do better defining an index:

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "create index user_rank on users(score);"
1) 1) "DONE"
2) 1) (integer) 0
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "explain query plan select * from users where score = 3;"
1) 1) "RESULT"
2) 1) "id"
   2) "parent"
   3) "notused"
   4) "detail"
3) 1) "INT"
   2) "INT"
   3) "INT"
   4) "TEXT"
4) 1) (integer) 3
   2) (integer) 0
   3) (integer) 0
   4) "SEARCH TABLE users USING INDEX user_rank (score=?)"
```

## Conclusion

In this article, we show how to use secondary indexes in zeeSQL.

They are very powerful and useful when you are simplifying your queries against Redis Hashes. Moreover, they allow you to think only about the main data, it is the query engine that finds the best way to query your data for you.

The important takeaway from this article should be that the table creates as zeeSQL secondary indexes are just standard tables. As such, those tables can be manipulated in whichever way the application developer finds more opportune.

