# SQL vs NoSQL

## The Filing Cabinet vs. the Sticky-Note Wall

Imagine two offices.

**Office A** has a classic filing cabinet. Every drawer is labelled. Every folder inside follows
the same template — client name, date, project number, status. To pull a report across 500 client
folders, an assistant can flip through them systematically because every folder is identical in
structure. If someone tries to file a folder without a client name, the system won't allow it.
Order and predictability are baked in.

**Office B** has a wall covered in sticky notes. Each note can say anything — a quick idea, a
phone number, a diagram, a to-do list with sub-items. There's no fixed template. New kinds of
notes can be added instantly. Searching the whole wall for a specific note requires reading every
single one, but for capturing fast-moving, varied information, the wall is incredibly flexible.

**SQL databases are the filing cabinet.** Structure first, flexibility second.
**NoSQL databases are the sticky-note wall.** Flexibility first, structure second.

Both approaches are genuinely useful. The skill is knowing which to reach for.

---

## What Makes SQL (Relational) Databases Strong?

```
  Strength              Why it matters
  ──────────────────────────────────────────────────────────────────────────
  Structured schema     Column types enforced → less bad data
  Relationships         Foreign keys link tables → no duplication
  ACID transactions     Bank-grade reliability for critical operations
  Complex queries       JOINs, aggregations, window functions
  Standardised          Skills transfer across PostgreSQL, MySQL, etc.
  Decades of tooling    BI tools, ORMs, ETL pipelines all speak SQL
```

SQL is the right default for:
- Structured data with clear relationships (users → orders → products)
- Applications where data correctness is critical (finance, healthcare)
- Reporting and analytics that needs aggregations and joins
- Situations where your data model is known upfront and changes slowly

---

## What Is NoSQL?

**NoSQL** means "Not Only SQL" — a family of databases that store data in formats other than
relational tables. There is no single NoSQL standard. There are four major categories:

### 1. Document Stores
Store data as JSON-like documents. Each document can have different fields.

```json
// MongoDB document for a user
{
  "_id": "u_7482",
  "name": "Alice Nguyen",
  "email": "alice@example.com",
  "preferences": {
    "theme": "dark",
    "notifications": ["email", "push"]
  },
  "addresses": [
    { "type": "home", "city": "Toronto" },
    { "type": "work", "city": "Montreal" }
  ]
}
```

A SQL table cannot store nested arrays and objects naturally. A document database handles them
effortlessly. **Examples:** MongoDB, CouchDB, Firestore.

### 2. Key-Value Stores
The simplest model: a key maps to a value. Extremely fast.

```
  Key                Value
  ──────────────────────────────────────────
  session:abc123     { user_id: 7482, ... }
  cache:product:99   { name: "Headphones", price: 89.99 }
  rate_limit:alice   42
```

Used for caches, sessions, leaderboards, real-time counters.
**Examples:** Redis, DynamoDB (also document), Memcached.

### 3. Column-Family Stores
Optimised for writing and reading massive volumes of time-series or event data across distributed
nodes. Think billions of rows where you rarely need complex joins.

```
  Row key           Column family: activity
  ───────────────────────────────────────────────────────
  user:7482         { 2024-03-01: "login",
                      2024-03-01: "purchase",
                      2024-03-02: "search" }
```

**Examples:** Apache Cassandra, HBase, ScyllaDB.

### 4. Graph Databases
Store data as nodes (entities) and edges (relationships). Perfect when the *connections* between
data matter more than the data itself.

```
  (Alice) --[FOLLOWS]--> (Bob)
  (Alice) --[PURCHASED]--> (Product: Headphones)
  (Bob)   --[REVIEWED]--> (Product: Headphones)
```

**Examples:** Neo4j, Amazon Neptune, TigerGraph.

---

## The Decision Table

```
  Your situation                             Reach for...
  ──────────────────────────────────────────────────────────────────────────────
  User accounts, orders, products            PostgreSQL or MySQL
  Banking, payments, ledgers                 PostgreSQL (ACID critical)
  CMS content with flexible fields           MongoDB
  Real-time session storage / caching        Redis
  Social graph (followers, friends)          Neo4j
  Billions of IoT sensor readings            Cassandra
  Mobile app with offline sync               Firestore or CouchDB
  Full-text search                           Elasticsearch (or Postgres + tsvector)
  Analytics on petabytes of data             BigQuery, Redshift (SQL but columnar)
```

---

## Real-World: Instagram's Data Stack

Instagram is a great case study because it uses both SQL and NoSQL at scale — for different
reasons:

```
  What data?                       Technology          Why
  ──────────────────────────────────────────────────────────────────────────────
  User accounts, profiles          PostgreSQL          Structured, ACID needed
  Comments, posts (structured)     PostgreSQL          Relational queries required
  Photo/video metadata             PostgreSQL          Schema defined, joins needed
  User feed (activity stream)      Cassandra           Billions of writes/reads,
                                                       time-ordered, distributed
  Caching hot data (profiles,      Memcached / Redis   Sub-millisecond reads,
  follower counts)                                     no persistence needed
  Search                           Elasticsearch       Full-text, fuzzy matching
```

The lesson: **SQL and NoSQL are not enemies.** Production systems routinely use both. SQL handles
the *source of truth* — the structured, relational, reliable core. NoSQL handles the *edges* —
caching, feeds, search, flexible content.

---

## ACID vs BASE — The Trade-off

SQL databases prioritise **ACID** (reliability). Many NoSQL databases prioritise **BASE**
(availability and speed):

```
  ACID (SQL)                        BASE (many NoSQL)
  ──────────────────────────────────────────────────────────────────
  Atomic transactions               Basically Available
  Consistent state always           Soft state (can be stale)
  Isolated concurrent operations    Eventually consistent
  Durable (written to disk)
```

"Eventually consistent" means: if you write to a Cassandra cluster, a read from a different node
might return the old value for a few milliseconds until all nodes sync. For a bank balance, that's
unacceptable. For a "like count" on a post, it's perfectly fine.

---

## SQL Is Not Slow at Scale

A common misconception: "NoSQL is for big data, SQL is for small projects." This is false.

- **PostgreSQL** handles billions of rows with proper indexing and partitioning
- **Google Spanner** is a globally distributed relational SQL database
- **Amazon Aurora** runs SQL at massive scale in the cloud
- **Snowflake and BigQuery** are columnar SQL databases for petabyte analytics

The real question is not SQL vs NoSQL — it's "what are your access patterns?" A document
database is faster for fetching a single deeply-nested document. A relational database is faster
for aggregating relationships across millions of rows.

---

## Quick Summary of NoSQL Types

```
  Type             Best for                  Popular options
  ──────────────────────────────────────────────────────────────────────────
  Document         Flexible content, CMS     MongoDB, Firestore, CouchDB
  Key-value        Cache, sessions, counters  Redis, DynamoDB, Memcached
  Column-family    Time-series, event logs   Cassandra, HBase, ScyllaDB
  Graph            Social networks, fraud    Neo4j, Amazon Neptune
```

---

## Summary

```
┌──────────────────────────────────────────────────────────────────────┐
│  SQL VS NOSQL — KEY TAKEAWAYS                                        │
├──────────────────────────────────┬───────────────────────────────────┤
│  SQL strengths                   │  Structure, relationships, ACID   │
│  NoSQL strengths                 │  Flexibility, scale, speed        │
│  4 NoSQL types                   │  Document, K-V, Column, Graph     │
│  SQL at scale                    │  Entirely possible with indexing  │
│  Real production systems         │  Usually use BOTH SQL + NoSQL     │
│  ACID vs BASE                    │  Reliability vs availability      │
│  When SQL wins                   │  Critical data, complex queries   │
│  When NoSQL wins                 │  Flexible schema, massive writes  │
└──────────────────────────────────┴───────────────────────────────────┘
```

---

**[🏠 Back to README](../README.md)**

**Prev:** [← Databases and RDBMS](./databases_and_rdbms.md) &nbsp;|&nbsp; **Next:** [SELECT and FROM →](../02_querying_basics/select_and_from.md)

**Related Topics:** [What is SQL](./what_is_sql.md) · [Databases and RDBMS](./databases_and_rdbms.md)

---

## 📝 Practice Questions

- 📝 [Q1 · sql-vs-nosql](../sql_practice_questions_100.md#q1--normal--sql-vs-nosql)

