# How to check if an index is used in zeeSQL and SQLite

This tutorial will cover both zeeSQL and SQLite. As you might know, zeeSQL is actually based on SQLite, so everything we say about zeeSQL can also be applied to raw SQLite.

## What are indexes

Indexes are secondary data-structures that sit next to the main data in your database.

They don't store the main data, but only a subset of them, usually a few columns, and the primary key.

The main point of indexes is that the data is always store sorted for the columns they are indexing.

For instance, if we create an index for the `score` of some users, in the index, the scores will be stored sorted, from the smallest to the largest \(or vice-versa\).

Storing the field sorted, allows a fast comparison for those fields. For instance, if we want to select all the users with a score greater than 20, we can look up in the index where the users with a score greater than 20 starts, and return all the users after that.

However, an index on the score, won't help if we are looking for all the users that played more than 15 games. For that query, you will need a different index.

As your use cases become complex, and the table larger, eventually a lot of indexes will be accumulated.

Then, for complex queries, it becomes very difficult to know what index, if any, is going to be used.

The SQL engine has its own algorithms and heuristic to figure out, what it thinks is the fastest way to execute a query.

Please note that while indexes help in retrieving information, they need to be maintained, which implies more load when the data are updated or added. There is a tradeoff between insertion speed, \(no index, fastest insertion\) and query speed \(the more indexes they are, the more query will run optimally\).

When no index can be used to speed up a query, SQLite will fallback in a full table scan, which means that it will look at every row in the table. For big tables, this implies a higher latency and slower results.

## Am I using some index?

SQLite, and so zeeSQL, comes with a handy command to check what the SQL engine is going to do, in other to retrieve the data.

`EXPLAIN QUERY PLAN $your_query` returns a human-readable representation of what the SQL engine is going to do to fetch the data.

An example will help clarify, suppose we have two tables, `foo` and `bar`.

```text
127.0.0.1:6379> ZEESQL.CREATE_DB DB
1) 1) "OK"
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "create table foo(a int, b int, c int);"
1) 1) "DONE"
2) 1) (integer) 0
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "create table bar(x int, y int, z int);"
1) 1) "DONE"
2) 1) (integer) 0
```

At this point, we don't have any indexes, so any query will go through a full table scan.

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "EXPLAIN QUERY PLAN select a from foo where a = 3 and b = 4 and c = 5;" NO_HEADER
1) 1) "RESULT"
2) 1) (integer) 2
   2) (integer) 0
   3) (integer) 0
   4) "SCAN TABLE foo"
```

In this example, were are scanning the whole `foo` table.

Adding a simple index to the `foo` table will help SQLite in searching more efficiently the table.

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "CREATE INDEX foo_a on foo(a);" NO_HEADER
1) 1) "DONE"
2) 1) (integer) 0
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "EXPLAIN QUERY PLAN select a from foo where a = 3 and b = 4 and c = 5;" NO_HEADER
1) 1) "RESULT"
2) 1) (integer) 3
   2) (integer) 0
   3) (integer) 0
   4) "SEARCH TABLE foo USING INDEX foo_a (a=?)"
```

This works also when you are working with multiple tables.

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "EXPLAIN QUERY PLAN select a from foo, bar where a = 3 and b = 4 and c = 5 and c = z;" NO_HEADER
1) 1) "RESULT"
2) 1) (integer) 4
   2) (integer) 0
   3) (integer) 0
   4) "SCAN TABLE bar"
3) 1) (integer) 10
   2) (integer) 0
   3) (integer) 0
   4) "SEARCH TABLE foo USING INDEX foo_a (a=?)"
```

On `foo`, we have the index we created previously that simplifies the search, in `bar` we don't have any indexes, so we are forced to do a full table scan.

As soon as we add an index:

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "CREATE INDEX bar_z on bar(z);" NO_HEADER
1) 1) "DONE"
2) 1) (integer) 0
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "EXPLAIN QUERY PLAN select a from foo, bar where a = 3 and b = 4 and c = 5 and c = z;" NO_HEADER
1) 1) "RESULT"
2) 1) (integer) 4
   2) (integer) 0
   3) (integer) 0
   4) "SEARCH TABLE foo USING INDEX foo_a (a=?)"
3) 1) (integer) 13
   2) (integer) 0
   3) (integer) 0
   4) "SEARCH TABLE bar USING COVERING INDEX bar_z (z=?)"
```

Now searches on both tables are using an index.

One detail, on `bar` we are using a `COVERING INDEX` instead of a standard index.

A standard index makes it faster to look up the id of the rows that we are interested in, but then we still need to fetch those rows from the database. A covering index means that you are not going to fetch extra rows because all the data you care about are already stored in the index itself. Note that in the query above we don't ask for any columns from the `bar` table in the result set, but we only compare that one column of `foo` is equal to one of `bar`.

## Composite indexes

Indexes can be against a single column or against multiple columns.

The more the index is specific, the faster will be your query.

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "CREATE INDEX foo_b_c on foo(b, c);" NO_HEADER
1) 1) "DONE"
2) 1) (integer) 0
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "EXPLAIN QUERY PLAN select a from foo where a = 3 and b = 4 and c = 5;" NO_HEADER
1) 1) "RESULT"
2) 1) (integer) 3
   2) (integer) 0
   3) (integer) 0
   4) "SEARCH TABLE foo USING INDEX foo_b_c (b=? AND c=?)"
```

In this case, instead of using the index `foo_a` the SQLite engine prefers the one on both `b` and `c` columns.

It is important to be careful between `AND` and `OR` conditions.

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "EXPLAIN QUERY PLAN select a from foo where a = 3 OR b = 4 and c = 5;" NO_HEADER                                                     1) 1) "RESULT"
2) 1) (integer) 4
   2) (integer) 0
   3) (integer) 0
   4) "MULTI-INDEX OR"
3) 1) (integer) 10
   2) (integer) 4
   3) (integer) 0
   4) "SEARCH TABLE foo USING INDEX foo_a (a=?)"
4) 1) (integer) 21
   2) (integer) 4
   3) (integer) 0
   4) "SEARCH TABLE foo USING INDEX foo_b_c (b=? AND c=?)"
```

The query seems the same, but instead of an `AND` we have an `OR`, in this case, one more search is necessary.

If they were all `OR`, then a full table scan is almost guarantee, unless you don't have a separated index for each column in the table.

```text
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "EXPLAIN QUERY PLAN select a from foo where a = 3 OR b = 4 OR c = 5;" NO_HEADER
1) 1) "RESULT"
2) 1) (integer) 2
   2) (integer) 0
   3) (integer) 0
   4) "SCAN TABLE foo"
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "CREATE INDEX foo_b on foo(b);" NO_HEADER
1) 1) "DONE"
2) 1) (integer) 0
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "CREATE INDEX foo_c on foo(c);" NO_HEADER
1) 1) "DONE"
2) 1) (integer) 0
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "EXPLAIN QUERY PLAN select a from foo where a = 3 OR b = 4 OR c = 5;" NO_HEADER
1) 1) "RESULT"
2) 1) (integer) 4
   2) (integer) 0
   3) (integer) 0
   4) "MULTI-INDEX OR"
3) 1) (integer) 10
   2) (integer) 4
   3) (integer) 0
   4) "SEARCH TABLE foo USING COVERING INDEX foo_a_b_c (a=?)"
4) 1) (integer) 20
   2) (integer) 4
   3) (integer) 0
   4) "SEARCH TABLE foo USING INDEX foo_b (b=?)"
5) 1) (integer) 30
   2) (integer) 4
   3) (integer) 0
   4) "SEARCH TABLE foo USING INDEX foo_c (c=?)"
```

## End

Indexes are fundamental for every database system, however, they might be confusing and not intuitive.

zeeSQL, and SQLite, provide a very simple way to check what indexes are going to be used for each of your queries, and when in doubt, it is a good idea to check it.

Do not add too many indexes, they slow down insertion. Moreover, in small datasets, a full table scan can be fast enough.

Always verify your assumption about indexes.

## About zeeSQL

zeeSQL is a Redis Module that provides SQL capabilities to Redis. It allows the creation and management of several SQL databases, each one independent from the other. Moreover, zeeSQL provides out-of-the-box [secondary indexes](../secondary-indexes.md) capabilities, allowing fast and easy search by value in Redis.

