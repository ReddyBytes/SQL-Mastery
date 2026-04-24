# DISTINCT and Aliases

## Two Analogies in One

**DISTINCT is like deduplicating a mailing list.**

You export 5,000 email addresses from your CRM to send a newsletter. Problem: some customers
appear three times because they made three purchases. You don't want to spam them. So you run a
deduplicate step — keep only unique email addresses. That's `SELECT DISTINCT`.

**Aliases are like nickname labels on a report.**

You're handing a spreadsheet to your CEO. Column name `usr_acct_ctry_cd` means nothing to her.
You rename it to `Country` for the report. The underlying data doesn't change — only the label.
That's `AS`.

---

## The Data We'll Use

```
  TABLE: orders
  ┌─────┬─────────────┬──────────────┬────────────┬────────────┐
  │ id  │ customer_id │ total_amount │ status     │ created_at │
  ├─────┼─────────────┼──────────────┼────────────┼────────────┤
  │  1  │      1      │    149.99    │ delivered  │ 2024-01-05 │
  │  2  │      2      │    349.00    │ delivered  │ 2024-01-18 │
  │  3  │      1      │     49.50    │ refunded   │ 2024-02-02 │
  │  4  │      3      │    899.95    │ delivered  │ 2024-02-20 │
  │  5  │      4      │     22.00    │ pending    │ 2024-03-01 │
  │  6  │      2      │    220.00    │ shipped    │ 2024-03-08 │
  │  7  │      5      │    115.00    │ delivered  │ 2024-03-15 │
  │  8  │      1      │   1250.00    │ delivered  │ 2024-03-22 │
  │  9  │      3      │     78.50    │ pending    │ 2024-03-28 │
  │ 10  │      4      │     62.00    │ cancelled  │ 2024-03-30 │
  └─────┴─────────────┴──────────────┴────────────┴────────────┘

  TABLE: customers
  ┌─────┬────────────┬───────────┬─────────────────────────┬─────────┐
  │ id  │ first_name │ last_name │ email                   │ country │
  ├─────┼────────────┼───────────┼─────────────────────────┼─────────┤
  │  1  │ Alice      │ Nguyen    │ alice@example.com       │ Canada  │
  │  2  │ Bob        │ Martinez  │ bob@example.com         │ USA     │
  │  3  │ Chen       │ Wei       │ chen.wei@example.com    │ China   │
  │  4  │ Diana      │ Okafor    │ diana.o@example.com     │ Nigeria │
  │  5  │ Ethan      │ Brooks    │ ethan.b@example.com     │ USA     │
  └─────┴────────────┴───────────┴─────────────────────────┴─────────┘
```

---

## SELECT DISTINCT — Unique Values Only

Without `DISTINCT`, every row is returned:

```sql
SELECT status
FROM   orders;
```

Result:

```
  status
  ──────────
  delivered
  delivered
  refunded
  delivered
  pending
  shipped
  delivered
  delivered
  pending
  cancelled
```

Ten rows, lots of repeats. To see just the unique statuses in use:

```sql
SELECT DISTINCT status
FROM   orders;
```

Result:

```
  status
  ──────────
  delivered
  refunded
  pending
  shipped
  cancelled
```

Five distinct values. Perfect for populating a dropdown, validating data, or quick discovery.

---

## DISTINCT Across Multiple Columns

`DISTINCT` applies to the **combination** of all selected columns, not just the first:

```sql
SELECT DISTINCT customer_id, status
FROM   orders;
```

This returns unique (customer_id, status) pairs — not unique customer_ids alone:

```
  customer_id │ status
  ────────────┼───────────
       1      │ delivered
       1      │ refunded
       2      │ delivered
       2      │ shipped
       3      │ delivered
       3      │ pending
       4      │ pending
       4      │ cancelled
       5      │ delivered
```

Nine rows — every unique combination, even though customer_id alone only has 5 distinct values.

---

## COUNT(DISTINCT column) — Count Unique Values

One of the most practical patterns in analytics:

```sql
-- How many unique customers have placed at least one order?
SELECT COUNT(DISTINCT customer_id) AS unique_customers
FROM   orders;
```

Result: `5` (customers 1, 2, 3, 4, 5 each appear at least once).

Compare to `COUNT(customer_id)` without DISTINCT:

```sql
SELECT COUNT(customer_id)          AS total_order_rows,
       COUNT(DISTINCT customer_id) AS unique_customers
FROM   orders;
```

```
  total_order_rows │ unique_customers
  ─────────────────┼─────────────────
        10         │        5
```

This pattern is everywhere in analytics: "how many unique users visited today?",
"how many distinct products were sold?", "how many countries do our customers come from?"

---

## Column Aliases with AS

An alias renames a column in the query output. It doesn't change the table or stored data.

```sql
SELECT
    first_name                          AS "First Name",
    last_name                           AS "Last Name",
    country                             AS "Country"
FROM customers;
```

Result:

```
  First Name  │ Last Name  │ Country
  ────────────┼────────────┼─────────
  Alice       │ Nguyen     │ Canada
  Bob         │ Martinez   │ USA
  ...
```

Aliases are especially important on computed columns, which would otherwise have an ugly
auto-generated name:

```sql
-- Without alias: column header is "?column?" or "total_amount * 0.9"
SELECT
    product_name,
    unit_price * 0.90   AS discounted_price,
    unit_price * 0.10   AS savings
FROM products;
```

**Quoting rules for aliases:**

```
  Style               When to use
  ────────────────────────────────────────────────────────────────────
  AS simple_word      No spaces, no caps — no quotes needed
  AS "Full Name"      Contains spaces or mixed case — use double quotes
  AS revenue_2024     Numbers OK as long as alias starts with a letter
```

> **MySQL note:** MySQL allows both `"double quotes"` and backticks for aliases.
> SQLite behaves like PostgreSQL here.

---

## The AS Keyword Is Optional

SQL allows you to omit `AS` — the alias still works:

```sql
-- These are identical:
SELECT first_name AS name FROM customers;
SELECT first_name    name FROM customers;
```

**Best practice:** always write `AS` explicitly. It makes the alias immediately obvious to the
reader and avoids any parser ambiguity.

---

## Table Aliases — A Teaser for Joins

You can alias table names too:

```sql
SELECT c.first_name, c.email
FROM   customers AS c;
```

Here `c` is a shorthand for `customers`. This becomes essential when joining multiple tables —
without aliases, column references become long and ambiguous:

```sql
-- With table aliases (readable)
SELECT c.first_name, o.total_amount
FROM   customers AS c
JOIN   orders    AS o ON c.id = o.customer_id;

-- Without table aliases (verbose and error-prone)
SELECT customers.first_name, orders.total_amount
FROM   customers
JOIN   orders ON customers.id = orders.customer_id;
```

Table aliases are covered fully in the Joins section (Section 5). For now, just know they exist
and follow the same `AS` syntax.

---

## When DISTINCT Is Expensive

`DISTINCT` forces the database to sort or hash the entire result set to remove duplicates. On
large tables this is expensive. Before reaching for `DISTINCT`, ask yourself why duplicates
exist:

```
  Cause of duplicates              Better solution than DISTINCT
  ──────────────────────────────────────────────────────────────────────────
  JOIN is multiplying rows         Fix the JOIN condition
  Missing GROUP BY clause          Add GROUP BY
  Data quality issue in source     Fix upstream, don't paper over it
  You genuinely need unique vals   DISTINCT is appropriate here
```

Example: if a query with a JOIN returns duplicate customer rows, adding `DISTINCT` is a
code smell — fix the JOIN instead.

---

## Putting It All Together

```sql
-- Marketing task: list the unique countries our customers are from,
-- plus how many customers are in each country
SELECT
    country                             AS "Customer Country",
    COUNT(DISTINCT id)                  AS "Customer Count"
FROM   customers
GROUP  BY country
ORDER  BY "Customer Count" DESC;
```

Result:

```
  Customer Country │ Customer Count
  ─────────────────┼───────────────
  USA              │       2
  Canada           │       1
  China            │       1
  Nigeria          │       1
```

(`GROUP BY` is covered fully in Section 3 — this is just a preview of alias usage in context.)

---

## Summary

```
┌──────────────────────────────────────────────────────────────────────┐
│  DISTINCT AND ALIASES — KEY TAKEAWAYS                                │
├──────────────────────────────────┬───────────────────────────────────┤
│  SELECT DISTINCT col             │  Unique values only               │
│  DISTINCT on multiple cols       │  Unique combination of all cols   │
│  COUNT(DISTINCT col)             │  Count unique non-null values     │
│  col AS alias                    │  Rename output column             │
│  "Quoted Alias"                  │  For spaces or mixed case         │
│  table AS t                      │  Short name for table (joins)     │
│  AS is optional but recommended  │  Always write it for clarity      │
│  DISTINCT is expensive           │  Diagnose root cause first        │
└──────────────────────────────────┴───────────────────────────────────┘
```

---

**[🏠 Back to README](../README.md)**

**Prev:** [← ORDER BY and LIMIT](./order_by_and_limit.md) &nbsp;|&nbsp; **Next:** [Aggregate Functions →](../03_aggregation/aggregate_functions.md)

**Related Topics:** [SELECT and FROM](./select_and_from.md) · [WHERE and Filtering](./where_and_filtering.md) · [ORDER BY and LIMIT](./order_by_and_limit.md)

---

## 📝 Practice Questions

- 📝 [Q11 · distinct](../sql_practice_questions_100.md#q11--normal--distinct)
- 📝 [Q12 · aliases](../sql_practice_questions_100.md#q12--normal--aliases)
- 📝 [Q95 · debug-distinct-perf](../sql_practice_questions_100.md#q95--debug--debug-distinct-perf)

