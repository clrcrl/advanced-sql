# Advanced SQL challenges


## Contents:
- [Subscription price changes](#subscription-price-changes)

## Subscription price changes

### Background

At my seltzer subscription box startup, customers get billed once a month. In between billing dates, customers can upgrade or downgrade their subscription box (which impacts how many seltzers they'll be sent).

These upgrade and downgrades are captured in a table, named `subscription_price_changes`:

```sql
select * from `advanced-sql-challenges`.`subscription_price_changes`.`subscription_price_changes`
```

| change_id | subscription_id | price | changed_at |
|-----------|-----------------|-------|------------|
| 1         | 1               | 50    | 2020-01-10 |
| 2         | 1               | 60    | 2020-01-15 |
| 3         | 2               | 40    | 2020-01-12 |
| ...       | ...             | ...   | ...        |

However, the change only takes effect on the date the customer gets rebilled — as above, this only happens once a month (the day of the month varies per customer).

Fortunately, I also have a table of `rebillings`, with one record per billing date.
```sql
select * from `advanced-sql-challenges`.`subscription_price_changes`.`rebillings`
```

| rebilling_id | subscription_id | rebilled_at |
|--------------|-----------------|-------------|
| 1            | 1               | 2020-02-01  |
| 2            | 1               | 2020-03-01  |
| 3            | 3               | 2020-02-12  |

I need to produce a table that tells me the effective dates of each subscription price change:

```sql
select * from `advanced-sql-challenges`.`subscription_price_changes`.`effective_subscription_changes`
```

| subscription_id | new_price | changed_at | effective_at |
|-----------------|-----------|------------|--------------|
| 1               | 60        | 2020-02-15 | 2020-02-01   |


Things worth noting in the final table.
- We don't care about `change_id = 1` from the original table — that's because it was superseded by `change_id = 2` before it took effect.
- We also haven't included `change_id = 3` here, as it's associated with `subscription_id = 2`, which doesn't have an associated rebill (perhaps the subscription is actually cancelled, and the change was needlessly applied).
- The `effective_at` date comes from the `rebillings` table


### Instructions

1. Write the SQL to produce the `effective_subscription_changes` table.
2. Bonus: Write SQL to prove that _your_ `effective_subscription_changes` table matches the sample one

Note: the above tables can all be queried from BigQuery — the [dbt tutorial](https://docs.getdbt.com/tutorial/setting-up#create-a-bigquery-project) has a handy section on how to create a BigQuery account and query a public dataset.

I recommend that you use the public BigQuery tables to perform this task. You may also load the CSVs in the [subscription-price-changes](/subscription-price-changes) directory of this repo into the warehouse of your choosing.
