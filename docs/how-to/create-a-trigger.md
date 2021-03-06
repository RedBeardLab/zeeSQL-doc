# How to create a trigger

Triggers are one way to keep a consistent state of your data.

They are not the only way, and somehow, they are looked upon, however they can be very powerful.

zeeSQL is based on SQLite, so all that we are saying, apply equally to both zeeSQL and SQLite itself.

When you modify your database, with an `UPDATE` or a `DELETE` or a `INSERT` triggers can be invoked and they can modify your databases.

To create a trigger, we need to define:

1. When to invoke it
2. What the trigger should do

A trigger can be invoked in response to either an UPDATE or a DELETE or an INSERT.

We can also specify if we want the trigger to be invoked before the action takes place, or just after.

The action that the trigger should do, is a simple SQL command, it can be an INSERT or an UPDATE or a DELETE.

Trigger are very useful to keep the database consistent with some view of the world that was not possible to express in the SQL schema, or to keep counters.

For instance, suppose we want to have a very quick way to know how many rows are in a table. We can either run a count\(\), or we can keep track of each row with a trigger.

