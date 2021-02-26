# How to check if an index is used in zeeSQL and SQLite

This tutorial will cover both zeeSQL and SQLite.
As you might know zeeSQL is actually based on SQLite, so everything we says about zeeSQL can also be applied to raw SQLite.

## What are indexes

Indexes are secondary data-structures that sit next to the main data in your database.

They don't store the main data, but only a subset of them, usually few columns, and the primary key.

The main point of indexes is that the data is always store sorted with the respect of the columns they are indexing.

For instance if we create an index for the `score` of some users, in the index, the scores will be stored sorted, from the smallest to the largest (or vice-versa).

Storing the field sorted, allow a fast comparison for those fields.
For instance, if we want to select all the users with score greater than 20, we can lookup in the index where the users with score greater than 20 starts, and return all the user after that.

However, an index on the score, won't help if we are looking for all the users that played more than 15 games.
For that query you will need a different index.

As your uses cases become complex, and the table larger, eventually a lot of index will be accumulated.

Then, for complex queries, it becomes very difficult to know what index, if any, is going to be used.

The SQL engine has its own algorithms and heuristic to figure out, what it think is the fastest way to execute a query.

Please note that while index help in retrieving informations, they need to be maintained, which implies more load when the data are updated or added.
There is a tradeoff between insertion speed, (no index, fastest insertion) and query speed (the more indexes they are, the more query will run optimally).

When no index can be used to speed up a query, SQLite will fallback in a full table scan, which means that it will look at every row in the table.
For big tables, this implies an higher latency and slower results.

## Am I using some index?
