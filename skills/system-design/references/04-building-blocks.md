# Building Blocks

**Sources:** *Grokking the System Design Interview* (system design basics); *System Design
Interview* Ch. 4–7 (Xu).

A catalog of the standard components, with selection criteria and the failure mode each one
introduces. **Every component you add is permanent operational cost** — the "when NOT to
use" column is as important as the rest.

---

## 1. Load balancers

Distribute traffic across backends, provide failover, and enable horizontal scaling.
Typically deployed at three points: client→web tier, web tier→internal services, and
service→database.

**Algorithms:** round robin · weighted round robin (heterogeneous capacity) · least
connections (long-lived, uneven connections) · least response time · least bandwidth ·
IP hash (stickiness, at the cost of even distribution).

**Health checks are mandatory** — backends that fail probes are removed from the pool
automatically and re-added when they recover.

**Failure mode introduced:** the load balancer is itself a SPOF. Run a redundant pair
monitoring each other, or use a managed service.

**Layer 4 vs. Layer 7:** L4 balances on IP/port (faster, protocol-agnostic); L7 balances on
HTTP content (path/header routing, TLS termination, richer but more expensive).

---

## 2. Caching

Exploits **locality of reference**: recently requested data is likely to be requested again.
Used at every layer — CPU, OS page cache, DNS, browser, CDN, application, database.

**Placement:**
- **Client/browser:** free, but you cannot invalidate it. Use short TTLs or versioned URLs.
- **CDN:** static and edge-cacheable content (Section 3).
- **Application-server cache:** local memory is fastest but each server has a different
  cache, so hit rate degrades as you add servers and load balancing becomes cache-hostile.
- **Distributed cache (Redis/Memcached):** shared across servers, consistent hit rate,
  one network hop. The usual default.
- **Database cache:** buffer pool / query cache; already there, often the cheapest win.

**Patterns:**
- **Cache-aside (lazy loading):** application checks cache, on miss reads the store and
  populates. Simple, resilient, but every miss costs a round trip and the first request
  after a write is stale.
- **Read-through:** the cache itself loads on miss. Cleaner application code.
- **Write-through:** write to cache and store together — consistent, slower writes.
- **Write-behind:** write to cache, asynchronously flush. Fast, and it **can lose data**.
- **Refresh-ahead:** proactively refresh popular items before expiry. Avoids stampedes on
  hot keys; wastes work on cold ones.

**Invalidation** is the hard part. Options: TTL (simple, always somewhat stale),
explicit invalidation on write (precise, easy to miss a path), versioned keys (safe, leaves
garbage to evict), or invalidation derived from the database change log (most reliable —
see `data-systems-design/references/11-stream-processing.md`).

**Eviction policies:** LRU (default) · LFU (stable popularity) · FIFO · random · TTL-based.

**Failure modes introduced:**
- **Stampede / thundering herd** on expiry or cold start. Mitigate with request coalescing,
  soft TTLs (serve stale while refreshing), and jittered expiry.
- **Staleness** wherever the invalidation path is incomplete.
- **The origin must survive a cold cache.** If it cannot, the cache is load-bearing with no
  redundancy — a latent outage.

**When NOT to cache:** write-heavy data, data read once, anything where staleness is
unacceptable and you have no invalidation path.

---

## 3. CDN

Geographically distributed edge servers for static assets — and, increasingly, dynamic
content assembled at the edge.

**Use for:** images, video, CSS, JS, fonts, downloads; any large static payload where egress
cost and latency matter.

**Considerations:** cost (CDN traffic is billed — cache only what pays for itself), TTL
tuning, a **fallback path to origin** when the CDN fails, and invalidation. Prefer
**versioned/fingerprinted URLs** over purge APIs: purge propagation is slow and best-effort,
whereas a new URL is correct immediately.

---

## 4. Proxies

- **Forward proxy:** sits in front of clients; used for egress control, filtering, and
  collapsing identical outbound requests.
- **Reverse proxy:** sits in front of servers; used for TLS termination, compression,
  caching, request routing, and hiding backend topology. (Load balancers are typically
  reverse proxies with a selection algorithm.)

**Collapsed forwarding** — merging concurrent identical requests into one upstream call —
is the same idea as cache request coalescing and is a cheap defense against stampedes.

---

## 5. Indexes

An index makes reads faster by adding a derived structure keyed on a searchable field.
The library-catalog analogy: without one you scan every shelf.

**The trade every time:** an index speeds reads and **slows writes**, because every write
must update every index on the table. A table with a dozen indexes has a dozen-fold write
amplification. Indexes also consume storage and memory.

Add indexes from measured query plans, not intuition. Audit and drop unused ones — they are
pure cost. Details: `data-systems-design/references/03-storage-and-retrieval.md`.

---

## 6. Message queues and streams

**Task queues (RabbitMQ, SQS, Celery):** work distribution, per-message ack, redelivery on
failure. Load balancing across consumers **destroys ordering** — do not fan messages for the
same entity across consumers if order matters.

**Partitioned logs (Kafka, Kinesis, Pulsar):** ordered within a partition, retained after
consumption, replayable by resetting offsets. Partition by entity key to get per-entity
ordering. Parallelism is bounded by partition count.

**Choose a log when** you need replay, ordering, or multiple independent consumers.
**Choose a task queue when** you need per-message retry semantics and order does not matter.

**Non-negotiables for either:** idempotent consumers (delivery is at-least-once), bounded
queue size with a defined overflow behavior, a dead-letter path with an alert and an owner,
and monitored consumer lag with retention sized against your worst realistic outage.

---

## 7. Consistent hashing

Solves the resharding problem. With `hash(key) mod N`, changing `N` remaps nearly every key
— catastrophic during a scale-out, and the canonical mistake.

**How it works:** map both keys and servers onto a hash ring. A key belongs to the first
server encountered moving clockwise. Adding or removing a server only remaps the keys
between it and its neighbor — roughly `k/n` keys instead of nearly all of them.

**Virtual nodes** are what make it work in practice: each physical server occupies many
points on the ring, which evens out distribution and makes rebalancing smoother. More
virtual nodes means more even distribution and more metadata.

**Used by:** Cassandra/DynamoDB partitioning, Memcached client sharding, load balancer
sticky routing, CDN request routing.

---

## 8. Rate limiting

Prevents resource starvation from DoS (intentional or not), reduces cost (especially with
paid third-party APIs), and protects servers from overload.

| Algorithm | How it works | Pros | Cons |
|---|---|---|---|
| **Token bucket** | Tokens refill at a fixed rate up to a capacity; each request consumes one | Simple, memory-efficient, **allows short bursts**. Used by Amazon and Stripe | Two parameters (size, refill rate) that are hard to tune |
| **Leaking bucket** | FIFO queue drained at a fixed rate; full queue drops requests | Memory-efficient; **stable outflow rate** — good for protecting a fixed-capacity downstream. Used by Shopify | A burst fills the queue with old requests, starving recent ones |
| **Fixed window counter** | Counter per fixed time window | Trivial to implement and reason about | **Bursts at window edges** can pass up to 2× the quota |
| **Sliding window log** | Timestamp log per client, evicting old entries | Precise; no edge problem | Memory grows with request rate, including rejected requests |
| **Sliding window counter** | Weighted blend of current and previous window | Smooths the edge problem; memory-efficient | Approximate |

**Token bucket is the sensible default.** Choose leaking bucket when a downstream needs a
steady rate. Choose sliding window log only when exactness is required.

**Bucket granularity:** per user per endpoint, per IP, and a global bucket are different
buckets serving different purposes — a user allowed 1 post/second, 150 friend-adds/day, and
5 likes/second needs three buckets.

**Also specify:** where counters live (Redis is typical; local counters do not work across a
fleet), the response on limit (429 with `Retry-After`, plus `X-RateLimit-*` headers), and
whether excess requests are dropped or queued.

---

## 9. Unique ID generation in distributed systems

Auto-increment does not work across shards. Options:

| Approach | Properties | Cost |
|---|---|---|
| **UUID v4** | 128-bit, no coordination, no collisions in practice | Not sortable; 128 bits bloats indexes; random insert order fragments B-trees |
| **Multi-master auto-increment** | Each server increments by `k` | Hard to scale, breaks when servers are added or removed |
| **Ticket server** | A central server hands out IDs | Simple; **a SPOF** and a scaling bottleneck |
| **Snowflake** | 41-bit timestamp + 10-bit machine ID + 12-bit sequence | 64-bit, **time-sortable**, no coordination on the hot path. Requires machine-ID assignment and depends on clock behavior |
| **ULID / KSUID** | Timestamp prefix + randomness | Sortable and coordination-free; larger than 64 bits |

**Time-sortable IDs are worth a lot** — they give you chronological ordering for free and
keep B-tree inserts sequential. But they leak creation time, and Snowflake-style schemes
must handle clock skew and backwards clock jumps explicitly (see
`data-systems-design/references/08-distributed-systems-faults.md`).

---

## 10. Client-server communication

| Technique | How | Use when | Cost |
|---|---|---|---|
| **Short polling (Ajax)** | Client asks repeatedly on a timer | Simplicity; low update frequency | Wasted requests; latency bounded by the interval |
| **Long polling** | Server holds the request open until data or timeout | Near-real-time without WebSocket support | Holds a connection per client; reconnect churn |
| **WebSocket** | Persistent bidirectional TCP connection | True bidirectional, low latency — chat, gaming, live collaboration | Stateful connections complicate load balancing, deploys, and scaling |
| **Server-Sent Events (SSE)** | Persistent one-way server→client stream over HTTP | Server push only — feeds, notifications, progress | Unidirectional; connection-limit issues on HTTP/1.1 |

**Choose the weakest one that meets the requirement.** WebSockets are frequently chosen for
problems SSE or long polling would solve, and they make every subsequent operational
decision harder: load balancers need sticky or connection-aware routing, deploys drop
connections, and reconnect storms after a deploy are a real load event you must plan for.

---

## 11. Redundancy and replication

**Redundancy** duplicates components to remove single points of failure. **Replication**
copies data across nodes.

- **Active-passive:** the standby takes over on failure. Simpler; the standby's readiness is
  untested until you need it — so test it on a schedule.
- **Active-active:** all instances serve traffic. Better utilization and proven readiness;
  requires conflict handling for writes.

**Build redundancy at every tier**, and remember that untested failover is not failover.
The failure path is the least-exercised code in the system and runs at the worst moment.

---

## 12. Choosing a component: the checklist

Before adding anything to the diagram:

1. **What number justifies it?** Cite the estimate from `02-estimation.md`.
2. **What does it cost?** Money, latency (one more hop), operational burden, and an
   availability multiplier on the request path.
3. **What breaks when it fails?** Every component needs a defined degraded mode.
4. **What is the simpler alternative, and why is it insufficient?**
5. **Who operates it?** A component nobody on the team knows how to debug at 3 a.m. is a
   liability regardless of its technical merits.
6. **Can we remove it later?** Reversibility is a feature; prefer reversible choices when
   the numbers are uncertain.
