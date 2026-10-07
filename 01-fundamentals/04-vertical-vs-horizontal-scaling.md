# Vertical vs Horizontal Scaling

We touched on these concepts in the "-ilities" note. This goes deeper — what actually happens when you scale each way, where each breaks down, and how real systems combine both.

---

## Vertical scaling (scale up)

You take an existing machine and make it more powerful: more CPU cores, more RAM, faster disks, faster network card.

```
         Before                      After
   ┌─────────────────┐        ┌─────────────────┐
   │  4 CPU cores    │   →    │  32 CPU cores   │
   │  16 GB RAM      │        │  256 GB RAM     │
   │  500 GB SSD     │        │  4 TB NVMe SSD  │
   └─────────────────┘        └─────────────────┘
```

### What you're actually buying

| Resource | Why it helps |
|----------|-------------|
| More CPU cores | Handle more concurrent requests; parallelize computation |
| More RAM | Cache more data in memory; avoid disk reads; run larger processes |
| Faster/larger SSD | Reduce disk I/O latency; store more data locally |
| Faster network | Higher throughput between services |

### When vertical scaling makes sense

- **Early stage** — your system is simple and one big machine is easier to operate than ten small ones
- **Stateful workloads** — anything that's genuinely hard to distribute (more on this below)
- **Databases** — scaling a database horizontally is complex; buying a bigger DB server buys significant headroom cheaply
- **Buying time** — you're under pressure; scaling up is fast and requires zero code changes

### Where vertical scaling breaks down

**1. There's a hard ceiling.**  
The biggest cloud instances top out around 448 vCPUs and 24 TB RAM (AWS u-24tb1.metal). Beyond that, there's nothing left to buy.

**2. Cost grows non-linearly.**  
Doubling the specs of a machine doesn't double the price — it often triples or quadruples it. Horizontal scaling with commodity hardware is almost always cheaper at scale.

```
2x performance vertically ≈ 4x the price
2x performance horizontally ≈ 2x the price
```

**3. It's still a single point of failure.**  
One giant machine going down takes everything with it. Vertical scaling does nothing for availability.

**4. Upgrades require downtime.**  
Resizing a VM or swapping hardware means taking the machine offline, even briefly. Horizontal scaling lets you do rolling deploys with zero downtime.

---

## Horizontal scaling (scale out)

Instead of making one machine bigger, you add more machines and spread the load across them.

```
         Before                        After

    ┌──────────────┐          ┌──────────────────────────┐
    │   Server     │          │      Load Balancer       │
    └──────────────┘          └──────────────────────────┘
                                   /        |        \
                            [Server 1] [Server 2] [Server 3]
```

Each server is identical. The load balancer (covered in Phase 2) distributes incoming requests across all of them. Add more servers → handle more traffic. Remove servers → save money during quiet periods. This is **elastic scaling** — the cloud makes it easy to automate.

### The critical requirement: statelessness

Horizontal scaling only works cleanly if your servers are **stateless** — they hold no memory of previous requests. If Server 1 stored your session in its local memory and your next request goes to Server 2, Server 2 has no idea who you are.

```
❌ Stateful servers (broken at scale)

Request 1 → Server 1  (stores session locally)
Request 2 → Server 2  (no session found → user logged out)

✅ Stateless servers (correct)

Request 1 → Server 1  (reads session from Redis)
Request 2 → Server 2  (reads same session from Redis)
```

The fix: **externalize all state**. Sessions go in Redis. User data goes in a database. Files go in object storage (S3). The servers themselves become interchangeable, disposable units.

### Where horizontal scaling breaks down

**1. Stateful services are hard to distribute.**  
Databases, caches, and message queues carry state by definition. Scaling them horizontally requires sharding, replication, and consistency trade-offs. This is where most of the complexity in distributed systems lives.

**2. Coordination overhead.**  
Ten servers need to agree on things that one server handled trivially. Distributed locks, consistent caching, and leader election all add complexity.

**3. Network becomes a bottleneck.**  
More machines means more inter-service communication. At very large scale, you start optimizing for network topology, rack placement, and bandwidth.

**4. Operational complexity.**  
Ten machines need monitoring, logging, deployment pipelines, and health checks. One machine is simpler to operate.

---

## Scaling the database — the hard problem

App servers are easy to scale horizontally because they're stateless. Databases are hard because they hold your data and you can't just add more identical copies without thinking carefully.

### Read replicas

Most applications read far more than they write. A simple and powerful pattern:

```
                  ┌─────────────────┐
   Writes ──────> │  Primary (RW)   │
                  └────────┬────────┘
                           │ replicates
              ┌────────────┴────────────┐
              ▼                         ▼
   ┌──────────────────┐      ┌──────────────────┐
   │  Replica 1 (RO)  │      │  Replica 2 (RO)  │
   └──────────────────┘      └──────────────────┘
              ▲                         ▲
   Reads ─────┘                         └───── Reads
```

All writes go to the **primary**. Reads are spread across **replicas**. This works well when reads dominate — social media feeds, product pages, dashboards.

**Trade-off:** replicas lag slightly behind the primary (replication delay). A user might write something and immediately read a replica that hasn't caught up yet. For most use cases this is fine; for some (financial transactions, inventory counts) it isn't.

### Sharding (horizontal partitioning)

When even the primary can't keep up with writes, you split the data across multiple databases. Each **shard** owns a slice of the data.

```
User IDs 1–10M    → Shard 1
User IDs 10M–20M  → Shard 2
User IDs 20M–30M  → Shard 3
```

Now write load is distributed across shards. But:
- Queries that span multiple shards are expensive (you have to ask all shards and merge results)
- Rebalancing when a shard gets too large is painful
- Application code needs to know which shard to talk to

Sharding is a last resort — exhaust vertical scaling and read replicas first.

---

## The hybrid reality

Real systems don't choose one or the other. They use both:

```
                        ┌─────────────────────────────┐
                        │       Load Balancer         │
                        └──────────────┬──────────────┘
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                  ▼
             [API Server]       [API Server]       [API Server]
             (medium size)      (medium size)      (medium size)
                    │                  │                  │
                    └──────────────────┼──────────────────┘
                                       ▼
                             ┌──────────────────┐
                             │  Primary DB       │  ← vertically scaled
                             │  (large instance) │    (32 CPU, 256GB RAM)
                             └──────────┬───────┘
                                        │
                             ┌──────────┴───────┐
                             ▼                  ▼
                         [Replica 1]        [Replica 2]
```

- App servers: horizontally scaled (stateless, cheap commodity instances)
- Primary database: vertically scaled (one beefy machine handles all writes)
- Database reads: spread across replicas

This is the standard architecture for most mid-to-large scale web applications.

---

## Decision framework

When you're asked "how would you scale this system?" in an interview, think through this:

1. **What's the bottleneck?** CPU? Memory? Disk I/O? Network? Database?
2. **Is the bottleneck stateless?** If yes, horizontal scaling is straightforward.
3. **If stateful (database), can read replicas help?** If reads dominate, yes.
4. **Is vertical scaling still viable?** If you're not near the ceiling and writes are the problem, scaling up the DB buys time.
5. **Is sharding necessary?** Only if you've exhausted the above.

Don't reach for sharding on day one. Most systems never need it.

---

## Summary

| | Vertical | Horizontal |
|-|---------|-----------|
| **What changes** | Bigger machine | More machines |
| **Complexity** | Low | Higher |
| **Ceiling** | Hard hardware limit | Virtually unlimited |
| **Cost curve** | Non-linear (expensive at top) | Linear |
| **Failure risk** | SPOF | Resilient |
| **Stateful services** | Works fine | Requires careful design |
| **Zero-downtime deploys** | Hard | Easy (rolling deploys) |

---

## What's next

The last concept in Phase 1 is **stateless vs stateful services** — a deeper look at what it means to externalize state, and the patterns (sessions, tokens, shared caches) that make horizontal scaling possible in practice.
