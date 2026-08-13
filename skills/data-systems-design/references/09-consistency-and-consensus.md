# Consistency and Consensus

**Source:** DDIA Ch. 9, p. 321.

The question this chapter answers: **what is the weakest guarantee that still makes your
application correct?** Buying more than you need is expensive; assuming more than you have
is a corruption bug.

---

## 1. The consistency ladder

| Model | Guarantee | Cost | Available during partition? |
|---|---|---|---|
| **Eventual consistency** | Replicas converge if writes stop | Cheapest | Yes |
| **Read-your-writes / monotonic reads / consistent prefix** | Session-level guarantees (see `05-replication.md`) | Low | Yes |
| **Causal consistency** | Causally related operations are seen in order | Moderate; version vectors / Lamport timestamps | **Yes** |
| **Linearizability** | Behaves as if there were one single copy, and all operations are atomic at a point in time | Consensus round trip; unavailable on the minority side of a partition | **No** |

**Key result: causal consistency is the strongest model that remains available during a
network partition.** If causal ordering is enough for your use case, you can have both
correctness and availability. Reach for linearizability only when you actually need it.

---

## 2. Linearizability

**Definition:** the system behaves as though there is only one copy of the data and all
operations on it are atomic. Once a read returns a new value, all subsequent reads (by any
client) must return the new value or newer. It is a **recency guarantee**.

Not the same as serializability: serializability is about *transactions* behaving as some
serial order; linearizability is about *single-object* recency. (Strict serializability =
both.)

### 2.1 When you genuinely need it
- **Locking and leader election.** Preventing split brain requires linearizable
  coordination — this is exactly what ZooKeeper/etcd provide.
- **Uniqueness constraints:** usernames, email addresses, seat/room booking, bank balance
  ≥ 0, inventory not oversold. All are essentially a lock on a value.
- **Cross-channel timing dependencies:** the classic bug — a service writes a file to
  object storage, publishes a message to a queue, and the consumer reads the file before
  the write is visible. Two communication channels racing each other. Either make the
  storage read linearizable, or carry the data in the message, or have the consumer retry
  until the object appears.

### 2.2 What is and isn't linearizable
- Single-leader replication *can* be, **if you always read from the leader** — but a node
  that believes it is leader may not be (see fencing, `08-...faults.md`).
- **Asynchronous replicas are not.** Reading from a follower breaks it.
- **Quorums (`w + r > n`) are not**, in general — see `05-replication.md` §4.1. Making
  Dynamo-style quorums linearizable requires read repair synchronously on read and reading
  the latest state before writes, and it costs performance.
- **Multi-leader and leaderless are not.**
- Consensus algorithms (Raft, ZAB, Paxos, VSR) **are** — this is what they are for.

### 2.3 The cost — CAP, stated correctly
CAP is best read as: **if the network is partitioned, you must choose between consistency
(linearizability) and availability.** It says nothing when there is no partition, it
ignores latency, and "pick two of three" is a misleading formulation.

More importantly: **linearizability is slow, partition or no partition.** Every
linearizable operation costs at least a quorum round trip, and response time is
proportional to network delay uncertainty. In a multi-region deployment this is tens to
hundreds of milliseconds, on every operation.

---

## 3. Ordering and causality

**Causality imposes a partial order.** Some operations are ordered (cause before effect);
concurrent operations are incomparable. Linearizability imposes a **total** order — which
is stronger than causality requires.

- **Lamport timestamps** (counter, node ID) give a total order **consistent with
  causality** — cheap, and enough for many purposes.
  - But a total order of timestamps is **not enough to implement a uniqueness constraint**:
    to reject a duplicate username you must know, *at the moment of the request*, that no
    other node is concurrently taking it. Knowing the eventual total order later is too
    late. You need **total order broadcast**.
- **Version vectors** capture causality precisely and detect true concurrency.

### 3.1 Total order broadcast (atomic broadcast)
Messages delivered to all nodes, **reliably** and in the **same total order**, with the
order fixed at delivery time. This is what a replication log is, and it is equivalent to
consensus.

Uses: state machine replication, serializable transactions, fencing tokens (the sequence
number is monotonically increasing by construction), and implementing linearizable storage
(append your intended operation to the log, then read the log to see whether yours or
someone else's came first — the "compare-and-set via log" pattern).

---

## 4. Distributed transactions and consensus

### 4.1 Two-phase commit (2PC) — know its failure mode before choosing it
A coordinator sends prepare; participants that reply "yes" have made an **irrevocable
promise** to commit; the coordinator then makes an irrevocable decision and writes it to
its own log; participants must apply it.

**The problem: if the coordinator crashes after participants voted yes but before it
distributes the decision, participants are stuck — "in doubt".** They hold their locks
and cannot unilaterally decide, blocking any transaction touching those rows, potentially
indefinitely (until an operator resolves it manually). Some implementations offer
heuristic decisions, which **violate atomicity by design** and can corrupt data.

**XA / distributed transactions in practice** additionally: cannot deduplicate across
systems, cannot detect deadlocks across participants, do not work with SSI, and amplify
failures (the whole system becomes as available as its least-available participant).

**Recommendation:** avoid 2PC/XA for application-level cross-service consistency. Prefer
**one transactional source of truth + derived data** (outbox/CDC), or an explicit
**saga** with compensating actions and idempotent steps, and accept the temporary
inconsistency window in writing.

### 4.2 Fault-tolerant consensus
One or more nodes propose values; the algorithm decides on one. Properties: uniform
agreement, integrity (no node decides twice), validity (only proposed values), and
**termination** (a decision is eventually reached, requiring a quorum — consensus cannot
make progress if fewer than a majority of nodes are up).

Real algorithms (Raft, ZAB, Paxos/Multi-Paxos, VSR) all work by **electing a leader per
epoch** and requiring a **quorum vote both to elect the leader and on each decision**.
The two quorums must overlap, which is how a stale leader is detected. Consensus is
essentially total order broadcast, which is essentially repeated rounds of agreement.

**Limitations to state honestly:** the voting is a synchronous replication cost; consensus
requires a **strict majority** to operate; the membership set is usually static (dynamic
membership is possible but less mature); and consensus algorithms are **sensitive to
network timing** — a flaky network causes frequent leader elections and can bring
throughput to nearly zero (a "frequently re-electing" cluster is a real and confusing
outage mode).

### 4.3 Membership and coordination services (ZooKeeper / etcd)
They hold small amounts of data that fits in memory (not your application data) and provide
a bundle of features that are hard to build correctly:

- **Linearizable atomic compare-and-set** → distributed lock/lease (with an auto-expiring
  lease and, importantly, a **fencing token** in the form of `zxid`/`cversion`).
- **Total ordering of operations** → fencing tokens.
- **Failure detection** via sessions and heartbeats, with ephemeral nodes auto-removed.
- **Change notifications** (watches) so clients learn about membership changes without polling.

Use them for **coordination and configuration**, not for data. Typical uses: leader
election, partition assignment, service discovery, and cluster membership.

---

## 5. Decision procedure

1. Write down the **invariant** you must uphold (e.g. "no two bookings overlap",
   "balance never negative", "one leader at a time").
2. Ask: does it require knowing about **concurrent** operations at decision time?
   - **Yes** → you need linearizability (a single leader, a unique index in a single-master
     database, or a consensus service). Accept the availability and latency cost, and
     confine it to the smallest possible scope.
   - **No, only ordering of related events matters** → causal consistency suffices; use
     version vectors / a partitioned ordered log, and stay available during partitions.
   - **No, staleness is fine** → eventual consistency; document the anomalies.
3. Prefer to **push the constraint into a single partition** so a local transaction can
   enforce it, rather than coordinating across partitions. Partition by the constraint's
   key (e.g. partition bookings by room ID) and you often turn a distributed problem into
   a single-node one.
4. If you cannot avoid multi-service consistency: outbox/CDC first, saga with compensation
   second, 2PC last and reluctantly.

---

## 6. Review checklist

- [ ] Every invariant classified: needs linearizability / causal / eventual
- [ ] Linearizable operations confined to the smallest possible scope
- [ ] Uniqueness constraints enforced by a single-partition unique index or consensus store,
      never by an application-level `SELECT`-then-`INSERT`
- [ ] No assumption that quorum reads/writes are linearizable
- [ ] No reads from an async replica in a path requiring recency
- [ ] Cross-channel races (write to store + publish message) identified and handled
- [ ] Leader election uses a consensus service; fencing tokens are enforced at the resource
- [ ] Distributed locks have lease expiry and fencing, and a defined mid-operation-expiry behavior
- [ ] 2PC/XA avoided, or its in-doubt/coordinator-failure behavior explicitly handled
- [ ] Sagas have idempotent steps and defined compensating actions for every step
- [ ] Consensus cluster sizing gives a strict majority; behavior when quorum is lost is defined
- [ ] Coordination service holds coordination data only, not application data
- [ ] Ordering across nodes uses logical clocks, not wall-clock timestamps
