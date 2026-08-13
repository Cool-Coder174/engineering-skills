# The Scaling Ladder: From One Server to Millions of Users

**Sources:** *System Design Interview*, Chapter 1 (Xu). Reliability terms from *Grokking the
System Design Interview*.

Scaling is a repeated process. It has no end state. This reference gives the standard order
of steps. Each step includes **the signal that tells you to take it**.

**The signal matters more than the step.** If you take a step before its signal, the design
is too large. If you take it after the signal, you have an incident.

An architecture fits one level of load. It usually fails at 10 times that load. Expect to
change the architecture at each 10-fold increase.

---

## Step 0 — One server

One machine runs everything: the application, the database, and the cache.

The request flow has four parts:

1. The user asks DNS to resolve a domain name.
2. DNS returns an IP address.
3. The client sends an HTTP request to that address.
4. The server returns HTML or JSON.

**This is the correct architecture for most new systems.** It is simple. It is cheap. It is
easy to understand and to operate. Do not leave it without a reason.

**Signal to continue:** the server cannot handle the traffic, or you cannot accept its
failure.

---

## Step 1 — Separate the data tier

Move the database to its own machine. Then the application and the database scale
separately.

**Reason:** they use different resources. The application needs CPU. The database needs
memory and disk operations. They also scale differently. Application servers hold no state,
so you can add them easily. Databases hold state, so you cannot.

**Which database?** A relational database is the default. It remains correct for most
applications. Consider a non-relational database in four cases: you need very low latency,
your data is unstructured, you only serialize and deserialize records, or you store an
enormous volume. Read `05-database-selection.md`.

---

## Step 2 — Larger machine, or more machines

**Larger machine (scale up).** Simple. It needs no software change. It has three limits. It
has a hard ceiling. It becomes expensive at the top end. It gives **no redundancy**. When one
machine fails, the whole service stops.

**More machines (scale out).** More complex. It is the only way past the ceiling. It is the
only way to get redundancy.

**Rule:** use a larger machine first, because it is simpler. Move to more machines when you
reach the ceiling, or when you need redundancy. For a database, delay this step as long as
the numbers permit. Distributed state is where the complexity is.

---

## Step 3 — Load balancer

Put a load balancer in front of several application servers. Users reach the load balancer at
a public IP address. The servers use private addresses on an internal network.

**Two benefits:** failover when one server stops, and capacity from more servers.

**Selection algorithms** (from Grokking):

- **Round robin.** Send each request to the next server in the list. Best when the servers
  are identical and connections are short.
- **Weighted round robin.** Each server has an integer weight for its capacity. Use this
  algorithm for servers of different sizes.
- **Least connections.** Send to the server with the fewest open connections. Best for long
  connections that spread unevenly.
- **Least response time.** Send to the server with the fewest connections and the lowest
  average response time.
- **Least bandwidth.** Send to the server that carries the least traffic in Mbps.
- **IP hash.** Hash the client IP address to select a server. This gives a stable mapping. It
  costs even distribution.

**Health checks are necessary.** The load balancer must test each server. It must remove a
server that fails the test. Without health checks, you distributed the traffic but not the
resilience.

**The load balancer is itself a single point of failure.** Run two of them, and let each one
monitor the other. A managed service does this for you.

---

## Step 4 — Database replication

One primary node accepts writes. Replica nodes serve reads.

**Three benefits:**

- Read throughput grows with the replica count.
- Availability improves, because you can promote a replica.
- The data survives the loss of one site.

**Three costs. Design for all three.**

1. **Replication lag.** Replicas hold older data. The lag has no upper bound during heavy
   load or during an incident. A user who writes and then reads may not see the change.

2. **Failover is complex.** Consider one replica. If the replica stops, reads move to the
   primary. If the primary stops, you promote the replica. With asynchronous replication,
   **you can lose writes that the system already confirmed to clients.** Recovery scripts
   must replay the lost writes.

3. **Read-after-write needs an explicit mechanism.** Use one of these three:
   - Read from the primary for a fixed period after the user writes.
   - Send the whole session to one node.
   - Wait until the replica reaches the log position of that write.

For the anomalies and the mechanisms, read `data-systems-design/references/05-replication.md`.

---

## Step 5 — Cache

A cache is a memory store for results that are expensive or frequent. A cache works because
of locality of reference: a recent request is likely to repeat.

**Read-through** is the common pattern. Check the cache. On a miss, read the database and
write the result to the cache.

**Use a cache when** the data changes rarely and the system reads it often.

**Five items that designs usually omit:**

1. **Expiry.** A short expiry sends most reads to the database. A long expiry serves old
   data. Also, do not expire many items at the same moment. Add a random offset to each
   expiry time.

2. **Consistency.** The cache and the database do not update together. Across regions this is
   harder. Decide how old the data may be. Write that decision down.

3. **Eviction.** LRU is the default. LFU and FIFO exist for specific access patterns.

4. **Redundancy.** One cache node is a single point of failure. A cache sized exactly to the
   working set will evict constantly. Add capacity, and run several nodes in separate failure
   domains.

5. **The cold cache problem.** After a deploy, a restart, or heavy eviction, the cache is
   empty. All traffic reaches the database at once. **The database must survive an empty
   cache.** If it cannot, the cache is a necessary component with no redundancy, and that is
   a future outage. To reduce the risk, merge identical concurrent requests, and serve old
   data while you refresh it.

---

## Step 6 — CDN

A CDN is a set of servers in many locations. It caches static content. Some CDNs also run
code at the edge for dynamic content.

**Two benefits:** lower latency because content is near the user, and less load on the
origin.

**Four items to decide:** the cost, because CDN traffic is billed, so cache only what pays
for itself. The expiry time. A path to the origin for the case where the CDN fails. The
invalidation method.

**Prefer versioned URLs over a purge API.** A purge propagates slowly, and it can fail. A new
URL is correct at once.

---

## Step 7 — Remove state from the application tier

Move session state out of the application servers. Put it in a shared store.

**This step controls every later step.** With state on each server, every request from one
user must reach the same server. That constraint makes it hard to add or remove servers, and
it makes a server failure visible to users.

Without state, **any request can reach any server**. Autoscaling becomes simple. A server
failure becomes invisible.

Take this step early. It costs little, and later steps depend on it.

---

## Step 8 — More than one datacenter

Route each user to the nearest datacenter. On failure, route the traffic to a healthy one.

**Three hard problems:**

1. **Traffic routing.** Use GeoDNS or anycast.
2. **Data synchronization.** Datacenters hold different data. The standard method is
   replication between them. That method brings lag and conflicts. Read
   `data-systems-design/references/05-replication.md`.
3. **Test and deploy.** You must test from several locations. You must deploy the same
   version everywhere. A partial deploy across regions is its own outage.

Do not take this step for latency alone. First measure that the network is the bottleneck.
Take this step when you need to survive the loss of a region, or when a rule requires it.

---

## Step 9 — Message queue

A message queue is a durable buffer for asynchronous work. Producers write messages.
Consumers read them.

**Three benefits:**

- Producers and consumers scale separately. They also fail separately.
- The queue absorbs load when a consumer is down.
- The system can process slow work outside the request path. Examples of slow work are image
  processing, notifications, fan-out, and machine-learning inference.

**The rule that keeps a queue correct: delivery is at-least-once, so every consumer must be
idempotent.**

Every queue also needs these four items: a maximum size, a defined behavior when it is full,
a dead-letter destination with an alert, and a monitor on consumer lag. Read
`data-systems-design/references/11-stream-processing.md`.

---

## Step 10 — Logging, metrics, and automation

These items become necessary above a certain size. They are cheap to add early.

- **Logging.** Collect structured logs from all servers in one place. Per-server logs stop
  being useful above a few servers.
- **Metrics.** Collect three kinds. Host metrics: CPU, memory, disk. Aggregate metrics:
  database performance, cache hit rate. **Business metrics: daily active users, retention,
  revenue.** The last kind tells you that the system works.
- **Automation.** Continuous integration, plus automated build, test, and deploy.

---

## Step 11 — Partition the database

This is the last step and the hardest one. Take it when one primary cannot accept the write
volume, or cannot hold the data.

Partitioning divides one large database into smaller parts. Each part uses the same schema
and holds different rows.

**The partition key controls the routing. Selecting it is the most important decision in this
step.** The key must spread the load evenly.

**Partitioning creates three problems. Plan for all three.**

1. **Repartitioning.** You need it when a partition fills, or when the load becomes uneven.
   It moves data and updates the routing map. Consistent hashing reduces how much data moves.
   Read `04-building-blocks.md`.

2. **Hot partitions.** One key can receive far more traffic than the others. Then one
   partition limits the whole system, and hashing does not fix it. The fix may be a dedicated
   partition for that key, or a split of the key itself.

3. **Joins and duplicate data.** A join across partitions is impractical. So teams duplicate
   data. Duplicate data creates a consistency obligation that you now own.

**Two actions can delay this step.** Move non-relational data out of the database: media to
object storage, and search to a search index. Also ask whether a different database, rather
than more partitions, is the correct answer.

For partition strategies, hot-key fixes, and rebalancing, read
`data-systems-design/references/06-partitioning.md`.

---

## Step 12 — Divide the system into services

Divide the application into services that deploy separately. Each service owns its data.

**Take this step for organizational reasons, not technical ones.** The reasons are
independent deployment and clear ownership.

**The costs are real and permanent.** More network calls. Partial failures. Distributed
transactions. Lower total availability, because four services at 99.9% in series give about
99.6%.

**Do not divide a system that you do not yet understand.** Boundaries drawn too early become
permanent and wrong. A set of services that cannot deploy separately is worse than one
application.

---

## Summary

Xu's list for scaling to millions of users:

- Keep the application tier **stateless**
- Add **redundancy** at every tier
- **Cache** as much data as you can
- Support **more than one datacenter**
- Serve static files from a **CDN**
- **Partition** the database
- Divide the system into **services**
- **Monitor** the system and automate the operations

---

## How to use this ladder in a review

Find the current step of the system. Then ask two questions.

**Question 1. Did the team skip a step?** Partitioning before caching is a skipped step. So
is separate services before a stateless tier. A skipped step usually means the architecture
is too large. It often hides a different bottleneck.

**Question 2. Is the system late for the next step?** Match the symptom to the step.

| Symptom | Step |
|---|---|
| Replication lag causes wrong reads | Step 4 |
| The database stops when the cache empties | Step 5 |
| Deploys break user sessions | Step 7 |
| Writes saturate the primary | Step 11 |

**Every step is a cost as well as a capability.** Never ask whether the system is large
enough. Ask this question instead: which measured number requires the next step?
