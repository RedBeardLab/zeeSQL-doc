# Copying RediSQL databases

One undervalued capability of RediSQL redise in the \[REDISQL.COPY\]\[copy\] command.

As you know, RediSQL databases comes in two different shapes, memory-backends database and file-backend databases.

The memory-backend databases operate only in RAM, all the database is stored in memory and \(excluding RDB and AOF files\) they disapear when the redis instance is shutted down. The file-backend databases store all their content in a standard file, so when the redis instance is shuted down, the file will still be in the filesystem. The file-backend databases can store an huge amount of data since all the data is stored on disk and not on memory, the trade-off is clearly performance.

RediSQL databases support both RDB and AOF files, so the data stored maintain all the persistency guarantees of Redis.

File-backend databases provide an easy way to get data in and out of RediSQL.

## Creating a file-backend database

The command `REDISQL.CREATE_DB $DB_NAME [$path]` take the optional argument $path. If a path is provided, RediSQL try to open an SQLite database from the specified path. If the file does not exists, it creates a new database. If the file is a SQLite database, the database, with all its data, is loaded into RediSQL. If the file is not a SQLite database, a simple error is raised.

This behaviour provide a simple yet very effective way to load data into RediSQL. It is possible to create an SQLite database that already contains all the necessary data, then, we pass this database to the `REDISQL.CREATE_DB` command as path argument. This will create a file-backend database with already all your data loaded.

The fastest and most efficient way to load tons of files inside RediSQL.

The trade-off is clear, the database just created will have the performance of a file-backend database.

## Copying databases

If is it necessary to have faster performance, than a file-backend database, the solution is to copy the database into a memory-backend database. The copy of a database is rather efficient, since it works in memory batches, copying pages of memory at the time and it is not a naive row by row copy.

The `REDISQL.COPY $source $destination` takes as argument a `$source` database and a `$destination` database, creating a copy of the `$source` database into the `$destination` database. The `$destination` database is overwritten and its content is lost.

Overall, the procedure to load a database into RediSQL would be similar to:

```text
# create the SQLite database in /home/ubuntu/input.sqlite
redis> REDISQL.CREATE_DB input /home/ubuntu/input.sqlite
redis> REDISQL.CREATE_DB fast_production
redis> REDISQL.COPY input fast_production
```

We first create two database, `input` and `fast_production`. The `input` database is a file-backend database using our original SQLite database while the `fast_production` is a memory-backend database. Then we copy the content of `input` into `fast_production`.

Now we can query the `fast_production` database that will have all the data in the original `/home/ubuntu/input.sqlite` database.

## Going the other way

While this procedure is convenient to get data inside RediSQL, it can be used also to get the data out of it, for backup reason or for shipping the data to some analytic pipeline.

Again, the procedure is rather simple, we create two databases: `production` and `copy`. `production` is a memory-backend database used during normal operatios, `copy` is a file-backend database used for backup. Then we copy the database from `production` into `copy`.

```text
# create the SQLite database in /home/ubuntu/input.sqlite
redis> REDISQL.CREATE_DB production
redis> REDISQL.CREATE_DB copy /home/ubuntu/copy.sqlite
redis> REDISQL.COPY production copy
```

The last step can be to delete the `copy` database, the file will stay intact in the filesystem, but the database won't use any more resorces from Redis. The database can be deleted using the standard `DEL` command, `DEL copy`.

## Recap

In this article we explore a simple way to use the `REDISQL.COPY` command.

The `REDISQL.COPY` command has also different uses, it can be used to create a stale copy of a database to distribute some traffic. Or it can be used to create a complex structure in a "template" database and quickly replicate the template for different users.

\[copy\]:

