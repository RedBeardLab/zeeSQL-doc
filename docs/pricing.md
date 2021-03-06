# Pricing for zeeSQL

zeeSQL is **not** a product free to run.

However, it comes with a **generous free plan**, so that small and medium applications can work without paying anything.

This document explains how the pricing for `zeeSQL` works.

It starts explaining how we charge for `zeeSQL` and then what is included in the free plan.

## How zeeSQL charges users

zeeSQL uses a credit system linked to a license key.

In `zeeSQL` there are two main concepts, `database`s and `secondary indexes`.

Running either one \(database or one secondary index\) costs 1 credit every hour.

One credit costs 0,01€

We devise this pricing schema to scale on the **complexity** that zeeSQL is managing not on the size of your datasets.

There is an allowance of **2160 free credit each month** for each license.

## zeeSQL free plan

zeeSQL reports its usage \(number of credit used\) only if the user provided a license key.

It is not necessary to provide a license key to use `zeeSQL`.

However, without a license key, it is not possible to spend more than 3 credits each hour. This is enforced by avoiding creating a new database or a new index if you are already using 3 credits for an hour.

To spend more than 3 credits for an hour, you need to input a valid license key.

The 2160 free credit, are there to allow users to input a valid license while still benefit from the free plan.

```text
3 credits each hours * 24 hours a day * 30 days a months = 2160 credit / month.
```

Which is exactly the amount of free credit available.

This will allow to test the system and do maintenance operations.

The 2160 credit/month can be spent also together during a single hour. This allows testing the product with multiple databases and secondary indexes.

## [Getting a license](https://license.zeesql.com)

You can obtain a license from [this website](https://license.zeesql.com).

The license can be shared between multiple `zeeSQL` processes.

## Example

These examples show how much you will be charged for using `zeeSQL`.

They all assume steady-state operation where databases and secondary indexes are not generate or deleted.

| Number of databases | Numbre of indexes | Credit usage | Cost per hour | Cost per month |
| :--- | :--- | :--- | :--- | :--- |
| 1 | 0 | 1 | FREE | FREE |
| 3 | 0 | 3 | FREE | FREE |
| 4 | 0 | 4 | 0.01 | 7.20 |
| 5 | 0 | 5 | 0.02 | 14.40 |
| 1 | 2 | 3 | FREE | FREE |
| 1 | 3 | 4 | 0.01 | 7.20 |
| 2 | 3 | 5 | 0.02 | 14.40 |
| 2 | 4 | 6 | 0.03 | 21.60 |

## Code examples

These code examples show what happens when the user tries to create more than 3 databases without one valid license key.

```text
127.0.0.1:6379> ZEESQL.CREATE_DB DB_1
1) 1) "OK"
127.0.0.1:6379> ZEESQL.CREATE_DB DB_2
1) 1) "OK"
127.0.0.1:6379> ZEESQL.CREATE_DB DB_3
1) 1) "OK"
127.0.0.1:6379> ZEESQL.CREATE_DB DB_4
(error) Not enough credit in your license, please upgrade.
127.0.0.1:6379> ZEESQL.LICENSE SET $one_valid_license
OK
127.0.0.1:6379> ZEESQL.CREATE_DB DB_4
1) 1) "OK"
```

`zeeSQL` returns a simple error and does not allows the user to create more databases.

Similarly with secondary indexes.

```text
127.0.0.1:6379> ZEESQL.CREATE_DB DB_1
1) 1) "OK"
127.0.0.1:6379> ZEESQL.INDEX DB_1 NEW TABLE users PREFIX users:* SCHEMA id INT name STRING
OK
127.0.0.1:6379> ZEESQL.INDEX DB_1 NEW TABLE games PREFIX games:* SCHEMA id INT game_name STRING player INT
OK
127.0.0.1:6379> ZEESQL.INDEX DB_1 NEW TABLE credit PREFIX credit:* SCHEMA user_id INT available_credits INT
(error) Not enough credit in your license, please upgrade.
127.0.0.1:6379> ZEESQL.LICENSE SET $one_valid_license
OK
127.0.0.1:6379> ZEESQL.INDEX DB_1 NEW TABLE credit PREFIX credit:* SCHEMA user_id INT available_credits INT
OK
```

## Pathological cases

Please [get in touch](mailto:simone@redbeardlab.com) if your application is a pathological case.

For instance, if you design your application to have one database for each user.

## Air gapped instances

`zeeSQL` needs to communicate to a specific hostname to communicate how many credits each instance is using.

If your system is without external connectivity, please [get in touch](mailto:simone@redbeardlab.com).

We can provide builds of `zeeSQL` that don't need to report back their usage.

