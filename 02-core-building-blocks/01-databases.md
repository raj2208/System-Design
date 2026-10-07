# Databases

Every system stores data. The database is where that data lives — and the choice of database, and how you use it, shapes almost every other architectural decision you make. This is the most important building block to understand deeply.

---

## What a database actually does

A database gives you four guarantees:

- **Store** data durably (it survives a server restart)
- **Retrieve** data efficiently (by key, by query, by range)
- **Update** data consistently (changes are reflected immediately)
- **Delete** data reliably (it's actually gone)

Everything else — indexes, replication, transactions, sharding — is built on top of these four operations.

---

## Relational databases (SQL)

A relational database stores data in **tables** — rows and columns, like a spreadsheet, but with strict rules and relationships between tables.

```
users table                         orders table
┌────┬──────────┬───────────────┐   ┌────┬─────────┬────────┬────────────┐
│ id │ name     │ email         │   │ id │ user_id │ total  │ created_at │
├────┼──────────┼───────────────┤   ├────┼─────────┼────────┼────────────┤
│  1 │ Raj      │ raj@email.com │   │  1 │    1    │ $49.99 │ 2026-10-01 │
│  2 │ Priya    │ p@email.com   │   │  2 │    1    │ $12.00 │ 2026-10-05 │
└────┴──────────┴───────────────┘   │  3 │    2    │ $99.00 │ 2026-10-06 │
                                    └────┴─────────┴────────┴────────────┘
```

The `orders.user_id` is a **foreign key** — it references `users.id`. This relationship is enforced by the database. You can query across both tables:

```sql
SELECT users.name, orders.total
FROM users
JOIN orders ON users.id = orders.user_id
WHERE users.id = 1;
```

**Popular relational databases:** PostgreSQL, MySQL, SQLite, Oracle, SQL Server.

### ACID — the reliability guarantee

Relational databases are built around **ACID transactions**. A transaction is a group of operations that either all succeed or all fail together.

**Atomicity** — all operations in a transaction succeed, or none of them do. No partial writes.
```
Transfer $100 from Account A to Account B:
  1. Deduct $100 from A
  2. Add $100 to B

If step 2 fails, step 1 is rolled back. Money doesn't disappear.
```

**Consistency** — the database is always in a valid state. Rules (foreign keys, unique constraints, not-null) are never violated.

**Isolation** — concurrent transactions don't interfere with each other. Two users buying the last item at the same time — only one wins.

**Durability** — once a transaction is committed, it's permanent. A crash immediately after a commit doesn't lose the data.

ACID is why you trust a bank's database with your money. These guarantees are not free — they require coordination that costs performance. But for data that absolutely cannot be wrong, ACID is what you want.

### Indexing — how databases stay fast

Without an index, finding a row means scanning every row in the table (a **full table scan**). With a million users, that's slow.

An **index** is a separate data structure (usually a B-tree) that the database maintains alongside the table. It maps column values to the rows that contain them — like the index in the back of a book.

```
Without index: SELECT * FROM users WHERE email = 'raj@email.com'
→ scan all 1,000,000 rows                                 O(n)

With index on email: same query
→ look up in B-tree → jump directly to the row            O(log n)
```

**Trade-offs:**
- Indexes speed up **reads** (queries, lookups)
- Indexes slow down **writes** (every insert/update/delete must also update the index)
- Indexes use **disk space**

Rule of thumb: index columns you filter or sort by frequently. Don't index every column — you'll pay the write penalty for no benefit.

### When to use a relational database

- Data has clear relationships (users have orders, orders have items)
- You need transactions (financial data, inventory, anything where partial writes are dangerous)
- You need complex queries across multiple tables (JOINs, aggregations, reporting)
- Data integrity is critical (foreign keys, constraints, no orphaned records)

---

## Non-relational databases (NoSQL)

NoSQL databases trade some of SQL's guarantees for flexibility, scale, or performance. "NoSQL" is an umbrella term — there are several distinct types.

### Document stores

Store data as JSON-like documents. Each document is self-contained and can have a different structure.

```json
{
  "id": "user_raj",
  "name": "Raj",
  "email": "raj@email.com",
  "preferences": {
    "theme": "dark",
    "notifications": true
  },
  "tags": ["premium", "early-adopter"]
}
```

No fixed schema — you can add new fields to some documents without changing others. Good for data that varies in shape.

**Examples:** MongoDB, CouchDB, Firestore.  
**Use when:** user profiles, content management, product catalogs where each item has different attributes.

### Key-value stores

The simplest model: a key maps to a value. Like a dictionary/hashmap, but persistent and distributed.

```
SET  user:raj:session  "abc123xyz"   (with 1hr TTL)
GET  user:raj:session  → "abc123xyz"
```

Extremely fast — lookups are O(1). No complex queries — you can only look up by key.

**Examples:** Redis, DynamoDB (also supports more), Memcached.  
**Use when:** caching, sessions, rate limiting counters, leaderboards, feature flags.

### Wide-column stores

Data is stored in rows, but each row can have a different set of columns. Optimized for writing and reading massive amounts of data across many machines.

```
Row key: "user_raj"
  Columns: login:2026-10-01, login:2026-10-05, login:2026-10-07
  (sparse — only the logins that exist are stored)
```

**Examples:** Apache Cassandra, Google Bigtable, HBase.  
**Use when:** time-series data, activity logs, IoT sensor data, analytics at massive scale (billions of rows).

### Graph databases

Data is stored as **nodes** (entities) and **edges** (relationships between them). Optimized for traversing relationships — "who are the friends of friends of Raj who also bought product X?"

**Examples:** Neo4j, Amazon Neptune.  
**Use when:** social networks, recommendation engines, fraud detection, knowledge graphs.

---

## SQL vs NoSQL — the honest comparison

This is one of the most common interview questions. The honest answer is: **it depends on the use case**.

| | SQL (Relational) | NoSQL |
|-|-----------------|-------|
| **Schema** | Fixed, enforced | Flexible, varies per record |
| **Transactions** | Full ACID | Varies (some support it, many don't) |
| **Scaling** | Vertical (primarily), horizontal is complex | Designed for horizontal scale |
| **Query power** | Very high (JOINs, aggregations, sub-queries) | Limited (usually query by key or index only) |
| **Consistency** | Strong by default | Often eventual |
| **Best for** | Structured, relational data; financial systems; anything needing complex queries | High-scale, flexible schema; caching; time-series; unstructured data |

### The myth that NoSQL is "faster"

NoSQL databases are not inherently faster than SQL databases. PostgreSQL with good indexing handles tens of thousands of queries per second. NoSQL wins on **horizontal scalability** — you can add nodes and distribute data in ways that are harder with SQL. But for most applications, PostgreSQL is fast enough and gives you far more powerful querying.

**Don't switch to NoSQL because you think SQL is slow. Switch because your data model or scale genuinely doesn't fit SQL.**

---

## Replication

**Replication** means keeping copies of your data on multiple machines. Two main reasons:

1. **Availability** — if the primary database dies, a replica can take over
2. **Read scaling** — spread read queries across multiple replicas

```
         Writes
           │
           ▼
    ┌─────────────┐
    │   Primary   │ ──── replicates ────> [Replica 1]
    └─────────────┘                  └──> [Replica 2]
                                          ↑
                                        Reads
```

**Replication lag:** replicas are slightly behind the primary. A write goes to primary → syncs to replicas in milliseconds. This means a user might write something and immediately read a replica that hasn't caught up yet. For most apps, this is fine. For financial transactions or inventory ("is this item still in stock?"), you need to read from the primary.

---

## Sharding

When one database can't handle the volume of writes, you split the data across multiple databases. Each **shard** owns a partition of the data.

```
Shard by user_id:

user_id 1–10M    → Shard A (its own database server)
user_id 10M–20M  → Shard B
user_id 20M–30M  → Shard C
```

Your application (or a routing layer) directs each query to the right shard.

**Problems sharding introduces:**
- **Cross-shard queries** — "find all users who signed up this week" now requires asking all shards and merging results
- **Hotspots** — if user IDs 1–10M are all active celebrities, Shard A is overwhelmed while B and C are idle. Good shard key selection matters
- **Rebalancing** — when a shard gets too big, splitting it and migrating data is painful

Sharding is a last resort. Most systems never need it. Exhaust vertical scaling and read replicas first.

---

## Choosing a database — a practical framework

In an interview, when asked "what database would you use?", work through this:

```
1. Is the data relational? (entities with relationships, need JOINs)
   └── Yes → start with PostgreSQL

2. Do you need ACID transactions?
   └── Yes → relational database, full stop

3. Is the schema highly variable or document-like?
   └── Yes → consider a document store (MongoDB)

4. Is this a caching layer or session storage?
   └── Yes → Redis (key-value)

5. Is this write-heavy time-series or event log data at massive scale?
   └── Yes → Cassandra or a purpose-built time-series DB

6. Are you traversing complex relationships (social graph, fraud detection)?
   └── Yes → graph database (Neo4j)
```

When in doubt: **start with PostgreSQL**. It's extraordinarily capable, handles most workloads, and you can always migrate later when you actually understand your bottleneck.

---

## Key terms to remember

| Term | Meaning |
|------|---------|
| ACID | Atomicity, Consistency, Isolation, Durability — reliability guarantees for transactions |
| Index | Data structure for fast lookups; speeds reads, slows writes |
| Foreign key | A column that references a row in another table; enforces relationships |
| Replication | Copying data to multiple machines for availability and read scaling |
| Replication lag | The delay between a write on primary and its appearance on replicas |
| Sharding | Splitting data across multiple database servers to distribute write load |
| Full table scan | Reading every row in a table — what happens without an index |

---

## What's next

Next up: **Caching** — one of the most powerful tools for reducing database load and lowering latency, and also one of the trickiest to get right.
