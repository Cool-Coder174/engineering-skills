# Replication

**Source:** DDIA Ch. 5, p. 151.

Replication exists for three reasons: keeping data geographically close to users
(latency), tolerating machine failure (availability), and scaling out reads (throughput).
All the difficulty comes from handling **changes** to replicated data.

---

## 1. Single-leader replication

One replica is the leader; all writes go to it. Followers apply the leader's replication
log in the same order. Reads may go to any replica.

### 1.1 Synchronous vs. asynchronous

- **Synchronous:** the leader waits for the follower to confirm before reporting success.
  The follower is guaranteed up to date, but **if the follower does not respond, writes
  block**. Making all followers synchronous is impractical: any one node's outage halts
  the system.
- **Semi-synchronous:** one follower synchronous, the rest asynchronous. If the
  synchronous follower fails, another is promoted to synchronous. Guarantees an up-to-date
  copy on at least two nodes. **This is usually the right default when durability matters.**
- **Asynchronous:** the leader does not wait. Fast and resilient to slow followers, but
  **writes acknowledged to the client can be lost if the leader fails** — durability is
  not what the client was told.

**Decision to record:** whether an acknowledged write may be lost on leader failure. If
the answer is "no", asynchronous replication alone is not acceptable and you must say
what makes it safe (sync follower, quorum commit, or write-ahead archive).

### 1.2 Setting up new followers
Snapshot the leader (without downtime), copy it to the new node, have the follower request
all changes since the snapshot's **log position** (log sequence number / binlog
coordinates), then catch up. The log position is the essential piece — without it, a
consistent handoff from snapshot to stream is impossible.

### 1.3 Handling outages

**Follower failure — catch-up recovery.** The follower knows its last processed
transaction; on restart it requests changes from there.

**Leader failure — failover.** Determine that the leader failed (usually a timeout —
there is no reliable way to distinguish "dead" from "slow"), choose a new leader (the one
with the most recent changes; ideally by consensus), and reconfigure clients and the old
leader. Failover is dangerous and full of sharp edges:

- **Lost writes:** with async replication, the new leader may not have all the old
  leader's writes. Discarding them violates clients' durability expectations — and is
  especially destructive if other storage (a cache, a search index) already reflects them.
- **Split brain:** two nodes both believe they are leader, both accept writes, and there is
  no conflict-resolution mechanism → data is lost or corrupted. Mechanisms that shut one
  node down must not be able to shut *both* down.
- **Timeout tuning:** too long a timeout means a long outage; too short means unnecessary
  failovers, which are themselves harmful — especially under load spikes, when failing
  over makes the load problem worse.
- **Auto-increment ID collisions** between old and new leader are a classic corruption path.

**For this reason, some teams deliberately perform failover manually.** That is a
legitimate, documentable choice.

### 1.4 Replication log implementations
- **Statement-based:** replays SQL. Breaks on `NOW()`, `RAND()`, auto-increment, and
  side-effecting triggers. Avoid.
- **Write-ahead log (WAL) shipping:** byte-level; ties replicas to the same storage
  engine version, which prevents zero-downtime version upgrades.
- **Logical (row-based) log:** decoupled from the storage engine, so it enables both
  version-independent replicas and **external consumers — this is what CDC is built on**
  (see `11-stream-processing.md`).
- **Trigger-based:** flexible, application-controlled, but higher overhead and more
  bug-prone.

---

## 2. Replication lag and the guarantees you may need

Asynchronous followers are **eventually consistent**, with no bound on the lag. Under load
or failure, seconds become minutes. Three named anomalies, each with a named fix.

### 2.1 Read-after-write (read-your-own-writes)
The user writes, then immediately reads a stale replica, and their own change appears to
have vanished. Fixes:

- Read anything the user may have modified **from the leader** (e.g. own profile).
- Track the **time of the last update**; read from the leader for the next N seconds, or
  route away from replicas whose lag exceeds a threshold.
- Client remembers the **logical timestamp** (log position) of its write and requires the
  replica to have caught up to it; if not, wait or fail over to another replica.
- Cross-device complications: the "last update timestamp" must be centralized, and the
  same user's devices may be routed to different datacenters.

### 2.2 Monotonic reads
The user makes successive reads, hits a lagging replica the second time, and **sees time
move backwards** (a comment they just saw disappears). Fix: **each user always reads from
the same replica**, chosen by a hash of the user ID (with rerouting when that replica fails).

### 2.3 Consistent prefix reads
An observer sees an answer before the question because the writes were partitioned and
applied in different orders. Fix: ensure causally-related writes go to the same partition,
or track causal dependencies explicitly.

### 2.4 The engineering rule
**If your application needs one of these guarantees, it must be provided by a mechanism.**
"The lag is usually small" is not a mechanism, and it will not be true during an incident.
And "eventual consistency" is fine only when you have written down which of these anomalies
the user may experience, and confirmed it is acceptable.

---

## 3. Multi-leader replication

Multiple nodes accept writes and replicate to each other. Reasonable use cases:

- **Multi-datacenter operation:** each DC has a leader; local write latency, DC-outage
  tolerance, tolerance of inter-DC network problems.
- **Offline-capable clients:** every device is effectively a leader with an
  indefinite replication lag (calendar apps, note apps).
- **Real-time collaborative editing.**

The cost is **write conflicts**, which you must design for explicitly.

### 3.1 Handling write conflicts
- **Avoid them.** The best strategy: route all writes for a particular record to the same
  leader (e.g. by user ID). Conflicts disappear if the same record is only ever written in
  one place. Breaks down when a leader must change (failover, user relocation).
- **Converge:** all replicas must reach the same final value.
  - **Last write wins (LWW)** by timestamp or ID — simple, **and lossy**. It silently
    discards writes and depends on clocks you cannot trust (see `08-distributed-systems-faults.md`).
    Acceptable only when data loss is genuinely acceptable.
  - Merge values (e.g. concatenate, union).
  - Record the conflict and resolve it later — in application code or by asking the user.
- **Custom resolution logic**, on write (as soon as a conflict is detected) or on read
  (all versions returned; application or user resolves; e.g. shopping cart union).
- **CRDTs / mergeable data structures / operational transformation** for automatic,
  sensible convergence.

**Conflict detection is subtler than it looks:** with asynchronous replication, a
uniqueness constraint enforced independently on two leaders will accept both writes and
detect the conflict only later — often too late to be fixed automatically.

### 3.2 Topologies
Circular, star, and all-to-all. All-to-all avoids single points of failure but suffers
from **messages arriving out of causal order** (an update arriving before the insert it
depends on). Version vectors are the correct fix; relying on wall-clock timestamps is not.

---

## 4. Leaderless replication (Dynamo-style quorums)

The client (or a coordinator) writes to several replicas in parallel and reads from
several in parallel. Version numbers determine which value is newer.

- **Quorum condition:** with `n` replicas, `w` write acks and `r` read responses,
  `w + r > n` guarantees the read set overlaps a node with the latest write.
  Common: `n = 3, w = r = 2`.
- **Read repair** (client writes back stale values it observes) and **anti-entropy**
  background processes bring stale replicas up to date. Note: without anti-entropy, a
  value that is rarely read may stay stale on some replica indefinitely, effectively
  reducing durability.

### 4.1 What quorums do NOT give you
`w + r > n` is **not** linearizability. Documented edge cases where stale reads occur even
with a formally satisfied quorum:

- A **sloppy quorum** was used (writes accepted by nodes outside the designated home
  nodes during a partition), so the read quorum and the write quorum may not overlap at all.
- Two writes occur concurrently — it is undefined which one "happened first", and merging
  by timestamp loses data.
- A write happens concurrently with a read — the write may be reflected on only some replicas.
- A write succeeded on fewer than `w` replicas and was **not rolled back** on the ones
  where it did succeed.
- A node with new data fails and is restored from a replica with old data, dropping the
  write below `w`.
- Unlucky timing.

**Conclusion:** if you need read-your-writes, monotonic reads, or a uniqueness constraint,
do not assume a quorum store gives it to you. It does not.

### 4.2 Sloppy quorums and hinted handoff
During a network interruption, writes go to *any* reachable nodes, not the home nodes, and
are later handed off. This increases **write availability** at the cost of **read
guarantees** — a read from the home nodes may not see a recent write until handoff
completes. Whether this is on by default varies by database: check, and record which.

### 4.3 Detecting concurrent writes
- **Version numbers per key:** the server assigns a version; the client must read before
  writing and pass the version back; concurrent writes produce **siblings** that the
  application must merge.
- **Merging siblings:** union is often right (shopping cart), but deletions require
  **tombstones** — you cannot represent "removed" by simply omitting the item, because a
  concurrent sibling would resurrect it.
- **Version vectors** (a version number per replica per key) are the correct structure for
  multi-replica concurrency detection. A single version number is insufficient.
- "Concurrent" means **neither happened-before the other** — not "at the same wall-clock
  time". Wall-clock time cannot establish this relationship.

---

## 5. Review checklist

- [ ] Replication topology chosen and written down (single-leader / multi-leader / leaderless)
- [ ] Whether an acknowledged write may be lost on failover is explicitly answered
- [ ] Failover procedure defined: automatic or manual, timeout value, split-brain prevention
- [ ] Fencing/leader-election mechanism prevents two active leaders (see `08-distributed-systems-faults.md`)
- [ ] Replication lag is monitored and alerted on, with a defined threshold
- [ ] Read-after-write handled for every path where a user reads what they just wrote
- [ ] Monotonic reads handled where the UI polls or paginates
- [ ] Causally-dependent writes are co-located or causally tracked
- [ ] Multi-leader: conflict resolution policy is explicit and is not silent LWW on wall clocks
- [ ] Leaderless: `n`, `w`, `r` documented; sloppy-quorum behavior known; anti-entropy running
- [ ] Concurrent writes detected with version vectors, not timestamps
- [ ] Deletions in mergeable data use tombstones
- [ ] Replicas are not assumed to be linearizable anywhere in application logic
