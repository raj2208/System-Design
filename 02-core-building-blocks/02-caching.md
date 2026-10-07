# Caching

If databases are where data lives, caches are where frequently-used data hangs out so you don't have to go all the way to the database every time. Caching is one of the highest-leverage optimizations in system design — done well, it can reduce database load by 90% and cut response times from hundreds of milliseconds to single digits.

It also has a famous reputation for being tricky: *"There are only two hard things in computer science: cache invalidation and naming things."* — Phil Karlton

---

## Why caching exists

Recall the latency numbers from Phase 1:

| Operation | Latency |
|-----------|---------|
| RAM access | ~100 ns |
| SSD read | ~100 µs |
| Database query (disk) | ~1–10 ms |
| Cross-datacenter network call | ~150 ms |

A cache stores data **in memory** — orders of magnitude faster than hitting disk or the network. If 1,000 users per second are all requesting the same homepage, there's no reason to run 1,000 identical database queries. Run one, store the result in a cache, serve the other 999 from memory.

```
Without cache:
User 1 ──> API ──> Database (10ms query) ──> response
User 2 ──> API ──> Database (10ms query) ──> response
User 3 ──> API ──> Database (10ms query) ──> response
... × 1,000

With cache:
User 1 ──> API ──> Database (10ms query) ──> cache ──> response
User 2 ──> API ──> cache hit (0.1ms)    ──────────> response
User 3 ──> API ──> cache hit (0.1ms)    ──────────> response
... × 999
```

---

## Cache hit and cache miss

- **Cache hit** — the data you need is already in the cache. Fast. No database call.
- **Cache miss** — the data isn't in the cache. Fetch from the database, store it in the cache, return it.

**Hit rate** = hits / (hits + misses). A good cache has a high hit rate (90%+). If your hit rate is low, you're paying the overhead of checking the cache on every request for little benefit.

---

## Where caches live

There are several layers where caching happens, each serving a different purpose:

### 1. Client-side / browser cache

The browser stores resources (images, CSS, JS files) locally. Controlled by HTTP headers:

```
Cache-Control: max-age=86400    ← browser can cache this for 24 hours
ETag: "abc123"                  ← fingerprint of the content; browser can validate freshness
```

The request never even reaches your server. Great for static assets.

### 2. CDN (Content Delivery Network)

A CDN is a geographically distributed network of cache servers. When a user in Mumbai requests content, it comes from a CDN node in Mumbai rather than your origin server in Virginia — dramatically lower latency.

```
User in Mumbai
       │
       ▼
  CDN node in Mumbai  ──── cache hit ──── serves response (fast)
       │
       │ cache miss (first request only)
       ▼
  Origin server in Virginia  ──── fetches and stores in CDN node
```

CDNs cache static files (images, videos, JS bundles) and increasingly dynamic content too. Examples: Cloudflare, AWS CloudFront, Fastly.

### 3. Application / in-process cache

A cache living inside your application server's memory. No network call needed — the data is right there in RAM.

```python
_cache = {}

def get_user(user_id):
    if user_id in _cache:
        return _cache[user_id]        # instant
    user = db.query(user_id)
    _cache[user_id] = user
    return user
```

**Problem:** each server instance has its own cache. If you have 10 app servers, they each independently cache data and can have different versions. Not suitable for distributed systems.

### 4. Distributed cache (Redis, Memcached)

A dedicated caching service that all your app servers share. The most common pattern for production systems.

```
[App Server 1] ──┐
[App Server 2] ──┼──> [Redis] ──(miss)──> [Database]
[App Server 3] ──┘
```

All servers read and write the same cache. A cache hit on Server 1 benefits a subsequent request on Server 3.

**Redis vs Memcached:**

| | Redis | Memcached |
|-|-------|-----------|
| Data structures | Rich (strings, lists, sets, sorted sets, hashes) | Simple key-value only |
| Persistence | Optional (can write to disk) | In-memory only |
| Replication | Built-in | No |
| Use cases | Caching, sessions, queues, leaderboards, pub/sub | Pure caching |

**Choose Redis** for almost everything. Memcached is faster for the simplest key-value use case but Redis's versatility makes it the default.

---

## Cache invalidation strategies

This is the hard part. When the underlying data changes, how does the cache know to update?

### 1. TTL (Time To Live)

Give every cache entry an expiry time. When it expires, the next request fetches fresh data from the database.

```
SET user:raj "{ name: Raj, email: raj@... }"  EX 300    ← expires in 5 minutes
```

**Pros:** Simple. No coordination needed.  
**Cons:** Stale data for up to TTL duration. Raj changes his email — for up to 5 minutes, the cache serves the old email.

**When to use:** Data that changes infrequently and where brief staleness is acceptable. Product descriptions, user profiles, public content.

### 2. Cache-aside (lazy loading)

The application manages the cache manually. On a miss, it fetches from the DB and populates the cache. On a write, it **invalidates** (deletes) the cache entry so the next read fetches fresh data.

```
READ:
  1. Check cache
  2. Hit → return data
  3. Miss → fetch from DB → write to cache → return data

WRITE:
  1. Update database
  2. Delete (invalidate) cache entry
  3. Next read will be a miss → fetches fresh from DB → repopulates cache
```

**Pros:** Cache only contains data that's actually been requested. Resilient — if the cache goes down, the app still works (just slower).  
**Cons:** First request after invalidation is always a miss (cache miss penalty). Possible race condition: two requests simultaneously miss and both hit the DB.

**Most common pattern** for web applications.

### 3. Write-through

Every write goes to both the cache and the database simultaneously.

```
WRITE:
  1. Update cache
  2. Update database (synchronously)
  3. Return success
```

**Pros:** Cache is always up-to-date. No stale reads.  
**Cons:** Every write pays the penalty of writing to two places. Cache fills up with data that may never be read.

**When to use:** Read-heavy systems where stale data is never acceptable.

### 4. Write-back (write-behind)

Write to the cache immediately, write to the database **asynchronously** later.

```
WRITE:
  1. Update cache ← immediate
  2. Return success ← fast response to user
  3. Background job writes to DB later
```

**Pros:** Very fast writes — the user doesn't wait for the database.  
**Cons:** Risk of data loss if the cache goes down before the DB write happens. Complex to implement correctly.

**When to use:** Write-heavy workloads where brief data loss is acceptable (gaming leaderboards, analytics counters, likes/views counts).

---

## Cache eviction policies

A cache has limited memory. When it's full and you need to add something new, you have to evict something old. The policy determines what gets kicked out.

| Policy | What it does | When to use |
|--------|-------------|-------------|
| **LRU** (Least Recently Used) | Evicts the item not accessed for the longest time | General purpose — most common default |
| **LFU** (Least Frequently Used) | Evicts the item accessed least often overall | When some items are always popular (hot keys) |
| **FIFO** (First In, First Out) | Evicts the oldest item regardless of access pattern | Simple; rarely optimal |
| **TTL-based** | Evicts expired items first | Good for time-sensitive data |

Redis supports LRU, LFU, and several variants. The right choice depends on your access pattern.

---

## The thundering herd problem

Imagine a cached result expires at exactly midnight. Thousands of users are active. All of them send a request in the same second. All of them get a cache miss. All of them hit the database simultaneously. The database gets crushed.

```
Cache entry expires
        │
        ▼
1,000 simultaneous requests → 1,000 DB queries → database overwhelmed
```

Solutions:

**1. Cache lock / mutex:** Only one request populates the cache. Others wait for it to finish and then serve from cache. Most requests queue briefly instead of all hitting the DB.

**2. Probabilistic early expiration:** Before the TTL expires, a small percentage of requests proactively refresh the cache. The cache never fully expires for all users at once.

**3. Staggered TTLs:** Add random jitter to expiry times — `TTL = 300 + random(0, 60)`. Entries don't all expire simultaneously.

**4. Background refresh:** A separate process refreshes popular cache entries before they expire. The cache never goes cold.

---

## What not to cache

Caching is powerful but not appropriate for everything:

- **User-specific sensitive data** (passwords, payment info) — caching increases the attack surface
- **Data that changes on every request** (real-time stock prices, live seat availability) — TTL is always stale
- **Large objects** — a 50MB object in Redis is expensive; cache IDs and fetch the object when needed
- **Unique queries** — if every request has different parameters, the hit rate will be near zero and the cache just wastes memory

---

## Putting it together — a real example

A user visits a product page on an e-commerce site:

```
Request: GET /products/iphone-16

1. Check CDN → hit (static assets: images, CSS, JS served from edge)

2. API request hits app server
   → Check Redis: "product:iphone-16"
   → Cache hit → return JSON in 0.5ms

   (If miss:)
   → Query PostgreSQL: SELECT * FROM products WHERE slug='iphone-16'
   → Store result in Redis with TTL=600 (10 min)
   → Return JSON in 8ms

3. Browser renders page using cached assets + fresh product data
```

The database query only runs once per 10 minutes regardless of traffic volume. The CDN means static assets never hit your servers at all.

---

## Key terms

| Term | Meaning |
|------|---------|
| Cache hit | Requested data found in cache |
| Cache miss | Data not in cache; must fetch from source |
| Hit rate | % of requests served from cache (higher = better) |
| TTL | How long a cache entry is valid before expiry |
| Cache-aside | App checks cache first; populates on miss; invalidates on write |
| Write-through | Writes go to cache and DB simultaneously |
| Write-back | Writes go to cache immediately; DB update is async |
| Eviction | Removing entries from a full cache to make space |
| LRU | Evict the least recently used entry |
| Thundering herd | Simultaneous cache misses overwhelming the database |
| CDN | Geographically distributed cache for static/dynamic content |

---

## What's next

Next: **Load Balancing** — how traffic is distributed across multiple servers, and the algorithms that determine where each request goes.
