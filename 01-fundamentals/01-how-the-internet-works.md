# How the Internet Works

Before designing systems, you need to understand the environment they live in. Every web application, API, and microservice you'll ever design rides on top of the same set of protocols. This note explains what actually happens when a browser (or a service) sends a request somewhere.

---

## The big picture

When you type `https://google.com` into a browser and hit enter, a lot happens before you see a result. At a high level:

1. Your browser figures out the **IP address** for `google.com` (DNS)
2. Your machine opens a **connection** to that IP address (TCP)
3. Your browser sends an **HTTP request** over that connection
4. Google's server sends back an **HTTP response**
5. Your browser renders the response

Each of those steps has its own protocol and its own failure modes. System designers care about all of them.

---

## IP Addresses

Every machine on the internet has an IP address — a unique number that identifies it, like a street address for a computer.

- IPv4 looks like: `142.250.80.46` (four numbers, 0–255 each)
- IPv6 looks like: `2001:4860:4860::8888` (newer, longer, more addresses available)

When you send data anywhere on the internet, every packet is stamped with a source IP and a destination IP. Routers use these to forward packets toward their destination.

**Why it matters for system design:** When you scale a service across multiple servers, those servers all have different IPs. Something has to decide which IP a client should talk to — that's what load balancers and DNS do.

---

## DNS — The Internet's Phone Book

You type `google.com`, not `142.250.80.46`. DNS (Domain Name System) translates human-readable names into IP addresses.

**The lookup chain:**
1. Your browser checks its own cache first.
2. If not cached, it asks your OS (which checks its own cache).
3. If still not found, it asks your ISP's **DNS resolver**.
4. The resolver asks the **root nameservers** → they point to the `.com` nameservers.
5. The `.com` nameservers point to **Google's authoritative nameserver**.
6. Google's nameserver returns the IP address.
7. Your browser caches it and connects.

This whole process typically takes **20–120ms**. The result is cached based on the record's **TTL (Time to Live)** — a value Google sets that tells your machine how long to trust that answer.

**Why it matters for system design:**
- DNS is often used for **geographic routing** — returning different IPs to users in different regions so they connect to a nearby server.
- TTL is a trade-off: low TTL means changes propagate quickly but DNS gets hammered with lookups; high TTL means faster lookups but slow propagation when IPs change.
- DNS is a **single point of failure** if not handled carefully — hence why large services use multiple DNS providers.

---

## TCP/IP — How Data Actually Travels

IP gets a packet to the right machine. **TCP (Transmission Control Protocol)** makes sure it arrives correctly and in order.

TCP is a **connection-oriented** protocol. Before sending any data, the two machines do a **3-way handshake**:

```
Client → Server:  SYN        ("I want to connect")
Server → Client:  SYN-ACK    ("OK, I'm ready")
Client → Server:  ACK        ("Great, starting now")
```

After the handshake, data flows. TCP guarantees:
- **Delivery** — if a packet is lost, it gets retransmitted.
- **Order** — packets are reassembled in the right sequence.
- **No duplicates** — duplicate packets are discarded.

This reliability comes at a cost: the handshake adds latency before any data is sent.

**UDP** is the alternative — no handshake, no guarantees, just fire and forget. Faster, but you can lose data. Used for video calls, gaming, DNS lookups — anywhere where speed matters more than perfect delivery.

**Why it matters for system design:**
- Every TCP connection costs a handshake. High-traffic systems **reuse connections** (connection pooling) rather than opening a new one per request.
- The handshake latency is why **CDNs** terminate connections close to the user — less distance = less round-trip time for the handshake.
- Choosing TCP vs UDP is a real design decision for real-time systems.

---

## HTTP — The Language of the Web

HTTP (HyperText Transfer Protocol) is the application-layer protocol that runs on top of TCP. It's how browsers and servers talk to each other — and how most APIs work.

An HTTP request looks like:

```
GET /search?q=system+design HTTP/1.1
Host: google.com
User-Agent: Mozilla/5.0
Accept: text/html
```

An HTTP response looks like:

```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 42345

<html>...</html>
```

Key parts:
- **Method**: `GET` (fetch), `POST` (create), `PUT`/`PATCH` (update), `DELETE` (remove)
- **Status code**: `200` OK, `301` redirect, `404` not found, `500` server error
- **Headers**: metadata about the request/response (content type, auth tokens, caching rules)
- **Body**: the actual payload

**HTTPS** is just HTTP with **TLS encryption** on top. The connection is encrypted end-to-end so that nobody in between can read the data. This adds a TLS handshake on top of the TCP handshake — more latency upfront, but it only happens once per connection.

**Why it matters for system design:**
- HTTP is **stateless** — each request is independent; the server remembers nothing about previous requests. This is why sessions, cookies, and tokens exist.
- Status codes matter: a `503` means your service is overloaded; a `429` means a client is being rate-limited. These are signals systems use to communicate failure.
- HTTP/2 and HTTP/3 improve on HTTP/1.1 by multiplexing multiple requests over a single connection — important for performance at scale.

---

## Putting it all together

Here's the full journey for `https://google.com`:

```
Browser
  │
  ├─ DNS lookup: google.com → 142.250.80.46   (~50ms, cached after)
  │
  ├─ TCP handshake to 142.250.80.46:443        (~30ms round trip)
  │
  ├─ TLS handshake (for HTTPS)                (~60ms)
  │
  ├─ HTTP GET / request sent
  │
  └─ HTTP 200 response received → page renders
```

Total before you see anything: **~140ms** on a good day, more on a slow connection or if servers are far away.

This is why system design cares so much about **latency**: every hop in this chain adds time, and at scale (millions of users), small optimizations compound enormously.

---

## Key terms to remember

| Term | What it means |
|------|---------------|
| IP address | Unique identifier for a machine on a network |
| DNS | Translates domain names to IP addresses |
| TTL | How long a DNS record (or cached response) is trusted |
| TCP | Reliable, ordered, connection-based data transfer |
| UDP | Fast, connectionless, no delivery guarantees |
| HTTP | Request/response protocol for web communication |
| HTTPS | HTTP + TLS encryption |
| Stateless | Server holds no memory of previous requests |
| Latency | Time for one request/response round trip |

---

## What's next

With this foundation, the next concept to cover is the **client-server model** — how services are structured around the idea of requesters and responders, and why that model shapes almost every system design decision.
