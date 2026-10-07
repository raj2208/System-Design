# The "-ilities": Latency, Throughput, Availability, Reliability, Scalability

These five terms come up in every system design conversation. They're the vocabulary for describing how well a system performs and how it behaves under stress. Before you can design a good system, you need to be able to measure and reason about these properties.

---

## Latency

**Latency** is the time it takes for one operation to complete — from the moment a request is sent to the moment a response is received.

It's measured in milliseconds (ms) or microseconds (µs).

```
Client sends request  ──────────────────>  Server
                      <──────────────────  Server sends response
│←────────── latency ────────────────────→│
```

### What contributes to latency?

- **Network time** — physical distance, number of hops between machines
- **Processing time** — CPU work the server does to handle the request
- **Queue time** — how long the request waited before being picked up
- **Database time** — query execution, disk reads
- **Serialization** — converting data to/from JSON, Protobuf, etc.

### Latency benchmarks worth knowing

These rough numbers help you reason about what's fast vs slow:

| Operation | Approximate latency |
|-----------|-------------------|
| L1 cache reference | ~0.5 ns |
| RAM access | ~100 ns |
| SSD read | ~100 µs |
| HDD read | ~10 ms |
| Same datacenter network round trip | ~0.5 ms |
| Cross-country network round trip (US) | ~40 ms |
| Cross-ocean round trip | ~150 ms |

The takeaway: **disk is ~100x slower than memory, and the network is somewhere in between**. This is why caching exists — keep data in memory so you don't pay disk or network latency every time.

### Percentiles matter more than averages

Don't use average latency — it hides the worst cases. Use percentiles:

- **p50** (median): 50% of requests are faster than this
- **p95**: 95% of requests are faster than this
- **p99**: 99% of requests are faster than this
- **p99.9**: the "tail latency" — the worst 1-in-1000 requests

If your p50 is 20ms but your p99 is 2000ms, you have a problem — 1% of users are having a terrible experience, and at scale, 1% is a lot of people.

**In interviews:** when someone says "the system should be fast," ask "what's the latency target — p99? p99.9?" That's the kind of clarifying question that impresses interviewers.

---

## Throughput

**Throughput** is how much work a system can do per unit of time.

- For a web server: requests per second (RPS)
- For a database: queries per second (QPS)
- For a data pipeline: records per second, or bytes per second (GB/s)

```
Throughput = amount of work / time

e.g. 10,000 requests/second
```

### Latency vs throughput — they're not the same

A common confusion:

- **Low latency** = each individual request is fast
- **High throughput** = the system handles many requests at once

You can have high throughput with high latency (a batch processing system that crunches millions of records but each takes 5 seconds). You can have low latency with low throughput (a system that responds in 1ms but can only handle 10 requests/second).

The goal is usually **low latency AND high throughput** — fast responses AND lots of them. Achieving both requires careful architecture (connection pooling, async processing, horizontal scaling).

### Bottlenecks limit throughput

Throughput is always limited by the slowest part of the system — the **bottleneck**. Common bottlenecks:

- A single database getting hammered with queries
- A slow external API your service depends on
- A CPU-bound operation with no parallelism
- Network bandwidth

Identifying bottlenecks is a core skill in system design. You profile, find the constraint, and fix it — then a new bottleneck appears somewhere else. This is normal.

---

## Availability

**Availability** is the percentage of time a system is operational and able to handle requests.

```
Availability = uptime / (uptime + downtime)
```

### The nines

Availability is usually expressed in "nines":

| Availability | Downtime per year | Downtime per month |
|-------------|------------------|-------------------|
| 99% (two nines) | ~3.65 days | ~7.2 hours |
| 99.9% (three nines) | ~8.7 hours | ~43 minutes |
| 99.99% (four nines) | ~52 minutes | ~4 minutes |
| 99.999% (five nines) | ~5 minutes | ~26 seconds |

Going from 99.9% to 99.99% is not "a little bit better" — it means 10x less downtime. Each nine is dramatically harder (and more expensive) to achieve.

### What causes downtime?

- Hardware failures (disk dies, server crashes)
- Software bugs (bad deploy, memory leak)
- Network partitions (two parts of your system can't talk)
- Traffic spikes (system gets overloaded)
- Planned maintenance (taking servers down to upgrade them)

### How to increase availability

- **Redundancy** — run multiple instances; if one dies, others keep serving
- **Failover** — automatically switch to a backup when the primary fails
- **No single point of failure** — if one component dying takes down the whole system, that's a SPOF; eliminate them
- **Health checks** — detect failures fast so traffic is rerouted quickly
- **Graceful degradation** — if a non-critical part fails, keep serving degraded functionality instead of failing completely

---

## Reliability

**Reliability** and availability are related but distinct:

- **Availability**: is the system up?
- **Reliability**: does the system do the right thing when it is up?

A system can be highly available but unreliable — it's always running but produces wrong results half the time. A reliable system consistently does what it's supposed to do, handles errors gracefully, and doesn't corrupt data.

Reliability properties:
- **Correctness** — returns the right answer
- **Consistency** — same input → same output (no random failures)
- **Fault tolerance** — handles errors without crashing
- **Durability** — once data is written, it stays written (doesn't disappear)

In databases, **ACID** (Atomicity, Consistency, Isolation, Durability) is the classic set of reliability guarantees. More on that when we cover databases.

---

## Scalability

**Scalability** is the ability of a system to handle increased load without degrading in performance.

"Load" can mean:
- More users
- More requests per second
- Larger data volumes
- More complex queries

### Vertical scaling (scale up)

Add more resources to the **same machine**: bigger CPU, more RAM, faster SSD.

```
Before:  [Server: 4 CPU, 16GB RAM]
After:   [Server: 32 CPU, 128GB RAM]
```

**Pros:** Simple — no code changes, no architectural complexity.  
**Cons:** Hardware has physical limits. There's a ceiling. Also, one big machine is still a single point of failure.

### Horizontal scaling (scale out)

Add **more machines** and distribute the load across them.

```
Before:  [Server]
After:   [Server 1] [Server 2] [Server 3]
            ↑ Load balancer distributes traffic ↑
```

**Pros:** Theoretically no ceiling — just keep adding machines. Resilient — one machine dying doesn't take everything down.  
**Cons:** More complex. Requires a load balancer. Your application needs to be stateless (see client-server model). Data needs to be shared across instances.

### Which to use?

Start with vertical scaling — it's simpler and buys you time. Switch to horizontal when you've hit the ceiling or when you need redundancy for availability.

Most large-scale systems end up with a mix: reasonably large individual machines, deployed horizontally across many instances.

### Scalability is not just about servers

Databases, caches, queues — every component needs to scale. A common failure mode: you scaled your API servers horizontally but forgot your database is still a single machine getting 10x the queries. The database becomes the bottleneck. Scaling is a whole-system concern.

---

## How these properties trade off against each other

These properties often pull in opposite directions:

| Trade-off | Example |
|-----------|---------|
| Availability vs consistency | A distributed database can be always-available or always-consistent, but not always both (CAP theorem — coming later) |
| Latency vs throughput | Batching requests increases throughput but adds latency |
| Scalability vs simplicity | Horizontal scaling is powerful but operationally complex |
| Reliability vs cost | Redundancy costs money; more nines = more spend |

**In interviews:** you'll be asked to make trade-offs. There is no universally right answer — it depends on the requirements. The skill is being able to articulate *which* trade-off you're making and *why* given the constraints.

---

## Key numbers to have internalized

Interviewers often expect you to reason about scale. Keep these in your head:

| Fact | Value |
|------|-------|
| Seconds in a day | ~86,400 |
| Requests a single server can handle (rough) | ~10,000 RPS |
| Average web page weight | ~2 MB |
| Twitter at peak | ~500,000 tweets/day |
| YouTube uploads | ~500 hours of video/minute |

These let you do back-of-envelope estimates during interviews — a skill we'll practice explicitly in the interview prep phase.

---

## Summary

| Property | One-line definition |
|----------|-------------------|
| **Latency** | How long one request takes |
| **Throughput** | How many requests the system handles per second |
| **Availability** | What percentage of the time the system is up |
| **Reliability** | Does the system do the right thing consistently |
| **Scalability** | Can the system handle more load without breaking |

---

## What's next

The next concept is **vertical vs horizontal scaling** in more depth, followed by **stateless vs stateful services** — which completes the Phase 1 foundations before we move into the core building blocks.
