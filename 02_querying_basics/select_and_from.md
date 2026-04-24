# SELECT and FROM

## The Question-and-Answer Analogy

Every SQL query is a question. When you want information from a database, you ask in a very
specific structure:

> "Give me [these pieces of information] from [this table]."

That maps directly to SQL:

```
  Give me these pieces of information   →   SELECT  first_name, email
  from this table                        →   FROM    customers
```

The librarian analogy from earlier applies perfectly here. `FROM` tells the database *which shelf*
to look at. `SELECT` tells it *which columns to pull off that shelf* and hand back to you.

You're not moving data. You're not changing anything. A `SELECT` query is purely a question — a
read operation that leaves the database completely unchanged.

---

## The Customers Table We'll Use

Throughout this section, assume this table exists and contains this data:

```
  TABLE: customers
  ┌─────┬────────────┬───────────┬─────────────────────────┬─────────┬─────────────┐
  │ id  │ first_name │ last_name │ email                   │ country │ signup_date │
  ├─────┼────────────┼───────────┼─────────────────────────┼─────────┼─────────────┤
  │  1  │ Alice      │ Nguyen    │ alice@example.com       │ Canada  │ 2022-03-15  │
  │  2  │ Bob        │ Martinez  │ bob@example.com         │ USA     │ 2022-07-01  │
  │  3  │ Chen       │ Wei       │ chen.wei@example.com    │ China   │ 2023-01-20  │
  │  4  │ Diana      │ Okafor    │ diana.o@example.com     │ Nigeria │ 2023-06-08  │
  │  5  │ Ethan      │ Brooks    │ ethan.b@example.com     │ USA     │ 2024-02-14  │
  └─────┴────────────┴───────────┴─────────────────────────┴─────────┴─────────────┘
```

---

## SELECT * — Get Everything

The asterisk `*` is a wildcard meaning "all columns":

```sql
SELECT *
FROM   customers;
```

Result: all 5 rows, all 6 columns, in the order they're stored in the table.

**When to use `SELECT *`:**
- Exploring an unfamiliar table for the first time
- Quickly checking what's in a table during debugging

**When NOT to use `SELECT *` in production:**
- It returns columns you may not need, wasting bandwidth and memory
- If someone adds or removes a column later, your application may break
- It prevents the database from using covering indexes efficiently

---

## SELECT Specific Columns

Name only the columns you want, separated by commas:

```sql
SELECT first_name, email
FROM   customers;
```

Result:

```
  first_name  │ email
  ────────────┼─────────────────────────
  Alice       │ alice@example.com
  Bob         │ bob@example.com
  Chen        │ chen.wei@example.com
  Diana       │ diana.o@example.com
  Ethan       │ ethan.b@example.com
```

The column order in your `SELECT` list controls the order in the output — it does not need to
match the order columns appear in the table:

```sql
SELECT email, last_name, id
FROM   customers;
```

Result has columns in exactly that order: email, last_name, id.

---

## SELECT With Expressions

You're not limited to raw column values. You can compute new values on the fly:

```sql
-- Combine first and last name into a single string
SELECT
    first_name || ' ' || last_name  AS full_name,
    email
FROM customers;
```

Result:

```
  full_name       │ email
  ────────────────┼──────────────────────────
  Alice Nguyen    │ alice@example.com
  Bob Martinez    │ bob@example.com
  Chen Wei        │ chen.wei@example.com
  Diana Okafor    │ diana.o@example.com
  Ethan Brooks    │ ethan.b@example.com
```

> **MySQL / SQLite note:** MySQL and SQLite use `CONCAT(first_name, ' ', last_name)` instead of
> the `||` concatenation operator. PostgreSQL supports both.

You can also do maths:

```sql
-- Orders table: show price with 10% discount applied
SELECT
    product_name,
    unit_price,
    unit_price * 0.90  AS discounted_price
FROM products;
```

---

## SELECT Constant Values

You can SELECT literal values — useful for testing or adding fixed labels:

```sql
SELECT
    first_name,
    'active'        AS account_status,
    2024            AS report_year
FROM customers;
```

Result:

```
  first_name  │ account_status │ report_year
  ────────────┼────────────────┼─────────────
  Alice       │ active         │ 2024
  Bob         │ active         │ 2024
  ...
```

---

## How SQL Actually Executes: FROM Before SELECT

Here's a subtlety that trips up beginners: the SQL you *write* is not the order SQL *evaluates*
in. The conceptual execution order is:

```
  Written order:    SELECT → FROM → WHERE → GROUP BY → HAVING → ORDER BY → LIMIT
  Execution order:  FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```

`FROM` runs first. The database identifies the table and loads the relevant rows into memory.
Then `SELECT` runs to pick which columns to show in the final output.

This matters because of a common mistake — you cannot use a column alias defined in `SELECT`
inside a `WHERE` clause:

```sql
-- This FAILS — WHERE runs before SELECT, so 'full_name' doesn't exist yet
SELECT first_name || ' ' || last_name AS full_name
FROM   customers
WHERE  full_name LIKE 'Alice%';   -- ERROR

-- This works — use the original expression in WHERE
SELECT first_name || ' ' || last_name AS full_name
FROM   customers
WHERE  (first_name || ' ' || last_name) LIKE 'Alice%';
```

---

## Progressive Example: Building a Query Step by Step

**Goal:** Get each customer's full name and how many days they've been a member.

**Step 1:** Start with `FROM` — identify the table.

```sql
SELECT *
FROM   customers;
```

**Step 2:** Pick the columns you need.

```sql
SELECT first_name, last_name, signup_date
FROM   customers;
```

**Step 3:** Add an expression for days since signup.

```sql
SELECT
    first_name,
    last_name,
    signup_date,
    CURRENT_DATE - signup_date  AS days_as_member
FROM customers;
```

Result:

```
  first_name │ last_name │ signup_date │ days_as_member
  ───────────┼───────────┼─────────────┼───────────────
  Alice      │ Nguyen    │ 2022-03-15  │ 731
  Bob        │ Martinez  │ 2022-07-01  │ 623
  Chen       │ Wei       │ 2023-01-20  │ 419
  Diana      │ Okafor    │ 2023-06-08  │ 280
  Ethan      │ Brooks    │ 2024-02-14  │ 29
```

(Numbers approximate — calculated from a 2024-03-14 reference date.)

---

## Formatting Conventions

SQL keywords are case-insensitive (`select` and `SELECT` are identical), but convention says:

```
  UPPERCASE    SQL keywords    SELECT, FROM, WHERE, JOIN, ORDER BY
  lowercase    table names     customers, orders, products
  lowercase    column names    first_name, email, total_amount
```

Indentation makes complex queries readable:

```sql
-- Compact and hard to read:
select id,first_name,email from customers where country='USA';

-- Properly formatted:
SELECT id, first_name, email
FROM   customers
WHERE  country = 'USA';
```

---

## Summary

```
┌──────────────────────────────────────────────────────────────────────┐
│  SELECT AND FROM — KEY TAKEAWAYS                                     │
├──────────────────────────────────┬───────────────────────────────────┤
│  SELECT *                        │  All columns (use sparingly)      │
│  SELECT col1, col2               │  Specific columns only            │
│  SELECT expr AS alias            │  Computed column with a label     │
│  FROM table_name                 │  Which table to read from         │
│  Execution order                 │  FROM runs before SELECT          │
│  Alias in WHERE                  │  NOT allowed (WHERE runs first)   │
│  || for concatenation (PG)       │  CONCAT() in MySQL/SQLite         │
│  CURRENT_DATE                    │  Today's date in PostgreSQL       │
└──────────────────────────────────┴───────────────────────────────────┘
```

---

**[🏠 Back to README](../README.md)**

**Prev:** [← SQL vs NoSQL](../01_fundamentals/sql_vs_nosql.md) &nbsp;|&nbsp; **Next:** [WHERE and Filtering →](./where_and_filtering.md)

**Related Topics:** [WHERE and Filtering](./where_and_filtering.md) · [ORDER BY and LIMIT](./order_by_and_limit.md) · [DISTINCT and Aliases](./distinct_and_aliases.md)

---

## 📝 Practice Questions

- 📝 [Q2 · select-basics](../sql_practice_questions_100.md#q2--normal--select-basics)
- 📝 [Q3 · execution-order](../sql_practice_questions_100.md#q3--thinking--execution-order)

