# zeeSQL, SQL and search by value fore Redis. Fast, Simple and Reliable.

This is a short introduction to zeeSQL, a Redis modules that brings SQL capabilities into Redis.

This document is your entry point for the documentation and will guide you to what to read next.

## Quickstart

The very first thing is to get the binary.

```
wget https://s3.eu-central-1.amazonaws.com/zeesql.com/releases/latest/zeesql.so -o $HOME/zeesql.so
```

Once you got the module you can start a Redis instance loading the module.

```
redis-server --loadmodule $HOME/zeesql.so
```

At this point you can start a Redis client to interact with the module.

Below we are creating a database, create a new table, insert new rows, and query those rows back.

```
$ redis-cli
> ZEESQL.CREATE_DB DB
1) 1) "OK"
> ZEESQL.EXEC DB COMMAND "CREATE TABLE users(id STRING, score INT);"
> ZEESQL.EXEC DB COMMAND "INSERT INTO users VALUES('jausten', 3), ('hugo', 5);"
> ZEESQL.EXEC DB COMMAND "SELECT * FROM users;"
```

`zeeSQL` allows searching Redis hashes by values, making much simpler to express complex queries.

```
> ZEESQL.INDEX DB NEW PREFIX products:* TABLE products SCHEMA id INT price INT name STRING
> HMSET products:123 id 123 price 2345 name "set of glasses"
> HMSET products:471 id 471 price 3459 name "wall clock"
> ZEESQL.EXEC DB COMMAND "SELECT name FROM products WHERE price > 2500;"
```

If you think that `zeeSQL` can solve some of your problem, please keep reading for more context and details.

Or you can [read the Tutorial](tutorial.md) to get a better overview on how `zeeSQL` works and what it can do for you.

## What is zeeSQL

`zeeSQL` is a Redis module. 
In version 4, Redis introduce this new features of modules.

A module, once loaded into Redis, provides new capabilities to Redis itself.
Most of the time this means new commands that the user can access using the standard Redis interface.

Either by command line or by API access.

`zeeSQL` embeds SQLite into Redis.

This allows to create SQL databases that can be easily accessed via Redis.
The main motivation for the project, back then, was to create a simple and easy to operate data layer.
Instead of having, a DBMS, a cache layer and a queue, you can just use `zeeSQL` with Redis and have all the tools to run a rather big web service.

`zeeSQL` outgrow this first stage.
Our users were to keep demanding tighter integration between the data stored in Redis and what was accessible over SQL.
So, `zeeSQL` secondary indexes were born.

Secondary indexes allow getting data from Redis hashes using standard SQL.
It is possible to look into all your keys and get only the one that have the eg. `score` field set to a number greater than 20.

Similarly, it is possible to do aggregation and filtering.

## How zeeSQL can help

`zeeSQL` can help whenever your infrastructure is too complex.

If you are having issue keeping your main database and its cache in sync, `zeeSQL` help.

If your Redis datamodel is too complex, `zeeSQL` can help.

If anytime you change a piece of information inside Redis you need to also modify dozen of other Redis keys, `zeeSQL` can simplify that.

If you need a fast SQL engine that works in memory, that you can integrate now in your infrastructure, `zeeSQL` will be fast.

## Getting the binary

The binary is distributed freely.

It is available on the website: 

```
wget https://zeesql.com/releases/latest/zeesql.so -o $HOME/zeesql.so
```

The URL is stable and provides always the latest released version of `zeeSQL`.

More information on [how to get zeeSQL.](how-to/get-zeesql.md)

## Loading the module

To use `zeeSQL` you need to load it into your Redis instances.

After you download the binary, and placed somewhere accessible, you can load the module in different way.

The first way is to pass the module as argument to Redis.

```
redis-server --loadmodule $HOME/zeesql.so
```

The second way is to configure Redis to start with the module.
It is sufficient to add the following line to your `redis.conf` file.

```
loadmodule $HOME/zeesql.so
```

The final way is to load the module issuing a command against your Redis instance.

```
redis-cli MODULE LOAD $HOME/zeesql.so
```

Each of these three way, if successful, will load `zeeSQL` into Redis making it ready to use.

## Getting a license

In order to offer a great product, with great documentation and with support, `zeeSQL` must limits its capabilities for free users and let people pay a fair price for what we believe being good software.

Free users can use the product as much as they like and we will provide support to them without any issue.
But `zeeSQL` will be limited in the amount of databases and secondary indexes that can be created.

Note how the lack of license does not limit the size of your dataset, but only the complexity tha `zeeSQL` manages for you.

In order to remove these limitations, it is possible to [buy a license][license].

More information about the [pricing in the dedicate page](pricing.md).


[license]: https://license.zeesql.com
