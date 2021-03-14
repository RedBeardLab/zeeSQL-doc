
# zeeSQL now runs on SQLite 3.35

SQLite is the SQL workhorse and engine that powers zeeSQL.

SQLite was chosen since it is fast, reliable, simple to operate, widely available, widely known in the tech community, and very well maintained.

The latest release of SQLite (3.35) is being exciting.
SQLite 3.35 introduces three really interesting features.
[The release notes are available on the SQLite website.](https://sqlite.org/releaselog/3_35_0.html)

zeeSQL version 1.0.1 includes SQLite 3.35 with all the improvements listed above.

zeeSQL version 1.0.1 is available as docker container [(redbeardlab/zeesql:1.0.1)](https://hub.docker.com/layers/redbeardlab/zeesql/1.0.1/images/sha256-6a1aafcb6d1285355af0c75737aa4920a4365cb03f31b1ea3f1160135079e807?context=explore) and as [direct download (https://zeesql.com/releases/v1.0.1/zeesql.so)](https://zeesql.com/releases/v1.0.1/zeesql.so)

## SQLite with RETURNING

RETURNING is a novel clause for `INSERT`, `UPDATE` and, `DELETE` statements.

It appears after the main statement and specifies a set of columns, or expressions, to return after the insertion, update or delete of rows.

The RETURNING clauses returns a row, for each row that the main statement insert, update or deletes.

In the case of zeeSQL, the RETURNING clauses can be invaluable.

Suppose you are letting zeeSQL generate random IDs for your columns, without the RETURNING clauses, you have no way to know what ID has been generated.
The only way would be to query back those rows.
With the `RETURNING` clauses it is possible to insert those rows and having zeeSQL returns the random ID generated.

```
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "create table random_id(id STRING DEFAULT (hex(randomblob(16))), name STRING, score INT);"
1) 1) "DONE"
2) 1) (integer) 0
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "insert into random_id(name, score) values('john', 3),('mary', 4) RETURNING id"
1) 1) "RESULT"
2) 1) "id"
3) 1) "TEXT"
4) 1) "A3400CE9270D6F7AFAE3FA1F09DD5798"
5) 1) "0FC0FA5557C3F873A80F6949EF01AEA0"
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "select * from random_id;"
1) 1) "RESULT"
2) 1) "id"
   2) "name"
   3) "score"
3) 1) "TEXT"
   2) "TEXT"
   3) "INT"
4) 1) "A3400CE9270D6F7AFAE3FA1F09DD5798"
   2) "john"
   3) (integer) 3
5) 1) "0FC0FA5557C3F873A80F6949EF01AEA0"
   2) "mary"
   3) (integer) 4
```

The RETURNING clause it is maybe more useful for UPDATEs and DELETEs.

Suppose you are deleting different some old values based on a timestamp, maybe a cache, you might be interested in knowing which element you removed.
Either you run two queries, the first to get all the rows you are going to delete and the second to delete them.
Or you use the RETURNING clauses, to delete the rows and get them at the same time.

```
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "create table cache(key STRING, value STRING, TTL INT);"
1) 1) "DONE"
2) 1) (integer) 0
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "insert into cache(key, value, TTL) VALUES('a', 'aaa', 3),('b', 'bbb', 4),('c', 'ccc', 2), ('d', 'ddd', 0);"
1) 1) "DONE"
2) 1) (integer) 4
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "DELETE FROM cache WHERE TTL < 3 RETURNING key, value;"
1) 1) "RESULT"
2) 1) "key"
   2) "value"
3) 1) "TEXT"
   2) "TEXT"
4) 1) "c"
   2) "ccc"
5) 1) "d"
   2) "ddd"
```

These are just a tiny sample of what is possible with the RETURNING clauses, your application can benefit from it a lot.

Also, the RETURNING clauses, break a small hidden assumption in zeeSQL.

Now also statements that are not `READ ONLY` can return rows.
This means that we have no constraints anymore of the statements and command that can use the [`INTO $stream`][into] clauses in zeeSQL.

More information about the RETURNING clauses is available on the [main SQLite page.](https://sqlite.org/lang_returning.html)

## SQLite with math functions

Another great addition to SQLite from the 3.35 release is about math functions.

The 3.35 release added a lot of math functions.
The whole list is available on the main [SQLite documentation.](https://sqlite.org/lang_mathfunc.html)

All these functions are now available in zeeSQL.

zeeSQL is compiled with the `SQLITE_ENABLE_MATH_FUNCTIONS` compile-time option enable.

## SQLite with DROP column

Another great feature of this release is the DROP column.

Before this release, to drop a column from a table, it was necessary to copy the whole table in a temporary table without the column to delete.
Then drop the original table.

With the 3.35 release, all this can be done with a simpler command which is an important quality of life improvement.


## Conclusion

The first minor release of zeeSQL updates the SQLite engine to one of the most feature-rich SQLite update ever.

The latest zeeSQL version (1.0.1) works with all the licenses already purchased. 
It is available as docker container [(redbeardlab/zeesql:1.0.1)](https://hub.docker.com/layers/redbeardlab/zeesql/1.0.1/images/sha256-6a1aafcb6d1285355af0c75737aa4920a4365cb03f31b1ea3f1160135079e807?context=explore) and as [direct download (https://zeesql.com/releases/v1.0.1/zeesql.so)](https://zeesql.com/releases/v1.0.1/zeesql.so)


[into]: ../references.md#into-stream
