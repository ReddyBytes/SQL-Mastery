# SQL Scenario-Based Interview Questions

> 25 real-world scenarios asked at top tech companies (FAANG, startups, and data-heavy interviews).
> Each question shows the schema, the ask, the solution, and what the interviewer is testing.

---

## Quick Index

| # | Scenario | Difficulty | Key Concept |
|---|----------|-----------|-------------|
| S1 | Customers who never ordered | 🟢 Beginner | LEFT JOIN anti-join |
| S2 | Find and delete duplicate emails | 🟢 Beginner | ROW_NUMBER + DELETE |
| S3 | Top 3 best-selling products | 🟢 Beginner | ORDER BY + LIMIT |
| S4 | Monthly revenue for the current year | 🟢 Beginner | DATE_TRUNC + GROUP BY |
| S5 | Users active in Jan but not Feb | 🟢 Beginner | EXCEPT / NOT EXISTS |
| S6 | Second-highest salary | 🔵 Intermediate | DENSE_RANK |
| S7 | Running total of sales | 🔵 Intermediate | SUM OVER (ORDER BY) |
| S8 | Month-over-month revenue growth | 🔵 Intermediate | LAG window function |
| S9 | Highest salary per department | 🔵 Intermediate | PARTITION BY |
| S10 | Users active every month for 6 months | 🔵 Intermediate | COUNT DISTINCT + HAVING |
| S11 | Detect gaps in sequential IDs | 🔵 Intermediate | LEAD window function |
| S12 | Pivot: monthly sales as columns | 🔵 Intermediate | CASE WHEN conditional agg |
| S13 | Diagnose a slow 50M-row query | 🔵 Intermediate | EXPLAIN ANALYZE |
| S14 | Design a Twitter/X schema | 🔴 Advanced | Schema design |
| S15 | Design a hotel booking schema | 🔴 Advanced | EXCLUDE constraint |
| S16 | Employees earning more than their manager | 🔴 Advanced | Self-join |
| S17 | Print the full org chart hierarchy | 🔴 Advanced | Recursive CTE |
| S18 | Monthly cohort retention analysis | 🔴 Advanced | Cohort CTE + pivot |
| S19 | Delete 200 million rows without killing the DB | 🔴 Advanced | Batch DELETE |
| S20 | Median salary without PERCENTILE_CONT | 🔴 Advanced | ROW_NUMBER + COUNT |
| S21 | Fraud: same card, two countries, 10 minutes | 🔴 Advanced | LAG + timestamp diff |
| S22 | Nth highest salary (generalised) | 🔴 Advanced | DENSE_RANK |
| S23 | Optimise a slow checkout query | 🔴 Advanced | Sargability |
| S24 | Average session duration from event log | 🔴 Advanced | LEAD on paired events |
| S25 | Products sold in every region (relational division) | 🔴 Advanced | EXCEPT / double NOT EXISTS |

---

## 🟢 Beginner Scenarios

---

### S1 — Customers Who Never Ordered

**Schema**
```sql
users(id, name, email, created_at)
orders(id, user_id, total, created_at)
```

**Question:** Return the names and emails of all users who have never placed an order.

**Solution**
```sql
SELECT u.name, u.email
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.id IS NULL;
```

**Alternative (NOT EXISTS)**
```sql
SELECT name, email
FROM users u
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);
```

**What the interviewer tests:** Understanding that LEFT JOIN + `WHERE right.id IS NULL` is the anti-join pattern. NOT EXISTS is often faster on large tables.

---

### S2 — Find and Delete Duplicate Emails

**Schema**
```sql
users(id, name, email, created_at)
```

**Question:** Find all users with duplicate emails. Then delete the duplicates, keeping only the earliest record (lowest `id`).

**Find duplicates**
```sql
SELECT email, COUNT(*) AS cnt
FROM users
GROUP BY email
HAVING COUNT(*) > 1;
```

**Delete duplicates (keep oldest)**
```sql
DELETE FROM users
WHERE id NOT IN (
    SELECT MIN(id)
    FROM users
    GROUP BY email
);
```

**PostgreSQL alternative using CTE + ROW_NUMBER**
```sql
WITH ranked AS (
    SELECT id,
           ROW_NUMBER() OVER (PARTITION BY email ORDER BY id) AS rn
    FROM users
)
DELETE FROM users
WHERE id IN (SELECT id FROM ranked WHERE rn > 1);
```

**What the interviewer tests:** Subquery in DELETE, ROW_NUMBER for deduplication, understanding of set-based operations.

---

### S3 — Top 3 Best-Selling Products

**Schema**
```sql
order_items(id, order_id, product_id, quantity, price)
products(id, name, category)
```

**Question:** Return the top 3 products by total quantity sold, with their names.

**Solution**
```sql
SELECT p.name, SUM(oi.quantity) AS total_sold
FROM order_items oi
JOIN products p ON oi.product_id = p.id
GROUP BY p.id, p.name
ORDER BY total_sold DESC
LIMIT 3;
```

**Follow-up:** Top 3 per category?
```sql
WITH ranked AS (
    SELECT p.category,
           p.name,
           SUM(oi.quantity) AS total_sold,
           RANK() OVER (PARTITION BY p.category ORDER BY SUM(oi.quantity) DESC) AS rnk
    FROM order_items oi
    JOIN products p ON oi.product_id = p.id
    GROUP BY p.category, p.id, p.name
)
SELECT category, name, total_sold
FROM ranked
WHERE rnk <= 3;
```

**What the interviewer tests:** Basic aggregation + JOIN, then escalates to top-N per group using window functions.

---

### S4 — Monthly Revenue for the Current Year

**Schema**
```sql
orders(id, user_id, total, created_at)
```

**Question:** Show total revenue per month for the current calendar year. Include months with no orders (show 0).

**Solution (without zero-fill)**
```sql
SELECT DATE_TRUNC('month', created_at) AS month,
       SUM(total) AS revenue
FROM orders
WHERE created_at >= DATE_TRUNC('year', NOW())
  AND created_at <  DATE_TRUNC('year', NOW()) + INTERVAL '1 year'
GROUP BY 1
ORDER BY 1;
```

**With zero-fill using generate_series**
```sql
WITH months AS (
    SELECT generate_series(
        DATE_TRUNC('year', NOW()),
        DATE_TRUNC('year', NOW()) + INTERVAL '11 months',
        INTERVAL '1 month'
    ) AS month
),
revenue AS (
    SELECT DATE_TRUNC('month', created_at) AS month, SUM(total) AS revenue
    FROM orders
    WHERE EXTRACT(year FROM created_at) = EXTRACT(year FROM NOW())
    GROUP BY 1
)
SELECT m.month, COALESCE(r.revenue, 0) AS revenue
FROM months m
LEFT JOIN revenue r ON m.month = r.month
ORDER BY 1;
```

**What the interviewer tests:** DATE_TRUNC, range filtering with indexes, handling sparse data with generate_series.

---

### S5 — Users Active in January but Not February

**Schema**
```sql
logins(user_id, login_date)
```

**Question:** Find users who logged in during January 2024 but did NOT log in during February 2024.

**Solution using EXCEPT**
```sql
SELECT DISTINCT user_id FROM logins
WHERE login_date >= '2024-01-01' AND login_date < '2024-02-01'

EXCEPT

SELECT DISTINCT user_id FROM logins
WHERE login_date >= '2024-02-01' AND login_date < '2024-03-01';
```

**Alternative using NOT EXISTS**
```sql
SELECT DISTINCT user_id
FROM logins
WHERE login_date >= '2024-01-01' AND login_date < '2024-02-01'
  AND NOT EXISTS (
      SELECT 1 FROM logins l2
      WHERE l2.user_id = logins.user_id
        AND l2.login_date >= '2024-02-01'
        AND l2.login_date < '2024-03-01'
  );
```

**What the interviewer tests:** Set operations (EXCEPT), date range filtering, understanding sargable predicates.

---

## 🔵 Intermediate Scenarios

---

### S6 — Second-Highest Salary

**Schema**
```sql
employees(id, name, department, salary)
```

**Question:** Find the employee with the second-highest salary. If multiple employees share that salary, return all of them.

**Solution**
```sql
WITH ranked AS (
    SELECT name, salary,
           DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM employees
)
SELECT name, salary
FROM ranked
WHERE rnk = 2;
```

**Why DENSE_RANK not RANK:** RANK skips numbers on ties (1, 1, 3). DENSE_RANK does not (1, 1, 2) — so the "2nd highest" is always `rnk = 2`.

**What the interviewer tests:** Knowing the difference between RANK, DENSE_RANK, and ROW_NUMBER.

---

### S7 — Running Total of Sales

**Schema**
```sql
daily_sales(sale_date, amount)
```

**Question:** Show each day's sales amount and the cumulative total up to that date.

**Solution**
```sql
SELECT sale_date,
       amount,
       SUM(amount) OVER (ORDER BY sale_date) AS running_total
FROM daily_sales
ORDER BY sale_date;
```

**Running total per product category**
```sql
SELECT category, sale_date, amount,
       SUM(amount) OVER (PARTITION BY category ORDER BY sale_date) AS category_running_total
FROM daily_sales
ORDER BY category, sale_date;
```

**What the interviewer tests:** Window frame default (RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) and PARTITION BY for grouped running totals.

---

### S8 — Month-over-Month Revenue Growth

**Schema**
```sql
orders(id, total, created_at)
```

**Question:** For each month, show total revenue and the percentage change from the previous month.

**Solution**
```sql
WITH monthly AS (
    SELECT DATE_TRUNC('month', created_at) AS month,
           SUM(total) AS revenue
    FROM orders
    GROUP BY 1
),
with_prev AS (
    SELECT month,
           revenue,
           LAG(revenue) OVER (ORDER BY month) AS prev_revenue
    FROM monthly
)
SELECT month,
       revenue,
       prev_revenue,
       ROUND(
           100.0 * (revenue - prev_revenue) / NULLIF(prev_revenue, 0),
           2
       ) AS pct_change
FROM with_prev
ORDER BY month;
```

**What the interviewer tests:** LAG function, NULLIF to avoid division by zero, CTE chaining.

---

### S9 — Highest Salary Per Department

**Schema**
```sql
employees(id, name, department, salary)
```

**Question:** Return each employee whose salary equals the highest salary in their department.

**Solution using window function**
```sql
WITH ranked AS (
    SELECT name, department, salary,
           MAX(salary) OVER (PARTITION BY department) AS dept_max
    FROM employees
)
SELECT name, department, salary
FROM ranked
WHERE salary = dept_max;
```

**Alternative using correlated subquery**
```sql
SELECT name, department, salary
FROM employees e
WHERE salary = (
    SELECT MAX(salary)
    FROM employees e2
    WHERE e2.department = e.department
);
```

**What the interviewer tests:** PARTITION BY for per-group max, correlated subquery as alternative — and knowing which is more readable.

---

### S10 — Users Active Every Month for 6 Consecutive Months

**Schema**
```sql
events(user_id, event_date)
```

**Question:** Find users who were active (had at least one event) in each of the last 6 calendar months.

**Solution**
```sql
SELECT user_id
FROM events
WHERE event_date >= DATE_TRUNC('month', NOW()) - INTERVAL '5 months'
  AND event_date <  DATE_TRUNC('month', NOW()) + INTERVAL '1 month'
GROUP BY user_id
HAVING COUNT(DISTINCT DATE_TRUNC('month', event_date)) = 6;
```

**What the interviewer tests:** COUNT(DISTINCT ...) inside HAVING, date range logic, understanding of "active in each month" vs "active sometime in 6 months".

---

### S11 — Detect Gaps in Sequential IDs

**Schema**
```sql
tickets(id)  -- id should be 1,2,3,... with no gaps
```

**Question:** Find all missing IDs in the sequence (gaps).

**Solution using LEAD**
```sql
SELECT id + 1 AS gap_start,
       next_id - 1 AS gap_end
FROM (
    SELECT id,
           LEAD(id) OVER (ORDER BY id) AS next_id
    FROM tickets
) t
WHERE next_id > id + 1;
```

**Solution using generate_series**
```sql
SELECT s.id AS missing_id
FROM generate_series(
    (SELECT MIN(id) FROM tickets),
    (SELECT MAX(id) FROM tickets)
) s(id)
LEFT JOIN tickets t ON s.id = t.id
WHERE t.id IS NULL;
```

**What the interviewer tests:** LEAD for comparing a row to its successor, generate_series as a number generator, anti-join.

---

### S12 — Pivot: Monthly Sales as Columns

**Schema**
```sql
sales(product, month, amount)  -- month is 'Jan','Feb',...'Dec'
```

**Question:** Show each product's sales for Jan, Feb, and Mar as separate columns.

**Solution using CASE WHEN**
```sql
SELECT product,
       SUM(CASE WHEN month = 'Jan' THEN amount ELSE 0 END) AS jan,
       SUM(CASE WHEN month = 'Feb' THEN amount ELSE 0 END) AS feb,
       SUM(CASE WHEN month = 'Mar' THEN amount ELSE 0 END) AS mar
FROM sales
GROUP BY product
ORDER BY product;
```

**PostgreSQL crosstab (tablefunc extension)**
```sql
-- Requires: CREATE EXTENSION tablefunc;
SELECT * FROM crosstab(
    'SELECT product, month, amount FROM sales ORDER BY 1,2',
    'SELECT DISTINCT month FROM sales ORDER BY 1'
) AS ct(product TEXT, jan NUMERIC, feb NUMERIC, mar NUMERIC);
```

**What the interviewer tests:** Conditional aggregation with CASE WHEN, awareness of crosstab for true pivots.

---

### S13 — Diagnose a Slow 50 Million Row Query

**Question:** This query takes 45 seconds. How do you diagnose and fix it?

```sql
SELECT * FROM orders
WHERE EXTRACT(year FROM created_at) = 2023
  AND status = 'pending';
```

**Step 1: Run EXPLAIN ANALYZE**
```sql
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE EXTRACT(year FROM created_at) = 2023
  AND status = 'pending';
```

**What you'll see:** `Seq Scan on orders (cost=0.00..2400000.00 rows=50000000)`

**Root cause:** `EXTRACT(year FROM created_at) = 2023` wraps the indexed column in a function — the index on `created_at` cannot be used. This is a **non-sargable predicate**.

**Fix: rewrite as a range**
```sql
SELECT * FROM orders
WHERE created_at >= '2023-01-01'
  AND created_at <  '2024-01-01'
  AND status = 'pending';
```

**Add a composite index**
```sql
CREATE INDEX idx_orders_date_status
ON orders (created_at, status);
```

**After fix:** `Index Scan using idx_orders_date_status` — query time drops from 45s to <100ms.

**What the interviewer tests:** Reading EXPLAIN output, knowing what sargability means, composite index column order (equality first, then range).

---

## 🔴 Advanced Scenarios

---

### S14 — Design a Twitter/X Schema

**Question:** Design the core database schema for Twitter. Support: users, tweets, follows, likes, retweets, hashtags.

**Solution**
```sql
CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    username    VARCHAR(50) UNIQUE NOT NULL,
    display_name VARCHAR(100),
    bio         TEXT,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE tweets (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT REFERENCES users(id) ON DELETE CASCADE,
    content     VARCHAR(280) NOT NULL,
    reply_to_id BIGINT REFERENCES tweets(id),  -- NULL = original tweet
    retweet_id  BIGINT REFERENCES tweets(id),  -- NULL = not a retweet
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE follows (
    follower_id BIGINT REFERENCES users(id) ON DELETE CASCADE,
    followee_id BIGINT REFERENCES users(id) ON DELETE CASCADE,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (follower_id, followee_id),
    CHECK (follower_id != followee_id)
);

CREATE TABLE likes (
    user_id    BIGINT REFERENCES users(id) ON DELETE CASCADE,
    tweet_id   BIGINT REFERENCES tweets(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (user_id, tweet_id)
);

CREATE TABLE hashtags (
    id   SERIAL PRIMARY KEY,
    tag  VARCHAR(100) UNIQUE NOT NULL
);

CREATE TABLE tweet_hashtags (
    tweet_id   BIGINT REFERENCES tweets(id) ON DELETE CASCADE,
    hashtag_id INT REFERENCES hashtags(id) ON DELETE CASCADE,
    PRIMARY KEY (tweet_id, hashtag_id)
);

-- Key indexes
CREATE INDEX idx_tweets_user ON tweets(user_id, created_at DESC);
CREATE INDEX idx_tweets_reply ON tweets(reply_to_id);
CREATE INDEX idx_follows_followee ON follows(followee_id);
```

**What the interviewer tests:** Self-referencing tables (reply/retweet as same table), composite primary keys for junction tables, ON DELETE CASCADE, index on `(user_id, created_at DESC)` to efficiently fetch a user's timeline.

---

### S15 — Design a Hotel Booking Schema

**Question:** Design a schema that prevents double-booking of the same room for overlapping dates.

**Solution**
```sql
CREATE TABLE hotels (
    id   SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL
);

CREATE TABLE rooms (
    id       SERIAL PRIMARY KEY,
    hotel_id INT REFERENCES hotels(id),
    number   VARCHAR(10) NOT NULL,
    type     VARCHAR(50),  -- 'single','double','suite'
    price    DECIMAL(10,2),
    UNIQUE (hotel_id, number)
);

CREATE TABLE bookings (
    id          BIGSERIAL PRIMARY KEY,
    room_id     INT REFERENCES rooms(id) ON DELETE RESTRICT,
    guest_name  VARCHAR(200) NOT NULL,
    check_in    DATE NOT NULL,
    check_out   DATE NOT NULL,
    CHECK (check_out > check_in),
    EXCLUDE USING GIST (
        room_id WITH =,
        daterange(check_in, check_out, '[)') WITH &&
    )
);
```

**The `EXCLUDE` constraint** (PostgreSQL only, requires `btree_gist` extension) prevents any two bookings for the same room whose date ranges overlap:
```sql
CREATE EXTENSION btree_gist;
```

**Application-level double-booking check** (portable SQL)
```sql
SELECT id FROM bookings
WHERE room_id = $1
  AND check_in  < $3   -- new check_out
  AND check_out > $2;  -- new check_in
-- Returns rows if conflict exists
```

**What the interviewer tests:** EXCLUDE constraint for range overlap, btree_gist, or application-level overlap detection logic.

---

### S16 — Employees Earning More Than Their Manager

**Schema**
```sql
employees(id, name, manager_id, salary)
-- manager_id references employees(id) — self-referencing
```

**Question:** List all employees who earn more than their direct manager.

**Solution**
```sql
SELECT e.name AS employee,
       e.salary AS employee_salary,
       m.name AS manager,
       m.salary AS manager_salary
FROM employees e
JOIN employees m ON e.manager_id = m.id
WHERE e.salary > m.salary;
```

**What the interviewer tests:** Self-join using table aliases. This is one of the most-asked SQL questions at all levels.

---

### S17 — Print the Full Org Chart Hierarchy

**Schema**
```sql
employees(id, name, manager_id)
```

**Question:** Print every employee along with their depth in the hierarchy (CEO = depth 0).

**Solution using recursive CTE**
```sql
WITH RECURSIVE org_chart AS (
    -- Base case: root node (CEO has no manager)
    SELECT id, name, manager_id, 0 AS depth, name::TEXT AS path
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive case: join children to their parents
    SELECT e.id, e.name, e.manager_id,
           oc.depth + 1,
           oc.path || ' → ' || e.name
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT depth, name, path
FROM org_chart
ORDER BY path;
```

**Output:**
```
depth | name     | path
0     | Alice    | Alice
1     | Bob      | Alice → Bob
2     | Carol    | Alice → Bob → Carol
1     | Dave     | Alice → Dave
```

**What the interviewer tests:** Recursive CTE structure (base case + UNION ALL + recursive join), handling tree-shaped data.

---

### S18 — Monthly Cohort Retention Analysis

**Schema**
```sql
users(id, created_at)
events(user_id, event_date)
```

**Question:** For each signup cohort (grouped by signup month), show what percentage of users were still active in months 1, 2, and 3 after signup.

**Solution**
```sql
WITH cohorts AS (
    SELECT id AS user_id,
           DATE_TRUNC('month', created_at) AS cohort_month
    FROM users
),
activity AS (
    SELECT e.user_id,
           c.cohort_month,
           DATE_PART('month', AGE(DATE_TRUNC('month', e.event_date), c.cohort_month)) AS months_since_signup
    FROM events e
    JOIN cohorts c ON e.user_id = c.user_id
    WHERE DATE_TRUNC('month', e.event_date) >= c.cohort_month
),
cohort_sizes AS (
    SELECT cohort_month, COUNT(*) AS cohort_size
    FROM cohorts
    GROUP BY cohort_month
)
SELECT a.cohort_month,
       cs.cohort_size,
       SUM(CASE WHEN a.months_since_signup = 0 THEN 1 ELSE 0 END) AS month_0,
       SUM(CASE WHEN a.months_since_signup = 1 THEN 1 ELSE 0 END) AS month_1,
       SUM(CASE WHEN a.months_since_signup = 2 THEN 1 ELSE 0 END) AS month_2,
       SUM(CASE WHEN a.months_since_signup = 3 THEN 1 ELSE 0 END) AS month_3
FROM activity a
JOIN cohort_sizes cs ON a.cohort_month = cs.cohort_month
GROUP BY a.cohort_month, cs.cohort_size
ORDER BY a.cohort_month;
```

**What the interviewer tests:** Multi-step CTE pipeline, AGE() for interval arithmetic, conditional aggregation for pivot, cohort analysis pattern used by every growth team.

---

### S19 — Delete 200 Million Rows Without Killing the Database

**Question:** A `logs` table has 200 million rows and you need to delete everything older than 90 days. How do you do it safely?

**Wrong approach:**
```sql
DELETE FROM logs WHERE created_at < NOW() - INTERVAL '90 days';
-- Locks the table for hours, fills transaction log, kills production
```

**Correct approach: batch deletes in a loop**
```sql
-- Run this in a script with a loop (Python/shell)
DO $$
DECLARE
    deleted INT;
BEGIN
    LOOP
        DELETE FROM logs
        WHERE id IN (
            SELECT id FROM logs
            WHERE created_at < NOW() - INTERVAL '90 days'
            LIMIT 10000
        );

        GET DIAGNOSTICS deleted = ROW_COUNT;
        EXIT WHEN deleted = 0;

        PERFORM pg_sleep(0.1);  -- brief pause between batches
    END LOOP;
END $$;
```

**Even better: partition the table by date**
```sql
-- Partition logs by month, then DROP whole partitions
-- DROP PARTITION is instant vs row-by-row DELETE
ALTER TABLE logs PARTITION BY RANGE (created_at);
-- Then: DROP TABLE logs_y2023m01; -- instant!
```

**What the interviewer tests:** Understanding of lock contention, transaction log explosion, batch delete pattern, table partitioning as the real solution at scale.

---

### S20 — Median Salary Without PERCENTILE_CONT

**Schema**
```sql
employees(id, salary)
```

**Question:** Calculate the median salary without using `PERCENTILE_CONT`.

**Solution**
```sql
WITH ordered AS (
    SELECT salary,
           ROW_NUMBER() OVER (ORDER BY salary) AS rn,
           COUNT(*) OVER () AS total
    FROM employees
)
SELECT AVG(salary) AS median
FROM ordered
WHERE rn IN (
    (total + 1) / 2,
    (total + 2) / 2
);
```

**How it works:**
- For odd N (e.g., 9): `(9+1)/2 = 5` and `(9+2)/2 = 5` → picks row 5
- For even N (e.g., 10): `(10+1)/2 = 5` and `(10+2)/2 = 6` → averages rows 5 and 6

**PostgreSQL native (simpler)**
```sql
SELECT PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) AS median
FROM employees;
```

**What the interviewer tests:** Understanding of median definition for odd/even counts, ROW_NUMBER for positional access — tests fundamental SQL reasoning skills.

---

### S21 — Fraud Detection: Same Card, Two Countries in 10 Minutes

**Schema**
```sql
transactions(id, card_id, country, amount, txn_time)
```

**Question:** Flag potentially fraudulent cases where the same card was used in two different countries within 10 minutes.

**Solution**
```sql
WITH consecutive AS (
    SELECT card_id,
           country,
           txn_time,
           LAG(country)  OVER (PARTITION BY card_id ORDER BY txn_time) AS prev_country,
           LAG(txn_time) OVER (PARTITION BY card_id ORDER BY txn_time) AS prev_time
    FROM transactions
)
SELECT card_id,
       prev_country,
       country AS current_country,
       prev_time,
       txn_time,
       txn_time - prev_time AS time_gap
FROM consecutive
WHERE prev_country IS NOT NULL
  AND prev_country != country
  AND txn_time - prev_time <= INTERVAL '10 minutes'
ORDER BY card_id, txn_time;
```

**What the interviewer tests:** LAG with PARTITION BY for per-card ordering, timestamp arithmetic, real-world fraud pattern.

---

### S22 — Nth Highest Salary (Generalised)

**Schema**
```sql
employees(id, salary)
```

**Question:** Write a query to find the Nth highest salary (e.g., N=3).

**Solution**
```sql
-- Replace 3 with any N
WITH ranked AS (
    SELECT DISTINCT salary,
           DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM employees
)
SELECT salary
FROM ranked
WHERE rnk = 3;
```

**As a reusable function (PostgreSQL)**
```sql
CREATE OR REPLACE FUNCTION nth_highest_salary(n INT)
RETURNS DECIMAL AS $$
    SELECT salary FROM (
        SELECT DISTINCT salary,
               DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
        FROM employees
    ) t
    WHERE rnk = n;
$$ LANGUAGE sql;

SELECT nth_highest_salary(3);
```

**What the interviewer tests:** DENSE_RANK vs RANK, wrapping logic in a function, handling ties correctly.

---

### S23 — Optimise a Slow Checkout Query

**Question:** Your checkout query scans millions of rows. The query is:

```sql
SELECT * FROM orders
WHERE TO_CHAR(created_at, 'YYYY-MM') = '2024-03';
```

**Problem:** `TO_CHAR(created_at, ...)` is a function applied to the column — the index on `created_at` cannot be used. This is a **non-sargable** predicate.

**Fix: use a range predicate**
```sql
SELECT * FROM orders
WHERE created_at >= '2024-03-01'
  AND created_at <  '2024-04-01';
```

**Rule of sargability:** Never wrap an indexed column in a function or expression on the left side of a WHERE clause:

| Non-sargable ❌ | Sargable ✅ |
|----------------|------------|
| `YEAR(created_at) = 2024` | `created_at BETWEEN '2024-01-01' AND '2024-12-31'` |
| `LOWER(email) = 'a@b.com'` | `email = 'a@b.com'` (store lowercase) |
| `price * 1.2 > 100` | `price > 83.33` |
| `SUBSTRING(code, 1, 3) = 'ABC'` | `code LIKE 'ABC%'` |

**What the interviewer tests:** Knowing the term "sargable", ability to rewrite queries to enable index use.

---

### S24 — Average Session Duration from an Event Log

**Schema**
```sql
events(user_id, event_type, event_time)
-- event_type: 'session_start' or 'session_end'
-- Each start is paired with the next end for the same user
```

**Question:** Calculate the average session duration per user.

**Solution**
```sql
WITH sessions AS (
    SELECT user_id,
           event_time AS start_time,
           LEAD(event_time) OVER (
               PARTITION BY user_id ORDER BY event_time
           ) AS end_time,
           event_type
    FROM events
)
SELECT user_id,
       AVG(end_time - start_time) AS avg_session_duration
FROM sessions
WHERE event_type = 'session_start'
  AND end_time IS NOT NULL
GROUP BY user_id
ORDER BY user_id;
```

**What the interviewer tests:** LEAD to look ahead to the next event, filtering to only start events (which now have their paired end time), timestamp arithmetic.

---

### S25 — Products Sold in Every Region (Relational Division)

**Schema**
```sql
sales(product_id, region)
regions(id, name)
```

**Question:** Find all products that have been sold in every region.

**Solution using double NOT EXISTS (relational division)**
```sql
SELECT DISTINCT s.product_id
FROM sales s
WHERE NOT EXISTS (
    SELECT r.id
    FROM regions r
    WHERE NOT EXISTS (
        SELECT 1
        FROM sales s2
        WHERE s2.product_id = s.product_id
          AND s2.region = r.id
    )
);
```

**Alternative using COUNT DISTINCT**
```sql
SELECT product_id
FROM sales
GROUP BY product_id
HAVING COUNT(DISTINCT region) = (SELECT COUNT(*) FROM regions);
```

**What the interviewer tests:** Relational division — one of the hardest classic SQL patterns. Double NOT EXISTS: "there is no region for which this product has no sale." The COUNT DISTINCT approach is simpler and usually preferred in practice.

---

## Summary

| Pattern | Used in |
|---------|---------|
| LEFT JOIN anti-join | S1, S11 |
| ROW_NUMBER + DELETE | S2 |
| RANK / DENSE_RANK | S6, S22 |
| Window SUM / running total | S7 |
| LAG / LEAD | S8, S11, S21, S24 |
| PARTITION BY | S9, S17, S21 |
| COUNT DISTINCT + HAVING | S10, S25 |
| CASE WHEN pivot | S12, S18 |
| EXPLAIN ANALYZE | S13, S23 |
| Recursive CTE | S17 |
| Self-join | S16 |
| Batch DELETE | S19 |
| Schema design | S14, S15 |
| Sargability | S13, S23 |
| generate_series | S4, S11 |
| EXCEPT / NOT EXISTS | S5, S25 |

---

**[🏠 Back to README](../README.md)**

**Prev:** [← SQL Interview Questions](./sql_questions.md) &nbsp;|&nbsp; **Next:** — (last file)

**Related Topics:** [JOIN Patterns](../05_joins/join_patterns.md) · [Window Functions](../03_aggregation/window_functions.md) · [CTEs](../06_advanced_queries/ctes.md) · [Query Optimization](../08_performance/query_optimization.md) · [Transactions](../07_data_modification/transactions.md) · [Indexes](../04_schema_design/indexes.md)
