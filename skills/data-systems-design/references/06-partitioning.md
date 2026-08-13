# Partitioning (Sharding)

**Source:** DDIA Ch. 6, p. 199.

Partitioning exists for scalability: each partition is its own small database, so query
load and data volume distribute across nodes. Partitioning is normally combined with
replication (each partition has replicas on different nodes).

**The goal is even distribution.** A partitioning scheme that produces **skew** puts
disproportionate load on one node — a **hot spot** — and the whole cluster runs at the
speed of that one node, which means you paid for N machines and got one.

---

## 1. Partitioning key-value data

### 1.1 By key range
Assign a continuous range of keys to each partition (like volumes of an encyclopedia).
Boundaries must adapt to the data distribution, so they are chosen manually or by the
system (auto-splitting).

- **Advantage:** range scans are efficient and in sort order.
- **Hazard:** **sequential keys create hot spots.** A timestamp key means all of today's
  writes go to one partition while the others idle. An auto-increment ID has the same
  problem.
- **Fix:** prefix the key with something that distributes — sensor name, tenant ID,
  region — so the time range becomes the *second* component (`(sensor_id, timestamp)`).

### 1.2 By hash of key
Apply a hash function (one with good distribution; note that some languages' built-in
hashes are not stable across processes — Java's `Object.hashCode`, Ruby's `Object#hash` —
so use an explicit stable hash like MD5/MurmurHash for partitioning) and assign hash ranges
to partitions.

- **Advantage:** distributes uniformly; hot spots become rare.
- **Cost:** **range queries are lost** — adjacent keys land on different partitions, so a
  range scan must hit every partition.
- **Compound key compromise:** hash the *first* column to choose the partition, and store
  the remaining columns as a sort key within it (Cassandra). `(user_id, timestamp)` gives
  you even distribution across users and efficient time-range scans for one user. This is
  usually the right shape for event/timeline data.

### 1.3 Skewed workloads and relieving hot spots
Even hashing does not fix a genuinely hot **single key** — the celebrity problem, where
one user ID receives an enormous share of traffic. All requests for that key go to the
same partition regardless of the hash.

Mitigation is application-level: add a random or bucketed prefix (e.g. two decimal digits)
to split the key across 100 partitions. The cost is that **reads must now fan out to all
100 buckets and combine**, plus bookkeeping about which keys are split. Apply it only to
the small number of keys that need it.

---

## 2. Partitioning and secondary indexes

Secondary indexes do not map neatly to partitions, and the choice between the two
approaches has direct latency consequences.

### 2.1 Local index (partitioning by document)
Each partition maintains an index over only its own documents.

- **Write:** touches only one partition. Simple, fast.
- **Read:** must query **every** partition and combine — **scatter/gather**.
- **Consequence:** read latency is the **maximum** over all partitions, so tail latency
  amplification is severe and gets worse as you add partitions. Widely used anyway
  (MongoDB, Cassandra, Elasticsearch, Riak) because writes stay simple.

### 2.2 Global index (partitioning by term)
A global index covering all partitions, itself partitioned by the indexed term (or by hash
of the term).

- **Read:** hits a single partition. Fast, no scatter/gather.
- **Write:** a single document write touches **multiple** partitions of the index, so
  writes are slower and — absent a distributed transaction — the index is usually updated
  **asynchronously**, meaning it can be stale.
- **Consequence:** if you choose this, state the staleness window and make sure the
  application does not read-after-write from the global index.

---

## 3. Rebalancing partitions

Rebalancing = moving load between nodes when nodes are added or removed, or data grows.

**Requirements:** load ends up fairly shared; the database keeps serving during the move;
only the minimum necessary data moves.

### 3.1 Strategies

- **`hash mod N` — never do this.** Changing `N` reshuffles almost every key. It is the
  canonical mistake.
- **Fixed number of partitions:** create many more partitions than nodes (e.g. 1,000
  partitions for 10 nodes) and assign several to each node. Adding a node steals a few
  partitions from each existing node; only whole partitions move, and the key→partition
  mapping never changes. Choosing the number up front is the trade-off: too few limits
  future growth; too many adds management overhead.
- **Dynamic partitioning:** split a partition when it exceeds a size threshold, merge when
  it shrinks. Adapts to data volume. Cold-start caveat: an empty database starts with one
  partition (all load on one node) unless you **pre-split**.
- **Partitioning proportionally to nodes:** fixed number of partitions per node; partition
  size grows with the dataset while node count is constant, and stabilizes as nodes are
  added.

### 3.2 Automatic vs. manual
Fully automatic rebalancing is convenient and dangerous: rebalancing is expensive
(re-routing, moving large amounts of data), and if it triggers off a false failure
detection during a load spike, it **adds load to an already overloaded system** and can
cascade. A human in the loop is slower and much safer. Whichever you choose, record it.

---

## 4. Request routing (service discovery)

Three approaches to "which node do I talk to for key K?":

1. Clients contact any node; that node forwards or replies.
2. A routing tier (partition-aware load balancer) sits in front.
3. Clients are partition-aware and connect directly.

The hard part in all three is **agreeing on the current assignment** while it changes.
Typically a coordination service (ZooKeeper/etcd) holds the authoritative mapping and
subscribers are notified of changes; some systems use a gossip protocol instead. Stale
routing information is a real failure mode — the client must handle "not my partition"
responses by refreshing the map, not by failing.

---

## 5. Parallel query execution

Analytical (MPP) query engines break a complex query into stages executed in parallel
across partitions. The practical implication for application design: a query that must
touch every partition has a latency floor set by the slowest partition, and it consumes
capacity across the entire cluster. Design access patterns so hot-path queries hit **one**
partition.

---

## 6. Review checklist

- [ ] Partition key stated explicitly, with the reason it distributes evenly
- [ ] No partitioning on a monotonically increasing value (timestamp, auto-increment ID)
      without a distributing prefix
- [ ] Known hot keys identified; mitigation stated (bucketed prefix) or accepted with reason
- [ ] Range-query requirements checked against a hash-partitioned key
- [ ] Hash function is stable across processes and versions
- [ ] Secondary index strategy chosen (local vs. global) with its latency/staleness cost stated
- [ ] Scatter/gather queries are off the hot path, or their p99 impact is accepted
- [ ] Rebalancing strategy defined; `hash mod N` is not used
- [ ] Rebalancing is manual or has guards against triggering during load spikes
- [ ] Routing/assignment changes handled by clients (refresh on "wrong partition")
- [ ] Cross-partition transactions identified — most partitioned stores do not support them
- [ ] Pre-splitting planned if starting from an empty dataset with dynamic partitioning
