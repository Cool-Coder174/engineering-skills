# Building Blocks

**Sources:** *Grokking the System Design Interview*, system design basics. *System Design
Interview*, Chapters 4 to 7 (Xu).

This is a catalog of standard components. Each entry gives the selection rule and the failure
mode that the component adds.

**Every component that you add is a permanent operational cost.** So the "do not use it when"
text matters as much as the rest.

---

## 1. Load balancers

A load balancer spreads traffic across servers. It provides failover. It permits horizontal
scaling.

Teams place load balancers at three points: between clients and the application tier, between
the application tier and internal services, and between services and databases.

**Algorithms.** Round robin. Weighted round robin, for servers of different capacity. Least
connections, for long connections that spread unevenly. Least response time. Least bandwidth.
IP hash, which gives a stable mapping and costs even distribution.

**Health checks are necessary.** The load balancer removes a server that fails a check. It
adds the server again when the server recovers.

**Failure mode added:** the load balancer is a single point of failure. Run two that monitor
each other, or use a managed service.

**Layer 4 compared to Layer 7.** Layer 4 routes on IP address and port. It is faster and it
ignores the protocol. Layer 7 routes on HTTP content. It supports routing by path or header,
and TLS termination. It costs more.

---

## 2. Caches

A cache works because of locality of reference: a recent request is likely to repeat. Caches
exist at every layer, from the CPU to the browser.

### Where to put a cache

| Location | Property |
|---|---|
| Browser | Free. **You cannot invalidate it.** Use short expiry times or versioned URLs. |
| CDN | For static content and edge content. See Section 3. |
| Application server memory | Fastest. Each server holds different data, so the hit rate falls as you add servers. |
| Distributed cache (Redis, Memcached) | Shared across servers. Stable hit rate. Costs one network call. **This is the usual default.** |
| Database buffer pool | Already present. Often the cheapest improvement. |

### Patterns

- **Cache-aside.** The application checks the cache. On a miss, it reads the database and
  writes to the cache. Simple and resilient. Every miss costs a round trip. The first read
  after a write returns old data.
- **Read-through.** The cache reads the database on a miss. The application code is simpler.
- **Write-through.** Write to the cache and the database together. Consistent. Writes are
  slower.
- **Write-behind.** Write to the cache. Write to the database later. Fast, and **it can lose
  data**.
- **Refresh-ahead.** Refresh popular items before they expire. It prevents a load spike on hot
  keys. It wastes work on cold keys.

### Invalidation

Invalidation is the hard part. Four methods exist:

| Method | Property |
|---|---|
| Expiry time | Simple. The data is always somewhat old. |
| Explicit invalidation on write | Exact. It is easy to miss one write path. |
| Versioned keys | Safe. It leaves old entries for the eviction policy to remove. |
| Invalidation from the database change log | Most reliable. Read `data-systems-design/references/11-stream-processing.md`. |

### Eviction policies

LRU is the default. LFU suits stable popularity. FIFO and random also exist. Expiry-based
eviction removes items after a fixed time.

### Failure modes added

1. **Load spike on expiry or on a cold start.** To reduce it, merge identical concurrent
   requests, serve old data during a refresh, and add a random offset to each expiry time.
2. **Old data** on every write path that does not invalidate.
3. **The database must survive an empty cache.** If it cannot, the cache is a necessary
   component with no redundancy. That is a future outage.

### Do not use a cache when

The data changes often. The system reads each item once. Old data is unacceptable and you
have no invalidation path.

---

## 3. CDN

A CDN is a set of servers in many locations that caches static content. Some CDNs also
assemble dynamic content at the edge.

**Use it for** images, video, CSS, JavaScript, fonts, and downloads. Use it for any large
static file where cost and latency matter.

**Four items to decide.** The cost, because CDN traffic is billed, so cache only what pays
for itself. The expiry time. **A path to the origin for the case where the CDN fails.** The
invalidation method.

**Prefer versioned URLs over a purge API.** A purge propagates slowly and can fail. A new URL
is correct at once.

---

## 4. Proxies

- **Forward proxy.** It sits in front of clients. Use it to control outbound traffic, to
  filter it, and to merge identical outbound requests.
- **Reverse proxy.** It sits in front of servers. Use it for TLS termination, compression,
  caching, request routing, and to hide the server layout.

A load balancer is a reverse proxy with a selection algorithm.

**Collapsed forwarding** merges identical concurrent requests into one upstream call. It is
the same idea as merging cache requests. It is a cheap defense against load spikes.

---

## 5. Indexes

An index makes reads faster. It is a second structure, ordered by a searchable field.

Use a library catalog as the comparison. Without a catalog, you examine every shelf.

**The trade is always the same. An index makes reads faster and writes slower.** Every write
must update every index on the table. A table with twelve indexes multiplies the write cost
by twelve. Indexes also consume disk and memory.

Add an index from a measured query plan. Do not add one from intuition. Find unused indexes
and remove them. An unused index is pure cost.

Read `data-systems-design/references/03-storage-and-retrieval.md`.

---

## 6. Message queues and logs

**Task queues** (RabbitMQ, SQS, Celery) distribute work. Each message has its own
acknowledgement. A failed message returns for retry.

**Warning: spreading messages across consumers destroys the order.** Do not spread messages
for one entity across consumers when the order matters.

**Partitioned logs** (Kafka, Kinesis, Pulsar) keep the order inside one partition. They keep
messages after a consumer reads them. A consumer can read them again by moving its offset.
Partition by entity key to keep the order for each entity. The partition count limits the
parallelism.

| Use a log when | Use a task queue when |
|---|---|
| You must read messages again | Each message needs its own retry behavior |
| You need order | Order does not matter |
| Several consumers read the same messages | One consumer group processes the work |

**Four requirements for both kinds:**

1. Consumers must be idempotent, because delivery is at-least-once.
2. The queue needs a maximum size and a defined behavior when it is full.
3. The queue needs a dead-letter destination, an alert, and an owner.
4. Consumer lag needs a monitor. Retention must exceed the longest outage that you expect.

---

## 7. Consistent hashing

Consistent hashing solves the repartitioning problem.

**The problem.** The formula `hash(key) mod N` maps almost every key to a new server when N
changes. That result makes scaling out very expensive. This is a frequent mistake.

**The method.** Place keys and servers on a hash ring. A key belongs to the first server
clockwise from it. Adding or removing a server moves only the keys between that server and
its neighbor. It moves about `k/n` keys instead of almost all of them.

**Virtual nodes make this method work in practice.** Each physical server occupies many
points on the ring. That placement evens out the distribution and smooths rebalancing. More
virtual nodes give a more even distribution and more metadata.

**Systems that use it:** Cassandra and DynamoDB for partitioning, Memcached clients for
sharding, load balancers for stable routing, and CDNs for request routing.

---

## 8. Rate limiting

A rate limiter has three purposes. It prevents resource exhaustion from denial-of-service
traffic, whether that traffic is deliberate or not. It reduces cost, which matters most for
paid third-party APIs. It protects servers from overload.

| Algorithm | Method | Advantage | Disadvantage |
|---|---|---|---|
| **Token bucket** | Tokens refill at a fixed rate up to a maximum. Each request uses one token. | Simple. Uses little memory. **Permits short bursts.** Amazon and Stripe use it. | Two parameters, and both are hard to tune |
| **Leaking bucket** | A FIFO queue drains at a fixed rate. A full queue rejects requests. | Uses little memory. **Output rate is steady.** Good for a downstream with fixed capacity. Shopify uses it. | A burst fills the queue with old requests and starves new ones |
| **Fixed window counter** | One counter for each fixed time window | Simple to build and to understand | **A burst at a window boundary can pass twice the limit** |
| **Sliding window log** | A list of timestamps for each client. Old entries are removed. | Exact. It has no boundary problem. | Memory grows with the request rate, including rejected requests |
| **Sliding window counter** | A weighted mix of the current and previous window | Reduces the boundary problem. Uses little memory. | Approximate |

**Use the token bucket as the default.** Use the leaking bucket when a downstream service
needs a steady rate. Use the sliding window log only when you need an exact count.

**Use separate buckets for separate purposes.** A user may post once each second, add 150
friends each day, and like 5 posts each second. That user needs three buckets. Buckets by IP
address and a global bucket serve different purposes again.

**Three more items to record.** Where the counters live, because local counters do not work
across many servers. Redis is the usual answer. The response when a client exceeds the limit,
which is status 429 with a `Retry-After` header and `X-RateLimit-*` headers. Whether excess
requests are rejected or queued.

---

## 9. Unique ID generation

An auto-increment column does not work across partitions. Five alternatives exist.

| Method | Property | Cost |
|---|---|---|
| **UUID v4** | 128 bits. No coordination. Collisions do not occur in practice. | Not sortable. 128 bits enlarge every index. Random order fragments a B-tree. |
| **Auto-increment on many nodes** | Each server increments by a step of k | Hard to scale. It breaks when you add or remove a server. |
| **Ticket server** | One server gives out IDs | Simple. **It is a single point of failure and a bottleneck.** |
| **Snowflake** | 41-bit timestamp, 10-bit machine ID, 12-bit sequence | 64 bits. **Sortable by time.** No coordination on the request path. It needs machine-ID assignment. It depends on clock behavior. |
| **ULID or KSUID** | A timestamp prefix and random bits | Sortable. No coordination. Larger than 64 bits. |

**An ID that sorts by time is valuable.** It gives chronological order at no cost. It keeps
B-tree inserts in order.

Two costs follow. The ID reveals its creation time. A Snowflake-style scheme must handle
clock skew and clocks that move backward. Read
`data-systems-design/references/08-distributed-systems-faults.md`.

---

## 10. Client-server communication

| Method | How it works | Use it when | Cost |
|---|---|---|---|
| **Short polling** | The client requests on a timer | The design must be simple. Updates are rare. | Wasted requests. The latency equals the interval. |
| **Long polling** | The server holds the request until data arrives or the request times out | You need low latency without WebSocket support | One connection for each client. Reconnections repeat often. |
| **WebSocket** | One connection carries data in both directions | You need two-way data with low latency, as in chat, games, and live editing | Connections hold state. That state complicates load balancing, deploys, and scaling. |
| **Server-Sent Events** | One connection carries data from server to client | The server sends data and the client does not, as in feeds and progress updates | One direction only. HTTP/1.1 limits the connection count. |

**Select the simplest method that meets the requirement.** Teams often select WebSockets for
a problem that Server-Sent Events or long polling would solve. WebSockets then make every
later decision harder. Load balancers need connection-aware routing. Deploys close
connections. **After a deploy, all clients reconnect at once. That reconnection is a real load
event. Plan for it.**

---

## 11. Redundancy and replication

**Redundancy** means more than one copy of a component. It removes single points of failure.
**Replication** means more than one copy of the data.

- **Active-passive.** A standby takes over on failure. Simpler. **Nothing tests the standby
  until you need it.** So test it on a schedule.
- **Active-active.** All instances serve traffic. Better use of hardware. The readiness is
  proven. It needs conflict handling for writes.

Add redundancy at every tier. **A failover path that nobody tested is not a failover path.**
It is the least-exercised code in the system, and it runs at the worst moment.

---

## 12. How to select a component

Answer these six questions before you add anything to the diagram.

1. **Which number requires it?** Cite the estimate from `02-estimation.md`.
2. **What does it cost?** Money. Latency, because it adds a network call. Operational work.
   It also lowers the availability of every request path that uses it.
3. **What breaks when it fails?** Every component needs a defined degraded mode.
4. **What is the simpler alternative? Why is that alternative not enough?**
5. **Who operates it?** A component that nobody can debug at 3 a.m. is a liability. Its
   technical merits do not change that.
6. **Can we remove it later?** A reversible choice is better than an irreversible one. Prefer
   reversible choices when the numbers are uncertain.
