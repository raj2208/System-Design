# Stateless vs Stateful Services

This concept keeps coming up — we touched on it in the client-server model and again in scaling. Now let's go deep. Understanding the difference between stateless and stateful services, and knowing how to design around state, is one of the most practical skills in distributed systems.

---

## What is "state"?

**State** is any data that a service needs to remember across multiple requests.

Examples of state:
- "This user is logged in" (session)
- "This user has added 3 items to their cart" (cart contents)
- "This job is 60% complete" (job progress)
- "This counter is at 1,042,873" (a running total)

If a service forgets everything between requests, it is **stateless**.  
If a service remembers something between requests, it is **stateful**.

---

## Stateless services

A stateless service treats every request as if it's the first one. It uses only what's in the request itself to produce a response.

```
Request A ──> [Server]  →  Response A   (server forgets everything)
Request B ──> [Server]  →  Response B   (server forgets everything)
```

**Examples of naturally stateless operations:**
- Convert a temperature from Celsius to Fahrenheit
- Resize an image
- Look up a product by ID (the data lives in a database, not the server)
- Validate a JWT token (the token carries all the information needed)

### Why stateless services are desirable at scale

Because every server instance is identical and holds no unique information, you can:

- **Route any request to any server** — the load balancer doesn't need to track which server a user was on before
- **Add/remove instances freely** — spinning up a new server works immediately; shutting one down loses nothing
- **Recover from crashes silently** — if a server dies, requests just go to another one
- **Deploy with zero downtime** — replace servers one at a time while others keep serving traffic

```
   User
    │
    ▼
[Load Balancer]
    │
    ├──> [Server 1]  ← handles request 1
    ├──> [Server 2]  ← handles request 2 (different server, no problem)
    └──> [Server 3]  ← handles request 3
```

All three servers are interchangeable because none of them hold anything unique.

---

## Stateful services

A stateful service remembers something between requests. That memory is what makes certain requests possible — but it's also what makes scaling hard.

```
Request 1 ──> [Server]  (remembers: user is logged in)
Request 2 ──> [Server]  (uses memory: yes, still logged in)
```

**Examples of inherently stateful services:**
- A database (stores your data permanently)
- A cache (stores data in memory for fast access)
- A message queue (holds messages until they're consumed)
- A WebSocket connection (maintains an open channel with a client)
- An in-progress file upload (partially received data must be held somewhere)

### The problem with stateful servers

If the state lives **inside the server process** (in memory, on its local disk), you get a hard constraint: **the same client must always talk to the same server.**

This is called **sticky sessions** or **session affinity**:

```
   User
    │
    ▼
[Load Balancer] ──── must always route this user to Server 1 ────> [Server 1] ← session here
                                                                    [Server 2]
                                                                    [Server 3]
```

Problems with sticky sessions:
- **Poor load distribution** — Server 1 might be overloaded while Server 2 is idle
- **Failure is catastrophic** — if Server 1 dies, that user's session is gone, they get logged out
- **Scaling is complicated** — you can't just add Server 4 and have it share the load for existing users

---

## The solution: externalize your state

The standard pattern for making stateful services scale like stateless ones is to **move state out of the server and into a shared store**.

```
         Before (state in server)                After (state externalized)

   [Server 1] ← session for user A        [Server 1] ──┐
   [Server 2] ← session for user B        [Server 2] ──┼──> [Redis / shared cache]
   [Server 3] ← session for user C        [Server 3] ──┘         (all sessions here)
```

Now all three servers can handle any user's request because they all read from the same place.

### Where to put externalized state

| Type of state | Where it goes |
|---------------|--------------|
| User sessions | Redis (fast, in-memory, supports TTL/expiry) |
| User data | Database (durable, queryable) |
| Files and uploads | Object storage (S3, GCS) |
| Temporary job state | Redis or a message queue |
| Configuration | Environment variables or a config service |

---

## JWT: carrying state in the token itself

Sessions stored server-side (even in Redis) require a round trip to the session store on every request. There's another approach: **put the state in the token** and send it with every request.

**JWT (JSON Web Token)** is the most common implementation. When a user logs in, the server creates a signed token containing the user's identity and permissions:

```
Header.Payload.Signature

Payload example:
{
  "user_id": "raj123",
  "role": "admin",
  "expires_at": "2026-11-01T00:00:00Z"
}
```

The token is **signed** with a secret key. The server can verify it is genuine without looking anything up — the proof is in the signature.

```
Request with JWT ──> [Any Server]
                         │
                         ├── verifies signature (no DB/Redis call needed)
                         └── extracts user_id, role from payload
```

**Pros:**
- Zero state on the server — truly stateless
- No session store to maintain
- Works well across multiple services (microservices all trust the same signature)

**Cons:**
- You can't invalidate a token before it expires (if a user is banned, their JWT still works until expiry)
- Larger payload than a session ID (sent with every request)
- Secret key rotation requires care

**Session store vs JWT:** For most apps, JWTs are fine. If you need instant revocation (security-critical apps, financial systems), use server-side sessions in Redis so you can delete the session immediately.

---

## WebSockets: the stateful connection problem

HTTP is stateless by design — request, response, done. But some features need a **persistent, bidirectional connection**:

- Live chat
- Real-time notifications
- Collaborative editing (Google Docs)
- Live sports scores

**WebSockets** solve this. The client opens a connection and it stays open — the server can push data at any time without the client polling.

```
Client ──── opens WebSocket ────> Server
              (connection stays open)
Server ──── pushes message ────> Client  (anytime)
Client ──── sends message  ────> Server  (anytime)
```

The problem: a WebSocket connection is **pinned to one server**. If that server dies, the connection dies. This makes WebSocket servers inherently stateful and harder to scale.

Common solutions:
- Use a **message broker** (Redis Pub/Sub, Kafka) so any server can broadcast to any connected client
- Route WebSocket connections through a dedicated **connection service** separate from your API servers

---

## Stateful services that can't be avoided

Some services are stateful by nature and no amount of clever design makes them stateless:

**Databases** — the whole point is to store state durably. Scaled via replication and sharding (covered in Phase 2).

**Caches** — hold data in memory. If a cache node dies, that data is gone (usually okay — it can be re-fetched from the DB).

**Message queues** — hold messages in flight. Scaled by partitioning (Kafka) or clustering (RabbitMQ).

For these, you accept the statefulness and design for **resilience** instead:
- Run multiple replicas so failure of one doesn't lose data
- Use leader election to decide which replica is authoritative
- Accept that distributed state comes with consistency trade-offs (CAP theorem — coming in Phase 3)

---

## Practical checklist for designing a scalable service

When designing any service, ask these questions:

1. **What state does this service need?**
2. **Where does that state live?** (In-process memory? Redis? Database? Client-side token?)
3. **What happens if a server instance dies?** Is any state lost?
4. **Can any server handle any request?** If not, why not — and is that necessary?
5. **How does state grow over time?** Does it need expiry/eviction policies?

If you can answer these five questions, you understand your service's scalability characteristics.

---

## Summary

| | Stateless | Stateful |
|-|-----------|---------|
| **Remembers across requests?** | No | Yes |
| **Horizontal scaling** | Easy | Requires externalizing state |
| **Failure impact** | None (another server takes over) | Depends on where state lives |
| **Examples** | API servers, image resizers | Databases, caches, WebSocket servers |
| **Design goal** | Make as much as possible stateless | Isolate state in dedicated, resilient stores |

The guiding principle: **stateless where possible, stateful where necessary, and the state always lives in a purpose-built store — not inside your application servers.**

---

## Phase 1 complete

You've now covered all five foundations:

1. How the internet works (DNS, TCP, HTTP)
2. The client-server model
3. The "-ilities" (latency, throughput, availability, reliability, scalability)
4. Vertical vs horizontal scaling
5. Stateless vs stateful services

**Phase 2** moves into the core building blocks — the components that show up in almost every real system: databases, caching, load balancing, message queues, and storage. This is where system design starts to get concrete.
