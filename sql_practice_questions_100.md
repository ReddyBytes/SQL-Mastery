# SQL Practice Questions — 100 Questions from Basics to Mastery

> Test yourself across the full SQL curriculum: querying, schema design, joins, aggregation, window functions, transactions, performance, and critical thinking.

---

## How to Use This File

1. **Read the question** — attempt your answer before opening the hint
2. **Use the framework** — run through the 5-step thinking process first
3. **Check your answer** — click "Show Answer" only after you've tried

---

## How to Think: 5-Step Framework

Apply this to every question:

1. **Restate** — what is this question actually asking?
2. **Identify the concept** — which SQL feature is being tested?
3. **Recall the rule** — what is the exact behaviour of that feature?
4. **Apply to the case** — trace through the query or scenario step by step
5. **Sanity check** — does the result make sense? What edge cases exist?

---

## Progress Tracker

- [ ] **Tier 1 — Basics** (Q1–Q33): SELECT, WHERE, JOINs, GROUP BY, schema fundamentals
- [ ] **Tier 2 — Intermediate** (Q34–Q66): Normalization, indexes, CTEs, subqueries, transactions
- [ ] **Tier 3 — Advanced** (Q67–Q75): Stored procedures, triggers, execution plans, performance
- [ ] **Tier 4 — Interview / Scenario** (Q76–Q90): Explain-it, compare-it, real-world problems
- [ ] **Tier 5 — Critical Thinking** (Q91–Q100): Predict output, debug, design decisions

---

## Question Type Legend

| Tag | Meaning |
|---|---|
| `[Normal]` | Recall + apply — straightforward concept check |
| `[Thinking]` | Requires reasoning about SQL internals or query behaviour |
| `[Logical]` | Predict output or trace query execution |
| `[Critical]` | Tricky gotcha, edge case, or non-obvious behaviour |
| `[Interview]` | Explain or compare in interview style |
| `[Debug]` | Find and fix the broken query |
| `[Design]` | Schema or query architecture decision |

---

## 🟢 Tier 1 — Basics

---

### Q1 · [Normal] · `sql-vs-nosql`

> **You have a social media app with user profiles, posts, comments, and likes — all heavily related. Your co-worker says "just use MongoDB, it's faster." What do you tell them?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
Use a relational database (PostgreSQL/MySQL). Highly related data with clear structure and integrity requirements (users have posts, posts have comments, comments have authors) is exactly what SQL was built for. MongoDB's flexibility is an advantage when your schema is fluid and relationships are minimal — not here.

**How to think through this:**
1. Identify the data shape: structured, relational, consistent schema → SQL wins
2. Identify the access patterns: joins are needed (posts + authors + comments) → SQL handles this natively
3. "NoSQL is faster" is a myth without context — PostgreSQL at millions of rows is fast with proper indexes

**Key takeaway:** Choose SQL when your data has clear relationships and integrity matters; choose NoSQL when your schema is unstructured or write throughput at massive scale is the constraint.

</details>

📖 **Theory:** [sql-vs-nosql](./01_fundamentals/sql_vs_nosql.md#sql-vs-nosql)


---

### Q2 · [Normal] · `select-basics`

> **Your manager asks you to pull a quick report: every customer's full name, their email, and their account balance multiplied by 1.1 (a 10% loyalty bonus preview). The table is `customers`. How do you write that SELECT?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
```sql
SELECT full_name, email, balance * 1.1 AS projected_balance
FROM customers;
```
**SELECT** lets you choose specific columns instead of `SELECT *` (which returns everything). You can also include **expressions** — arithmetic, string functions, CASE — directly in the SELECT list and alias them with **AS**.

**How to think through this:**
1. Name only the columns you need — avoids pulling unnecessary data over the wire
2. Expressions like `balance * 1.1` are evaluated per row at query time — no stored column needed
3. `AS projected_balance` gives the computed column a readable name in the result set

**Key takeaway:** SELECT is a projection — pick your columns, compute what you need, alias for clarity; `SELECT *` is fine for exploration but harmful in production queries.

</details>

📖 **Theory:** [select-basics](./02_querying_basics/select_and_from.md#select-and-from)


---

### Q3 · [Thinking] · `execution-order`

> **A junior engineer writes a query with a WHERE clause that references an alias defined in SELECT — and gets an error. They're confused because SELECT "comes before" WHERE visually. How do you explain what's actually happening?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
SQL has a **logical execution order** that is different from the written order. The engine processes clauses in this sequence: `FROM` → `WHERE` → `GROUP BY` → `HAVING` → `SELECT` → `ORDER BY`. Because WHERE runs before SELECT, the alias defined in SELECT doesn't exist yet when WHERE is evaluated — hence the error.

**How to think through this:**
1. Think of it as a pipeline: FROM builds the table, WHERE filters rows, GROUP BY collapses them, HAVING filters groups, SELECT projects columns, ORDER BY sorts the final result
2. Aliases defined in SELECT are only "born" at the SELECT step — anything running earlier (WHERE, GROUP BY, HAVING) cannot see them
3. The fix: repeat the expression in WHERE, or wrap the query in a subquery/CTE so the alias becomes a real column in the outer query

**Key takeaway:** SQL's written order is for readability, not execution — internalize `FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY` and alias errors become instantly obvious.

</details>

📖 **Theory:** [execution-order](./02_querying_basics/select_and_from.md#select-and-from)


---

### Q4 · [Normal] · `where-filtering`

> **You're querying an `orders` table and need: all orders placed in 2024, with a total above $500, that are NOT in 'cancelled' status. Write the WHERE clause and name every operator you used.**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
```sql
WHERE EXTRACT(YEAR FROM order_date) = 2024
  AND total > 500
  AND status != 'cancelled'
```
Operators used: `=` (equality), `>` (greater than), `!=` (not equal). SQL's **comparison operators** are `=`, `!=` (or `<>`), `<`, `>`, `<=`, `>=`.

**How to think through this:**
1. Each condition in WHERE is a boolean expression — the row is included only if ALL conditions evaluate to TRUE
2. `!=` and `<>` are equivalent; `<>` is the ANSI standard but `!=` is more readable and supported everywhere
3. Filtering on a function like `EXTRACT(YEAR FROM order_date)` prevents index use — for performance, prefer `order_date >= '2024-01-01' AND order_date < '2025-01-01'`

**Key takeaway:** WHERE conditions are AND'd together by default — each narrows the result set further, and wrapping column expressions in functions silently kills index performance.

</details>

📖 **Theory:** [where-filtering](./02_querying_basics/where_and_filtering.md#where-and-filtering)


---

### Q5 · [Normal] · `between-in-like`

> **You need to find all products where the price is between $10 and $50, the category is one of ('Electronics', 'Books', 'Toys'), and the product name starts with "Pro". Write the WHERE clause using BETWEEN, IN, and LIKE.**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
```sql
WHERE price BETWEEN 10 AND 50
  AND category IN ('Electronics', 'Books', 'Toys')
  AND product_name LIKE 'Pro%'
```
**BETWEEN** is inclusive on both ends (equivalent to `>= 10 AND <= 50`). **IN** checks membership in a list. **LIKE** matches patterns: `%` means zero or more characters, `_` means exactly one character.

**How to think through this:**
1. `BETWEEN 10 AND 50` is shorthand — always remember both bounds are inclusive, which can surprise you with date ranges
2. `IN (...)` is cleaner than chaining multiple `OR` conditions and performs the same way
3. `LIKE 'Pro%'` can use an index (leading wildcard `%Pro%` cannot) — pattern position matters for performance

**Key takeaway:** BETWEEN, IN, and LIKE are readability shortcuts for common filter patterns — BETWEEN is inclusive, and LIKE only uses indexes when the wildcard is at the end.

</details>

📖 **Theory:** [between-in-like](./02_querying_basics/where_and_filtering.md#between--range-filtering)


---

### Q6 · [Critical] · `is-null`

> **A data analyst writes `WHERE phone_number = NULL` to find customers with no phone on file. The query returns zero rows even though you can clearly see NULLs in the table. What's wrong and how do you fix it?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
`= NULL` never works. **NULL** represents the absence of a value — it is not a value itself. Any comparison using `=` with NULL yields NULL (not TRUE or FALSE), so no rows pass the WHERE filter. The correct syntax is `WHERE phone_number IS NULL`.

**How to think through this:**
1. In SQL's three-valued logic (TRUE / FALSE / NULL), `NULL = NULL` evaluates to NULL, not TRUE
2. WHERE only lets through rows where the condition is TRUE — NULL-result rows are silently dropped
3. Use `IS NULL` to check for absence of value; use `IS NOT NULL` to check for presence; never use `=` or `!=` with NULL

**Key takeaway:** NULL is not a value — it is unknown; `= NULL` always silently returns nothing, and the only correct test is `IS NULL` / `IS NOT NULL`.

</details>

📖 **Theory:** [is-null](./02_querying_basics/where_and_filtering.md#is-null-and-is-not-null)


---

### Q7 · [Critical] · `null-handling`

> **What does this query return — and why is it surprising?**

```sql
SELECT * FROM employees WHERE department != 'Engineering';
```

<details>
<summary>💡 Show Answer</summary>

**Answer:**
It returns all rows where department is NOT 'Engineering' — but it **silently excludes rows where department IS NULL**. `NULL != 'Engineering'` evaluates to NULL (not TRUE), so those rows are filtered out.

**How to think through this:**
1. In SQL, any comparison with NULL yields NULL, not TRUE or FALSE
2. WHERE only includes rows where the condition evaluates to TRUE
3. To include NULLs: `WHERE department != 'Engineering' OR department IS NULL`

**Key takeaway:** NULL is not a value — it is the absence of a value; never use `=` or `!=` with NULL, always use `IS NULL` / `IS NOT NULL`.

</details>

📖 **Theory:** [null-handling](./02_querying_basics/where_and_filtering.md#is-null-and-is-not-null)


---

### Q8 · [Thinking] · `and-or-not`

> **A query is supposed to return VIP customers OR customers who spent over $1000 AND live in California. The developer writes `WHERE is_vip = true OR total_spent > 1000 AND state = 'CA'` — but the results look wrong. Why, and how do you fix it?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
**AND has higher precedence than OR**, so the query is parsed as `is_vip = true OR (total_spent > 1000 AND state = 'CA')`. This returns all VIP customers regardless of state, plus non-VIP California high-spenders. If the intent was `(is_vip = true OR total_spent > 1000) AND state = 'CA'`, parentheses are required.

**How to think through this:**
1. SQL evaluates NOT first, then AND, then OR — same as most programming languages
2. Without parentheses, `A OR B AND C` is always `A OR (B AND C)` — never assume left-to-right
3. When mixing AND and OR, always use explicit parentheses even if you think you know the precedence — it makes intent clear and prevents bugs

**Key takeaway:** AND binds tighter than OR — always use parentheses when mixing them, because the "obvious" reading and SQL's actual evaluation are frequently different.

</details>

📖 **Theory:** [and-or-not](./02_querying_basics/where_and_filtering.md#and-or-not--combining-conditions)


---

### Q9 · [Normal] · `order-by`

> **You're building a leaderboard: sort users by `score` descending, then by `signup_date` ascending as a tiebreaker, and push any users with a NULL score to the bottom. Write the ORDER BY clause.**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
```sql
ORDER BY score DESC NULLS LAST, signup_date ASC
```
**ORDER BY** accepts multiple columns separated by commas — rows are sorted by the first column, then ties are broken by the second. **NULLS LAST** / **NULLS FIRST** controls where NULLs land (PostgreSQL supports this; MySQL does not natively).

**How to think through this:**
1. Multi-column ORDER BY is applied left to right — the second column only matters when the first column is equal
2. Default NULL position varies by database: in PostgreSQL, NULLs sort as if larger than any value (NULLS LAST for DESC, NULLS FIRST for ASC) — always be explicit
3. MySQL workaround for NULLS LAST: `ORDER BY score IS NULL, score DESC` (NULL IS NULL = 1, non-NULL = 0, so NULLs sort last)

**Key takeaway:** Multi-column ORDER BY breaks ties left to right, and NULL sort position is database-specific — always use NULLS FIRST/LAST explicitly when NULLs matter.

</details>

📖 **Theory:** [order-by](./02_querying_basics/order_by_and_limit.md#order-by-and-limit)


---

### Q10 · [Thinking] · `limit-offset`

> **Your API returns paginated results — page 1 is `LIMIT 20 OFFSET 0`, page 2 is `LIMIT 20 OFFSET 20`. A user complains that page 500 loads extremely slowly even though the table only has 100,000 rows. Why, and what's the better approach?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
**OFFSET** forces the database to scan and discard all preceding rows before returning your page. At `OFFSET 9980`, the engine reads and throws away 9,980 rows every time — it cannot jump directly to row 9,981. This makes deep pagination O(n) expensive. The better approach is **keyset pagination** (also called cursor-based): `WHERE id > :last_seen_id ORDER BY id LIMIT 20`.

**How to think through this:**
1. OFFSET is not a pointer — the database has no bookmark; it re-scans from the start on every request
2. Keyset pagination uses an indexed column (like `id` or `created_at`) as a cursor — the engine seeks directly to that value using the index, making every page equally fast
3. Trade-off: keyset pagination cannot jump to an arbitrary page number, only "next" — acceptable for infinite scroll, not for "go to page 47"

**Key takeaway:** LIMIT/OFFSET pagination degrades linearly with page depth — for large tables or deep pages, use keyset (cursor) pagination with an indexed column instead.

</details>

📖 **Theory:** [limit-offset](./02_querying_basics/order_by_and_limit.md#order-by-and-limit)


---

### Q11 · [Normal] · `distinct`

> **Your analytics query returns duplicate customer IDs because a customer can appear in multiple orders. You need: the count of unique customers, the unique cities they're from, and the unique (customer_id, city) pairs. How does DISTINCT handle each case?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
- Unique customer count: `SELECT COUNT(DISTINCT customer_id) FROM orders`
- Unique cities: `SELECT DISTINCT city FROM orders`
- Unique pairs: `SELECT DISTINCT customer_id, city FROM orders`

**SELECT DISTINCT** removes duplicate rows from the result. When applied to multiple columns, uniqueness is evaluated across the entire row — not per column. `COUNT(DISTINCT column)` counts unique non-NULL values within an aggregate.

**How to think through this:**
1. `DISTINCT` operates on the full output row — `SELECT DISTINCT a, b` deduplicates on the combination of (a, b), not separately
2. `COUNT(DISTINCT col)` and `SELECT DISTINCT` are different tools: one collapses to a count, the other returns the distinct rows themselves
3. DISTINCT has a sorting/hashing cost — if you're using it to paper over a JOIN that produces duplicates, fix the JOIN instead

**Key takeaway:** DISTINCT deduplicates on the full selected row; `COUNT(DISTINCT col)` counts unique non-NULL values — and if DISTINCT feels necessary, check whether a bad JOIN is the real problem.

</details>

📖 **Theory:** [distinct](./02_querying_basics/distinct_and_aliases.md#distinct-and-aliases)


---

### Q12 · [Normal] · `aliases`

> **You're writing a report query joining `customers` and `orders`. The column names are long, the table names repeat everywhere, and a computed column has no name at all. Show how aliases clean this up — and note when AS is optional.**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
```sql
SELECT
    c.full_name          AS customer_name,
    o.created_at         AS order_date,
    o.total * 0.9        AS discounted_total   -- ← computed column needs a name
FROM customers AS c                            -- ← table alias shortens repeated references
JOIN orders    AS o ON c.id = o.customer_id;
```
**AS** is the keyword for aliasing — but it is optional in most databases. `customers c` and `customers AS c` are equivalent. Column aliases defined here are available in ORDER BY but not in WHERE or GROUP BY (execution order).

**How to think through this:**
1. Table aliases are most valuable in JOINs — typing `c.` instead of `customers.` throughout a long query reduces noise significantly
2. Column aliases rename the output column for readability or to give computed expressions a usable name
3. AS is syntactically optional but always include it for column aliases — skipping it on table aliases is common and fine

**Key takeaway:** Aliases exist for readability and brevity — always alias computed columns (they have no name otherwise), and remember column aliases are invisible to WHERE and GROUP BY due to execution order.

</details>

📖 **Theory:** [aliases](./02_querying_basics/distinct_and_aliases.md#distinct-and-aliases)


---

### Q13 · [Critical] · `count`

> **Your team has two queries on the same `signups` table: `SELECT COUNT(*) FROM signups` returns 10,000. `SELECT COUNT(referral_code) FROM signups` returns 6,432. The table hasn't changed. Why are they different?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
**`COUNT(*)`** counts every row regardless of content — NULLs included. **`COUNT(column)`** counts only rows where that column is NOT NULL. The difference (3,568) is the number of signups where `referral_code` is NULL — users who signed up without a referral.

**How to think through this:**
1. `COUNT(*)` is a row counter — it doesn't look at any column values, just whether the row exists
2. `COUNT(column)` is a non-NULL value counter — it silently skips NULL values in that column
3. This distinction matters for data quality checks: `COUNT(*) - COUNT(col)` tells you exactly how many NULLs exist in a column

**Key takeaway:** `COUNT(*)` counts rows; `COUNT(column)` counts non-NULL values in that column — the difference reveals your NULL count, which is a useful data quality signal.

</details>

📖 **Theory:** [count](./03_aggregation/aggregate_functions.md#the-accountants-monthly-report)


---

### Q14 · [Normal] · `sum-avg-min-max`

> **You're computing sales metrics: total revenue, average order value, smallest order, and largest order from an `orders` table. Write the query — and explain what happens if some `total` values are NULL.**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
```sql
SELECT
    SUM(total)  AS total_revenue,
    AVG(total)  AS avg_order_value,
    MIN(total)  AS smallest_order,
    MAX(total)  AS largest_order
FROM orders;
```
**SUM**, **AVG**, **MIN**, and **MAX** all ignore NULL values. AVG divides by the count of non-NULL rows — so a NULL total is excluded from both the sum and the denominator, which can make the average higher than expected.

**How to think through this:**
1. All four aggregate functions silently skip NULLs — they operate only on rows where the column has a value
2. AVG(total) is equivalent to SUM(total) / COUNT(total) — NOT SUM(total) / COUNT(*), because COUNT(*) would include NULL rows in the denominator
3. To include NULLs as zero in SUM/AVG, use `COALESCE(total, 0)` — but do this intentionally, not as a default

**Key takeaway:** SUM, AVG, MIN, and MAX ignore NULLs — for AVG this means the denominator is the non-NULL count, which can skew results if NULLs represent "zero" in your domain.

</details>

📖 **Theory:** [sum-avg-min-max](./03_aggregation/aggregate_functions.md#min-and-max--the-extremes)


---

### Q15 · [Thinking] · `null-in-aggregates`

> **A finance team says their `AVG(transaction_amount)` is higher than expected. You suspect NULL values are distorting it. Explain exactly how NULL affects AVG — and write a query to expose the problem.**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
AVG ignores NULLs in both the numerator and denominator. If 200 of 1,000 rows have `transaction_amount = NULL`, AVG divides by 800, not 1,000. If those NULLs should be treated as zero (e.g., no transaction occurred), the average is artificially inflated.

Diagnostic query:
```sql
SELECT
    COUNT(*)                          AS total_rows,
    COUNT(transaction_amount)         AS non_null_rows,        -- ← count of non-NULL values
    COUNT(*) - COUNT(transaction_amount) AS null_rows,         -- ← how many are NULL
    AVG(transaction_amount)           AS avg_ignoring_nulls,
    AVG(COALESCE(transaction_amount, 0)) AS avg_treating_null_as_zero
FROM transactions;
```

**How to think through this:**
1. The gap between `COUNT(*)` and `COUNT(column)` is the NULL count — always run this check before trusting aggregates
2. Decide whether NULL means "unknown" (exclude it) or "zero" (replace with COALESCE) — that's a business logic decision, not a SQL decision
3. AVG(COALESCE(col, 0)) lowers the average; AVG(col) ignores NULLs entirely — pick the one that reflects reality

**Key takeaway:** All aggregate functions except `COUNT(*)` ignore NULLs — always audit `COUNT(*) vs COUNT(col)` before presenting aggregate results to stakeholders.

</details>

📖 **Theory:** [null-in-aggregates](./03_aggregation/aggregate_functions.md#visualising-the-null-difference)


---

### Q16 · [Critical] · `group-by`

> **A developer writes this query to get total sales per region and gets an error: `column "sales_rep_name" must appear in the GROUP BY clause or be used in an aggregate function`. They're confused — why can't they just SELECT it?**

```sql
SELECT region, sales_rep_name, SUM(amount) AS total
FROM sales
GROUP BY region;
```

<details>
<summary>💡 Show Answer</summary>

**Answer:**
**GROUP BY** collapses multiple rows into one per group. After grouping by `region`, a single output row might represent 50 sales rows — each potentially having a different `sales_rep_name`. The database doesn't know which name to display for that group, so it errors rather than silently pick one. Every column in SELECT must either be in GROUP BY or wrapped in an aggregate function.

**How to think through this:**
1. Think of GROUP BY as "one output row per unique combination of these columns" — any other selected column must be reducible to a single value for that group
2. Fix options: add `sales_rep_name` to GROUP BY (groups by region + rep), use `MAX(sales_rep_name)` (arbitrary pick), or remove it from SELECT
3. MySQL's `ONLY_FULL_GROUP_BY` mode enforces this; older MySQL (and SQLite) silently allowed it by picking an arbitrary value — a dangerous behavior

**Key takeaway:** Every non-aggregated column in SELECT must appear in GROUP BY — if it doesn't, the database can't determine which value to show for the collapsed group.

</details>

📖 **Theory:** [group-by](./03_aggregation/group_by_and_having.md#group-by-and-having)


---

### Q17 · [Thinking] · `having-vs-where`

> **You want regions where total sales exceed $50,000 — but only counting orders from 2024. A teammate writes `HAVING order_year = 2024 AND SUM(amount) > 50000`. You spot a bug. What is it?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
`order_year = 2024` belongs in **WHERE**, not **HAVING**. WHERE filters individual rows before grouping — so only 2024 rows ever enter the aggregation. HAVING filters groups after aggregation — putting a row-level filter there means the database first groups ALL years together and then tries to filter by year at the group level, which produces wrong results (or an error, depending on the database).

**How to think through this:**
1. Execution order: WHERE runs first (filters rows) → GROUP BY (builds groups) → HAVING (filters groups) — put the condition at the right stage
2. Rule of thumb: if the filter references an aggregate function (`SUM`, `COUNT`, etc.) → HAVING; if it references a raw column value → WHERE
3. Correct query: `WHERE order_year = 2024` + `HAVING SUM(amount) > 50000` — the year filter cuts rows early, making the aggregation faster too

**Key takeaway:** WHERE filters rows before grouping; HAVING filters groups after aggregation — mixing them up produces wrong answers, and putting row filters in WHERE also improves performance.

</details>

📖 **Theory:** [having-vs-where](./03_aggregation/group_by_and_having.md#where-vs-having--side-by-side)


---

### Q18 · [Interview] · `window-functions-intro`

> **An interviewer asks: "What is a window function and how is it different from GROUP BY?" Give a clear, precise answer with a concrete example.**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
A **window function** performs a calculation across a set of rows related to the current row — but unlike GROUP BY, it does NOT collapse those rows into one. Every row remains in the result set; the function just adds a computed column alongside it. The `OVER()` clause defines the "window" of rows the function sees.

Example: rank each employee by salary within their department, while keeping all employee rows:
```sql
SELECT name, department, salary,
       RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank
FROM employees;
```
GROUP BY would give you one row per department. The window function gives you one row per employee, with their rank attached.

**How to think through this:**
1. GROUP BY aggregates and collapses — you lose the individual rows
2. Window functions aggregate but preserve — every original row survives with a new computed column appended
3. Common use cases: rankings, running totals, moving averages, comparing each row to its group's aggregate

**Key takeaway:** Window functions compute across rows without collapsing them — use GROUP BY when you want summary rows, use window functions when you want per-row context alongside the aggregate.

</details>

📖 **Theory:** [window-functions-intro](./03_aggregation/window_functions.md#window-functions)


---

### Q19 · [Thinking] · `partition-by`

> **You have a sales table with `salesperson`, `region`, and `amount`. You want each row to show the salesperson's amount AND the total for their region — without losing any rows. How does PARTITION BY make this possible?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
**PARTITION BY** divides the rows into groups (partitions) for the window function — similar to GROUP BY, but without collapsing. Each row's window function sees only the rows in its partition.

```sql
SELECT
    salesperson,
    region,
    amount,
    SUM(amount) OVER (PARTITION BY region) AS region_total  -- ← each row gets its region's total
FROM sales;
```
Every row is returned. For a salesperson in the West region, `region_total` shows the sum of all West rows — computed in the background, attached to each row individually.

**How to think through this:**
1. Think of PARTITION BY as "reset the window for each group" — each partition is an independent calculation scope
2. Without PARTITION BY, `SUM(amount) OVER ()` uses the entire table as one window — you'd get the grand total on every row
3. You can combine PARTITION BY with ORDER BY inside OVER() — this creates a cumulative (running) total within each partition

**Key takeaway:** PARTITION BY is GROUP BY for window functions — it scopes the calculation to a subset of rows while keeping every original row visible in the output.

</details>

📖 **Theory:** [partition-by](./03_aggregation/window_functions.md#partition-by--splitting-into-groups-without-collapsing)


---

### Q20 · [Interview] · `row-number-rank-dense-rank`

> **Three colleagues with scores 100, 100, 90 are being ranked. What rank does each function assign — and when would you use each one in a real product?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
Given scores: Alice=100, Bob=100, Carol=90:

| Function | Alice | Bob | Carol | Gap? |
|---|---|---|---|---|
| `ROW_NUMBER()` | 1 | 2 | 3 | No ties — arbitrary |
| `RANK()` | 1 | 1 | 3 | Skips 2 |
| `DENSE_RANK()` | 1 | 1 | 2 | No skip |

**ROW_NUMBER** assigns unique sequential integers — no ties possible. **RANK** matches sports rankings (two gold medals → no silver). **DENSE_RANK** doesn't skip numbers after a tie.

**How to think through this:**
1. ROW_NUMBER: use when you need exactly one row per rank, like "top 1 order per customer" — ties broken arbitrarily
2. RANK: use for competitive ranking where skipping is meaningful ("they tied for 1st, so there's no 2nd place")
3. DENSE_RANK: use when you want continuous rank numbers and ties shouldn't create gaps — common in percentile bands or tier assignments

**Key takeaway:** ROW_NUMBER never ties, RANK ties and skips, DENSE_RANK ties and never skips — pick based on whether gaps in the ranking sequence are meaningful in your domain.

</details>

📖 **Theory:** [row-number-rank-dense-rank](./03_aggregation/window_functions.md#row_number-rank-dense_rank)


---

### Q21 · [Normal] · `lag-lead`

> **You have a `daily_revenue` table with `date` and `revenue` columns. Write a query that shows each day's revenue alongside the previous day's revenue, so stakeholders can see the day-over-day change in one row.**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
```sql
SELECT
    date,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY date)  AS prev_day_revenue,       -- ← previous row's value
    revenue - LAG(revenue, 1) OVER (ORDER BY date) AS day_over_day   -- ← difference
FROM daily_revenue
ORDER BY date;
```
**LAG(col, n)** looks back n rows in the window order. **LEAD(col, n)** looks forward. The first row's `prev_day_revenue` will be NULL (no prior row exists).

**How to think through this:**
1. LAG/LEAD are window functions — they need an OVER clause with ORDER BY to define "previous" and "next"
2. The second argument to LAG is the offset (default 1); the optional third argument is the default value when the lookback goes out of bounds: `LAG(revenue, 1, 0)`
3. Use LEAD when you need the next row's value (e.g., showing what comes next in a scheduled task list)

**Key takeaway:** LAG looks back, LEAD looks forward — both need ORDER BY in the OVER clause to define sequence, and they return NULL at boundaries unless you provide a default.

</details>

📖 **Theory:** [lag-lead](./03_aggregation/window_functions.md#lag-and-lead--looking-at-neighbouring-rows)


---

### Q22 · [Logical] · `running-total`

> **What does this query produce — trace through it row by row.**

```sql
SELECT
    order_date,
    amount,
    SUM(amount) OVER (ORDER BY order_date) AS running_total
FROM orders
ORDER BY order_date;
```

<details>
<summary>💡 Show Answer</summary>

**Answer:**
Each row's `running_total` is the cumulative sum of `amount` from the first row up to and including the current row (by `order_date` order). If the data is:

| order_date | amount |
|---|---|
| 2024-01-01 | 100 |
| 2024-01-02 | 200 |
| 2024-01-03 | 150 |

The result is:

| order_date | amount | running_total |
|---|---|---|
| 2024-01-01 | 100 | 100 |
| 2024-01-02 | 200 | 300 |
| 2024-01-03 | 150 | 450 |

**How to think through this:**
1. `SUM() OVER (ORDER BY col)` uses the default frame: `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` — all rows from the start up to the current one
2. The frame grows with each row — that's what makes it a running (cumulative) total
3. Ties in `order_date` will sum all tied rows together at the same step — add a tiebreaker like `ORDER BY order_date, order_id` to get strictly sequential accumulation

**Key takeaway:** `SUM() OVER (ORDER BY col)` computes a running total by expanding the frame one row at a time — the default window frame is unbounded preceding to current row.

</details>

📖 **Theory:** [running-total](./03_aggregation/window_functions.md#running-totals-with-order-by-in-the-window)


---

### Q23 · [Normal] · `inner-join`

> **You have a `customers` table and an `orders` table. You want a list of every customer who has placed at least one order, along with their order details. Which join do you use — and what happens to customers with no orders?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
Use **INNER JOIN**. It returns only rows where the join condition matches in both tables — customers with no orders are excluded entirely.

```sql
SELECT c.name, o.order_id, o.total
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id;  -- ← only rows matched on both sides
```
If a customer has 3 orders, they appear 3 times in the result. If a customer has 0 orders, they don't appear at all.

**How to think through this:**
1. INNER JOIN is an intersection — it only returns the overlap between two tables based on the join condition
2. A customer with multiple orders is "fan'd out" — they appear once per matching order row
3. If you need customers with no orders too, switch to LEFT JOIN

**Key takeaway:** INNER JOIN returns only matched rows from both tables — unmatched rows on either side are silently excluded, which is the most common source of "missing data" bugs in JOIN queries.

</details>

📖 **Theory:** [inner-join](./05_joins/inner_join.md#inner-join--the-matchmaker)


---

### Q24 · [Thinking] · `left-join`

> **Your marketing team wants a list of ALL customers — including those who have never ordered — to identify inactive accounts. An INNER JOIN misses them. What join do you use, and how do you identify the "never ordered" customers in the result?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
Use a **LEFT JOIN**. It returns all rows from the left table (customers) and matching rows from the right table (orders). Where no match exists, the right-side columns are NULL.

```sql
SELECT c.name, o.order_id
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id;  -- ← all customers, NULLs where no order exists
```
To find inactive customers specifically: `WHERE o.order_id IS NULL` — those are the customers with no matching order row.

**How to think through this:**
1. LEFT JOIN = "give me everything from the left table, fill in the right side where it matches, NULL where it doesn't"
2. The NULL on the right side is the signal — if `o.order_id IS NULL`, that customer has no orders
3. This pattern (LEFT JOIN + WHERE right.id IS NULL) is called an anti-join — a very common interview pattern

**Key takeaway:** LEFT JOIN preserves all left-table rows and fills unmatched right-side columns with NULL — filter on `right.id IS NULL` to isolate rows with no match.

</details>

📖 **Theory:** [left-join](./05_joins/left_right_outer_join.md#left-right-and-full-outer-join--the-inclusive-matchmaker)


---

### Q25 · [Interview] · `full-outer-join`

> **When would you actually use a FULL OUTER JOIN? Give a real-world scenario where neither LEFT nor RIGHT JOIN would give you the complete picture.**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
Use **FULL OUTER JOIN** when you need all rows from both tables — including unmatched rows from each side. A real-world example: reconciling two payment systems. System A has its transaction log; System B has its own. You want every transaction from both systems in one result — including transactions that exist in A but not B (possible missed sync) and transactions in B but not A.

```sql
SELECT a.txn_id AS a_txn, b.txn_id AS b_txn, a.amount, b.amount
FROM system_a a
FULL OUTER JOIN system_b b ON a.txn_id = b.txn_id;
-- rows where a_txn IS NULL: exist in B only
-- rows where b_txn IS NULL: exist in A only
-- rows where both non-null: matched in both
```

**How to think through this:**
1. LEFT JOIN gives you all of A + matched B. RIGHT JOIN gives you all of B + matched A. FULL OUTER gives you the union of both
2. MySQL doesn't support FULL OUTER JOIN natively — emulate it with `LEFT JOIN UNION ALL RIGHT JOIN WHERE left.id IS NULL`
3. Use it for data reconciliation, audit comparisons, or merging two partially overlapping datasets

**Key takeaway:** FULL OUTER JOIN is for reconciliation scenarios where you need to see what's missing on either side — both "A without B" and "B without A" in one query.

</details>

📖 **Theory:** [full-outer-join](./05_joins/left_right_outer_join.md#left-right-and-full-outer-join--the-inclusive-matchmaker)


---

### Q26 · [Critical] · `cross-join`

> **A clothing store has 4 sizes and 6 colors. A developer wants to generate every possible size/color combination for a product catalog. They use a CROSS JOIN and it works perfectly. But then they accidentally CROSS JOIN two 10,000-row tables. What happens?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
A **CROSS JOIN** produces a **Cartesian product** — every row from the left table paired with every row from the right table. 4 sizes × 6 colors = 24 rows (correct and useful). But 10,000 rows × 10,000 rows = **100,000,000 rows** — likely crashing the query or consuming all available memory.

```sql
-- Intentional: generate all size/color combinations
SELECT s.size, c.color
FROM sizes s
CROSS JOIN colors c;  -- ← 4 × 6 = 24 rows, perfectly fine
```

**How to think through this:**
1. CROSS JOIN is the only join with no ON condition — it's intentional by design, making it hard to accidentally write unless you truly mean it
2. Accidental Cartesian products often happen with implicit joins in old-style SQL: `FROM a, b WHERE ...` — if you forget the WHERE condition, you get a cross join
3. Always check row counts before CROSS JOINing — if either table is large, the result explodes quadratically

**Key takeaway:** CROSS JOIN is powerful for small combination tables (sizes × colors, dates × categories) and catastrophic for large ones — always compute left_rows × right_rows before running it.

</details>

📖 **Theory:** [cross-join](./05_joins/cross_and_self_join.md#cross-join-and-self-join--every-combination-and-self-reference)


---

### Q27 · [Normal] · `self-join`

> **Your `employees` table has an `id` column and a `manager_id` column (which references another employee's `id`). Write a query that shows each employee's name alongside their manager's name.**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
```sql
SELECT
    e.name        AS employee,
    m.name        AS manager
FROM employees e                                       -- ← the employee
LEFT JOIN employees m ON e.manager_id = m.id;         -- ← join same table as the manager
```
A **self-join** joins a table to itself using two different aliases. LEFT JOIN is used here so the CEO (who has no manager, `manager_id IS NULL`) still appears in the results with a NULL manager name.

**How to think through this:**
1. Give the table two different aliases — one for the "child" role (employee), one for the "parent" role (manager)
2. The join condition links the child's foreign key (`manager_id`) to the parent's primary key (`id`)
3. Use LEFT JOIN unless you intentionally want to exclude the top-level node (the one with no manager)

**Key takeaway:** Self-joins model hierarchical relationships by treating the same table as two different entities — always use distinct aliases and think carefully about whether the root node should appear in results.

</details>

📖 **Theory:** [self-join](./05_joins/cross_and_self_join.md#cross-join-and-self-join--every-combination-and-self-reference)


---

### Q28 · [Thinking] · `anti-join`

> **You want to find all customers who have NEVER placed an order. A subquery with `NOT IN` works but a colleague says it breaks silently when the subquery contains NULLs. Show both approaches and explain the NULL trap.**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
The **anti-join** pattern using LEFT JOIN is safer:

```sql
-- Safe: LEFT JOIN anti-join
SELECT c.*
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.customer_id IS NULL;                          -- ← no match = never ordered

-- Dangerous: NOT IN with potential NULLs
SELECT * FROM customers
WHERE id NOT IN (SELECT customer_id FROM orders);     -- ← breaks if any customer_id in orders is NULL
```

If any `customer_id` in the orders subquery is NULL, `NOT IN` evaluates `id != NULL` which yields NULL (not TRUE) — causing the entire NOT IN to return no rows at all, silently.

**How to think through this:**
1. `NOT IN (1, 2, NULL)` expands to `id != 1 AND id != 2 AND id != NULL` — that last condition is NULL, making the whole expression NULL, filtering out every row
2. The LEFT JOIN approach doesn't have this problem — it uses `IS NULL` which is safe
3. `NOT EXISTS` is another safe alternative: `WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id)`

**Key takeaway:** Anti-join via `LEFT JOIN ... WHERE right.id IS NULL` is always safe; `NOT IN` silently returns no rows if the subquery contains any NULL — use NOT EXISTS or LEFT JOIN instead.

</details>

📖 **Theory:** [anti-join](./05_joins/join_patterns.md#pattern-1--find-unmatched-rows-anti-join)


---

### Q29 · [Design] · `junction-table`

> **You're designing a schema for a course platform: students can enroll in many courses, and each course can have many students. How do you model this relationship — and what columns belong in the junction table beyond just the two foreign keys?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
Model it with a **junction table** (also called a bridge or associative table) that sits between `students` and `courses`:

```sql
CREATE TABLE enrollments (
    student_id  INT REFERENCES students(id),
    course_id   INT REFERENCES courses(id),
    enrolled_at TIMESTAMPTZ DEFAULT NOW(),    -- ← when the relationship was created
    status      VARCHAR(20) DEFAULT 'active', -- ← relationship-level attribute
    PRIMARY KEY (student_id, course_id)       -- ← composite PK prevents duplicate enrollments
);
```

The junction table breaks the many-to-many into two one-to-many relationships. Beyond the two foreign keys, include attributes that belong to the relationship itself — enrollment date, status, grade, progress — not to either entity alone.

**How to think through this:**
1. Many-to-many cannot be stored in either parent table — a student row can't hold a list of course IDs in one column (that violates first normal form)
2. The composite primary key `(student_id, course_id)` enforces that a student can only enroll in each course once
3. Relationship-level data (grade, enrollment date) belongs here, not on the student or course table

**Key takeaway:** Junction tables resolve many-to-many relationships by becoming their own entity — and they often deserve their own attributes representing the state of that relationship.

</details>

📖 **Theory:** [junction-table](./05_joins/join_patterns.md#pattern-2--many-to-many-via-a-junction-table)


---

### Q30 · [Normal] · `primary-key`

> **A new developer on your team creates a `sessions` table with no PRIMARY KEY because "it's just log data." Six months later, deduplication becomes impossible and a JOIN produces thousands of phantom rows. What should they have done, and when is a composite PK the right choice?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
Every table should have a **PRIMARY KEY** — a column or combination of columns that uniquely identifies each row. For a sessions table, a surrogate key (`session_id SERIAL PRIMARY KEY`) or a natural composite key (`(user_id, session_start)`) would have prevented duplicate rows entirely.

A **composite primary key** is right when no single column is unique on its own but the combination is. Example: `enrollments(student_id, course_id)` — neither column alone is unique, but the pair is.

**How to think through this:**
1. Without a PK, there's no guaranteed way to reference or deduplicate a specific row — `DELETE` without a unique identifier deletes all matching rows
2. A surrogate key (auto-increment `id`) is simple and always unique — use it when no natural unique column exists
3. Composite PKs are semantically richer — they encode the business rule ("a student can enroll once per course") directly in the schema

**Key takeaway:** Every table needs a primary key — it is the only reliable way to uniquely identify, update, delete, or reference a specific row.

</details>

📖 **Theory:** [primary-key](./04_schema_design/tables_and_constraints.md#primary-key)


---

### Q31 · [Thinking] · `foreign-key`

> **You have `orders.customer_id` that references `customers.id`. What does FOREIGN KEY actually enforce — and what happens when you try to delete a customer who still has orders, with and without ON DELETE CASCADE?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
A **FOREIGN KEY** enforces **referential integrity** — it guarantees that a value in the child table (`orders.customer_id`) always corresponds to an existing row in the parent table (`customers.id`). You can never insert an order for a customer that doesn't exist.

Without `ON DELETE CASCADE`: deleting a customer who has orders raises a foreign key violation error — the database refuses to leave orphaned order rows.

With `ON DELETE CASCADE`: deleting a customer automatically deletes all their orders too. The cascade propagates the delete down the chain.

**How to think through this:**
1. Foreign keys prevent orphan rows — without them, `customer_id = 9999` could silently reference a deleted customer, breaking JOIN queries
2. `ON DELETE CASCADE` is powerful but dangerous — one accidental DELETE of a parent row wipes all children; use it only when child data is truly meaningless without the parent
3. Alternatives: `ON DELETE SET NULL` (child's FK becomes NULL), `ON DELETE RESTRICT` (default — block the delete)

**Key takeaway:** Foreign keys enforce that relationships in your data are valid; ON DELETE CASCADE deletes children automatically — use it only when child rows have no meaning without their parent.

</details>

📖 **Theory:** [foreign-key](./04_schema_design/tables_and_constraints.md#foreign-key-referential-integrity)


---

### Q32 · [Normal] · `constraints`

> **You're defining a `users` table. The email must always be present and unique, the age must be 18 or older, and new users should default to 'active' status. Which constraint handles each rule?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
```sql
CREATE TABLE users (
    id         SERIAL PRIMARY KEY,
    email      VARCHAR(255) NOT NULL UNIQUE,    -- ← NOT NULL: must exist; UNIQUE: no duplicates
    age        INT          CHECK (age >= 18),  -- ← CHECK: enforces a condition on the value
    status     VARCHAR(20)  DEFAULT 'active'    -- ← DEFAULT: fills value if not provided
);
```

- **NOT NULL**: column must have a value — inserting NULL raises an error
- **UNIQUE**: no two rows can share this value (NULLs are typically exempt — multiple NULLs allowed)
- **CHECK**: any boolean expression — row is rejected if it evaluates to FALSE
- **DEFAULT**: used when INSERT omits the column — does not prevent explicit NULL insertion

**How to think through this:**
1. NOT NULL and DEFAULT work together: DEFAULT provides a fallback value; NOT NULL ensures even the fallback can't be bypassed by explicit `NULL`
2. UNIQUE allows multiple NULLs in most databases (NULL != NULL) — combine with NOT NULL if you want exactly one NULL allowed
3. CHECK constraints can reference multiple columns: `CHECK (end_date > start_date)` — useful for business rules that span columns

**Key takeaway:** Constraints encode business rules at the database level — NOT NULL, UNIQUE, CHECK, and DEFAULT are your first line of defense against bad data, running before any application code.

</details>

📖 **Theory:** [constraints](./04_schema_design/tables_and_constraints.md#tables-and-constraints)


---

### Q33 · [Interview] · `data-types`

> **A developer asks: when should I use VARCHAR vs TEXT, CHAR vs VARCHAR, INT vs NUMERIC, and TIMESTAMP vs TIMESTAMPTZ? Give a practical answer for each pair.**

<details>
<summary>💡 Show Answer</summary>

**Answer:**

**VARCHAR(n) vs TEXT:**
- `VARCHAR(n)` enforces a max length at the DB level — use it when length is a business rule (e.g., `VARCHAR(255)` for emails)
- `TEXT` has no length limit — use it for open-ended content (descriptions, blog posts, JSON blobs)
- In PostgreSQL, performance is identical; the difference is purely semantic constraint

**CHAR(n) vs VARCHAR(n):**
- `CHAR(n)` is fixed-width — always pads to n characters with spaces. Use only for truly fixed-length codes (ISO country codes `CHAR(2)`, `CHAR(10)` product SKUs)
- `VARCHAR(n)` is variable-width — stores only what's needed. Use for almost everything else
- CHAR is rarely the right choice in modern databases

**INT vs NUMERIC:**
- `INT` (or `BIGINT`) stores whole numbers with no decimals — use for counts, IDs, quantities
- `NUMERIC(p, s)` stores exact decimal numbers — use for money, measurements, anything where floating-point rounding is unacceptable (`NUMERIC(10, 2)` for dollars and cents)
- Never store currency in `FLOAT` — binary floating-point cannot represent 0.10 exactly

**TIMESTAMP vs TIMESTAMPTZ:**
- `TIMESTAMP` stores a date and time with no time zone — dangerous for multi-region apps
- `TIMESTAMPTZ` stores the moment in UTC and converts to the session time zone on display — always use this for application timestamps
- Rule: store in UTC (`TIMESTAMPTZ`), display in local time in the application layer

**How to think through this:**
1. Choose the most constrained type that correctly models the data — it documents intent and prevents invalid values
2. Money always deserves NUMERIC, never FLOAT — $0.10 stored as a float can become $0.0999999...
3. TIMESTAMPTZ is almost always right for any timestamp that a human interacts with — TIMESTAMP is for cases where you truly don't want timezone conversion (e.g., storing a "local wall clock time" independent of zone)

**Key takeaway:** Pick the type that encodes your constraints — NUMERIC for money, TIMESTAMPTZ for timestamps, TEXT when length is unbounded, VARCHAR(n) when it's a business rule, and never CHAR unless the data is truly fixed-width.

</details>

📖 **Theory:** [data-types](./04_schema_design/data_types.md#data-types)



---

## 🟡 Tier 2 — Intermediate

---

### Q34 · [Interview] · `normalization-1nf`

> **Your coworker stores multiple phone numbers in a single column like `"555-1234, 555-5678"`. Why does this violate First Normal Form, and how would you fix the schema?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
Storing multiple values in one cell violates **First Normal Form (1NF)**, which requires every column to hold a single, atomic value with no repeating groups.

**How to think through this:**
1. 1NF treats each cell like a single slot in a grid — one fact, one value; the phone column holds two facts crammed into one slot, making it impossible to query or index a specific number cleanly
2. The fix is to move the repeating group into its own table: a `user_phones` table with `(user_id, phone_number)` where each row holds exactly one phone number
3. Edge case: even a column named `phone1`, `phone2`, `phone3` violates 1NF in spirit — that's a horizontal repeating group; the relational answer is always a child table

**Key takeaway:** 1NF means one value per cell and no repeating columns — if you find yourself splitting on commas or adding numbered columns, you need a child table.

</details>

📖 **Theory:** [normalization-1nf](./04_schema_design/normalization.md#normalization)


---

### Q35 · [Interview] · `normalization-2nf`

> **You have a table `order_items(order_id, product_id, product_name, quantity)` with a composite primary key of `(order_id, product_id)`. A colleague says it violates Second Normal Form. Are they right, and why?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
Yes. `product_name` depends only on `product_id`, not on the full composite key `(order_id, product_id)`. That is a **partial dependency**, which violates **Second Normal Form (2NF)**.

**How to think through this:**
1. 2NF requires every non-key column to depend on the *whole* primary key, not just part of it; think of it as: every fact in the row should need *all* the key columns to identify it
2. `product_name` can be determined from `product_id` alone — you don't need `order_id` to know what a product is called, so it's in the wrong table
3. The fix: move `product_name` to a `products(product_id, product_name)` table and keep `order_items` lean with only `(order_id, product_id, quantity)`

**Key takeaway:** 2NF eliminates partial dependencies — any column that can be identified by only *part* of a composite key belongs in a separate table.

</details>

📖 **Theory:** [normalization-2nf](./04_schema_design/normalization.md#normalization)


---

### Q36 · [Interview] · `normalization-3nf`

> **A `employees` table has columns `(employee_id, department_id, department_name)`. It's already in 2NF. Does it satisfy Third Normal Form? Why or why not?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
No. `department_name` depends on `department_id`, which is a non-key column — this is a **transitive dependency**, which violates **Third Normal Form (3NF)**.

**How to think through this:**
1. 3NF says: non-key columns must depend on the primary key, the whole key, and *nothing but* the key; here the chain is `employee_id → department_id → department_name` — the name rides on a middle column, not the key directly
2. This causes update anomalies: rename a department and you must update every employee row; miss one and the data is inconsistent
3. The fix: create a `departments(department_id, department_name)` table; `employees` keeps only `department_id` as a foreign key

**Key takeaway:** 3NF eliminates transitive dependencies — if column C depends on column B which depends on the key, move B and C to their own table.

</details>

📖 **Theory:** [normalization-3nf](./04_schema_design/normalization.md#normalization)


---

### Q37 · [Design] · `denormalization`

> **Your analytics dashboard runs a report joining 6 tables to compute monthly revenue by region. It's timing out in production. A senior engineer suggests denormalizing. What does that mean, and what are the trade-offs?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
**Denormalization** means intentionally storing redundant data — pre-joining or pre-aggregating — to eliminate expensive runtime joins and speed up reads.

**How to think through this:**
1. Normalization optimizes for write correctness (no redundancy, no anomalies); denormalization trades some of that for read speed — you store `region_name` directly in the `orders` table even though it also lives in `regions`, so the dashboard query never needs to join
2. Common patterns: wide flat tables for reporting, pre-computed summary tables (or materialized views), or duplicating foreign key display values alongside the FK
3. The trade-off is real: every write now touches more places to stay consistent; denormalization is the right call for read-heavy analytical workloads where data changes infrequently, but it's the wrong call for transactional systems where correctness is paramount

**Key takeaway:** Denormalize when read performance is the bottleneck and the data doesn't change often — but always document the redundancy and enforce consistency in your write path.

</details>

📖 **Theory:** [denormalization](./04_schema_design/normalization.md#normalization)


---

### Q38 · [Thinking] · `btree-index`

> **Why does PostgreSQL use a B-tree index by default? Explain how it works and what kinds of queries benefit most from it.**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
A **B-tree (balanced tree) index** organizes values in a sorted, self-balancing tree structure. Every lookup, insert, and delete is O(log n). It's the default because it handles the widest range of queries efficiently.

**How to think through this:**
1. Picture a sorted phone book divided into sections, each section divided into pages — a B-tree is that hierarchy; to find a value the database walks down the tree levels (typically 3–4 levels even for millions of rows) instead of scanning every row
2. B-trees support equality (`=`), range (`>`, `<`, `BETWEEN`), prefix `LIKE 'abc%'`, `ORDER BY`, and `IS NULL` — basically any query that benefits from sorted order
3. They do *not* help with full-text search, geometric data, or `LIKE '%suffix'` (no leading anchor) — for those you'd use GIN, GiST, or other index types

**Key takeaway:** B-tree indexes work because they keep values sorted, making equality and range lookups fast by tree traversal instead of full-table scan.

</details>

📖 **Theory:** [btree-index](./04_schema_design/indexes.md#indexes)


---

### Q39 · [Critical] · `composite-index-order`

> **You have an index on `(department_id, salary)`. A colleague writes a query filtering only on `salary`. They're confused why the query is still slow. What's happening?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
The query skips the **leading column** `department_id`, so the database cannot use the composite index effectively — it either does a full index scan or falls back to a sequential table scan.

**How to think through this:**
1. A composite index `(department_id, salary)` is like a filing cabinet sorted first by department, then by salary within each department — to find a salary range you must open a specific department drawer first; without specifying department, there's no starting drawer
2. The **left-prefix rule**: a composite index on `(A, B, C)` can serve queries filtering on `A`, `A+B`, or `A+B+C` — but not `B` alone, `C` alone, or `B+C`
3. The fix: either add a separate index on `salary`, or reorder to `(salary, department_id)` if salary filters are more common — index column order should match your most frequent query patterns

**Key takeaway:** Composite indexes are only usable from the leftmost column outward — always put the most-filtered or highest-cardinality column first.

</details>

📖 **Theory:** [composite-index-order](./04_schema_design/indexes.md#composite-multi-column-index)


---

### Q40 · [Design] · `partial-index`

> **Your `orders` table has 50 million rows but only 10,000 are in `status = 'pending'`. Every operational query filters on pending orders. How can a partial index help here?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
A **partial index** includes a `WHERE` clause in the index definition, so only rows matching the condition are indexed. The result is a tiny, fast index that covers exactly the rows your queries care about.

**How to think through this:**
1. A regular index on `status` across 50 million rows is large and has low cardinality (maybe 5 status values) — it often won't be used; a partial index on `WHERE status = 'pending'` contains only 10,000 entries, making it dramatically smaller and cheaper to scan
2. Queries that include `WHERE status = 'pending'` will match the index condition and use this index; queries without that filter won't use it — which is fine, they weren't the target
3. Additional benefit: you can combine the partial index with other columns, e.g., `CREATE INDEX ON orders (created_at) WHERE status = 'pending'` — giving you a tiny, sorted index for the exact operational workload

**Key takeaway:** Partial indexes let you build an index over a relevant subset of rows, keeping it small, fast, and laser-focused on the queries that matter most.

</details>

📖 **Theory:** [partial-index](./04_schema_design/indexes.md#partial-index)


---

### Q41 · [Thinking] · `covering-index`

> **What is an index-only scan, and how do you design a covering index to enable it? Why does it matter for performance?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
An **index-only scan** is when PostgreSQL satisfies a query entirely from the index without touching the main table (heap). A **covering index** is one that includes all columns the query needs, enabling this scan.

**How to think through this:**
1. Normally an index lookup returns row pointers, then the database fetches the actual rows from disk — that second step (the heap fetch) is the expensive part; a covering index makes it unnecessary by storing the needed column values right in the index
2. Design example: if your query is `SELECT email FROM users WHERE account_id = 123`, an index on `(account_id, email)` covers the query — `account_id` is used for lookup, `email` is read directly from the index leaf node
3. In PostgreSQL you can use `INCLUDE` to add non-searchable columns: `CREATE INDEX ON users (account_id) INCLUDE (email)` — this keeps the index tree tight while still covering the query

**Key takeaway:** A covering index eliminates the heap fetch by bundling all needed column values into the index itself, turning two disk operations into one.

</details>

📖 **Theory:** [covering-index](./04_schema_design/indexes.md#indexes)


---

### Q42 · [Critical] · `index-when-not`

> **A junior engineer adds an index to every column "just to make things fast." What cases will actually hurt performance, and what would you tell them?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
Indexes slow down **writes** and waste storage. They also don't help (and can hurt) on **low-cardinality columns** or very **small tables**.

**How to think through this:**
1. Every `INSERT`, `UPDATE`, and `DELETE` must also update every index on the table — a table with 10 indexes on a high-write workload pays 10x the write overhead; this can turn a fast transactional table into a bottleneck
2. Low-cardinality columns like `is_active` (two values: true/false) or `gender` (few values) give the query planner a terrible selectivity ratio — if 60% of rows are `is_active = true`, a full table scan is faster than an index scan plus heap fetches for 60% of the table
3. Very small tables (a few hundred rows) are faster to scan sequentially than to navigate an index tree — the planner often ignores indexes on small tables entirely; indexes shine on large, frequently-read tables with high-cardinality filter columns

**Key takeaway:** Indexes are a read-write trade-off — add them surgically based on query patterns, not defensively on every column.

</details>

📖 **Theory:** [index-when-not](./04_schema_design/indexes.md#when-indexes-help)


---

### Q43 · [Normal] · `scalar-subquery`

> **You need to find all employees who earn more than the average salary of their own department. Write the query using a scalar subquery in the WHERE clause.**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
Use a correlated scalar subquery that computes the department average for each row being evaluated.

**How to think through this:**
1. A **scalar subquery** returns exactly one row and one column — it produces a single value that can be used in comparisons; placing it in `WHERE` lets you compare each row against a dynamically computed value
2. Here the subquery must reference the outer query's `department_id` to compute the average *for that employee's department* — making it correlated (re-evaluated per outer row)
3. The pattern is: `WHERE salary > (SELECT AVG(salary) FROM employees e2 WHERE e2.department_id = e1.department_id)` — straightforward to read but can be slow on large tables; a CTE or window function is often more efficient at scale

**Key takeaway:** A scalar subquery in `WHERE` is readable and correct for per-row comparisons, but watch performance on large tables — consider a JOIN to a pre-aggregated subquery instead.

</details>

📖 **Theory:** [scalar-subquery](./06_advanced_queries/subqueries.md#subquery-in-where--scalar-subquery)


---

### Q44 · [Normal] · `derived-table`

> **You want the top-earning employee in each department, but your database doesn't support window functions. How would you use a derived table (subquery in FROM) to solve this?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
Compute the max salary per department in a subquery inside `FROM`, then join it back to the employees table to get the matching rows.

**How to think through this:**
1. A **derived table** (also called an inline view) is a subquery placed in the `FROM` clause and given an alias — the outer query treats it exactly like a regular table; it's computed once and then joined
2. The approach: `FROM (SELECT department_id, MAX(salary) AS max_sal FROM employees GROUP BY department_id) AS dept_max` produces one row per department with the max salary; then `JOIN employees e ON e.department_id = dept_max.department_id AND e.salary = dept_max.max_sal` finds matching employees
3. Edge case: if two employees tie for the max salary in a department, both rows are returned — decide upfront whether that's desired or whether you need an additional tiebreaker

**Key takeaway:** A derived table lets you pre-aggregate or pre-filter data in the `FROM` clause, turning a complex multi-step problem into a clean join.

</details>

📖 **Theory:** [derived-table](./06_advanced_queries/subqueries.md#subquery-in-from--derived-table)


---

### Q45 · [Thinking] · `correlated-subquery`

> **Explain what makes a correlated subquery different from a regular subquery. Why can correlated subqueries be a performance problem on large tables?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
A **correlated subquery** references a column from the outer query, so it cannot be executed once and reused — it is re-evaluated for every row the outer query processes.

**How to think through this:**
1. A regular subquery runs once, produces a result set, and the outer query uses that result; a correlated subquery is like a function call embedded in the WHERE clause — the database calls it once per outer row, passing the current row's values as input
2. On a table with 1 million rows, a correlated subquery in `WHERE` runs 1 million times — if each sub-execution touches an index, that's 1 million index lookups; without an index on the correlated column it's 1 million full scans
3. The fix is usually to rewrite as a `JOIN` to a pre-aggregated subquery or CTE, or use window functions — this lets the planner compute the aggregation once and merge results in a single pass

**Key takeaway:** Correlated subqueries are N executions for N outer rows — they're readable but can be O(N²) performance traps; rewrite as a JOIN or window function when tables are large.

</details>

📖 **Theory:** [correlated-subquery](./06_advanced_queries/subqueries.md#subquery-in-select--correlated-subquery)


---

### Q46 · [Critical] · `exists-vs-in`

> **You're filtering orders to only those placed by customers in a VIP list. Should you use `WHERE customer_id IN (SELECT ...)` or `WHERE EXISTS (SELECT ...)`? Does it matter when the subquery can return NULLs?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
Both work when there are no NULLs, but `EXISTS` is safer with NULLs and often faster on large subquery result sets because it **short-circuits** on the first match.

**How to think through this:**
1. `IN` materializes the full subquery result into a list, then checks membership — if the subquery returns NULL values, `NULL IN (1, 2, NULL)` evaluates to `NULL` (not TRUE or FALSE), which causes rows to be silently filtered out; `IN` with NULLs in the list can give unexpected empty results
2. `EXISTS` runs the subquery for each outer row and stops as soon as one match is found — it never materializes the full list; it also evaluates to TRUE/FALSE only, so NULL rows in the subquery don't affect the outer result
3. Modern optimizers (PostgreSQL, MySQL 8+) often rewrite `IN` to a semi-join anyway, making them equivalent in execution plan — but `EXISTS` remains the safer semantic choice when subquery NULLs are possible

**Key takeaway:** Prefer `EXISTS` over `IN` when the subquery might return NULLs — `IN` against a NULL-containing list silently drops rows due to three-valued logic.

</details>

📖 **Theory:** [exists-vs-in](./06_advanced_queries/subqueries.md#exists-vs-in)


---

### Q47 · [Design] · `not-exists`

> **You want all customers who have never placed an order. You could use `LEFT JOIN ... WHERE order_id IS NULL` or `NOT EXISTS`. Which would you choose and why?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
Both are correct **anti-join** patterns. `NOT EXISTS` is often preferred for clarity and avoids a subtle NULL trap present in `NOT IN`.

**How to think through this:**
1. `LEFT JOIN ... WHERE orders.customer_id IS NULL` keeps all customers, joins orders if they exist, then filters to only the non-matching rows — semantically clear, and most optimizers execute it as a hash or merge anti-join (same plan as `NOT EXISTS`)
2. `NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id)` is equally efficient and arguably more readable as prose: "give me customers for which no matching order exists"
3. The important pattern to avoid is `NOT IN (SELECT customer_id FROM orders)` — if any row in orders has a NULL `customer_id`, the entire `NOT IN` list returns NULL, and the outer query returns zero rows; `NOT EXISTS` is immune to this because it checks existence, not list membership

**Key takeaway:** `NOT EXISTS` and `LEFT JOIN / IS NULL` are equivalent anti-joins — use either, but never `NOT IN` when the subquery column can contain NULLs.

</details>

📖 **Theory:** [not-exists](./06_advanced_queries/subqueries.md#not-exists--anti-join-alternative)


---

### Q48 · [Normal] · `cte-basics`

> **Rewrite this deeply nested subquery using a CTE. Why is the CTE version easier to work with?**

```sql
SELECT dept, avg_sal
FROM (
  SELECT department_id AS dept, AVG(salary) AS avg_sal
  FROM employees
  GROUP BY department_id
) AS dept_avgs
WHERE avg_sal > 80000;
```

<details>
<summary>💡 Show Answer</summary>

**Answer:**
A **Common Table Expression (CTE)** defined with `WITH` gives the subquery a name, hoisting it to the top so the main query reads like plain English.

**How to think through this:**
1. The `WITH` clause names the intermediate result: `WITH dept_avgs AS (SELECT department_id AS dept, AVG(salary) AS avg_sal FROM employees GROUP BY department_id)` — the name `dept_avgs` can then be referenced in the main `SELECT` just like a table, making the query read top-to-bottom like a recipe
2. Unlike derived tables, a named CTE can be **referenced multiple times** in the same query without repeating the subquery — if you need `dept_avgs` in both a `JOIN` and a `WHERE`, you write it once
3. Readability is the primary win — a deep chain of nested subqueries forces you to read inside-out; a CTE chain reads sequentially step by step, which is how humans think about data transformations

**Key takeaway:** CTEs replace inside-out nested subqueries with a named, top-to-bottom recipe that is easier to read, debug, and reuse within the same query.

</details>

📖 **Theory:** [cte-basics](./06_advanced_queries/ctes.md#ctes--sticky-notes-on-a-whiteboard)


---

### Q49 · [Design] · `recursive-cte`

> **Your `employees` table has a `manager_id` column that points back to `employee_id`. How would you use a recursive CTE to retrieve an entire reporting chain starting from a given employee?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
A **recursive CTE** has two parts joined by `UNION ALL`: an **anchor** (the starting row) and a **recursive member** (each iteration joins back to the CTE to find the next level).

**How to think through this:**
1. The anchor selects the root employee: `SELECT employee_id, name, manager_id, 1 AS depth FROM employees WHERE employee_id = :start_id` — this is the first row in the result and the seed for recursion
2. The recursive member joins the CTE back to `employees` to find direct reports: `SELECT e.employee_id, e.name, e.manager_id, r.depth + 1 FROM employees e JOIN reporting_chain r ON e.manager_id = r.employee_id` — each iteration adds one more level of the tree
3. The recursion stops automatically when the join produces no new rows (leaf nodes have no direct reports); always add a depth limit or cycle guard (`WHERE depth < 10`) to prevent infinite loops on bad data with circular references

**Key takeaway:** Recursive CTEs traverse hierarchies by self-joining the CTE in the recursive member — anchor sets the root, recursive member walks one level per iteration until no rows remain.

</details>

📖 **Theory:** [recursive-cte](./06_advanced_queries/ctes.md#recursive-ctes--walking-a-hierarchy)


---

### Q50 · [Interview] · `cte-vs-subquery`

> **A teammate always uses CTEs; another always uses subqueries. When is each actually the better choice?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
CTEs win on **readability and reuse**; subqueries win on **inline simplicity** and sometimes **optimizer flexibility**.

**How to think through this:**
1. Use a CTE when: the intermediate result is referenced more than once in the query, the logic has multiple sequential steps that read better as a recipe, or you need recursion — the `WITH` clause turns complex logic into named, debuggable chunks
2. Use a subquery when: the logic is a simple one-off filter or derivation that only appears once and is short enough to read inline without losing context — an inline subquery in `FROM` or `WHERE` avoids the overhead of naming something that's used exactly once
3. Performance nuance: in PostgreSQL, a plain CTE is an **optimization fence** in versions before 12 (the planner can't push predicates into it); from PostgreSQL 12+ CTEs are inlined by default unless you write `WITH ... AS MATERIALIZED (...)` — so in older versions a subquery could outperform a CTE because the planner had more freedom

**Key takeaway:** Default to CTEs for multi-step or reused logic; use inline subqueries for simple one-off derivations — and know that CTEs in PostgreSQL pre-12 are optimization fences that can block predicate pushdown.

</details>

📖 **Theory:** [cte-vs-subquery](./06_advanced_queries/ctes.md#cte-vs-subquery--when-to-use-each)


---

### Q51 · [Normal] · `case-searched`

> **You're generating a customer report and need to label each customer as 'High', 'Mid', or 'Low' value based on their total spend. How do you write the CASE expression for this row classification?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
Use the **searched CASE** form — `CASE WHEN condition THEN result ... END` — which evaluates independent boolean conditions in order and returns the first match.

**How to think through this:**
1. The searched CASE is more flexible than the simple CASE (which only tests equality against one value) — each `WHEN` can use any boolean expression including `>`, `<`, `BETWEEN`, `IS NULL`, or even subqueries
2. Order matters: conditions are evaluated top to bottom and the first TRUE wins; put the most specific or highest-value conditions first (`WHEN total_spend > 10000 THEN 'High'`) so they're caught before the broader ranges
3. Always include `ELSE` as a catch-all — without it, rows matching no condition return NULL, which can silently corrupt a report column that should always have a value

**Key takeaway:** Searched CASE evaluates conditions in order and returns the first match — put specific conditions before general ones and always include ELSE to avoid NULL surprises.

</details>

📖 **Theory:** [case-searched](./06_advanced_queries/case_and_conditionals.md#1-searched-case--most-flexible)


---

### Q52 · [Thinking] · `conditional-aggregation`

> **You want a single-row summary showing how many orders are 'pending', how many are 'shipped', and how many are 'delivered' — without writing three separate queries. How does conditional aggregation solve this?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
**Conditional aggregation** uses `CASE` inside an aggregate function (`SUM` or `COUNT`) to count or sum only the rows matching each condition — producing a pivot-like result in one pass.

**How to think through this:**
1. The pattern is: `SUM(CASE WHEN status = 'pending' THEN 1 ELSE 0 END) AS pending_count` — for each row, the CASE emits 1 if the condition matches or 0 if not, and `SUM` adds them up; repeat for each status column you need
2. Alternatively use `COUNT(CASE WHEN status = 'pending' THEN 1 END)` — when CASE returns NULL (the implicit ELSE), COUNT ignores NULLs, so this is equivalent without the explicit ELSE 0
3. This is a single table scan with multiple aggregation expressions — far more efficient than three separate `COUNT(*) WHERE status = ?` queries and cleaner than a `GROUP BY status` pivot that would require the application layer to reshape the rows

**Key takeaway:** Conditional aggregation collapses multiple filter-and-count operations into one query pass by putting CASE expressions inside SUM or COUNT, rotating status values into columns.

</details>

📖 **Theory:** [conditional-aggregation](./06_advanced_queries/case_and_conditionals.md#case-in-group-by--conditional-aggregation)


---

### Q53 · [Critical] · `coalesce-nullif`

> **You're computing average response time but some rows have `0` stored as a sentinel for "unknown" — you want those treated as NULL. You also want to display "N/A" instead of NULL in the output. Which functions do you combine, and how?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
Use **`NULLIF`** to convert the sentinel `0` to NULL before the aggregation, then **`COALESCE`** to replace NULL in the display column with `'N/A'`.

**How to think through this:**
1. `NULLIF(response_time, 0)` returns NULL when `response_time = 0`, otherwise returns `response_time` unchanged — this cleanly converts the sentinel without a CASE expression; `AVG` then automatically ignores NULLs, so zeros are excluded from the average
2. `COALESCE(value, 'N/A')` returns the first non-NULL argument — wrapping the AVG result: `COALESCE(AVG(NULLIF(response_time, 0))::text, 'N/A')` produces a display-ready string
3. The combination `COALESCE(NULLIF(...))` is a common idiom: NULLIF creates the NULL, COALESCE catches it — remember their signatures: `NULLIF(a, b)` returns NULL if `a = b`; `COALESCE(a, b, c...)` returns the first non-NULL

**Key takeaway:** Use `NULLIF` to manufacture NULLs from sentinel values (silencing them in aggregates), and `COALESCE` to replace NULLs with a fallback display value.

</details>

📖 **Theory:** [coalesce-nullif](./06_advanced_queries/case_and_conditionals.md#coalesce--first-non-null-value)


---

### Q54 · [Normal] · `string-functions`

> **A legacy data load dumped customer names with inconsistent spacing and mixed case — like `"  john SMITH  "`. List the functions you'd chain to clean this, and what each does.**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
Chain `TRIM` → `LOWER` (or `INITCAP`) → optionally `REPLACE` to clean the string in one expression.

**How to think through this:**
1. `TRIM('  john SMITH  ')` removes leading and trailing whitespace, returning `'john SMITH'`; `LTRIM` and `RTRIM` strip only one side if needed
2. `LOWER('john SMITH')` returns `'john smith'`; PostgreSQL's `INITCAP('john smith')` capitalizes each word's first letter, returning `'John Smith'` — ideal for display names
3. For structural fixes: `REPLACE(name, '  ', ' ')` collapses double internal spaces; `SUBSTRING(name, 1, 50)` enforces a max length; `LENGTH(name)` validates the result — these are the core string toolkit: `LENGTH`, `UPPER`/`LOWER`/`INITCAP`, `TRIM`/`LTRIM`/`RTRIM`, `SUBSTRING`, `REPLACE`, `CONCAT` (or `||`)

**Key takeaway:** String cleaning chains TRIM (remove outer whitespace) + LOWER/INITCAP (normalize case) + REPLACE (fix internals) — apply them in that order since TRIM must come before case normalization for consistent results.

</details>

📖 **Theory:** [string-functions](./06_advanced_queries/string_and_date_functions.md#string-and-date-functions--power-tools-for-text-and-time)


---

### Q55 · [Normal] · `date-functions`

> **You want a monthly revenue report: group orders by calendar month, show how many days since each order, and add a 30-day expiry date to each order timestamp. Which date functions do you reach for?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
Use `DATE_TRUNC` for grouping by month, `EXTRACT` or `AGE` for elapsed time, and `INTERVAL` arithmetic for adding durations.

**How to think through this:**
1. `DATE_TRUNC('month', order_date)` truncates a timestamp to the first instant of its month — `'2024-03-15 10:30'` becomes `'2024-03-01 00:00:00'`; this makes all March orders group together under the same key
2. `AGE(NOW(), order_date)` returns a human-readable interval like `'15 days 04:30:00'`; `EXTRACT(DAY FROM NOW() - order_date)` returns a plain integer number of days — use EXTRACT when you need a number for comparison or arithmetic
3. `order_date + INTERVAL '30 days'` adds exactly 30 days to the timestamp; you can also use `order_date + INTERVAL '1 month'` for a calendar month — PostgreSQL handles month-length variation automatically

**Key takeaway:** For date work: `DATE_TRUNC` to snap to a period boundary for grouping, `EXTRACT`/`AGE` for elapsed measurements, and `+ INTERVAL` for adding durations.

</details>

📖 **Theory:** [date-functions](./06_advanced_queries/string_and_date_functions.md#string-and-date-functions--power-tools-for-text-and-time)


---

### Q56 · [Normal] · `insert-basics`

> **You need to populate a `archived_orders` table from `orders` where status is 'closed', and also insert a single new test record. Show both INSERT patterns.**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
Use `INSERT INTO ... VALUES (...)` for a single row and `INSERT INTO ... SELECT ...` to bulk-copy from another table.

**How to think through this:**
1. Single-row insert: `INSERT INTO archived_orders (order_id, customer_id, total, status) VALUES (9999, 42, 150.00, 'closed')` — list the target columns explicitly so the statement is resilient to table column reordering or future additions
2. Bulk insert from query: `INSERT INTO archived_orders SELECT * FROM orders WHERE status = 'closed'` — this runs as a single statement and is far more efficient than looping with individual inserts; the database can optimize it as a bulk operation
3. Multi-row VALUES shorthand: `INSERT INTO archived_orders (order_id, status) VALUES (1, 'closed'), (2, 'closed'), (3, 'closed')` — batching multiple value tuples in one statement reduces round-trip overhead versus separate INSERT statements

**Key takeaway:** Always list target columns explicitly in INSERT statements; use `INSERT INTO ... SELECT` for bulk copies — it's cleaner and faster than application-level loops.

</details>

📖 **Theory:** [insert-basics](./07_data_modification/insert.md#insert--adding-data-to-your-tables)


---

### Q57 · [Design] · `upsert-on-conflict`

> **You're syncing a product catalog from an external feed. If a product already exists (by `sku`), you want to update the price; if it doesn't exist, insert it. How does PostgreSQL's upsert handle this atomically?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
PostgreSQL's `INSERT ... ON CONFLICT` clause handles this atomically — it specifies which conflict target to detect and what to do when that conflict occurs.

**How to think through this:**
1. The syntax is: `INSERT INTO products (sku, name, price) VALUES (...) ON CONFLICT (sku) DO UPDATE SET price = EXCLUDED.price, updated_at = NOW()` — `EXCLUDED` is a special pseudo-table that holds the values that *would have been* inserted, making it easy to reference the new values in the update
2. `ON CONFLICT DO NOTHING` is the other variant — silently skip the insert if a conflict exists, useful for idempotent bulk loads where you don't want duplicates but don't need to update anything
3. The operation is atomic: there is no race condition between checking existence and inserting/updating — the entire check-and-act happens in a single lock-protected operation, unlike an application-level read-then-write

**Key takeaway:** `ON CONFLICT DO UPDATE` is a true atomic upsert — use `EXCLUDED` to reference the incoming row's values, and prefer it over application-level check-then-insert to eliminate race conditions.

</details>

📖 **Theory:** [upsert-on-conflict](./07_data_modification/insert.md#on-conflict--upsert-in-postgresql)


---

### Q58 · [Normal] · `update-join`

> **You need to update the `region` column on `orders` based on the `region` stored in a separate `customers` table. How do you write an UPDATE that joins to another table in PostgreSQL?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
PostgreSQL uses `UPDATE ... FROM ... WHERE` to join another table in an update statement — the `FROM` clause introduces the second table and the `WHERE` clause links the join condition.

**How to think through this:**
1. Syntax: `UPDATE orders SET region = c.region FROM customers c WHERE orders.customer_id = c.customer_id` — the `FROM` clause works like a JOIN source, and `SET` can reference columns from the joined table using the alias
2. Unlike standard SQL's `UPDATE ... JOIN` (MySQL syntax), PostgreSQL uses `FROM` — which is semantically a cross join filtered by `WHERE`; always include the join condition in `WHERE` or you'll accidentally update every row with values from arbitrary customer rows
3. You can join multiple tables by adding more tables to `FROM` and more conditions to `WHERE`; add a final `WHERE orders.region IS NULL` to restrict the update to only unset rows if needed

**Key takeaway:** PostgreSQL UPDATE joins use `FROM table_alias WHERE join_condition` — always include the join predicate in WHERE, not just the filter, or you'll produce a cartesian update.

</details>

📖 **Theory:** [update-join](./07_data_modification/update_and_delete.md#update-with-a-join-postgresql-syntax)


---

### Q59 · [Interview] · `delete-truncate`

> **You need to clear all rows from a large log table. A teammate says "just use DELETE." You suggest TRUNCATE. What's the difference, and when would DELETE still be the right choice?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
**TRUNCATE** removes all rows instantly by deallocating data pages rather than scanning and deleting row by row. **DELETE** scans every row, writes WAL log entries for each, and respects WHERE filters.

**How to think through this:**
1. TRUNCATE is dramatically faster on large tables because it doesn't log individual row deletions — it drops entire data pages; on a 100-million-row table, DELETE might run for minutes while TRUNCATE is milliseconds
2. TRUNCATE also resets sequences (identity columns) to their start value by default, fires no row-level triggers, and cannot be filtered with WHERE — it's all-or-nothing
3. Use DELETE when: you need a WHERE clause to remove only some rows, you need row-level triggers to fire, you need the operation to be rollback-friendly in a transaction with savepoints, or you need the row count returned; TRUNCATE is transactional in PostgreSQL (can be rolled back) but is not suitable for partial deletes

**Key takeaway:** TRUNCATE for clearing entire tables fast (no WHERE, resets sequences); DELETE for filtered removal, trigger-dependent workflows, or when you need exact row counts.

</details>

📖 **Theory:** [delete-truncate](./07_data_modification/update_and_delete.md#truncate-vs-delete)


---

### Q60 · [Design] · `soft-delete`

> **Your product manager wants to be able to "restore" deleted customer records. What is the soft delete pattern, how do you implement it, and what are the hidden costs?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
**Soft delete** adds a flag column (`is_deleted BOOLEAN` or `deleted_at TIMESTAMP`) to mark a row as deleted without physically removing it — the row stays in the table and can be restored by clearing the flag.

**How to think through this:**
1. Implementation: add `deleted_at TIMESTAMP DEFAULT NULL` to the table; deletions become `UPDATE customers SET deleted_at = NOW() WHERE customer_id = ?`; all application queries must add `WHERE deleted_at IS NULL` to exclude soft-deleted rows — a partial index on `WHERE deleted_at IS NULL` keeps these queries fast
2. The primary benefit is recoverability and audit trail — you can see who was deleted, when, and restore with a single UPDATE; hard deletes are irreversible and require separate audit tables
3. Hidden costs: every query must remember the `WHERE deleted_at IS NULL` filter (easy to forget, causing ghost data to appear); unique constraints need rethinking (`email` must be unique among *active* users — a partial unique index solves this); the table grows unboundedly and requires periodic archival

**Key takeaway:** Soft delete preserves data for recovery and auditing but adds filter discipline overhead to every query — enforce it with a partial index and consider a view that adds the filter automatically.

</details>

📖 **Theory:** [soft-delete](./07_data_modification/update_and_delete.md#the-soft-delete-pattern)


---

### Q61 · [Interview] · `acid-properties`

> **A payment system processes a bank transfer: debit one account, credit another. Walk through how each ACID property protects this transaction if something goes wrong mid-transfer.**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
**ACID** stands for Atomicity, Consistency, Isolation, and Durability — each property addresses a different failure mode for the transfer.

**How to think through this:**
1. **Atomicity**: the debit and credit are one indivisible unit — if the server crashes after the debit but before the credit, the entire transaction is rolled back; no money disappears into the void; it's all-or-nothing
2. **Consistency**: the database enforces constraints (no negative balances, total money conserved) after every transaction; a transaction that would violate a constraint is rejected, keeping the database in a valid state; **Isolation** ensures concurrent transfers on the same accounts don't see each other's in-progress state — one transfer reads committed balances, not half-complete ones
3. **Durability**: once the database returns "committed," the transfer is written to durable storage (WAL flushed to disk) — a crash immediately after commit does not undo the transfer; the data survives power loss

**Key takeaway:** ACID is a four-layer safety net: Atomicity prevents partial writes, Consistency enforces rules, Isolation prevents concurrent corruption, Durability survives crashes.

</details>

📖 **Theory:** [acid-properties](./07_data_modification/transactions.md#acid-properties-explained-simply)


---

### Q62 · [Normal] · `transaction-basics`

> **You're writing a script that inserts an order and then inserts the order's line items. Why should both inserts be wrapped in a single transaction, and what does the BEGIN/COMMIT/ROLLBACK lifecycle look like?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
Without a transaction, a crash or error between the two inserts leaves an orphan order with no line items — wrapping them in a **transaction** guarantees both succeed or both are rolled back.

**How to think through this:**
1. `BEGIN` starts the transaction and puts the session in a pending state — changes are written to a transaction buffer and not yet visible to other sessions; `COMMIT` makes all changes permanent and visible atomically; `ROLLBACK` undoes everything back to the `BEGIN`
2. If the line item insert fails (e.g., foreign key violation, constraint error), the application calls `ROLLBACK` and neither the order nor the items are persisted — the database stays consistent
3. In PostgreSQL every statement outside an explicit `BEGIN` runs in its own auto-committed single-statement transaction; once you wrap multiple statements in `BEGIN...COMMIT`, they share a single transaction ID and are treated as one atomic unit

**Key takeaway:** Wrap related DML statements in a transaction so that failure of any step triggers a full ROLLBACK — never let your data end up in a half-written state.

</details>

📖 **Theory:** [transaction-basics](./07_data_modification/transactions.md#transactions--all-or-nothing)


---

### Q63 · [Normal] · `savepoint`

> **Inside a large transaction you want to attempt a risky UPDATE but be able to undo just that step without rolling back the entire transaction if it fails. What SQL feature enables this?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
**SAVEPOINTs** create named checkpoints within a transaction that you can roll back to without abandoning the entire transaction.

**How to think through this:**
1. Syntax flow: `BEGIN;` → do some work → `SAVEPOINT before_risky_update;` → attempt the risky UPDATE → if it fails, `ROLLBACK TO SAVEPOINT before_risky_update;` → continue the transaction with the state restored to the savepoint — then optionally `RELEASE SAVEPOINT before_risky_update;` to free the savepoint when no longer needed
2. `ROLLBACK TO SAVEPOINT` does not end the transaction — it rewinds the transaction's state to the savepoint marker and keeps the session inside the active transaction; you can still COMMIT the work done before the savepoint
3. SAVEPOINTs are useful in stored procedures, ETL pipelines, and application code that processes rows in a loop — mark a savepoint before each row, catch exceptions, roll back just that row, log the error, and continue processing the rest without aborting the whole batch

**Key takeaway:** SAVEPOINTs give you a partial undo within a transaction — rollback to the marker without losing earlier work, then keep going or commit what succeeded.

</details>

📖 **Theory:** [savepoint](./07_data_modification/transactions.md#savepoint--partial-rollbacks)


---

### Q64 · [Interview] · `isolation-levels`

> **Your reporting query reads a balance twice in the same transaction and gets different values each time because another transaction committed in between. What isolation problem is this, and how do the standard isolation levels address it?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
This is a **non-repeatable read** — a row read twice in the same transaction returns different values because another transaction committed a change in between.

**How to think through this:**
1. The four standard isolation levels address progressively stronger anomalies: **READ UNCOMMITTED** (allows dirty reads — seeing uncommitted changes), **READ COMMITTED** (prevents dirty reads but allows non-repeatable reads — PostgreSQL default), **REPEATABLE READ** (prevents non-repeatable reads by snapshotting all rows at transaction start), **SERIALIZABLE** (prevents phantom reads too — new rows inserted by others don't appear mid-transaction)
2. A **phantom read** is similar but affects a range: transaction A counts rows matching a filter, transaction B inserts a new matching row and commits, transaction A counts again and gets a different number — REPEATABLE READ prevents row changes but not phantom inserts in some databases; PostgreSQL REPEATABLE READ also blocks phantoms
3. Higher isolation = safer reads but more lock contention and potential serialization failures; READ COMMITTED is the pragmatic default for most applications; bump to REPEATABLE READ or SERIALIZABLE only for financial calculations or reports where consistency across multiple reads is critical

**Key takeaway:** READ COMMITTED is the default and prevents dirty reads; REPEATABLE READ locks the snapshot so the same row always returns the same value; SERIALIZABLE is the strongest and treats concurrent transactions as if they ran one-at-a-time.

</details>

📖 **Theory:** [isolation-levels](./07_data_modification/transactions.md#isolation-levels)


---

### Q65 · [Design] · `views`

> **Three different application teams are writing queries against a complex 5-table join. You want to give them a simple interface without duplicating the join logic. What do views offer here, and what's the key limitation?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
A **view** is a named, saved query — teams query it like a table, and the database substitutes the view definition at runtime. It centralizes complex join logic in one place.

**How to think through this:**
1. `CREATE VIEW active_order_summary AS SELECT ... FROM orders JOIN customers ... JOIN ...` — each team runs `SELECT * FROM active_order_summary WHERE region = 'west'` without knowing or repeating the join logic; when the underlying schema changes you update the view in one place
2. Views also act as access control boundaries — grant SELECT on the view without granting access to the underlying tables; teams see only the columns you expose, not sensitive fields like `payment_method` or `internal_notes`
3. Key limitation: most views are **not updatable** — `INSERT`/`UPDATE`/`DELETE` through a view is only allowed when the view maps directly to one base table with no aggregation, DISTINCT, GROUP BY, or joins; a view over 5 joined tables is not updatable; use it for reads only

**Key takeaway:** Views encapsulate complex query logic behind a simple name and are ideal for read access control — but views over joins, aggregates, or DISTINCT are not updatable.

</details>

📖 **Theory:** [views](./09_real_world/views.md#views--a-window-into-your-data)


---

### Q66 · [Interview] · `materialized-views`

> **A regular view over your analytics join is taking 45 seconds to query. A colleague suggests a materialized view. What is it, how is it different from a regular view, and what's the operational trade-off?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
A **materialized view** physically stores the query result on disk at creation (or refresh) time, so queries read pre-computed rows instead of re-executing the underlying query each time.

**How to think through this:**
1. A regular view is a stored query — every time you SELECT from it, the database executes the full underlying SQL from scratch; a materialized view is a stored *result* — the expensive join and aggregation runs once at `REFRESH` time, and all subsequent queries read the cached rows like a regular table
2. `CREATE MATERIALIZED VIEW monthly_revenue AS SELECT ...` followed by `REFRESH MATERIALIZED VIEW monthly_revenue` re-executes the query and replaces the stored data; `REFRESH MATERIALIZED VIEW CONCURRENTLY` allows reads during refresh (requires a unique index on the view) — without CONCURRENTLY, the view is locked during refresh
3. The trade-off is **staleness**: a materialized view is a snapshot in time; data changes in the base tables are invisible until the next REFRESH; you must schedule refreshes (cron job, trigger, or Airflow DAG) and decide how stale is acceptable — for a daily dashboard, hourly refresh may be fine; for real-time balances, it's not

**Key takeaway:** Materialized views trade freshness for speed — they pre-compute and store results for instant reads, but require scheduled REFRESH to stay current with base table changes.

</details>

📖 **Theory:** [materialized-views](./09_real_world/views.md#materialized-views-postgresql)



---

---

### Q67 · [Thinking] · `stored-procedures`

> **What is the difference between a PostgreSQL function and a stored procedure? When would you use PL/pgSQL?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
In PostgreSQL, a **function** returns a value and can be used inside a SELECT. A **stored procedure** (added in PostgreSQL 11) does not return a value directly but can manage transactions — it can COMMIT and ROLLBACK inside its body. Functions cannot control transactions.

**How to think through this:**
1. Ask: does this routine need to return data for use in a query? If yes, use a function. If it's a batch operation (ETL, bulk update), use a procedure.
2. Ask: does it need to commit intermediate work inside the routine? Only procedures can do that.
3. PL/pgSQL is the procedural language that enables IF/ELSE, loops, variables, and exception handling inside both functions and procedures — use it when pure SQL isn't expressive enough.

**Key takeaway:** Use a function when you need a return value in SQL; use a stored procedure when you need transaction control inside the routine.

</details>

📖 **Theory:** [stored-procedures](./09_real_world/stored_procedures.md#stored-procedures--reusable-logic-in-the-database)


---

### Q68 · [Thinking] · `triggers`

> **What is the difference between a BEFORE and AFTER trigger? How would you use a trigger to automatically update an `updated_at` timestamp?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
A **BEFORE trigger** fires before the row is written to disk — you can modify the row inside the trigger function before it lands. An **AFTER trigger** fires after the write and is used for side effects (auditing, cascading actions) where modifying the row is not needed.

For auto-stamping `updated_at`, use a BEFORE trigger with `FOR EACH ROW`:

```sql
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_updated_at
BEFORE UPDATE ON orders
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();
```

**How to think through this:**
1. You need to modify `NEW` (the incoming row) before it is written — that requires BEFORE, not AFTER.
2. `FOR EACH ROW` means the trigger fires once per row changed; `FOR EACH STATEMENT` fires once per statement regardless of row count.
3. Return `NEW` from the trigger function — returning NULL cancels the operation.

**Key takeaway:** Use a BEFORE trigger when you need to modify the row being written; use AFTER when you only need to react to the change.

</details>

📖 **Theory:** [triggers](./09_real_world/triggers.md#triggers--automatic-actions-when-data-changes)


---

### Q69 · [Thinking] · `explain-plan`

> **How do you read EXPLAIN output in PostgreSQL? What is the difference between a Seq Scan and an Index Scan, and what do cost, rows, and width mean?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
`EXPLAIN` shows the **query plan** PostgreSQL will use without executing the query. Each node shows the operation, estimated cost, estimated rows, and average row width.

- **Seq Scan**: reads every page of the table from start to finish. Appropriate for small tables or when the query returns most rows.
- **Index Scan**: follows the B-tree index to find specific rows, then fetches the heap. Appropriate for high-selectivity filters.
- **cost**: shown as `startup_cost..total_cost`. Units are arbitrary (not milliseconds) but relative to each other. Lower is faster.
- **rows**: estimated number of rows returned by that node.
- **width**: estimated average bytes per row.

**How to think through this:**
1. Scan the plan top-down — the outermost node is the final operation; indented inner nodes feed into it.
2. Look for Seq Scans on large tables with selective WHERE clauses — those are index opportunities.
3. Compare `rows` estimates to actual results (use EXPLAIN ANALYZE for actuals) — a large mismatch means stale statistics.

**Key takeaway:** EXPLAIN shows what PostgreSQL plans to do; a Seq Scan on a large table with a selective filter is the most common sign that an index is missing.

</details>

📖 **Theory:** [explain-plan](./08_performance/execution_plans.md#explain--see-the-plan-without-running-it)


---

### Q70 · [Thinking] · `explain-analyze`

> **What does EXPLAIN ANALYZE add over EXPLAIN? Why does the gap between estimated and actual rows matter?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
`EXPLAIN ANALYZE` actually **executes** the query and adds real timing and row counts alongside the planner's estimates. You see both `rows=estimated` and `actual rows=N, loops=1`.

The gap between estimated and actual rows matters because the planner chooses join strategies and scan types based on its row estimates. If it thinks 10 rows will match but 1,000,000 actually do, it may choose a Nested Loop (fast for small sets) over a Hash Join (fast for large sets), causing catastrophic slowdowns.

**How to think through this:**
1. Run `EXPLAIN ANALYZE` and compare `rows=X` (estimate) to `actual rows=Y` for each node.
2. A ratio greater than 10x or less than 0.1x signals stale or missing statistics — run `ANALYZE table_name` to refresh them.
3. Check `actual time` on the slowest nodes to find the bottleneck; high loops count on a Nested Loop with large inner sets is a red flag.

**Key takeaway:** The estimate-vs-actual row gap is the leading diagnostic for bad query plans — large gaps mean statistics need refreshing.

</details>

📖 **Theory:** [explain-analyze](./08_performance/execution_plans.md#explain-analyze--run-it-and-measure)


---

### Q71 · [Thinking] · `slow-query-causes`

> **What are the most common causes of slow SQL queries in production?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
The top causes of slow queries are:

1. **Missing index**: the query does a full table scan instead of using an index. Most common fix.
2. **SELECT ***: fetches all columns including large text/JSON blobs; wastes I/O and network bandwidth.
3. **N+1 queries**: application fetches one parent row then issues one query per child row in a loop — 1000 orders = 1001 queries. Fix with a JOIN or a single batch query.
4. **Functions on indexed columns**: `WHERE LOWER(email) = ...` or `WHERE DATE(created_at) = ...` prevents index use because the index stores the raw value, not the function output.
5. **Stale statistics**: planner makes bad plan choices based on outdated row count estimates.
6. **Unbounded queries**: no LIMIT on a query that returns millions of rows.
7. **Lock contention**: query waits behind a long-running transaction holding a lock.

**How to think through this:**
1. Start with `pg_stat_statements` to find the highest total-time queries.
2. Run `EXPLAIN ANALYZE` on each to classify the cause (Seq Scan, poor estimate, etc.).
3. Fix the worst offender first — usually a missing index or an N+1 in application code.

**Key takeaway:** Missing indexes and N+1 query patterns cause the majority of production slow query incidents.

</details>

📖 **Theory:** [slow-query-causes](./08_performance/query_optimization.md#query-optimization--making-slow-queries-fast)


---

### Q72 · [Thinking] · `index-on-function`

> **Why does `WHERE LOWER(email) = 'user@example.com'` prevent PostgreSQL from using a standard index on the `email` column? How do you fix it?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
A standard B-tree index on `email` stores the original values (e.g., `'User@Example.com'`). When you apply `LOWER()` in the WHERE clause, PostgreSQL must compute `LOWER(email)` for every row to check the condition — it cannot use the index because the index doesn't contain lowercased values.

The fix is an **expression index** (also called a **functional index**):

```sql
CREATE INDEX idx_users_email_lower ON users (LOWER(email));
```

Now the index stores the precomputed `LOWER(email)` values, and `WHERE LOWER(email) = 'user@example.com'` can use it directly.

**How to think through this:**
1. The rule: if a function wraps an indexed column in a WHERE clause, the index is invisible to the planner.
2. The fix: build the index on the same expression used in the query — they must match exactly.
3. Alternatively, enforce lowercase at write time (store emails already lowercased) and use a plain index.

**Key takeaway:** Indexes store raw values — applying any function to a column in a WHERE clause bypasses the index unless you create an expression index on that exact function.

</details>

📖 **Theory:** [index-on-function](./08_performance/indexing_strategies.md#covering-indexes-index-only-scans)


---

### Q73 · [Thinking] · `exists-in-join-perf`

> **When should you use EXISTS vs IN vs JOIN for existence checks and anti-patterns? What is the NULL trap with IN?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
All three can check "does a matching row exist" but behave differently:

- **EXISTS**: stops scanning as soon as one match is found (short-circuit). Best for large subqueries. Works correctly with NULLs because it only checks row presence.
- **IN with subquery**: historically evaluated as a full list; modern PostgreSQL optimizes it similarly to EXISTS in most cases. The NULL trap: `WHERE id NOT IN (SELECT user_id FROM banned WHERE user_id IS NULL)` returns zero rows if any `user_id` in the sublist is NULL, because `x NOT IN (1, 2, NULL)` evaluates to UNKNOWN (NULL), not TRUE.
- **JOIN**: most flexible; lets you project columns from both tables. Can produce duplicate rows if the right table has multiple matches — use `DISTINCT` or `EXISTS` if you only want existence.

**How to think through this:**
1. For "does this user exist in another table?" — prefer EXISTS.
2. For NOT IN: always check whether the subquery column is nullable. If it is, use NOT EXISTS instead.
3. For anti-join (rows in A with no match in B): `LEFT JOIN ... WHERE b.id IS NULL` or `NOT EXISTS` are both idiomatic; avoid `NOT IN` on nullable columns.

**Key takeaway:** The NULL trap in NOT IN is one of the most common subtle bugs in SQL — use NOT EXISTS when the subquery column can contain NULLs.

</details>

📖 **Theory:** [exists-in-join-perf](./08_performance/query_optimization.md#exists-vs-in-vs-join-performance)


---

### Q74 · [Thinking] · `index-bloat`

> **What is index bloat in PostgreSQL? What causes it on write-heavy tables, and how do you fix it?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
**Index bloat** is when an index grows much larger than needed because it contains many dead (logically deleted) entries. PostgreSQL uses MVCC — when a row is updated or deleted, the old version is not immediately removed; it stays on disk as a dead tuple. The index still holds pointers to these dead tuples.

On write-heavy tables (frequent UPDATEs and DELETEs), dead tuples accumulate faster than **VACUUM** can clean them up, causing the index to consume far more disk space than the live data warrants. This slows index scans because PostgreSQL must traverse bloated index pages.

Fixes:
- **VACUUM**: removes dead tuples from the table and marks index pages as reclaimable. Runs automatically (autovacuum) but may lag on very busy tables.
- **VACUUM FULL**: rewrites the table and its indexes from scratch, reclaiming disk space. Requires an exclusive lock — avoid on live production without a maintenance window.
- **REINDEX**: rebuilds the index from scratch, eliminating bloat without touching the table. Use `REINDEX CONCURRENTLY` to avoid locking.

**How to think through this:**
1. Check bloat with `pgstattuple` extension or query `pg_stat_user_indexes` for unusual index sizes.
2. For live tables, `REINDEX CONCURRENTLY` is the safest fix.
3. Long-term: tune autovacuum aggressiveness (`autovacuum_vacuum_scale_factor`) for write-heavy tables.

**Key takeaway:** Index bloat is caused by MVCC dead tuples accumulating faster than autovacuum clears them — REINDEX CONCURRENTLY is the live-safe fix.

</details>

📖 **Theory:** [index-bloat](./08_performance/indexing_strategies.md#index-bloat)


---

### Q75 · [Thinking] · `sql-with-python`

> **How do you safely run parameterized queries in Python with psycopg2? Why is building SQL with f-strings dangerous?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
**Parameterized queries** pass user input as separate parameters to the database driver, which handles escaping. The query template and the data never get concatenated into a single string before being sent to the database.

```python
# SAFE — parameterized
import psycopg2
conn = psycopg2.connect(dsn)
cur = conn.cursor()
cur.execute("SELECT * FROM users WHERE email = %s", (user_email,))

# UNSAFE — f-string SQL injection risk
cur.execute(f"SELECT * FROM users WHERE email = '{user_email}'")
# If user_email = "'; DROP TABLE users; --" you've lost the table.

# Bulk insert with executemany
cur.executemany(
    "INSERT INTO events (user_id, event) VALUES (%s, %s)",
    [(1, 'login'), (2, 'purchase')]
)
```

f-string SQL is dangerous because a malicious input like `' OR '1'='1` or `'; DROP TABLE users; --` gets executed as SQL. The database cannot distinguish your intended query from injected commands.

**How to think through this:**
1. Always use `%s` placeholders (psycopg2 style) or named parameters — never concatenate.
2. Use `executemany` for batch inserts rather than looping `execute` — it's faster and still parameterized.
3. For dynamic table or column names (which cannot be parameterized), use `psycopg2.sql.Identifier` to safely quote identifiers.

**Key takeaway:** Parameterized queries are non-negotiable — they are the only reliable defense against SQL injection.

</details>

📖 **Theory:** [sql-with-python](./09_real_world/sql_with_python.md#sql-with-python--connecting-your-code-to-your-database)


## 🔵 Tier 4 — Interview / Scenario

---

### Q76 · [Interview] · `explain-window-functions`

> **Explain window functions to a junior developer. Use an analogy. What makes them fundamentally different from GROUP BY?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
Imagine a classroom where the teacher calculates the class average grade. With GROUP BY, the teacher collapses all students into a single summary row — you no longer see individual students, just the aggregate. Window functions are like the teacher writing the class average on every student's own report card. Each student's row stays intact, but now carries a computed value derived from the group.

**GROUP BY** eliminates detail rows and returns one row per group. **Window functions** (using the `OVER` clause) compute across a set of rows related to the current row, but every original row survives in the output.

```sql
-- GROUP BY: one row per department
SELECT dept, AVG(salary) FROM employees GROUP BY dept;

-- Window: every employee row + their department average
SELECT name, dept, salary,
       AVG(salary) OVER (PARTITION BY dept) AS dept_avg
FROM employees;
```

Common window functions: `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`, `LAG()`, `LEAD()`, `SUM() OVER`, `AVG() OVER`.

**How to think through this:**
1. Ask: do I need one summary row per group? Use GROUP BY. Do I need the original rows plus a computed value? Use a window function.
2. The `PARTITION BY` inside OVER is like GROUP BY but scoped to the window computation only.
3. `ORDER BY` inside OVER defines a frame — useful for running totals and moving averages.

**Key takeaway:** Window functions compute across a group while keeping every row — GROUP BY compresses rows into one per group.

</details>

📖 **Theory:** [explain-window-functions](./03_aggregation/window_functions.md#window-functions)


---

### Q77 · [Interview] · `compare-cte-subquery`

> **Compare CTEs and subqueries. When does each win on readability, reuse, and optimizer behavior?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
A **subquery** is a query nested inline inside another query. A **CTE** (Common Table Expression, using `WITH`) is a named temporary result set defined before the main query.

**Readability**: CTEs win for complex multi-step logic. Naming each step (`WITH monthly_revenue AS (...), growth AS (...)`) reads like pseudocode. Subqueries become unreadable when deeply nested.

**Reuse**: CTEs can be referenced multiple times in the same query by name. A subquery must be copy-pasted if needed more than once (or use a lateral join).

**Optimizer behavior (important nuance)**:
- In PostgreSQL prior to version 12, CTEs were **optimization fences** — the planner could not push WHERE clauses into them. They were always materialized (executed once, result stored).
- Since PostgreSQL 12, non-recursive CTEs are **inlined by default** unless you add `MATERIALIZED` keyword. The optimizer can now see through them.
- Subqueries are always subject to optimization — the planner may merge them with the outer query.
- Use `WITH cte AS MATERIALIZED (...)` when you explicitly want the CTE computed once (e.g., an expensive subquery referenced twice).

**How to think through this:**
1. Default to CTEs for readability when a query has more than two logical steps.
2. If the CTE is referenced twice and is expensive, check whether adding `MATERIALIZED` improves performance.
3. For simple one-off filters, an inline subquery is fine — don't over-engineer.

**Key takeaway:** CTEs win on readability and reuse; since PostgreSQL 12 the optimizer inlines them by default, removing the old performance penalty.

</details>

📖 **Theory:** [compare-cte-subquery](./06_advanced_queries/ctes.md#cte-vs-subquery--when-to-use-each)


---

### Q78 · [Interview] · `explain-indexes`

> **Explain B-tree indexes to a non-technical manager using an analogy. What does an index cost?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
Think of a database table as a textbook with 10,000 pages. Without an index, finding every mention of "revenue" means reading every single page. That is a **sequential scan**. A **B-tree index** is the book's index at the back — it alphabetically lists every term with the page numbers where it appears. Instead of reading 10,000 pages, you flip to "R", find "revenue", and go directly to pages 42, 107, and 891.

B-tree stands for "balanced tree" — it keeps entries sorted in a tree structure where every lookup takes roughly the same number of steps (logarithmic time) regardless of table size.

**The cost of an index:**
- Every INSERT, UPDATE, or DELETE must also update the index — writes become slightly slower.
- Indexes consume disk space.
- Too many indexes on a write-heavy table slows down writes and increases bloat.

Rule of thumb: index columns that appear in WHERE clauses, JOIN conditions, and ORDER BY on large tables with selective filters.

**How to think through this:**
1. Indexes make reads faster but writes slower — it's a deliberate trade-off.
2. A table with 500 rows rarely needs an index — the overhead isn't worth it.
3. The most impactful indexes are on foreign key columns and high-cardinality filter columns.

**Key takeaway:** An index is a sorted lookup structure that trades slower writes for dramatically faster reads on large tables.

</details>

📖 **Theory:** [explain-indexes](./04_schema_design/indexes.md#indexes)


---

### Q79 · [Interview] · `compare-joins`

> **Compare INNER JOIN, LEFT JOIN, and FULL OUTER JOIN. When would you pick each?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
Think of two circles in a Venn diagram — table A on the left, table B on the right.

- **INNER JOIN**: returns only rows where a match exists in both tables. The intersection of the two circles. Use when you only care about records that have a corresponding entry in both tables (e.g., orders with a valid customer).
- **LEFT JOIN** (LEFT OUTER JOIN): returns all rows from the left table, plus matching rows from the right table. Rows with no match get NULL for all right-side columns. The full left circle. Use when you want all records from the primary table regardless of whether a related record exists (e.g., all customers, even those with zero orders).
- **FULL OUTER JOIN**: returns all rows from both tables, with NULLs where there is no match on either side. Both circles combined. Use when you need to identify records that exist in A but not B, B but not A, and matched pairs all in one result (e.g., reconciling two data sources).
- **RIGHT JOIN**: same as LEFT JOIN with tables swapped. Rarely used in practice — most developers rewrite it as a LEFT JOIN for consistency.

**How to think through this:**
1. Start with what you want as your "anchor" table — the one where you want all rows preserved. That table goes on the LEFT side of a LEFT JOIN.
2. If missing rows from either table are meaningful (data quality check, reconciliation), use FULL OUTER JOIN.
3. If you only care about confirmed matches, use INNER JOIN — it produces fewer rows and is easier to reason about.

**Key takeaway:** LEFT JOIN is the most commonly needed join in reporting — it preserves all rows from your primary table and surfaces gaps as NULLs.

</details>

📖 **Theory:** [compare-joins](./05_joins/inner_join.md#inner-join--the-matchmaker)


---

### Q80 · [Interview] · `explain-acid`

> **Explain ACID to a product manager who asks "why can't we just skip transactions for performance?"**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
Imagine a bank transfer: take $100 from Alice's account and add it to Bob's account. That is two separate writes. Without transactions, the server could crash after the debit but before the credit — Alice loses $100 and Bob gets nothing. ACID prevents that class of disaster.

**ACID** stands for:

- **Atomicity**: the transaction is all-or-nothing. Either both writes succeed or neither does. No partial state.
- **Consistency**: the database moves from one valid state to another. Rules (constraints, foreign keys) are enforced.
- **Isolation**: concurrent transactions don't see each other's in-progress work. One transaction's half-written data is invisible to another.
- **Durability**: once a transaction commits, it survives crashes. The data is on disk.

"Skip transactions for performance" is a false trade-off. The performance cost of transactions is small (milliseconds). The cost of ACID violations is corrupted data, incorrect balances, duplicate orders, or phantom records — which require expensive incident response, data backups, and customer refunds.

**How to think through this:**
1. The "performance" argument only applies to non-critical, idempotent operations where partial failure is acceptable (e.g., logging clicks). Even then, most databases handle this efficiently.
2. Ask the PM: "What is the cost when we ship a customer two orders but charge them once?" That is the real cost of skipping transactions.
3. Modern databases (PostgreSQL, MySQL InnoDB) implement ACID with minimal overhead for typical workloads — the trade-off is rarely worth it.

**Key takeaway:** ACID is not a performance tax — it is the guarantee that your data stays correct when servers crash and users click simultaneously.

</details>

📖 **Theory:** [explain-acid](./07_data_modification/transactions.md#acid-properties-explained-simply)


---

### Q81 · [Interview] · `compare-view-matview`

> **Compare views and materialized views. What are the trade-offs and when would you use each in a real system?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
A **view** is a saved SQL query — a named shortcut. Every time you query a view, the underlying query runs from scratch against the live base tables. The view stores no data itself; it is always fresh and always reflects current state.

A **materialized view** stores the query result physically on disk, like a snapshot. Querying it is fast because it reads pre-computed data. But it goes stale — it only updates when you explicitly run `REFRESH MATERIALIZED VIEW`.

**Trade-offs:**

| | View | Materialized View |
|---|---|---|
| Data freshness | Always current | Stale until refreshed |
| Query performance | Depends on base query | Fast (pre-computed) |
| Storage | None | Disk space required |
| Indexes | Not indexable | Can add indexes |
| Refresh cost | None (no data stored) | Refresh can be expensive |

**When to use each:**
- **View**: permission boundaries (expose only certain columns to an analyst role), simplify complex joins used across many queries, no tolerance for stale data.
- **Materialized view**: expensive aggregation queries (daily revenue summaries, large joins) that don't need real-time data. Dashboard queries where users can tolerate data that is 1 hour or 1 day old. Use `REFRESH MATERIALIZED VIEW CONCURRENTLY` to avoid locking reads during refresh.

**How to think through this:**
1. If the query takes 30+ seconds and users can tolerate hourly data, a materialized view dramatically improves user experience.
2. Schedule refreshes during off-peak hours or trigger them after ETL jobs complete.
3. Never use a materialized view for financial or transactional data that must be real-time.

**Key takeaway:** A view is a live query alias; a materialized view is a cached snapshot — choose based on whether staleness is acceptable for your use case.

</details>

📖 **Theory:** [compare-view-matview](./09_real_world/views.md#views--a-window-into-your-data)


---

### Q82 · [Interview] · `compare-norm-denorm`

> **Compare normalization and denormalization. Which would you choose for a reporting system and why?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
**Normalization** organizes data to eliminate redundancy by splitting it into related tables. A customer's name is stored once in `customers`, not repeated in every `orders` row. This protects data integrity (update one place, not thousands), saves storage, and avoids update anomalies.

**Denormalization** intentionally introduces redundancy — duplicating columns, pre-joining tables, or storing computed values — to make reads faster. The `orders` table might store `customer_name` directly alongside `customer_id` so a report query doesn't need a JOIN.

**For a transactional system (OLTP)**: normalize. Frequent inserts and updates need integrity guarantees. Redundancy causes inconsistencies.

**For a reporting system (OLAP)**: denormalize or use a dimensional model (star/snowflake schema). Reports join many tables across millions of rows. Pre-joining and pre-aggregating data into wide fact tables dramatically speeds up queries. Tools like BigQuery, Redshift, and Snowflake are optimized for columnar reads of denormalized data.

**Practical middle ground**: normalize the transactional source, then run an ETL pipeline to a denormalized reporting layer (data warehouse or materialized views). This is the standard modern data stack pattern.

**How to think through this:**
1. Ask: who writes to this table and how often? Writes favor normalization.
2. Ask: what queries run against this table and how complex are they? Complex analytical reads favor denormalization.
3. The two are not mutually exclusive — use both by maintaining separate operational and analytical stores.

**Key takeaway:** Normalize for data integrity in transactional systems; denormalize for query performance in reporting systems — the mature pattern uses both with an ETL layer in between.

</details>

📖 **Theory:** [compare-norm-denorm](./04_schema_design/normalization.md#when-to-denormalize)


---

### Q83 · [Design] · `slow-table-scenario`

> **Production scenario: a 50 million row `orders` table is getting slow on queries that were fast six months ago. Walk through your investigation and fixes.**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
This is a systematic performance investigation. Here is the full walkthrough:

**Step 1 — Identify the slow queries**
Use `pg_stat_statements` to find queries with the highest `total_exec_time` or `mean_exec_time` against the orders table:
```sql
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
WHERE query ILIKE '%orders%'
ORDER BY total_exec_time DESC
LIMIT 10;
```

**Step 2 — EXPLAIN ANALYZE the top offenders**
For each slow query, run `EXPLAIN (ANALYZE, BUFFERS)` to see the actual plan, row estimates, and buffer hits vs. disk reads.

**Step 3 — Common findings and fixes**

- **Seq Scan on a filtered column**: add an index. For time-range queries, a B-tree index on `created_at` is usually the fix.
- **Stale statistics (estimated rows far off from actual)**: run `ANALYZE orders;` to rebuild statistics. May also need to adjust `default_statistics_target`.
- **Index bloat**: if the table has high churn (lots of updates/deletes), indexes may be bloated. Run `REINDEX CONCURRENTLY idx_orders_created_at`.
- **Table bloat**: run `VACUUM ANALYZE orders;`. Check autovacuum is keeping up with `pg_stat_user_tables`.
- **Missing composite index**: a query filtering on both `status` and `created_at` may benefit from a composite index `(status, created_at)`.
- **SELECT * fetching unnecessary columns**: large JSON or text columns add I/O. Switch to explicit column lists.
- **Partition the table**: for 50M rows with time-based access patterns, consider range partitioning by month on `created_at`. Queries scoped to recent months only scan the relevant partition.

**Step 4 — Monitor after each fix**
Re-run `EXPLAIN ANALYZE` after each change. Watch `pg_stat_statements` over 24 hours to confirm mean execution time drops.

**How to think through this:**
1. Never guess — start with evidence from `pg_stat_statements` and `EXPLAIN ANALYZE`.
2. Fix one thing at a time and measure — stacking changes makes causality unclear.
3. Prevention: set up autovacuum alerts, track index bloat weekly, and add `pg_stat_statements` to your monitoring dashboard.

**Key takeaway:** Systematic investigation with `pg_stat_statements` and `EXPLAIN ANALYZE` beats guessing — identify the bottleneck before touching anything.

</details>

📖 **Theory:** [slow-table-scenario](./08_performance/query_optimization.md#query-optimization--making-slow-queries-fast)


---

### Q84 · [Design] · `duplicate-rows-scenario`

> **Production scenario: a race condition caused 50,000 duplicate order rows. How do you find them and safely remove them?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
This is a data recovery operation. Safety and auditability matter as much as speed.

**Step 1 — Find the duplicates**
Use a CTE to identify which `order_id` values appear more than once:
```sql
SELECT order_id, COUNT(*) AS cnt
FROM orders
GROUP BY order_id
HAVING COUNT(*) > 1;
```

To see the actual duplicate rows including their internal `ctid` (physical row identifier in PostgreSQL):
```sql
SELECT ctid, order_id, created_at, status
FROM orders
WHERE order_id IN (
  SELECT order_id FROM orders GROUP BY order_id HAVING COUNT(*) > 1
)
ORDER BY order_id, created_at;
```

**Step 2 — Decide which row to keep**
Common strategies: keep the earliest `created_at`, keep the row with the most complete data, or keep the row with the lowest `ctid`. Define this rule clearly before deleting anything.

**Step 3 — Safe deletion with a backup**
Never delete directly. First, move duplicates to a backup table:
```sql
CREATE TABLE orders_duplicates_backup AS
SELECT * FROM orders
WHERE ctid NOT IN (
  SELECT MIN(ctid)
  FROM orders
  GROUP BY order_id
);
```

Then delete using the same logic:
```sql
DELETE FROM orders
WHERE ctid NOT IN (
  SELECT MIN(ctid)
  FROM orders
  GROUP BY order_id
);
```

Wrap the DELETE in a transaction, verify the count, then COMMIT:
```sql
BEGIN;
  DELETE FROM orders WHERE ctid NOT IN (...);
  -- check: SELECT COUNT(*) FROM orders;
COMMIT;  -- or ROLLBACK if counts look wrong
```

**Step 4 — Prevent recurrence**
Add a UNIQUE constraint on `order_id` (or the natural key that should be unique). If the race condition is in application code, add database-level protection:
```sql
ALTER TABLE orders ADD CONSTRAINT orders_order_id_unique UNIQUE (order_id);
```

**How to think through this:**
1. Always back up before deleting — create `orders_duplicates_backup` first, not after.
2. Use a transaction with an explicit COUNT check before committing the DELETE.
3. Fix the root cause (missing unique constraint, application-level race) immediately after the cleanup.

**Key takeaway:** Backup first, delete inside a transaction, verify counts before committing, then add the constraint that should have been there from the start.

</details>

📖 **Theory:** [duplicate-rows-scenario](./06_advanced_queries/ctes.md#ctes--sticky-notes-on-a-whiteboard)


---

### Q85 · [Design] · `blocking-transaction-scenario`

> **Production scenario: a long-running transaction is blocking all writes to a critical table. What caused it and how do you resolve it?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
This is a lock contention emergency. The priority is to identify and terminate the blocker with minimal collateral damage.

**Step 1 — Find the blocking transaction**
```sql
SELECT pid, now() - pg_stat_activity.query_start AS duration,
       query, state, wait_event_type, wait_event
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;
```

Look for a long-running query in `state = 'idle in transaction'` — this means the application opened a transaction, did some work, and never committed or rolled back. The transaction holds its locks indefinitely.

To see what is blocking what:
```sql
SELECT blocked.pid AS blocked_pid,
       blocked.query AS blocked_query,
       blocking.pid AS blocking_pid,
       blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid));
```

**Step 2 — Terminate the blocker**
First try a graceful cancel (stops the query but keeps the connection):
```sql
SELECT pg_cancel_backend(blocking_pid);
```

If that doesn't work (e.g., it's `idle in transaction`, not actively running), terminate the connection:
```sql
SELECT pg_terminate_backend(blocking_pid);
```

**Common causes:**
- Application opened a transaction, did a slow external call (API, file I/O) inside it, never committed.
- Application crashed mid-transaction and the connection was not cleaned up.
- A schema migration (ALTER TABLE) that holds an ACCESS EXCLUSIVE lock.
- An explicit LOCK TABLE command that was never released.

**Prevention:**
- Set `idle_in_transaction_session_timeout` in PostgreSQL to auto-terminate stale transactions after N seconds.
- Set connection timeouts in the application layer.
- Never do non-database work (API calls, file I/O) inside an open transaction.

**How to think through this:**
1. `pg_stat_activity` + `pg_blocking_pids` gives you the full picture in under 60 seconds.
2. Try `pg_cancel_backend` first — it is less disruptive than termination.
3. After the incident, set `idle_in_transaction_session_timeout = '30s'` to prevent recurrence.

**Key takeaway:** `idle in transaction` is the most common blocker — terminate it with `pg_terminate_backend` and set `idle_in_transaction_session_timeout` to prevent it happening again.

</details>

📖 **Theory:** [blocking-transaction-scenario](./07_data_modification/transactions.md#transactions--all-or-nothing)


---

### Q86 · [Design] · `bulk-delete-scenario`

> **You need to delete 200 million old log rows from a live production table without killing the database. How?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
A single `DELETE FROM logs WHERE created_at < '2023-01-01'` on 200M rows will: hold a massive transaction lock for hours, generate enormous WAL (write-ahead log) traffic, potentially crash the database, and definitely impact application performance. Never do this.

**The correct approach: batched deletes**

Delete in small batches with a sleep between each to let autovacuum keep up and avoid lock buildup:

```sql
DO $$
DECLARE
  deleted INT;
BEGIN
  LOOP
    DELETE FROM logs
    WHERE id IN (
      SELECT id FROM logs
      WHERE created_at < '2023-01-01'
      LIMIT 10000
    );
    GET DIAGNOSTICS deleted = ROW_COUNT;
    EXIT WHEN deleted = 0;
    PERFORM pg_sleep(0.1);  -- brief pause to let autovacuum breathe
  END LOOP;
END $$;
```

Or from application code / a cron job that runs this loop over hours or days.

**Alternative: table partitioning (best long-term solution)**

If logs are range-partitioned by month on `created_at`, dropping old partitions is instantaneous and lock-free:
```sql
DROP TABLE logs_2022_12;  -- drops the whole partition, no row-by-row DELETE
```

This is the industry-standard approach for time-series log data and is orders of magnitude faster.

**Other considerations:**
- Archive before deleting: copy rows to S3 (via `COPY TO` or `aws_s3` extension) before the DELETE for compliance/audit.
- Run during off-peak hours if batched deletes are unavoidable on an unpartitioned table.
- Monitor autovacuum progress after deletion — 200M dead tuples need to be cleaned up.

**How to think through this:**
1. Never delete 200M rows in a single transaction on a live table — batch at 5,000–50,000 rows per transaction.
2. If this will be a recurring operation, partition the table now and make future deletions a DROP PARTITION.
3. Add a small sleep between batches to prevent sustained I/O saturation.

**Key takeaway:** Batch deletes in small transactions with pauses between them; the long-term fix is table partitioning so old data can be dropped with a single DDL statement.

</details>

📖 **Theory:** [bulk-delete-scenario](./07_data_modification/update_and_delete.md#update-and-delete--correcting-and-removing-records)


---

### Q87 · [Design] · `schema-design-twitter`

> **Design a minimal schema for a Twitter-like app: users, posts, follows, likes. What tables, primary keys, foreign keys, and indexes would you create?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**

```sql
CREATE TABLE users (
  id          BIGSERIAL PRIMARY KEY,
  username    TEXT NOT NULL UNIQUE,
  email       TEXT NOT NULL UNIQUE,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE posts (
  id          BIGSERIAL PRIMARY KEY,
  user_id     BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  content     TEXT NOT NULL CHECK (char_length(content) <= 280),
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE follows (
  follower_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  followee_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (follower_id, followee_id),
  CHECK (follower_id != followee_id)  -- can't follow yourself
);

CREATE TABLE likes (
  user_id    BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  post_id    BIGINT NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (user_id, post_id)  -- one like per user per post
);

-- Indexes
CREATE INDEX idx_posts_user_id ON posts(user_id);               -- user's own posts
CREATE INDEX idx_posts_created_at ON posts(created_at DESC);    -- timeline ordering
CREATE INDEX idx_follows_followee_id ON follows(followee_id);   -- "who follows me"
CREATE INDEX idx_likes_post_id ON likes(post_id);               -- like count per post
```

**Design decisions:**
- `follows` and `likes` use **composite primary keys** — this enforces uniqueness (no duplicate follows/likes) and serves as the index for the most common lookup direction.
- `ON DELETE CASCADE` propagates deletes — deleting a user removes their posts, follows, and likes automatically.
- `CHECK (follower_id != followee_id)` prevents self-follows at the database level.
- `posts.created_at DESC` index supports the home timeline query efficiently.

**How to think through this:**
1. Identify the entities (users, posts) and the relationships (follows = user-to-user, likes = user-to-post).
2. Many-to-many relationships (follows, likes) become junction tables with composite primary keys.
3. Think about the most common read queries and add indexes to support them.

**Key takeaway:** Junction tables with composite primary keys are the idiomatic pattern for many-to-many relationships — they enforce uniqueness and index both directions simultaneously.

</details>

📖 **Theory:** [schema-design-twitter](./04_schema_design/tables_and_constraints.md#schema-diagram)


---

### Q88 · [Design] · `schema-design-booking`

> **Design a hotel booking schema: hotels, rooms, bookings, availability. What constraints prevent double-bookings?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**

```sql
CREATE TABLE hotels (
  id         BIGSERIAL PRIMARY KEY,
  name       TEXT NOT NULL,
  city       TEXT NOT NULL
);

CREATE TABLE rooms (
  id         BIGSERIAL PRIMARY KEY,
  hotel_id   BIGINT NOT NULL REFERENCES hotels(id) ON DELETE CASCADE,
  room_number TEXT NOT NULL,
  room_type  TEXT NOT NULL,   -- 'single', 'double', 'suite'
  price_per_night NUMERIC(10,2) NOT NULL,
  UNIQUE (hotel_id, room_number)
);

CREATE TABLE bookings (
  id           BIGSERIAL PRIMARY KEY,
  room_id      BIGINT NOT NULL REFERENCES rooms(id),
  guest_name   TEXT NOT NULL,
  check_in     DATE NOT NULL,
  check_out    DATE NOT NULL,
  status       TEXT NOT NULL DEFAULT 'confirmed'
                 CHECK (status IN ('confirmed', 'cancelled', 'completed')),
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  CHECK (check_out > check_in)  -- must check out after check in
);

-- The double-booking prevention constraint (PostgreSQL EXCLUDE)
CREATE EXTENSION IF NOT EXISTS btree_gist;

ALTER TABLE bookings
ADD CONSTRAINT no_double_booking
EXCLUDE USING gist (
  room_id WITH =,
  daterange(check_in, check_out) WITH &&
)
WHERE (status = 'confirmed');
```

**The double-booking constraint explained:**
PostgreSQL's `EXCLUDE` constraint with `btree_gist` extension blocks any two confirmed bookings for the same `room_id` where their date ranges **overlap** (`&&` is the overlap operator for ranges). Cancelled bookings are excluded from the constraint (`WHERE status = 'confirmed'`), so cancellations free up the dates.

**Alternative without btree_gist** (portable, application-level):
Use a serializable transaction that checks for overlapping bookings before inserting:
```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT COUNT(*) FROM bookings
WHERE room_id = $1
  AND status = 'confirmed'
  AND check_in < $3 AND check_out > $2;
-- if 0, proceed with INSERT
COMMIT;
```

**How to think through this:**
1. Double-booking requires checking date range overlap — the condition is `existing.check_in < new.check_out AND existing.check_out > new.check_in`.
2. The `EXCLUDE` constraint enforces this at the database level, making it race-condition-safe.
3. Without the extension, use `SERIALIZABLE` isolation to prevent concurrent inserts from both passing the overlap check.

**Key takeaway:** PostgreSQL's EXCLUDE constraint with range overlap is the correct database-level solution for preventing double-bookings — it is enforced atomically unlike application-level checks.

</details>

📖 **Theory:** [schema-design-booking](./04_schema_design/tables_and_constraints.md#schema-diagram)


---

### Q89 · [Design] · `fraud-detection-query`

> **Write a query to detect suspicious transactions: the same card used in two different countries within 10 minutes.**

<details>
<summary>💡 Show Answer</summary>

**Answer:**

```sql
-- Detect: same card, different country, within 10 minutes of each other
SELECT
  t1.transaction_id  AS txn_1,
  t2.transaction_id  AS txn_2,
  t1.card_id,
  t1.country         AS country_1,
  t2.country         AS country_2,
  t1.txn_at          AS time_1,
  t2.txn_at          AS time_2,
  EXTRACT(EPOCH FROM (t2.txn_at - t1.txn_at)) / 60 AS minutes_apart
FROM transactions t1
JOIN transactions t2
  ON  t1.card_id  = t2.card_id
  AND t1.country != t2.country
  AND t2.txn_at   > t1.txn_at
  AND t2.txn_at  <= t1.txn_at + INTERVAL '10 minutes'
ORDER BY t1.card_id, t1.txn_at;
```

**Alternative using LAG() window function** (more efficient for ordered scan):

```sql
WITH ordered_txns AS (
  SELECT
    transaction_id,
    card_id,
    country,
    txn_at,
    LAG(country) OVER (PARTITION BY card_id ORDER BY txn_at) AS prev_country,
    LAG(txn_at)  OVER (PARTITION BY card_id ORDER BY txn_at) AS prev_txn_at
  FROM transactions
)
SELECT *
FROM ordered_txns
WHERE country    != prev_country
  AND prev_txn_at IS NOT NULL
  AND txn_at <= prev_txn_at + INTERVAL '10 minutes';
```

The self-join approach is conceptually clear but expensive on large tables. The LAG window function approach is more efficient — it does a single ordered pass per card rather than a cross-join.

**How to think through this:**
1. The core logic: for each card, find consecutive transactions where country changed and time gap is under 10 minutes.
2. Self-join with `t2.txn_at > t1.txn_at` avoids matching a row with itself and avoids duplicates (only look forward).
3. For production at scale: run this as a batch job on a window of the last 30 minutes of transactions, not the full history.

**Key takeaway:** The LAG window function is the production-appropriate pattern for sequential transaction analysis — it avoids the O(n²) self-join cost.

</details>

📖 **Theory:** [fraud-detection-query](./03_aggregation/window_functions.md#window-functions)


---

### Q90 · [Design] · `cohort-retention`

> **Write a query for monthly cohort retention: users who signed up in month X and returned in month X+1.**

<details>
<summary>💡 Show Answer</summary>

**Answer:**

```sql
-- Step 1: Assign each user to their signup cohort (month of first activity)
WITH cohorts AS (
  SELECT
    user_id,
    DATE_TRUNC('month', MIN(created_at)) AS cohort_month
  FROM users
  GROUP BY user_id
),

-- Step 2: Get all months each user was active (had any event/login)
user_activity AS (
  SELECT DISTINCT
    user_id,
    DATE_TRUNC('month', event_at) AS activity_month
  FROM events
),

-- Step 3: Join cohort to activity and calculate months since signup
cohort_activity AS (
  SELECT
    c.cohort_month,
    ua.activity_month,
    COUNT(DISTINCT c.user_id) AS active_users,
    EXTRACT(EPOCH FROM (ua.activity_month - c.cohort_month)) / 2592000 AS months_since_signup
  FROM cohorts c
  JOIN user_activity ua ON c.user_id = ua.user_id
  GROUP BY c.cohort_month, ua.activity_month
),

-- Step 4: Get cohort sizes (month 0 = 100%)
cohort_sizes AS (
  SELECT cohort_month, active_users AS cohort_size
  FROM cohort_activity
  WHERE months_since_signup = 0
)

-- Step 5: Calculate retention rate
SELECT
  ca.cohort_month,
  ca.months_since_signup::INT AS month_number,
  ca.active_users,
  cs.cohort_size,
  ROUND(100.0 * ca.active_users / cs.cohort_size, 1) AS retention_pct
FROM cohort_activity ca
JOIN cohort_sizes cs ON ca.cohort_month = cs.cohort_month
WHERE ca.months_since_signup IN (0, 1, 2, 3)  -- show first 4 months
ORDER BY ca.cohort_month, ca.months_since_signup;
```

**Sample output:**
```
cohort_month | month_number | active_users | cohort_size | retention_pct
-------------|--------------|--------------|-------------|---------------
2024-01-01   | 0            | 1200         | 1200        | 100.0
2024-01-01   | 1            | 480          | 1200        | 40.0
2024-01-01   | 2            | 312          | 1200        | 26.0
2024-02-01   | 0            | 950          | 950         | 100.0
2024-02-01   | 1            | 361          | 950         | 38.0
```

**How to think through this:**
1. First define the cohort: each user belongs to the month they first appeared. Use `MIN(created_at)` per user.
2. Then find all months each user was active and join back to their cohort to compute the lag.
3. Divide active users in month N by cohort size in month 0 to get the retention percentage.

**Key takeaway:** Cohort retention queries are a chain of CTEs — define cohorts, find activity, compute lag, then divide by cohort size for the percentage.

</details>

📖 **Theory:** [cohort-retention](./06_advanced_queries/ctes.md#real-world-example--monthly-cohort-retention)


## 🔴 Tier 5 — Critical Thinking

---

### Q91 · [Logical] · `predict-null-aggregate`

> **What does `AVG(score)` return when half the rows have NULL for `score`?**

```sql
CREATE TABLE results (student TEXT, score INT);
INSERT INTO results VALUES
  ('Alice', 90),
  ('Bob',   80),
  ('Carol', NULL),
  ('Dave',  NULL);

SELECT AVG(score) FROM results;
```

<details>
<summary>💡 Show Answer</summary>

**Answer:**
`AVG(score)` returns `85.0`, not `42.5`.

**Aggregate functions ignore NULLs.** AVG sums only the non-NULL values and divides by the count of non-NULL rows. So: `(90 + 80) / 2 = 85.0`. The two NULL rows are silently excluded from both the numerator and denominator.

If you want to treat NULLs as zero: `AVG(COALESCE(score, 0))` returns `(90 + 80 + 0 + 0) / 4 = 42.5`.

**How to think through this:**
1. In SQL, NULLs propagate through most arithmetic but are **excluded** from aggregate functions (SUM, AVG, COUNT, MIN, MAX) — this is intentional per the SQL standard.
2. `COUNT(*)` counts all rows including NULLs; `COUNT(score)` counts only non-NULL values.
3. Always ask: "are NULLs missing data or meaningful zeros?" — the answer determines whether to use COALESCE.

**Key takeaway:** Aggregate functions silently ignore NULLs — AVG(score) averages only the rows where score is not NULL.

</details>

📖 **Theory:** [predict-null-aggregate](./03_aggregation/aggregate_functions.md#aggregate-functions)


---

### Q92 · [Logical] · `predict-having-where`

> **Trace this query: GROUP BY + HAVING + WHERE — which filters run first and why?**

```sql
SELECT dept, COUNT(*) AS headcount
FROM employees
WHERE salary > 50000
GROUP BY dept
HAVING COUNT(*) > 5;
```

<details>
<summary>💡 Show Answer</summary>

**Answer:**
The logical execution order is:

1. **FROM**: load the `employees` table
2. **WHERE `salary > 50000`**: filter rows before grouping — removes employees earning 50,000 or less
3. **GROUP BY `dept`**: group the surviving rows by department
4. **HAVING `COUNT(*) > 5`**: filter groups after aggregation — removes departments with 5 or fewer qualifying employees
5. **SELECT**: compute and return `dept` and `COUNT(*)`

The WHERE clause runs on individual rows before any grouping occurs. HAVING runs on groups after aggregation. This means `COUNT(*)` in HAVING counts only the rows that passed the WHERE filter.

Common mistake: putting a condition in HAVING when it belongs in WHERE. `HAVING dept = 'Engineering'` works but forces the database to aggregate all departments first, then discard most of them. `WHERE dept = 'Engineering'` filters before grouping — much more efficient.

**How to think through this:**
1. Memorize the logical order: FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT.
2. WHERE cannot reference aliases defined in SELECT (SELECT hasn't run yet). HAVING can reference aggregates. ORDER BY can reference SELECT aliases.
3. If a filter doesn't involve an aggregate, it belongs in WHERE, not HAVING.

**Key takeaway:** WHERE filters rows before grouping; HAVING filters groups after aggregation — putting non-aggregate filters in HAVING is a common performance mistake.

</details>

📖 **Theory:** [predict-having-where](./03_aggregation/group_by_and_having.md#where-vs-having--side-by-side)


---

### Q93 · [Logical] · `predict-join-nulls`

> **What rows appear in a LEFT JOIN when the right table has NULLs in the join column?**

```sql
CREATE TABLE orders   (id INT, customer_id INT);
CREATE TABLE customers (id INT, name TEXT);

INSERT INTO orders    VALUES (1, 100), (2, NULL), (3, 999);
INSERT INTO customers VALUES (100, 'Alice'), (200, 'Bob');

SELECT o.id, c.name
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.id;
```

<details>
<summary>💡 Show Answer</summary>

**Answer:**
```
id | name
---|------
1  | Alice   -- customer_id 100 matched
2  | NULL    -- customer_id is NULL: NULL = anything is UNKNOWN, no match
3  | NULL    -- customer_id 999: no matching customer row
```

All three orders appear because LEFT JOIN preserves all rows from the left table. But:
- Order 2 has `customer_id = NULL` — in SQL, `NULL = 100` evaluates to UNKNOWN (not TRUE), so no join match occurs. `name` is NULL.
- Order 3 has `customer_id = 999` — no customer with id=999 exists. `name` is NULL.

Both unmatched rows look identical in the output (NULL for all right-side columns), but for different reasons. To distinguish: order 2 has a NULL join key while order 3 has a valid but unresolvable key.

**How to think through this:**
1. LEFT JOIN never drops left-side rows — every left row appears at least once.
2. Any join condition involving NULL evaluates to UNKNOWN, which is treated as FALSE — no match.
3. To find rows with no match (including NULL join keys), filter `WHERE c.id IS NULL` after a LEFT JOIN.

**Key takeaway:** NULL in a join column never matches anything — not even another NULL — because NULL comparisons always evaluate to UNKNOWN.

</details>

📖 **Theory:** [predict-join-nulls](./05_joins/left_right_outer_join.md#left-right-and-full-outer-join--the-inclusive-matchmaker)


---

### Q94 · [Debug] · `debug-wrong-group-by`

> **Find the bug: this query uses a non-aggregated column not in GROUP BY.**

```sql
SELECT dept, name, COUNT(*) AS headcount
FROM employees
GROUP BY dept;
```

<details>
<summary>💡 Show Answer</summary>

**Answer:**
The bug is that `name` appears in SELECT but is not in GROUP BY and is not wrapped in an aggregate function. This is a **GROUP BY violation**.

In standard SQL (and PostgreSQL), every column in SELECT must either appear in GROUP BY or be wrapped in an aggregate (COUNT, SUM, MAX, etc.). Since `dept` groups multiple employees into one row, there is no single `name` value to display for that group — the database does not know which employee's name to show.

PostgreSQL will throw:
```
ERROR: column "employees.name" must appear in the GROUP BY clause
or be used in an aggregate function
```

**The fix depends on intent:**

Option A — Show any one name per department (non-deterministic):
```sql
SELECT dept, MAX(name) AS example_name, COUNT(*) AS headcount
FROM employees
GROUP BY dept;
```

Option B — Show all employee names (don't aggregate):
```sql
SELECT dept, name
FROM employees
ORDER BY dept;
```

Option C — Show headcount per department (drop name from SELECT):
```sql
SELECT dept, COUNT(*) AS headcount
FROM employees
GROUP BY dept;
```

**How to think through this:**
1. For every column in SELECT, ask: "is this in GROUP BY, or wrapped in an aggregate?" If neither, it's a bug.
2. MySQL in non-strict mode silently allows this and returns an arbitrary value — a dangerous behavior that masks bugs.
3. PostgreSQL enforces the standard — the error message is clear and actionable.

**Key takeaway:** Every non-aggregated column in SELECT must appear in GROUP BY — if it doesn't, you have either the wrong SELECT list or the wrong GROUP BY.

</details>

📖 **Theory:** [debug-wrong-group-by](./03_aggregation/group_by_and_having.md#group-by-and-having)


---

### Q95 · [Debug] · `debug-distinct-perf`

> **Find the anti-pattern: DISTINCT is being used to hide a broken JOIN that produces duplicates.**

```sql
SELECT DISTINCT o.order_id, o.total
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id;
```

<details>
<summary>💡 Show Answer</summary>

**Answer:**
The anti-pattern is using `DISTINCT` to suppress duplicate `order_id` rows that are caused by the JOIN fan-out — each order with multiple line items produces one row per item, so `order_id=1` with 5 items appears 5 times before DISTINCT.

`DISTINCT` here is a **symptom fix**, not a root-cause fix. It hides the fact that the JOIN is returning more rows than intended.

The problems with this approach:
1. `DISTINCT` sorts or hashes the entire result set, adding unnecessary CPU and memory cost.
2. It obscures the data model — a reader cannot tell why DISTINCT is needed.
3. If `order_items` ever has legitimate duplicates you care about, DISTINCT silently hides them.

**The correct fix** depends on what you need:

If you only need order totals and don't need anything from `order_items`:
```sql
-- Don't join at all if you don't use columns from order_items
SELECT order_id, total FROM orders;
```

If you need to confirm at least one item exists (existence check):
```sql
SELECT o.order_id, o.total
FROM orders o
WHERE EXISTS (
  SELECT 1 FROM order_items oi WHERE oi.order_id = o.order_id
);
```

If you need aggregate data from items:
```sql
SELECT o.order_id, o.total, COUNT(oi.id) AS item_count
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
GROUP BY o.order_id, o.total;
```

**How to think through this:**
1. When you see DISTINCT, ask: "why does this query produce duplicates without it?" The answer usually reveals a JOIN that needs restructuring.
2. DISTINCT is legitimate for deduplicating unions or genuinely non-unique data — it is a red flag when papering over fan-out from a JOIN.
3. EXISTS or GROUP BY almost always expresses the intent more clearly than DISTINCT after a JOIN.

**Key takeaway:** DISTINCT after a JOIN is often a sign of a JOIN that produces unintended fan-out — fix the query structure instead of hiding duplicates.

</details>

📖 **Theory:** [debug-distinct-perf](./02_querying_basics/distinct_and_aliases.md#distinct-and-aliases)


---

### Q96 · [Debug] · `debug-update-no-where`

> **Find the disaster: an UPDATE without WHERE just ran in production. What do you do now?**

```sql
UPDATE users SET subscription_tier = 'free';
```

<details>
<summary>💡 Show Answer</summary>

**Answer:**
This statement set every user's `subscription_tier` to `'free'` — including paying customers. This is a **data corruption incident**.

**Immediate response (in order):**

1. **Stop the bleeding**: check if any application processes are writing to `users` right now. If so, consider putting the app in maintenance mode to prevent new writes that would complicate recovery.

2. **Check if it was in a transaction**:
```sql
-- If still in the same session and transaction was not committed:
ROLLBACK;
-- This reverts the UPDATE completely. Always use explicit transactions for risky operations.
```

3. **Restore from backup** (if the transaction was committed):
   - Identify the last backup before the incident.
   - Use **point-in-time recovery (PITR)** if WAL archiving is enabled — you can restore to the exact second before the bad UPDATE ran.
   - In PostgreSQL with WAL archiving: restore the base backup, then replay WAL up to `2024-01-15 14:32:59` (one second before the UPDATE).

4. **Surgical fix if backup is unavailable**:
   - Check if the original values exist in an audit/history table, CDC log, or application-level event store.
   - If you have a `users_audit` table with the previous `subscription_tier` values, join and restore:
```sql
UPDATE users u
SET subscription_tier = a.old_subscription_tier
FROM users_audit a
WHERE u.id = a.user_id
  AND a.changed_at = (SELECT MAX(changed_at) FROM users_audit WHERE user_id = u.id AND changed_at < 'incident_timestamp');
```

**Prevention:**
- Always use transactions for risky DML: `BEGIN; UPDATE ...; SELECT COUNT(*) WHERE ...; COMMIT;`
- Use `SET LOCAL lock_timeout = '5s'` to avoid long locks.
- In psql, set `\set AUTOCOMMIT off` during maintenance work.
- Add a `require_where` policy or use pgaudit to log all DML.
- Run destructive queries with `LIMIT 1` first to verify they target the right rows.

**How to think through this:**
1. The first question is always: "was this committed?" — an uncommitted transaction can be rolled back instantly.
2. If committed, the recovery path is PITR from WAL archives — this is why WAL archiving must be enabled in production.
3. The post-incident fix: add the WHERE clause as a required review step in runbook/change management.

**Key takeaway:** An UPDATE without WHERE is a full-table corruption — the only clean recovery is ROLLBACK (if still in transaction) or point-in-time recovery from WAL archives.

</details>

📖 **Theory:** [debug-update-no-where](./07_data_modification/update_and_delete.md#rule-3-never-run-updatedelete-without-where-in-production)


---

### Q97 · [Design] · `design-index-strategy`

> **Given this query pattern, which index would you create and why? When would you choose composite vs partial vs covering?**

```sql
-- Query pattern A: high-volume production query
SELECT id, email, created_at
FROM users
WHERE status = 'active' AND created_at > NOW() - INTERVAL '30 days';
```

<details>
<summary>💡 Show Answer</summary>

**Answer:**

**Composite index** — the best choice for this query:
```sql
CREATE INDEX idx_users_status_created ON users (status, created_at DESC);
```
This index lets PostgreSQL filter on `status = 'active'` first (eliminating most rows if only a fraction are active), then scan the `created_at` range within the active subset. Put the equality column first, the range column second.

**Partial index** — better if `status = 'active'` is almost always the filter:
```sql
CREATE INDEX idx_users_active_created ON users (created_at DESC)
WHERE status = 'active';
```
This is a smaller, faster index because it only contains active user rows. If 90% of users are inactive, the partial index is ~10% the size of a full index on the same column. Any query that doesn't include `WHERE status = 'active'` cannot use this index.

**Covering index** — best if this exact query runs millions of times and you want to avoid heap fetches:
```sql
CREATE INDEX idx_users_active_covering ON users (status, created_at DESC)
INCLUDE (id, email);
```
The `INCLUDE` clause adds `id` and `email` to the index leaf pages (but not the tree structure). PostgreSQL can satisfy the entire SELECT from the index without touching the table heap — called an **index-only scan**. This is the fastest possible path for a high-volume query.

**Decision framework:**
- Single column filter: plain B-tree index on that column.
- Multiple filter columns: composite index (equality columns first, range columns last).
- Query always uses the same selective WHERE predicate: partial index.
- Same query runs at massive scale and SELECT columns are few: covering index with INCLUDE.

**How to think through this:**
1. Check query frequency and selectivity — a query running 10,000 times/second is worth a covering index.
2. Partial indexes are smaller and faster but narrower — they only help queries that match the partial condition exactly.
3. Too many indexes slow writes — only add indexes you have evidence are needed from `EXPLAIN ANALYZE`.

**Key takeaway:** Put equality conditions before range conditions in composite indexes; use partial indexes when a selective predicate is always present; use INCLUDE for covering indexes on hot read paths.

</details>

📖 **Theory:** [design-index-strategy](./04_schema_design/indexes.md#indexes)


---

### Q98 · [Design] · `design-pagination`

> **OFFSET-based pagination breaks at scale. What are the two better alternatives and when do you use each?**

<details>
<summary>💡 Show Answer</summary>

**Answer:**
**The OFFSET problem:**
```sql
-- This gets slower as the page number grows
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 100000;
```
PostgreSQL must scan and discard 100,000 rows to reach page 5,001. At page 10,000, it scans 200,000 rows and throws most away. Performance degrades linearly with page number.

---

**Alternative 1 — Keyset Pagination (Cursor-based)**

```sql
-- First page
SELECT id, title, created_at FROM posts
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- Next page: use the last row's values as the cursor
SELECT id, title, created_at FROM posts
WHERE (created_at, id) < ('2024-01-15 10:30:00', 5234)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

The WHERE clause seeks directly to the cursor position using the index — no rows are skipped and discarded. Performance is O(1) regardless of how deep you are.

**When to use:** infinite scroll, API pagination, any case where users page forward sequentially (social feeds, search results, audit logs).

**Limitation:** cannot jump to arbitrary page numbers ("go to page 500"). Can only go forward (or backward with reversed cursor logic).

---

**Alternative 2 — Seek with Approximate Count + Bounded OFFSET**

For UIs that need page number navigation but the total result set is manageable (under 10,000 rows per filter), cap the OFFSET:

```sql
-- Use an estimated count (fast) instead of COUNT(*) (slow)
SELECT reltuples::BIGINT AS approx_count FROM pg_class WHERE relname = 'posts';

-- Limit total navigable pages (e.g., max 200 pages of 50 = 10,000 rows)
SELECT * FROM posts WHERE category = 'tech'
ORDER BY created_at DESC
LIMIT 50 OFFSET LEAST($offset, 9950);
```

Or materialize the full filtered result in a temp table/CTE and use OFFSET on that bounded set.

**When to use:** admin panels, search results where users rarely go beyond page 20, reports with bounded result sets.

---

**How to think through this:**
1. Keyset pagination is the production-grade solution for any feed or list where users scroll forward.
2. The cursor must include a unique column (like `id`) alongside the sort column to handle ties in `created_at`.
3. For APIs, encode the cursor as an opaque base64 string — never expose raw row values to clients.

**Key takeaway:** Keyset pagination replaces OFFSET with a WHERE clause on the last seen value, giving O(1) performance at any depth — use it for all sequential-scroll use cases.

</details>

📖 **Theory:** [design-pagination](./02_querying_basics/order_by_and_limit.md#offset--skip-rows-for-pagination)


---

### Q99 · [Design] · `design-second-highest`

> **Find the second highest salary — show 3 different SQL approaches and explain the trade-offs of each.**

```sql
-- Table: employees(id, name, salary)
```

<details>
<summary>💡 Show Answer</summary>

**Answer:**

**Approach 1 — Subquery with MAX**
```sql
SELECT MAX(salary) AS second_highest
FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);
```
Simple and readable. Works in all SQL databases. Returns NULL if there is no second distinct salary (only one unique value). Requires two passes over the table (two MAX scans).

**Approach 2 — OFFSET (limit + skip)**
```sql
SELECT DISTINCT salary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET 1;
```
Clean and intuitive. `DISTINCT` handles ties — if three people earn the top salary, OFFSET 1 skips all of them and returns the true second-highest distinct value. Returns no rows (not NULL) if there is no second distinct salary. Can be generalized to Nth highest by changing OFFSET.

**Approach 3 — Window function with DENSE_RANK**
```sql
SELECT name, salary
FROM (
  SELECT name, salary,
         DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
  FROM employees
) ranked
WHERE rnk = 2;
```
The most powerful approach — returns the full row (name AND salary), not just the salary value. `DENSE_RANK` assigns the same rank to ties and does not skip numbers (rank 1, 1, 2 — not 1, 1, 3 like `RANK()`). Can be extended to show all people tied for second place. Easily generalizable: change `rnk = 2` to any N.

**Trade-off summary:**

| Approach | Returns | Handles Ties | Generalizable | Returns Row Data |
|---|---|---|---|---|
| Subquery MAX | Salary only | Yes (MAX ignores ties) | Harder | No |
| OFFSET | Salary only | Yes (DISTINCT) | Easy (change N) | No |
| DENSE_RANK | Full row | Yes | Easy (change N) | Yes |

**How to think through this:**
1. In an interview, start with the subquery approach (shows you know the basics), then offer DENSE_RANK as the production-appropriate solution.
2. Clarify the tie-handling requirement before writing code — "second highest distinct salary" vs "second row by salary" are different problems.
3. DENSE_RANK is preferred when you need full row data or must handle N-th highest for arbitrary N.

**Key takeaway:** DENSE_RANK with a window function is the most flexible and production-correct approach for N-th highest problems — it handles ties and returns full row data.

</details>

📖 **Theory:** [design-second-highest](./03_aggregation/window_functions.md#window-functions)


---

### Q100 · [Design] · `design-running-report`

> **Design a query from scratch: "monthly revenue, month-over-month growth rate, and 3-month rolling average" — all in one query.**

<details>
<summary>💡 Show Answer</summary>

**Answer:**

```sql
WITH monthly_revenue AS (
  -- Step 1: Aggregate raw revenue by month
  SELECT
    DATE_TRUNC('month', order_date)  AS month,
    SUM(total_amount)                 AS revenue
  FROM orders
  WHERE status = 'completed'
  GROUP BY DATE_TRUNC('month', order_date)
),

revenue_with_lag AS (
  -- Step 2: Add previous month's revenue using LAG
  SELECT
    month,
    revenue,
    LAG(revenue) OVER (ORDER BY month) AS prev_revenue
  FROM monthly_revenue
),

final AS (
  -- Step 3: Compute growth rate and 3-month rolling average
  SELECT
    month,
    revenue,
    prev_revenue,

    -- Month-over-month growth rate
    CASE
      WHEN prev_revenue IS NULL OR prev_revenue = 0 THEN NULL
      ELSE ROUND(100.0 * (revenue - prev_revenue) / prev_revenue, 2)
    END AS mom_growth_pct,

    -- 3-month rolling average (current month + 2 prior months)
    ROUND(
      AVG(revenue) OVER (
        ORDER BY month
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
      ),
      2
    ) AS rolling_3mo_avg

  FROM revenue_with_lag
)

SELECT
  TO_CHAR(month, 'YYYY-MM')  AS month,
  revenue,
  mom_growth_pct,
  rolling_3mo_avg
FROM final
ORDER BY month;
```

**Sample output:**
```
month   | revenue   | mom_growth_pct | rolling_3mo_avg
--------|-----------|----------------|----------------
2024-01 | 100000.00 | NULL           | 100000.00
2024-02 | 120000.00 | 20.00          | 110000.00
2024-03 | 110000.00 | -8.33          | 110000.00
2024-04 | 130000.00 | 18.18          | 120000.00
2024-05 | 150000.00 | 15.38          | 130000.00
```

**Design decisions explained:**

- **LAG()** over the ordered monthly result gives previous month's revenue without a self-join.
- **CASE WHEN prev_revenue = 0** prevents division by zero on months with zero prior revenue.
- **AVG() OVER (ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)** defines a 3-row sliding window. `ROWS` mode counts physical rows; `RANGE` mode would group equal values — `ROWS` is correct here since each month is one row.
- The CTE chain (monthly → lag → compute) keeps each step readable and independently testable.

**How to think through this:**
1. Break the problem into layers: aggregate first, then add window computations on the aggregated result — you cannot use LAG directly on raw orders.
2. Always handle NULL and division-by-zero in growth rate calculations — month 1 has no prior month.
3. Test the window frame definition: `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` on month 1 returns a 1-row average, month 2 a 2-row average, month 3+ a 3-row average — confirm this is the intended behavior.

**Key takeaway:** Complex reporting queries are built as CTE chains — aggregate first, then layer window functions on the aggregated result, keeping each step independently readable and testable.

</details>

📖 **Theory:** [design-running-report](./03_aggregation/window_functions.md#the-running-scoreboard)

