# How to get JSON output

By default zeeSQL returns nested arrays.

zeeSQL can also returns the exact same information as JSON output.

JSON output may be preferable since it is usually easier to parse and all languages offer full support for it.

The [`ZEESQL.EXEC`][exec] and [`ZEESQL.QUERY`][query] commands support the [`JSON`][json] flag, which instructs them to return JSON as output.

## Examples

The first example is about a command that does not return any rows but only `DONE`.

```
127.0.0.1:6379> ZEESQL.CREATE_DB DB
1) 1) "OK"
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "CREATE TABLE foo(col1 STRING, col2 INT, col3 STRING);"
1) 1) "DONE"
2) 1) (integer) 0
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "CREATE TABLE bar(col1 STRING, col2 INT, col3 STRING);" JSON
"{\"result\":\"done\",\"modified_rows\":0}"
```

Adding the `JSON` flags returns the exact same result but in JSON format.

```
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "INSERT INTO foo VALUES('AAA', 2, 'BBB'),('CCC', 3, 'DDD');"
1) 1) "DONE"
2) 1) (integer) 2
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "INSERT INTO bar VALUES('AAA', 2, 'BBB'),('CCC', 3, 'DDD');" JSON
"{\"result\":\"done\",\"modified_rows\":2}"
```

Another example is when the command returns some rows.

```
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "SELECT * FROM foo;"
1) 1) "RESULT"
2) 1) "col1"
   2) "col2"
   3) "col3"
3) 1) "TEXT"
   2) "INT"
   3) "TEXT"
4) 1) "AAA"
   2) (integer) 2
   3) "BBB"
5) 1) "CCC"
   2) (integer) 3
   3) "DDD"
127.0.0.1:6379> ZEESQL.EXEC DB COMMAND "SELECT * FROM bar;" JSON
"{\"rows\":[{\"col1\":\"AAA\",\"col2\":2,\"col3\":\"BBB\"},{\"col1\":\"CCC\",\"col2\":3,\"col3\":\"DDD\"}],\"number_of_rows\":2,\"columns\":{\"col1\":\"TEXT\",\"col2\":\"INT\",\"col3\":\"TEXT\"}}"
```

The returned JSON is formatted for saving bytes on the network, not for readability.

However, the result looks like this:

```
{
  "rows": [
    {
      "col1": "AAA",
      "col2": 2,
      "col3": "BBB"
    },
    {
      "col1": "CCC",
      "col2": 3,
      "col3": "DDD"
    }
  ],
  "number_of_rows": 2,
  "columns": {
    "col1": "TEXT",
    "col2": "INT",
    "col3": "TEXT"
  }
}
```

The `rows` key contains the actual result set, each row is an object with a key the name of the column and with value the actual value of the row.

Then there is the `number_of_rows` key, an integer that describes how many rows are returned in this result set.

The `columns` field maps the name of the column to their type.

Overall returning JSON from zeeSQL is very simple, just add the `JSON` flag to your command and you are done.

## About zeeSQL

zeeSQL is a Redis Module that provides SQL capabilities to Redis.
It allows the creation and management of several SQL databases, each one independent from the other.
Moreover, zeeSQL provides out-of-the-box [secondary indexes](../secondary-indexes.md) capabilities, allowing fast and easy search by value in Redis.



[exec]: ../references.md#zeesql-exec
[query]: ../references.md#zeesql-query
[json]: ../references.md#json-flag
