# Advanced SQL challenges


## Contents:
- [Subscription price changes](#subscription-price-changes)
- [Apportioning payments](#apportioning-payments)

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
| 1               | 60        | 2020-01-15 | 2020-02-01   |


Things worth noting in the final table.
- We don't care about `change_id = 1` from the original table — that's because it was superseded by `change_id = 2` before it took effect.
- We also haven't included `change_id = 3` here, as it's associated with `subscription_id = 2`, which doesn't have an associated rebill (perhaps the subscription is actually cancelled, and the change was needlessly applied).
- The `effective_at` date comes from the `rebillings` table


### Instructions

1. Write the SQL to produce the `effective_subscription_changes` table.
2. Bonus: Write SQL to prove that _your_ `effective_subscription_changes` table matches the sample one

Note: the above tables can all be queried from BigQuery — the [dbt tutorial](https://docs.getdbt.com/tutorial/setting-up#create-a-bigquery-project) has a handy section on how to create a BigQuery account and query a public dataset.

I recommend that you use the public BigQuery tables to perform this task. You may also load the CSVs in the [subscription-price-changes](/subscription-price-changes) directory of this repo into the warehouse of your choosing.

## Apportioning payments
A task marketplace startup is grappling with how to apportion their payments for accounting purposes.

On the platform:
- A user posts a task
- A worker offers to do it for a particular price
- A user accepts the offer, and the amount goes into escrow
- Upon successful completion of the task, the worker receives all the money¹

The associated payments for a typical task (`task_id = 1`) in the database look like this:
| payment_id | task_id | payment_type | amount |
|------------|---------|--------------|--------|
| 1          | 1       | inbound      | 50     |
| 2          | 1       | payout       | 50     |

For accounting purposes, it needs to be transformed to this:
| task_id | inbound_payment_id | inbound_amount | payout_amount   |
|---------|--------------------|----------------|-----------------|
| 1       | 1                  | 50             | 50              |

Seems do-able.

*However*, sometimes after the money is placed in escrow, the task gets cancelled, and all money is refunded to the user.

The associated payments for such a task (`task_id = 2`) looks like this:

| payment_id | task_id | payment_type | amount |
|------------|---------|--------------|--------|
| 3          | 2       | inbound      | 30     |
| 4          | 2       | refund       | 30     |

And for accounting purposes it needs to looks like this:

| task_id | inbound_payment_id | inbound_amount | payout_amount   | refund_amount |
|---------|--------------------|----------------|-----------------|---------------|
| 2       | 3                  | 0              | 30              | 30            |

Also seems do-able.

**However**, sometimes after the money is placed in escrow, a second inbound payment occurs because the task was bigger than expected. Assuming that the worker is paid out fully, the payments might look like this for such a task (`task_id = 3`):

| payment_id | task_id | payment_type | amount |
|------------|---------|--------------|--------|
| 5          | 3       | inbound      | 40     |
| 6          | 3       | inbound      | 20     |
| 7          | 3       | payout       | 60     |

And for accounting purposes it needs to looks like this:

| task_id | inbound_payment_id | inbound_amount | payout_amount   | refund_amount |
|---------|--------------------|----------------|-----------------|---------------|
| 3       | 5                  | 40             | 40              | 0             |
| 3       | 6                  | 20             | 20              | 0             |

Note: things are getting complicated here. For complicated accounting reasons required by the finance department, it's important that the `payout` payment is apportioned over each inbound payment such that the payout amount does not exceed the inbound amount — here is was split into $40 and $20 to match that requirement. Okay, getting tricky

**HOWEVER**, sometimes a partial refund needs to occur for a task — maybe only half the task was done. And every so often that occurs on a task that had two inbound payments.

The associated payments for such a task (`task_id in (4, 5)`) looks like this:

| payment_id | task_id | payment_type | amount |
|------------|---------|--------------|--------|
| 8          | 4       | inbound      | 20     |
| 9          | 4       | payout       | 15     |
| 10         | 4       | refund       | 5      |
| 11         | 5       | inbound      | 20     |
| 12         | 5       | inbound      | 40     |
| 13         | 5       | inbound      | 30     |
| 14         | 5       | payout       | 25     |
| 15         | 5       | refund       | 65     |


And for accounting purposes it looks like this:

| task_id | inbound_payment_id | inbound_amount | payout_amount   | refund_amount |
|---------|--------------------|----------------|-----------------|---------------|
| 4       | 8                  | 20             | 15              | 5             |
| 5       | 11                 | 20             | 20              | 0             |
| 5       | 12                 | 40             | 5               | 35            |
| 5       | 13                 | 30             | 0               | 30            |


_Y I K E S_, that `task_id = 5` is particularly nasty — there's _three_ inbound payments, and we need to split the payout payment across the first two inbound payments, and then split the refund across the second and third payments. And no, apparently we can't apportion them according to their ratios — we instead need to first fully "refund" the first inbound payment (`inbound_payment_id = 11`), and then partially refund the second inbound payment (`inbound_payment_id = 12)`).

## Exercise
1. Write the SQL to transform the ledger of payments into the format required by accounting. Use the following BigQuery tables, or load the CSVs in [apportioning-payments](/apportioning-payments) into the warehouse of your choice:
```sql
-- input data
select * from `advanced-sql-challenges`.`apportioning_payments`.`payments`

-- sample output for comparison
select * from `advanced-sql-challenges`.`apportioning_payments`.`inbound_payment_states`
```
2. List the assumptions that you are making about your source data as you write this

## Bonus exercises
1. Write SQL to prove that _your_ `inbound_payment_states` table matches the sample result.
2. Consider how this might work for in-progress tasks that only have `inbound` payments, where an additional column `in_escrow` tracks any money held in the bank account.
3. Consider what might happen if there are two refund payments for the one task


¹Perhaps their accounting problems are the least of their woes since currently the business does not capture any fees.
