# Execution Plans — How the Database Plans Your Route

## The GPS Analogy

Before you start driving to a new destination, you punch it into your GPS. It shows you the route it plans to take *before* you've moved an inch. You can see: take the highway (fast), or go through the city centre (slow). You can compare routes and pick the better one.

`EXPLAIN` is your database's GPS. It shows you the route — the *execution plan* — that the query planner has chosen before executing a single byte of data retrieval. `EXPLAIN ANALYZE` actually drives the route and reports back how long each segment really took.

---

## EXPLAIN — See the Plan Without Running It

```sql
EXPLAIN
SELECT u.name, COUNT(o.order_id) AS order_count
FROM   users u
JOIN   orders o ON o.user_id = u.user_id
WHERE  o.status = 'completed'
GROUP  BY u.user_id, u.name
ORDER  BY order_count DESC;
```

Sample output (PostgreSQL):

```
                              QUERY PLAN
------------------------------------------------------------------------
 Sort  (cost=1842.33..1852.33 rows=4000 width=40)
   Sort Key: (count(o.order_id)) DESC
   ->  HashAggregate  (cost=1582.00..1622.00 rows=4000 width=40)
         Group Key: u.user_id, u.name
         ->  Hash Join  (cost=450.00..1432.00 rows=30000 width=16)
               Hash Cond: (o.user_id = u.user_id)
               ->  Seq Scan on orders o  (cost=0.00..820.00 rows=30000 width=8)
                     Filter: (status = 'completed')
               ->  Hash  (cost=300.00..300.00 rows=12000 width=16)
                     ->  Seq Scan on users u  (cost=0.00..300.00 rows=12000 width=16)
```

---

## Reading the Output — Annotated

```
Sort  (cost=1842..1852 rows=4000 width=40)
  |         |     |      |       |
  |         |     |      |       +-- bytes per row estimate
  |         |     |      +---------- estimated output rows
  |         |     +----------------- estimated total cost (arbitrary units)
  |         +----------------------- estimated startup cost
  +--------------------------------- operation type

"->  Seq Scan on orders"
     ^
     Full table scan — reads every row
     (suspicious on large tables!)

"cost=0.00..820.00"
       |      |
       |      +-- total cost to return all rows
       +--------- startup cost (before first row returned)
```

Read plans from the **innermost/deepest** level outward — that's where execution starts. The tree reads bottom-up.

---

## EXPLAIN ANALYZE — Run It and Measure

`EXPLAIN ANALYZE` actually executes the query and adds real timing data alongside the estimates. This is where you catch bad row estimates.

```sql
EXPLAIN ANALYZE
SELECT u.name, COUNT(o.order_id) AS order_count
FROM   users u
JOIN   orders o ON o.user_id = u.user_id
WHERE  o.status = 'completed'
GROUP  BY u.user_id, u.name
ORDER  BY order_count DESC;
```

Sample output with `ANALYZE`:

```
 Sort  (cost=1842.33..1852.33 rows=4000 width=40)
       (actual time=245.123..246.891 rows=3847 loops=1)
   ->  HashAggregate  ...
       (actual time=230.001..238.445 rows=3847 loops=1)
         ->  Hash Join  ...
             (actual time=12.332..198.221 rows=28910 loops=1)
               ->  Seq Scan on orders o
                   (cost=0.00..820.00 rows=30000 width=8)
                   (actual time=0.021..85.443 rows=28910 loops=1)
                     Filter: (status = 'completed')
                     Rows Removed by Filter: 11090
               ->  Hash  (cost=300.00..300.00 rows=12000 width=16)
                   (actual time=11.891..11.891 rows=12000 loops=1)
                     ->  Seq Scan on users u  ...
                         (actual time=0.012..7.332 rows=12000 loops=1)

 Planning Time: 1.234 ms
 Execution Time: 247.112 ms
```

Key additions with ANALYZE:
- `actual time=X..Y` — real milliseconds (startup..total)
- `rows=N` — actual rows returned
- `loops=N` — how many times this node ran (in nested loops)
- `Rows Removed by Filter` — rows scanned but thrown away

---

## Common Node Types

```
+-------------------+--------------------------------------------------+
| Node Type         | What it means                                    |
+-------------------+--------------------------------------------------+
| Seq Scan          | Read every row in the table (full table scan)    |
|                   | Fine for small tables or when fetching most rows |
|                   | BAD on large tables with selective WHERE clauses |
+-------------------+--------------------------------------------------+
| Index Scan        | Use an index to find matching rows               |
|                   | Good: skips most of the table                    |
+-------------------+--------------------------------------------------+
| Index Only Scan   | Answer entirely from the index, no table read    |
|                   | Fastest possible: "covering index"               |
+-------------------+--------------------------------------------------+
| Bitmap Heap Scan  | Uses index to build a bitmap, then scans heap    |
|                   | Good for medium-selectivity queries              |
+-------------------+--------------------------------------------------+
| Nested Loop       | For each row in outer set, scan inner set        |
|                   | Great when inner set is tiny; terrible if large  |
+-------------------+--------------------------------------------------+
| Hash Join         | Build hash table from one side, probe with other |
|                   | Good for larger sets without usable index        |
+-------------------+--------------------------------------------------+
| Merge Join        | Sort both sides, then merge                      |
|                   | Good when both inputs are already sorted         |
+-------------------+--------------------------------------------------+
| Sort              | Sort rows in memory (or disk if too large)       |
|                   | Look for "Sort Method: external merge Disk"      |
+-------------------+--------------------------------------------------+
| HashAggregate     | Aggregate using a hash table (GROUP BY)          |
+-------------------+--------------------------------------------------+
```

---

## What to Look For: Red Flags

### 1. Sequential Scan on a Large Table

```
->  Seq Scan on orders  (cost=0.00..85000.00 rows=2000000 width=32)
      Filter: (status = 'completed')
      Rows Removed by Filter: 1800000
```

Reading 2 million rows to return 200,000 — an index on `status` (or a partial index) would help here.

### 2. Estimated Rows Way Off from Actual Rows

```
Seq Scan on products
  (cost=0.00..450.00 rows=12 width=64)     -- planner estimated 12
  (actual time=0.021..32.44 rows=8943 loops=1) -- actually 8943!
```

A massive mismatch means the planner's statistics are stale. Run `ANALYZE products;` to refresh them. The planner uses bad estimates to choose bad plans.

### 3. Sort Spilling to Disk

```
Sort  (cost=... rows=...)
  Sort Method: external merge  Disk: 14512kB
```

The sort ran out of `work_mem` and had to write to disk — much slower. Increase `work_mem` for the session or add an index to avoid the sort:

```sql
SET work_mem = '256MB';
-- then re-run your query
```

### 4. Nested Loop with a Large Inner Set

```
Nested Loop  (actual time=0.1..9823.4 rows=50000 loops=50000)
  -> Seq Scan on orders  (rows=50000)
  -> Seq Scan on order_items  (rows=50000 loops=50000)
```

50,000 × 50,000 = 2.5 billion row comparisons. An index on `order_items.order_id` would replace this with an Index Scan.

---

## Useful EXPLAIN Options (PostgreSQL)

```sql
-- Verbose: show column names, table schema details
EXPLAIN (VERBOSE) SELECT ...;

-- Buffers: show cache hits vs disk reads (requires ANALYZE)
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;

-- JSON format for programmatic parsing
EXPLAIN (FORMAT JSON) SELECT ...;

-- Full breakdown
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT TEXT) SELECT ...;
```

`Buffers` output tells you whether rows came from PostgreSQL's shared buffer cache (fast) or had to be read from disk (slow):

```
Buffers: shared hit=4821 read=293
                  ^           ^
                  cache hit   disk read (costly)
```

---

## Refreshing Statistics

The planner uses statistics gathered by `ANALYZE` to estimate row counts. If stats are stale, estimates are wrong and plans are suboptimal.

```sql
-- Update stats for one table
ANALYZE orders;

-- Update stats for the whole database
ANALYZE;

-- PostgreSQL's autovacuum runs ANALYZE automatically,
-- but after a large bulk insert you may want to run it manually
```

---

## Summary

```
+---------------------------+------------------------------------------------+
| Command                   | What it does                                   |
+---------------------------+------------------------------------------------+
| EXPLAIN query             | Show the query plan without executing          |
| EXPLAIN ANALYZE query     | Execute and show actual vs estimated timings   |
| EXPLAIN (BUFFERS) ...     | Show cache hit / disk read stats               |
| ANALYZE table_name        | Refresh table statistics for the planner       |
+---------------------------+------------------------------------------------+

Red flags to investigate:
  - Seq Scan on tables with millions of rows
  - Estimated rows << actual rows (stale stats — run ANALYZE)
  - Sort Method: external merge Disk (increase work_mem or add index)
  - Nested Loop with large row counts on both sides (missing index)
```

---

**[🏠 Back to README](../README.md)**

**Prev:** [← Query Optimization](./query_optimization.md) &nbsp;|&nbsp; **Next:** [Indexing Strategies →](./indexing_strategies.md)

**Related Topics:** [Query Optimization](./query_optimization.md) · [Indexing Strategies](./indexing_strategies.md)
