# The Scaling Ladder: Zero to Millions of Users

**Source:** *System Design Interview* Ch. 1 (Xu), with reliability framing from
*Grokking the System Design Interview*.

Scaling is an iterative process, not a destination. This is the standard progression, in
order, with **the signal that tells you to take each step**. The signal matters more than
the step: taking a step before its trigger is over-engineering, and taking it after is an
incident.

> Architectures are workload-specific. An architecture appropriate for one level of load is
> unlikely to cope with 10× that load. Expect to revisit the architecture on roughly every
> order-of-magnitude increase.

---

## Step 0 — Single server

Everything on one machine: web app, database, cache. The request flow: user resolves a
domain via DNS → gets an IP → sends HTTP to the web server → receives HTML or JSON.

**This is the correct architecture for most new systems.** It is simple, cheap, easy to
reason about, and easy to operate. Do not leave it without a reason.

**Trigger to move on:** the single server cannot handle the traffic, or you cannot tolerate
its failure.

---

## Step 1 — Separate the data tier

Split the web/mobile tier from the database so each can scale independently.

**Why now:** they have different resource profiles (CPU vs. IOPS/memory) and different
scaling characteristics. Web servers are stateless and easy to add; databases are not.

**Choosing the database:** relational is the default and remains the right answer for the
overwhelming majority of applications. Consider non-relational when you have super-low
latency requirements, unstructured or non-relational data, only need to serialize and
deserialize data, or need to store a massive amount of data. See `05-database-selection.md`.

---

## Step 2 — Vertical vs. horizontal scaling

**Vertical (scale up):** a bigger machine. Simple, and with no software changes. Limits:
there is a hard ceiling, it is expensive at the high end, and — critically — **it provides
no failover or redundancy.** One machine failing takes the whole service down.

**Horizontal (scale out):** more machines. More complex, but it is the only path past the
ceiling and the only way to get redundancy.

**Rule:** scale up first because it is simpler; scale out when you hit the ceiling or when
you need redundancy. For stateful data systems, delay scaling out as long as the numbers
allow — distributing state is where the complexity lives.

---

## Step 3 — Load balancer

Put a load balancer in front of multiple web servers. Users hit the load balancer's public
IP; servers hold private IPs and communicate over the internal network.

**What you gain:** failover (one server dies, traffic shifts) and horizontal capacity.

**Selection algorithms** (from Grokking):
- **Round robin** — cycles through the list. Best when servers are identical and there are
  few persistent connections.
- **Weighted round robin** — servers get an integer weight reflecting capacity; useful for
  heterogeneous hardware.
- **Least connections** — fewest active connections. Good with many long-lived, unevenly
  distributed connections.
- **Least response time** — fewest connections *and* lowest average response time.
- **Least bandwidth** — least traffic in Mbps.
- **IP hash** — hash of client IP picks the server. Gives stickiness, at the cost of even
  distribution.

**Health checks are mandatory.** The load balancer must probe backends and remove failures
from the pool automatically; otherwise you have distributed the traffic but not the
resilience.

**The load balancer itself is a SPOF.** Run a redundant pair that monitor each other, or
use a managed service that does.

---

## Step 4 — Database replication

Primary (leader) for writes, replicas (followers) for reads.

**Gains:** read throughput scales with replicas; better availability (a replica can be
promoted); data safety through geographic distribution.

**The costs you must design for:**
- **Replication lag.** Replicas are behind, with no bound during load or incidents. A user
  who writes then immediately reads may not see their own change.
- **Failover complexity.** With one replica, if it is down and the primary fails, reads go
  to the primary; if the primary fails, a replica is promoted — and with asynchronous
  replication, **writes acknowledged to clients may be lost**. Recovery scripts must
  replay what was missed.
- **Read-after-write.** Must be handled explicitly: read from the primary for a bounded
  window after a user's write, pin the session, or wait for the replica to reach the
  write's log position.

Anomalies, guarantees, and mechanisms: `data-systems-design/references/05-replication.md`.

---

## Step 5 — Cache

An in-memory store for expensive or frequently-accessed results. Caching exploits
**locality of reference**: recently requested data is likely to be requested again.

**Read-through cache** is the common pattern: check the cache; on a miss, read the database
and populate the cache.

**When to use a cache:** data is read frequently and modified infrequently.

**The considerations that are always underspecified:**
- **Expiration policy.** Too short and you hit the database constantly; too long and data
  goes stale. Also: expiring everything at the same moment creates a synchronized stampede
  — jitter the TTLs.
- **Consistency.** The cache and the store are not updated atomically. Across regions this
  gets harder. Decide what staleness is acceptable and state it.
- **Eviction policy.** LRU is the default; LFU and FIFO exist for specific access patterns.
- **Single point of failure.** A single cache node is a SPOF, and one sized exactly to the
  working set will thrash. Over-provision, and run multiple nodes across failure domains.
- **The cold-start problem.** When the cache is empty (deploy, restart, eviction storm), all
  traffic hits the origin at once. **Your origin must survive a cold cache**, or your cache
  is a load-bearing component with no redundancy. Request coalescing and soft TTLs mitigate.

---

## Step 6 — CDN

Geographically distributed servers caching static content — images, video, CSS, JS, and
increasingly dynamic content via edge computation.

**Gains:** latency (content served near the user) and origin offload.

**Considerations:** CDN traffic costs money, so cache only what is worth caching; set TTLs
carefully; have a **CDN fallback** so clients can reach the origin if the CDN fails; support
**invalidation** (API-based purge or versioned/fingerprinted URLs — versioned URLs are more
reliable and avoid the purge-propagation delay entirely).

---

## Step 7 — Stateless web tier

Move session state out of the web servers into a shared datastore.

**Why this is the pivotal step:** with state on individual servers, requests from one user
must always hit the same server (sticky sessions), which makes adding and removing servers
awkward and failure disruptive. With a stateless tier, **any request can go to any server**,
so autoscaling becomes trivial and a server dying is a non-event.

This is the step that makes every later step possible. It is worth doing early.

---

## Step 8 — Multiple data centers

Users are geo-routed to the nearest datacenter; on failure, traffic shifts to a healthy one.

**The hard problems:**
- **Traffic redirection:** GeoDNS or anycast.
- **Data synchronization:** different datacenters may have different data. The standard
  approach is replicating data across datacenters — with all the conflict and lag issues
  that implies (`data-systems-design/references/05-replication.md`).
- **Test and deployment:** you must test from different geographic locations and deploy
  consistently across all of them; a partial deploy across regions is its own outage class.

Do not take this step for latency alone until you have measured that the network is the
bottleneck. Take it when you need regional failure tolerance or have a compliance
requirement.

---

## Step 9 — Message queue

A durable buffer supporting asynchronous communication. Producers publish; consumers
subscribe and process.

**Gains:** decoupling (producer and consumer scale and fail independently), buffering
against downstream outages, and the ability to make slow work asynchronous — image
processing, notifications, fan-out, ML inference.

**The rule that keeps this correct:** delivery is **at-least-once**, so consumers must be
**idempotent**. Every queue also needs a bounded size, a defined overflow behavior, a
dead-letter path with an alert, and monitored consumer lag. See
`data-systems-design/references/11-stream-processing.md`.

---

## Step 10 — Logging, metrics, automation

Not optional past a certain size, and cheap to add early:

- **Logging:** centralized, structured, aggregated across servers. Per-server logs stop
  being usable the moment you have more than a handful of servers.
- **Metrics:** host-level (CPU, memory, disk I/O), aggregated (database performance, cache
  hit rate), and **business-level (DAU, retention, revenue)** — the last category is what
  actually tells you the system is working.
- **Automation:** CI, automated build/test/deploy, so that a team can ship reliably.

---

## Step 11 — Database scaling: sharding

The last and hardest step, taken when a single primary can no longer handle the write
volume or the data size.

**Sharding** splits large databases into smaller pieces that share a schema but hold
disjoint rows. The **sharding key** (partition key) determines routing, and choosing it is
the most consequential decision here: it must distribute evenly.

**The problems sharding introduces** — plan for all three:
- **Resharding:** needed when a shard fills up or distribution becomes uneven. Requires
  moving data and updating the routing map. Use consistent hashing to limit the movement
  (see `04-building-blocks.md`).
- **The celebrity/hotspot problem:** one key receiving disproportionate traffic makes one
  shard the bottleneck regardless of hashing. May require dedicated shards or key splitting.
- **Joins and denormalization:** cross-shard joins are impractical, so data gets
  denormalized — which creates consistency obligations you now own.

Also: **move non-relational data out of the relational database** (media to object storage,
search to a search index) to reduce the load before sharding, and consider whether the
answer is a different store rather than more shards.

Partitioning strategies, hot-key mitigation, rebalancing: `data-systems-design/references/06-partitioning.md`.

---

## Step 12 — Split tiers into services

Decompose the monolith into independently deployable services, each with its own datastore.

**Take this step for organizational reasons** (independent team deployment, ownership
boundaries) more than technical ones. Splitting a service adds network hops, partial
failure, distributed transactions, and multiplied availability math (four 99.9% services in
series ≈ 99.6%). Those are real, permanent costs.

**Do not split a system you do not yet understand.** Boundaries drawn before the domain is
clear become permanent and wrong, and a distributed monolith is strictly worse than a
monolith.

---

## Summary: scaling to millions

The consolidated list from Xu:

- Keep the web tier **stateless**
- Build **redundancy** at every tier
- **Cache** data as much as you can
- Support **multiple data centers**
- Host static assets in a **CDN**
- Scale the data tier by **sharding**
- Split tiers into **individual services**
- **Monitor** the system and use automation tools

---

## Using this ladder in a review

For an existing system, find its current rung and ask two questions:

1. **Has it skipped a rung?** Sharding before caching, or microservices before a stateless
   tier, is a strong signal of over-engineering and usually of an unresolved bottleneck
   elsewhere.
2. **Is it past due for the next rung?** Match the symptom to the step: replication lag →
   Step 4 handling; cache stampede → Step 5; deploy pain from sticky sessions → Step 7;
   write saturation → Step 11.

**Each rung is a cost as well as a capability.** The right question is never "are we
web-scale" but "which specific number, measured, justifies the next rung?"
