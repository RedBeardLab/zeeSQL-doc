# Motivation

I build `zeeSQL` to remove operational and software complexity from most small to medium application.

## Remove complexity from small and medium application

Even simple apps are becoming increasingly complex.

Applications today usually have at least 3 moving parts.

1. Some persistent layer like a database
2. A fast ephemeral storage
3. Some sort of queue

Modern database solution like Postres can usually manage almost everything, but they are never really quite enough.

You are not going to cache your data in Postgres, you are going to use Redis or Memcache.
Similarly for session information.

Moreover, databases are hard and complex to operate.
Complex, even before to create all the modern container orchestration infrastructure.

Redis is another very candidate, it works very well as fast ephemeral storage and as queue system.
Moreover, it has great persistency capabilities.
However, beside simple uses cases, it is hard to use as only database.

Most application need some form of complex data query and filtering.

`zeeSQL` is born to address this small niche.

It provides a fast, simple and easy to operate SQL engine that is embed in Redis.
Adding more capabilities on top of the Redis features.
It inherits all the persistency guarantees of Redis, and it is perfect to use as persistency layer in small to medium applications.

Using `zeeSQL` it is possible to use a single, easy to maintain, and easy to operate external process for all the application data needs.

Then working in-memory by default, `zeeSQL` turns out to be very fast. And I try my very best to keep performance as high as possible.

## Simplify Redis

The first version of `zeeSQL` kept completely separated the data belongings to `zeeSQL` and the data from Redis.

`zeeSQL` was not able to query and see data from Redis.

Then users start to ask how to query data that are stored in Redis.

People wanted to use the SQL capabilities of `zeeSQL` to query data in Redis.
It makes sense, the technology and the code is already ready for this use case, and it improve the uses cases of `zeeSQL`.

The latest version of `zeeSQL` can now integrate with Redis hashes allow people to search Redis data by value.

This is really another big simplification for developers and Redis users.

Without `zeeSQL`, if you needed to search value in Redis hash, you had two choices.

1. Get all the data from Redis to your code, and then implement search, filter and aggregation by hand.
2. Maintains separated data structure in Redis to quickly identify the elements you are interested in.

Of course neither choice is optimal.

Fetching all the data from Redis is a slow operation, that keep the Redis process busy, increments the tail latency and saturate the bandwidth.
Also implementing search, filtering and aggregation in code is a slow and error prone process.
It would be much better if Redis could return directly only the data we are interested in.

Keeping separated data structures is very cumbersome and error prone.
Whenever you update a Redis hash value, you need also to update all the other data structure, otherwise you will keep a wrong views of your data.
And these updates need to be done on insertion, deletions, and when you modify the data.
Moreover, it is not flexible when new bussiness requirements come along.
It would be much better if Redis could keep track itself of the data and figure out by itself how to query them.

With `zeeSQL` all of this is now possible.

Values from Redis hashes are pushed into a standard SQL table, and from there they can be queried.

This is a superior model to search Redis by value.
It keep the best of the two alternatives overcoming both limitation.

It allows users to specify what data they want, and `zeeSQL` find the best way to provide only those information.
Without the need to maintains separated data structures.

## Simplify, simplify, simplify

The motivation for creating `zeeSQL` is to simplify developers life.

First, it simplified operations at the data level, offering a single solution for small to medium applications to solve all their data needs.

Then `zeeSQL` simplifies Redis operations, solving how to search Redis by value and not only by key.

If you would like to learn more about `zeeSQL`, please visit the [README](README.md) or try to [follow the tutorial](tutorial.md).
You can also check out the [command references.](references.md)

If you got more questions you can contact me at [simone@redbeardlab.com](mailto:simone@redbeardlab.com)
