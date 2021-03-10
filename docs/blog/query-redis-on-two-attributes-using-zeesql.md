# Query Redis on two attributes

Somebody [asked on StackOverflow how to query Redis for two attributes](https://stackoverflow.com/questions/66310700/what-is-the-right-data-structure-to-use-in-redis-to-query-based-on-two-attribute/66555255#66555255).

The use case is rather simple, the developer is storing a set of physical events, each of them has a start time and a finish time.

We would like to know all the events that are in progress at any given time.

The solution in raw Redis would involve creating two sorted sets, insert in one of those sets the events that start after the input time, insert in the other set all the events that end after the input time, and finally take the intersection of the two sets.
The two sorted sets are not necessary anymore after we obtain our results, so they can be deleted.

This solution works, but it forces the developers to express the logic and all the steps necessary to obtain your results.
Using procedural programming (expressing the steps necessary to obtain a result, like in this case) is more error-prone and complex than using declarative programming (expressing the result that you want, no how-to obtain it).

With [zeeSQL, SQL and search by value for Redis](https://zeesql.com) we can obtain the same result procedurally, without thinking or taking care of temporary sorted sets in Redis.

In this case, we are going to use a zeeSQL secondary index.

## The data format

Building on top of the StackOverflow question, we might expect that each event has a unique id, a start time, an end time, and maybe the number of participants.

We will express the two times as Unix timestamps.

```
127.0.0.1:6379> HMSET event:100 start_time 1615225944 end_time 1615657944 participants 30
OK
127.0.0.1:6379> HMSET event:101 start_time 1615225844 end_time 1615658944 participants 67
OK
127.0.0.1:6379> HMSET event:102 start_time 1615744344 end_time 1616003544 participants 52
OK
```

Now is around ~1615402492 so only `event:100` and `event:101` are ongoing now.

With zeeSQL the very first step would be to create a new database where to store the data, we can create a database and called it `EventsDB`.

```
127.0.0.1:6379> ZEESQL.CREATE_DB EventsDB
1) 1) "OK"
```

After we have created the database, we can add a zeeSQL secondary index to it.
zeeSQL will automatically add all the keys to the newly created secondary index.

```
127.0.0.1:6379> ZEESQL.INDEX EventsDB NEW PREFIX event:* TABLE events SCHEMA start_time INT end_time INT participants INT
OK
```

With this command, we have created a new zeeSQL secondary index.

The secondary index is associated with the SQL table `events` and stores all the Redis Hashes that start with the prefix `event:`.

The index has 3 columns plus one.

The first column is the primary key and stores the Redis key of the hashes, in this case, it will store the string `event:100`, `event:101`, and `event:102`.

The other columns are the `start_time`, the `end_time`, and the `participants` column.
Each one is an integer column and will store the respective fields of the Redis Hash.

As soon as the index is created, zeeSQL automatically populates it with the Redis Hash already in Redis.

We can immediately visualize the data.

```
127.0.0.1:6379> ZEESQL.EXEC EventsDB COMMAND "SELECT * FROM events"
1) 1) "RESULT"
2) 1) "key"
   2) "start_time"
   3) "end_time"
   4) "participants"
3) 1) "TEXT"
   2) "INT"
   3) "INT"
   4) "INT"
4) 1) "event:102"
   2) (integer) 1615744344
   3) (integer) 1616003544
   4) (integer) 52
5) 1) "event:101"
   2) (integer) 1615225844
   3) (integer) 1615658944
   4) (integer) 67
6) 1) "event:100"
   2) (integer) 1615225944
   3) (integer) 1615657944
   4) (integer) 30
```

Now we can try to answer the original question.

What are the events that are on-going now, or at any given time?

```
127.0.0.1:6379> ZEESQL.EXEC EventsDB COMMAND "SELECT key FROM events WHERE start_time < ?1 AND end_time > ?1" ARGS 1615402492
1) 1) "RESULT"
2) 1) "key"
3) 1) "TEXT"
4) 1) "event:101"
5) 1) "event:100"
```

Simple.

No other data structure to manage, just a simple SQL query to write.

## Use the standard SQLite time functions

You might not know what time it is now, in this case, you can use the standard SQLite functions.

```
127.0.0.1:6379> ZEESQL.EXEC EventsDB COMMAND "SELECT key FROM events WHERE start_time < strftime('%s', 'now') AND end_time > strftime('%s', 'now')"
1) 1) "RESULT"
2) 1) "key"
3) 1) "TEXT"
4) 1) "event:101"
5) 1) "event:100"
```

Or maybe you need to answer more complex queries, like the original one.

We might want to know all the events that are on-going at the moment and will finish in one week.

We don't have any of them in our Redis instance, but we can add one.

```
127.0.0.1:6379> HMSET event:103 id 103 start_time 1615225944 end_time 1616176492 participants 22
OK
127.0.0.1:6379> ZEESQL.EXEC EventsDB COMMAND "SELECT count(*) FROM events"
1) 1) "RESULT"
2) 1) "count(*)"
3) 1) "INT"
4) 1) (integer) 4
```

zeeSQL automatically adds the new event to the secondary index.

And now we can express our more complex query:

```
127.0.0.1:6379> ZEESQL.EXEC EventsDB COMMAND "SELECT key FROM events WHERE start_time < strftime('%s', 'now') AND end_time > strftime('%s', 'now', '+7 day')"
1) 1) "RESULT"
2) 1) "key"
3) 1) "TEXT"
4) 1) "event:103"
```

## Using all SQL

At this point, is useful to clarify that in zeeSQL you can use all the SQL provided by SQLite.

For instance, you may want to know how many people are registered for all the events that are ongoing right now.

```
127.0.0.1:6379> ZEESQL.EXEC EventsDB COMMAND "SELECT key, participants FROM events WHERE start_time < ?1 AND end_time > ?1" ARGS 1615402492
1) 1) "RESULT"
2) 1) "key"
   2) "participants"
3) 1) "TEXT"
   2) "INT"
4) 1) "event:101"
   2) (integer) 67
5) 1) "event:100"
   2) (integer) 30
6) 1) "event:103"
   2) (integer) 22
127.0.0.1:6379> ZEESQL.EXEC EventsDB COMMAND "SELECT SUM(participants) FROM events WHERE start_time < ?1 AND end_time > ?1" ARGS 1615402492
1) 1) "RESULT"
2) 1) "SUM(participants)"
3) 1) "INT"
4) 1) (integer) 119
```

## Getting data as JSON

The last interesting step that I would like to make you aware, it is how to return data as JSON.

Working with Redis nested arrays is not always easy, and sometimes JSON is more convenient to deserialize.

In zeeSQL, you only need to add the `JSON` flag, and your result will be returned as a JSON object.

```
root@96ad2f8bdfe5:/data# redis-cli ZEESQL.EXEC EventsDB COMMAND "SELECT key, participants FROM events WHERE start_time < ?1 AND end_time > ?1" JSON ARGS 1615402492 | jq
{
  "rows": [
    {
      "key": "event:101",
      "participants": 67
    },
    {
      "key": "event:100",
      "participants": 30
    },
    {
      "key": "event:103",
      "participants": 22
    }
  ],
  "number_of_rows": 3,
  "columns": {
    "key": "TEXT",
    "participants": "INT"
  }
}
```

Here we pipe the result in `jq` for easier visualization.

## Conclusions

zeeSQL solves a lot of problems when working with Redis.

In this particular case simplify the querying of data using a zeeSQL secondary index.
This avoided the need to keeping separated indexes in Redis that can become cumbersome and error-prone.

This particular use case could have been solved without any Redis data structure, but simply using zeeSQL as the main datastore.

All the standard SQL commands are available like `SELECT`, `INSERT`, `UPDATE`, and `DELETE`.

zeeSQL by default works in-memory, like Redis, and it is blazing fast.
Moreover, being completely integrated with Redis, zeeSQL supports AOF and RDB persistency.
