# Learning Roadmap

A phased approach to system design — from zero to interview-ready.

The path goes bottom-up: understand the primitives first, then the components built on top of them, then how those components behave at scale, and finally how to design full systems under pressure. Each phase builds on the last.

---

## Phase 1 — Foundations

> Goal: understand the environment that every system lives in before touching any architecture.

- How the internet works: DNS, HTTP/HTTPS, TCP/IP (just enough)
- Client-server model
- The "-ilities": latency, throughput, availability, reliability, scalability
- Vertical vs horizontal scaling — what they mean and when each breaks down
- Stateless vs stateful services

**Why start here:** These are the words system designers use constantly. Without them, every conversation about architecture is confusing.

---

## Phase 2 — Core Building Blocks

> Goal: know the standard components that show up in nearly every system design interview and production system.

- **Databases**
  - Relational (SQL) vs non-relational (NoSQL) — when to use which
  - Indexing — what it is and why it matters
  - ACID properties
  - Replication and sharding basics
- **Caching**
  - What a cache is and why it exists
  - Cache invalidation strategies (hardest problem in CS, genuinely)
  - Where to cache: in-memory (Redis), CDN, browser
- **Load Balancing**
  - What a load balancer does
  - Algorithms: round-robin, least connections, consistent hashing
  - Layer 4 vs Layer 7
- **Message Queues**
  - Why queues exist (decoupling producers from consumers)
  - Async vs sync communication
  - Kafka, RabbitMQ — what makes them different
- **Storage**
  - Block vs object vs file storage
  - When to reach for S3 vs a database vs a filesystem

**Why this order:** These are the Lego bricks. Every real-world system is just a combination of these pieces.

---

## Phase 3 — Distributed Systems Concepts

> Goal: understand what changes (and what breaks) when a system runs across multiple machines.

- CAP theorem — consistency, availability, partition tolerance
- Eventual consistency vs strong consistency
- Consistent hashing — how data gets distributed without reshuffling everything
- Leader election and consensus (Raft/Paxos — conceptual level only)
- Idempotency — why it matters for retries and failures
- Single points of failure and how to design around them
- Data replication strategies: leader-follower, multi-leader, leaderless

**Why this matters for a platform engineer:** You've probably seen these problems in prod without knowing the names. This phase names them.

---

## Phase 4 — API Design

> Goal: understand how services talk to each other and how to design those contracts well.

- REST — principles, resource modeling, status codes
- GraphQL — when and why
- gRPC — why it exists and where it wins
- API versioning strategies
- Rate limiting — algorithms: token bucket, leaky bucket, sliding window
- Authentication patterns: API keys, OAuth 2.0, JWT

---

## Phase 5 — Designing Real Systems

> Goal: practice putting all the pieces together by designing systems that exist in the real world.

Each system is a chance to rehearse the full design loop: requirements → estimates → components → trade-offs → failure modes.

| System | Key concepts it exercises |
|--------|--------------------------|
| URL shortener | Hashing, databases, caching, redirects |
| Rate limiter | Algorithms, distributed counters, Redis |
| Notification system | Queues, fan-out, push vs pull |
| News feed (Twitter/Instagram) | Fan-out on write vs read, ranking, caching |
| Video streaming (YouTube) | Object storage, CDN, encoding, chunking |
| Distributed cache (Redis clone) | Consistent hashing, eviction policies |
| Search autocomplete | Trie, prefix trees, caching, ranking |
| Ride-sharing (Uber) | Geospatial indexing, real-time updates, matching |

---

## Phase 6 — Interview Prep

> Goal: translate knowledge into confident, structured answers under time pressure.

- The interview framework: clarify requirements → estimate scale → high-level design → deep dive → trade-offs
- How to talk through trade-offs without knowing the "right" answer
- What interviewers actually want to hear (hint: it's the reasoning, not the diagram)
- Timed mock designs: 45-minute end-to-end walkthroughs
- Common follow-up questions and how to handle them

---

## Rough time investment

This isn't a race. A realistic pacing for someone learning on the side:

| Phase | Suggested depth before moving on |
|-------|----------------------------------|
| 1 — Foundations | Comfortable explaining each concept out loud |
| 2 — Building Blocks | Can describe each component, its trade-offs, and a use case |
| 3 — Distributed Systems | Can explain CAP theorem and consistency trade-offs without notes |
| 4 — API Design | Can compare REST/GraphQL/gRPC and design a simple API contract |
| 5 — Real Systems | Has designed at least 3-4 systems end-to-end |
| 6 — Interview Prep | Can complete a full design in 45 minutes with trade-off discussion |

---

## Notes

- Topics will be covered in roughly this order but not rigidly — interesting tangents are fine.
- Each phase gets its own folder in the repo with markdown files per concept.
- The goal isn't memorization. It's building enough intuition that you can reason through unfamiliar systems.
