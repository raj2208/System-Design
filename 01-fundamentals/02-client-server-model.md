# The Client-Server Model

The client-server model is the architectural pattern that underlies almost every networked system. Understanding it deeply means you can reason about who is responsible for what in any system you design.

---

## The core idea

A **client** is anything that makes a request.  
A **server** is anything that handles that request and sends back a response.

That's it. The model is simple — but the nuances matter enormously at scale.

```
Client                        Server
  │                              │
  │──── request ────────────────>│
  │                              │ (does some work)
  │<─── response ────────────────│
  │                              │
```

The client initiates. The server waits and responds. This asymmetry is fundamental.

---

## What counts as a client?

A client is not just a browser. Anything that sends a request is a client:

- A **browser** visiting a website
- A **mobile app** fetching your timeline
- A **CLI tool** calling an API
- A **microservice** asking another microservice for data
- A **cron job** hitting an endpoint on a schedule
- A **load balancer** forwarding traffic to an upstream server

This last point is important: **in a distributed system, servers are also clients**. A service that handles your request often turns around and makes requests to a database, a cache, another service, or an external API. The roles are not fixed — they depend on the direction of the request.

---

## What counts as a server?

A server is any process that listens for incoming requests and responds to them. Common types:

| Server type | What it does |
|-------------|-------------|
| Web server | Serves static files (HTML, CSS, images) — e.g. Nginx, Apache |
| Application server | Runs business logic — e.g. a Node.js or Python API |
| Database server | Stores and retrieves data — e.g. Postgres, MySQL, MongoDB |
| Cache server | Stores frequently accessed data in memory — e.g. Redis, Memcached |
| Message broker | Passes messages between services — e.g. Kafka, RabbitMQ |
| CDN edge server | Serves cached content from a location close to the user |

In small systems, one machine might play multiple roles. At scale, each role typically gets its own fleet of machines.

---

## The request-response cycle in detail

Let's trace a real example: you open Instagram and your feed loads.

```
Your phone (client)
  │
  │  POST /api/feed?user=raj
  │  Authorization: Bearer <token>
  ▼
Instagram's API server (server + client)
  │
  ├──> queries database for your follows
  ├──> queries database for recent posts
  ├──> checks cache for already-ranked feed
  ├──> calls ML ranking service
  │
  └──> assembles response
  │
  ▼
Your phone receives JSON → renders the feed
```

The API server is a **server** to your phone and a **client** to the database, cache, and ML service simultaneously. This nesting of client-server relationships is what a microservices architecture looks like in practice.

---

## Multi-tier architecture

Most systems are organized into tiers, each with a clear responsibility:

```
┌─────────────────────────────────────────┐
│  Presentation tier (client)             │  Browser, mobile app
└─────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│  Application tier (server + client)     │  API servers, business logic
└─────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│  Data tier (server)                     │  Databases, caches, object storage
└─────────────────────────────────────────┘
```

**1-tier**: Everything on one machine. Fine for local dev tools, not for anything with users.

**2-tier**: Client talks directly to the database. Common in early desktop apps. The problem: clients need database credentials, business logic lives on the client (hard to update, easy to tamper with), and you can't scale the two layers independently.

**3-tier**: Client → Application server → Database. This is the standard model for web apps. The application server holds business logic and is the only thing that touches the database directly. This separation is why you can update your API without shipping a new app to every user.

**N-tier**: The application layer itself is split into many services. This is microservices — useful for large teams and complex systems, but adds operational overhead.

---

## The alternative: peer-to-peer

In the client-server model, servers are always on and waiting; clients initiate. In a **peer-to-peer (P2P)** model, every node is both a client and a server — it can initiate requests and handle them.

Examples: BitTorrent, blockchain networks, some video calling systems.

Trade-offs:
- No central server to go down (more resilient)
- Harder to control, enforce policy, or update consistently
- Coordination is much more complex

For most systems you'll design, client-server is the right default. P2P comes up in specific distributed systems problems (file sharing, decentralized storage, real-time communication).

---

## Why servers are stateless (and why that matters)

HTTP is stateless — the server remembers nothing about you between requests. Every request arrives cold. This is a deliberate design choice, and it has a huge impact on scalability.

**Stateless server:**
- Any instance can handle any request
- You can add or remove instances freely
- Load balancers can route to any server without caring which one served you last

```
Request 1 → Server A
Request 2 → Server B   ← different server, no problem
Request 3 → Server C
```

**Stateful server (the problem):**
- If session state lives on the server, the same client must always hit the same server
- This is called **sticky sessions** — it works but limits your ability to scale and recover from failures

The solution to needing state is to **move it out of the server**:
- Store sessions in a shared cache (Redis)
- Use signed tokens (JWTs) so the client carries their own state
- Store user data in a database all servers can reach

This is why "design for statelessness" is one of the first principles of scalable system design.

---

## Synchronous vs asynchronous communication

In the basic client-server model, the client **waits** for the server to respond before doing anything else. This is **synchronous** communication — the client is blocked.

```
Client ──request──> Server
Client (waiting...)
Client <──response── Server
Client continues
```

This works fine for most interactions. But it has limits:
- If the server is slow, the client is stuck
- If the server is down, the client fails immediately
- For long-running tasks (video encoding, report generation), keeping a connection open is wasteful

**Asynchronous** communication decouples the two sides. The client sends a request and moves on; the server processes it and notifies the client later (via webhook, polling, or a message queue).

```
Client ──request──> Queue
Client continues immediately

Server picks up job from Queue
Server ──result──> Client (via callback/webhook)
```

When to use async:
- Work that takes more than a few seconds
- Work that can be retried if it fails
- Decoupling two services that shouldn't be tightly coupled

You'll see this pattern constantly in system design interviews.

---

## Key takeaways

- **Client initiates, server responds** — roles are determined by the direction of the request, not the type of machine
- **Servers are also clients** — every layer of a distributed system is making requests to the layer below it
- **Statelessness enables horizontal scaling** — keep state out of your servers; put it in a shared store
- **Sync vs async is a design choice** — not every response needs to be immediate

---

## What's next

The next concept covers the **"-ilities"** — latency, throughput, availability, reliability, and scalability. These are the metrics that define whether a system is actually good, and they're the vocabulary that every system design conversation uses.
