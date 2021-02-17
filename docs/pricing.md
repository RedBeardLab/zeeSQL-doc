# Pricing for zeeSQL

zeeSQL use a credit system.

In `zeeSQL` there are two main concepts, `database`s and `secondary indexe`s.

Running either one \(database or one secondary index\) cost 1 credit every hour.

One credit costs 0,01€

There are 3 free credits hour available for each installation of `zeeSQL`.

To use beyond of the 3 credits, you need to input a license key.

We device this pricing schema to scale on the **complexity** that zeeSQL is managing not on the size of your datasets.

## Example

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

In case of replication, each replica get 3 free credits.

## Pathological cases

Please get in touch if your application is a patological cases.

For instance if you design your application to have one database for each user.

