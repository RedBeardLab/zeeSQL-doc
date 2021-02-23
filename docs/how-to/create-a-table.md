# How to create a new table in zeeSQL

## About zeeSQL

zeeSQL is a Redis Module that provides SQL capabilities to Redis.
It allows the creation and management of several SQL databases, each one independent from the other.
Moreover, zeeSQL provides out-of-the-box [secondary indexes](secondary-indexes.md) capabilities, allowing fast and easy search by value in Redis.

## How to create a new table in zeeSQL

After you have created a new database, the next likely action, will be to create a set of tables to store information.

zeeSQL uses SQLite under the hood, hence it can work with all the SQL queries that can be managed by SQLite.

Suppose you have created a new database with:

```
127.0.0.1:6379> ZEESQL.CREATE_DB DB
1) 1) "OK"
```

You can now create a new table executing a command against that database:

```
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "CREATE TABLE foo(col1 STRING, col2 INT, col3 STRING);"
1) 1) "DONE"
2) 1) (integer) 0
```

This command will create a table called `foo` with 3 columns.
The first column is called `col1` and has type `STRING`, the second column is called `col2` and has type `INT` and the last column is called `col3` again of type `STRING`.

We know that everything went well because the result was "DONE".
Moreover, zeeSQL is telling us that 0 rows were modified, this is correct since we only created a new table.

The [`CREATE TABLE`][create_table] documentation page of SQLite describes all the options and the syntax available to create a new table.
All of them are available in `zeeSQL`.

[create_table]: https://sqlite.org/lang_createtable.html
