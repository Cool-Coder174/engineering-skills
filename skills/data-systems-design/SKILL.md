---
name: data-systems-design
description: Design-decision engine for data-intensive and distributed systems, grounded in Designing Data-Intensive Applications (Kleppmann). Use when a task touches storage, schemas, migrations, replication, sharding, caching, transactions, queues, event streams, retries, background jobs, multi-service writes, or any non-functional requirement (latency, availability, throughput, durability, consistency). Provides decision tables, a named-hazard catalog, and deep-dive references per chapter.
---

# DATA SYSTEMS DESIGN ENGINE

**ROLE:** Principal Engineer for data-intensive systems.
**CORE FUNCTION:** Convert vague requirements ("make it fast", "keep it in sync", "don't lose data") into explicit, named, defensible design decisions — and reject designs that rely on guarantees the chosen technology does not provide.

This skill is a **knowledge and gate skill**. It does not run a workflow. It is consulted by
`planner`, `detail-planning`, `implement`, `verify`, and `code-review`, and can be invoked
directly for design questions.

**Companion skill: `system-design`.** The two divide the problem cleanly:

| | `system-design` | `data-systems-design` (this skill) |
|---|---|---|
| Question | *What should we build?* | *Will it stay correct?* |
| Covers | Requirements method, capacity estimation, component selection, scaling ladder, database selection, simplicity laws | Isolation, replication anomalies, partitioning, consensus, hazards |
| Output | Design document with trade-offs | Design Record with invariants and a hazard scan |

Run `system-design` first to decide the shape; run this skill to make sure the shape
survives concurrency and failure. Every deep dive in a design should end with this skill's
hazard scan.

---

# 1. ACTIVATION

Activate whenever the work involves any of:

| Trigger area | Examples |
|---|---|
| Persistence | new table, new collection, index, migration, schema change, ORM mapping |
| Scale & latency | "slow", "scale", "cache", "N+1", "timeout", "capacity", "p99" |
| Replication | read replicas, failover, multi-region, follower reads, standby |
| Partitioning | sharding, hot keys, consistent hashing, rebalancing, routing |
| Transactions | concurrency, race condition, double-charge, double-booking, counters |
| Distribution | RPC, service-to-service writes, distributed lock, leader election |
| Messaging | queues, Kafka, SQS, webhooks, retries, dead-letter, CDC, outbox |
| Correctness | idempotency, dedup, reconciliation, audit, "eventually consistent" |
| Interfaces | API versioning, protobuf/Avro/JSON payloads, rolling deploy compatibility |

If none of the above apply, this skill stays silent. **Do not apply distributed-systems
ceremony to a single-process CRUD change.** See Section 8 (Proportionality Rule).

---

# 2. THE THREE CONCERNS (Ch. 1, p. 3)

Every design decision must be justified against these three, and trade-offs between
them must be stated explicitly rather than assumed away.

| Concern | Definition | What "done" looks like |
|---|---|---|
| **Reliability** | Continues to work *correctly* (correct function, at desired performance) even when faults occur | A written fault model: which faults are tolerated, which are accepted, and the blast radius of each |
| **Scalability** | Reasonable ways to cope with growth in load | Load parameters named with numbers, and a stated answer to "what breaks at 10x?" |
| **Maintainability** | Operability, simplicity, evolvability | Change can be made and deployed without coordinated downtime or a rewrite |

**Fault vs. failure:** a *fault* is one component deviating from spec; a *failure* is the
system as a whole stopping service. The goal is fault-tolerance: prevent faults from
becoming failures. Faults you must consider: hardware faults, software errors
(correlated, systemic — e.g. the same bad input crashes every node), and human error
(the leading cause of outages).

## 2.1 Load and performance must be numeric

Never accept a performance requirement without **load parameters** and **percentiles**.

- **Load parameters:** requests/sec, reads:writes ratio, concurrent sessions, fan-out
  (e.g. followers per user), cache hit rate, rows per query, payload size. Pick the
  parameters that actually stress *this* system.
- **Response time is a distribution, not a number.** Use median (p50) plus p95/p99/p999.
  Averages hide the outliers, and the outliers are frequently the most valuable users
  (most data, most history).
- **Tail latency amplification:** when one user request fans out to N backend calls,
  a single slow backend slows the whole request. With enough fan-out, nearly every
  end-user request hits at least one slow call. Budget tail latency, not mean latency.
- **Percentiles do not average.** Averaging p95 across machines or time buckets is
  mathematically meaningless — aggregate histograms instead (t-digest, HdrHistogram).
- Measure response time **client-side**, including queueing delay. Server-side timing
  hides head-of-line blocking, which is often the real problem.

## 2.2 Load testing rule

Generate load **independently of response time**. If the load generator waits for the
previous request to finish before sending the next, it artificially shortens the queue
and produces a flattering, wrong result (coordinated omission).

Deep dive: `references/01-reliability-scalability-maintainability.md`

---

# 3. DECISION TABLES

Use these to make a decision quickly, then read the linked reference before committing
to anything expensive or hard to reverse.

## 3.1 Data model (Ch. 2, p. 27)

| If the data is… | Use | Because |
|---|---|---|
| One-to-many tree, loaded as a unit, no cross-entity queries | Document | Locality; schema flexibility; no joins needed |
| Many-to-many, or many-to-one references that need to stay consistent | Relational | Joins and FKs are cheap and correct; document stores emulate them badly in application code |
| Highly interconnected, arbitrary traversal depth | Graph | Recursive traversal is a first-class operation |
| Write-once, append-heavy, analytical scans over few columns | Column-oriented | Compression + scan efficiency (see Ch. 3) |

**Schema-on-read vs. schema-on-write:** document stores are not "schemaless"; the schema
is implicit and enforced by *every reader*. Prefer schema-on-read only when the structure
is genuinely heterogeneous or externally determined. When you choose it, you have
accepted the obligation to handle every historical shape in application code.

Deep dive: `references/02-data-models.md`

## 3.2 Storage engine (Ch. 3, p. 69)

| Workload | Engine | Trade-off |
|---|---|---|
| Write-heavy, high ingest | LSM-tree (log-structured) | Higher write throughput, lower write amplification; unpredictable read/compaction latency at high percentiles |
| Read-heavy, latency-sensitive, transactional | B-tree | Predictable reads, mature transactional support; writes go to disk at least twice (WAL + page) |
| Analytics over few columns of many rows | Column store | Huge compression + scan wins; poor for point writes |

**OLTP vs. OLAP is a real boundary.** Do not run analytical scans against the
transactional store because "it's the same data." Derive an analytical copy.

Deep dive: `references/03-storage-and-retrieval.md`

## 3.3 Encoding and schema evolution (Ch. 4, p. 111)

Every schema change must satisfy both directions during a rolling deploy:

- **Backward compatibility:** new code can read data written by old code.
- **Forward compatibility:** old code can read data written by new code.

Rolling upgrades and multi-version clients mean **both versions run simultaneously**.
The mandatory pattern for any breaking change is **expand → migrate → contract**:

1. **Expand:** add the new field/column/topic as optional; write to both; read from old.
2. **Migrate:** backfill; switch reads to new; verify.
3. **Contract:** stop writing old; only then remove it — in a later release.

Never combine expand and contract in one deploy. Never make a new field required
without a default.

Deep dive: `references/04-encoding-and-evolution.md`

## 3.4 Replication (Ch. 5, p. 151)

| Topology | Use when | Main hazard |
|---|---|---|
| Single-leader | Default. One region, standard OLTP | Replication lag anomalies; failover data loss |
| Multi-leader | Multi-datacenter writes, offline clients, collaborative editing | Write conflicts — you *must* define a resolution policy |
| Leaderless (quorum) | High write availability, tolerate node loss | No guarantee of read-your-writes without care; `w + r > n` is not linearizability |

**Read-after-write consistency is not automatic.** Under async replication, a user can
write and then read a stale replica. If your UI writes then immediately reads, you must
pick one: read from the leader for that user for a bounded window, pin the session to a
replica, or track a logical timestamp and wait for the replica to catch up.

The three lag guarantees you may need, each stronger than the last:
**read-your-own-writes** → **monotonic reads** (never go backwards in time) →
**consistent prefix reads** (causally ordered writes appear in order).

Deep dive: `references/05-replication.md`

## 3.5 Partitioning (Ch. 6, p. 199)

| Strategy | Range queries | Hot-spot risk |
|---|---|---|
| Key range | Efficient | High — sequential keys (timestamps, auto-increment IDs) all land on one partition |
| Hash of key | Lost (unless compound key) | Low, except for genuinely hot single keys (celebrity problem) |
| Compound key (hash prefix + range suffix) | Efficient within prefix | Low | 

**Secondary indexes force a choice:** local (by document — write to one partition, read
scatter/gather across all) vs. global (by term — write touches multiple partitions, read
hits one). Scatter/gather makes p99 latency the max of all partitions.

**Never partition by a monotonically increasing key** if writes are recent-biased.
For a genuinely hot key, prepend a random or bucketed prefix and accept the read fan-out.

Deep dive: `references/06-partitioning.md`

## 3.6 Transactions and isolation (Ch. 7, p. 221)

The isolation level must be **stated explicitly** for any multi-step read-then-write.
Defaults are usually weaker than developers assume.

| Anomaly | Prevented by | Notes |
|---|---|---|
| Dirty read | Read committed | Default in most databases |
| Dirty write | Read committed | Row-level locks |
| Read skew (non-repeatable read) | Snapshot isolation / repeatable read | MVCC |
| **Lost update** | Atomic write op, explicit lock (`SELECT … FOR UPDATE`), CAS, or automatic detection | Read-modify-write cycles are the classic bug |
| **Write skew** | **Serializable only** | Two transactions read the same set, then write *different* rows, invalidating each other's premise |
| **Phantoms** | Serializable, or materializing conflicts / index-range locks | An insert changes the result of a prior search |

**Write skew is the one to hunt for in reviews.** Signature: `SELECT` a condition, check
it in application code, then `INSERT`/`UPDATE` based on that check. Snapshot isolation
does **not** prevent it — not in PostgreSQL repeatable read, MySQL/InnoDB repeatable
read, Oracle "serializable", or SQL Server snapshot isolation. Examples: double-booking
a room, two doctors both going off-call, claiming a unique username, overdrawing a
balance across two accounts.

Deep dive: `references/07-transactions.md`

## 3.7 Distributed systems faults (Ch. 8, p. 273)

Assume as the default system model: **partially synchronous network, crash-recovery
nodes, unreliable clocks, no upper bound on message delay or process pause.**

- **You cannot distinguish** a slow node from a dead node, or a lost request from a lost
  response. Timeouts are a guess. Choose them deliberately and document the reasoning.
- **A node cannot trust its own judgment** about whether it is still the leader. It can
  be paused (GC, VM live migration, swap, SIGSTOP) for minutes past its lease expiry.
  Protect resources with **fencing tokens**: a monotonically increasing number issued
  with the lock, checked and enforced *by the resource*, rejecting any lower token.
  Client-side lock checks are not sufficient.
- **Do not use wall-clock time for ordering or correctness.** Time-of-day clocks jump
  (NTP steps, leap seconds) and drift. Use monotonic clocks for durations, and logical
  clocks/version vectors for ordering. Last-write-wins on timestamps silently drops data.
- Truth is defined by **quorum**, not by any single node's belief.

Deep dive: `references/08-distributed-systems-faults.md`

## 3.8 Consistency and consensus (Ch. 9, p. 321)

| You need | Then you need | Cost |
|---|---|---|
| Uniqueness constraint, single source of truth, leader election, "no split brain" | Linearizability → consensus (Raft/ZAB/Paxos) or a single leader | Unavailable during partition on the minority side (CAP); latency floor of a quorum round trip |
| Causal ordering only ("reply after post") | Causal consistency (version vectors, Lamport-ish ordering) | Available during partitions; much cheaper |
| Fire-and-forget analytics | Eventual consistency | Cheapest; reads may be arbitrarily stale |

Do not buy linearizability by accident (e.g. a distributed lock on a hot path) and do not
assume it where it does not exist (e.g. `w + r > n` quorums, or a cache in front of a
replica). Consensus is expensive and correct; ad-hoc coordination is cheap and wrong.

Deep dive: `references/09-consistency-and-consensus.md`

## 3.9 Batch and stream processing (Ch. 10 p. 389, Ch. 11 p. 439)

- Prefer **deterministic, replayable, immutable-input** jobs. Immutable input is what
  makes recovery, backfill, and debugging possible — you can re-run after a bug fix.
- **Do not dual-write.** Writing to the database *and* the search index/cache/queue from
  application code produces permanent divergence when one write fails or when two
  concurrent writes are applied in different orders. Use a **single ordered source of
  truth** and derive everything else via CDC or an outbox table in the same transaction.
- **Event time ≠ processing time.** Window on event time; define allowed lateness and an
  explicit policy for stragglers (drop, or emit a correction).
- "Exactly-once" is really **effectively-once**, achieved through idempotence plus
  atomic offset+output commit — never through "we won't retry."

Deep dive: `references/10-batch-processing.md`, `references/11-stream-processing.md`

## 3.10 Correctness and integrity (Ch. 12, p. 489)

- **End-to-end argument:** the application must have its own correctness check. A
  transaction in one hop does not make a multi-hop flow correct. Carry an **end-to-end
  operation identifier** (client-generated request ID) all the way through, and dedupe
  on it at the point of effect.
- **Timeliness vs. integrity.** Timeliness (being up to date) can be relaxed and
  repaired; **integrity** (no corruption, no lost or duplicated data) cannot. Spend the
  coordination budget on integrity; use apology/compensation for timeliness violations.
- **Trust, but verify.** Assume data gets corrupted eventually. Run auditing and
  reconciliation jobs continuously; don't wait for a user to report the discrepancy.

Deep dive: `references/12-correctness-and-integrity.md`

---

# 4. HAZARD GATE (MANDATORY)

Before any design is accepted, run the named-hazard catalog against it:
`references/hazard-catalog.md`.

Each hazard entry has a **detection signature** (what the code or design looks like),
the **consequence**, and the **required fix**. The catalog is the source of truth for the
`code-review` skill's data-hazard mode and for `verify`'s failure-mode matrix.

Report format:

```md
### Hazard Scan
| Hazard | Present | Evidence | Required Fix |
|---|---|---|---|
| H-07 Read-modify-write lost update | Yes | `wallet.py:88` reads balance, adds, writes | Atomic `UPDATE … SET balance = balance + ?` or `SELECT … FOR UPDATE` |
| H-13 Dual write | No | Single write path via outbox | — |
```

Only list hazards that are actually applicable. An honest "not applicable — this is a
single-process, single-writer script" is a valid and preferred result.

---

# 5. DESIGN RECORD OUTPUT

When invoked for a design decision, produce this record. It is short on purpose: every
line is a commitment that `verify` and `code-review` can later check.

```md
## Design Record: [Component]

### Requirements
- Load parameters: [writes/sec, reads/sec, ratio, fan-out, payload size, growth rate]
- Latency objective: p50 [x] ms, p99 [y] ms, measured [client-side / at the edge]
- Availability objective: [target, and what "down" means for this component]
- Durability objective: [RPO — acceptable data loss; RTO — acceptable recovery time]
- Consistency requirement: [linearizable / causal / read-your-writes / eventual] — because [user-visible reason]

### Fault Model
| Fault | Tolerated? | Behavior | Blast radius |
|---|---|---|---|
| Single node loss | Yes | Automatic failover, ≤Ns unavailability | One AZ |
| Network partition | Degraded | Minority side rejects writes | Region |
| Bad deploy / software bug | No | Requires rollback | Global |
| Operator error | Partial | Restore from PITR backup | Dataset |

### Decisions
| Decision | Choice | Rationale | Rejected alternative & why |
|---|---|---|---|
| Data model | [ ] | [ ] | [ ] |
| Storage engine | [ ] | [ ] | [ ] |
| Replication | [ ] | [ ] | [ ] |
| Partition key | [ ] | [ ] | [ ] |
| Isolation level | [ ] | [ ] | [ ] |
| Delivery semantics | [ ] | [ ] | [ ] |

### Invariants (must hold at all times)
- INV-1: [e.g. no two bookings overlap for the same room] — enforced by [mechanism]
- INV-2: [e.g. wallet balance never negative] — enforced by [mechanism]

> Every invariant needs a *mechanism*, not a hope. "The application checks it first"
> is not a mechanism under concurrency.

### Evolution Plan
- Schema change strategy: expand → migrate → contract
- Compatibility: backward [how] / forward [how]
- Rollback: [how to undo without data loss]

### What Breaks at 10x
[The specific parameter that saturates first, the symptom, and the next architecture]

### Hazard Scan
[Table from Section 4]
```

---

# 6. LANGUAGE DISCIPLINE (ANTI-HAND-WAVING)

These phrases are **forbidden** in a design without an accompanying mechanism:

| Forbidden | Must be replaced with |
|---|---|
| "eventually consistent" | Which anomalies are possible, what the convergence bound is, what the user sees meanwhile |
| "we'll retry" | Retry budget, backoff + jitter, idempotency key, terminal/dead-letter behavior |
| "it's atomic" | The exact scope of atomicity, and the isolation level |
| "exactly once" | Idempotence mechanism + dedup key + where the dedup state lives |
| "we'll keep them in sync" | Single source of truth + derivation mechanism (CDC/outbox), or the divergence you accept |
| "add a cache" | Invalidation strategy, staleness bound, stampede protection, and what happens on a cold cache |
| "use a distributed lock" | Fencing token enforced at the resource, plus what happens when the lease expires mid-operation |
| "scale horizontally" | Partition key, rebalancing plan, and what state prevents it |
| "the timestamp orders them" | Monotonic clock or logical clock — wall clocks do not order events across nodes |

---

# 7. REFERENCE INDEX

| Reference | DDIA source | Read when |
|---|---|---|
| `references/01-reliability-scalability-maintainability.md` | Ch. 1, p. 3 | Setting NFRs, SLOs, fault models, capacity |
| `references/02-data-models.md` | Ch. 2, p. 27 | Choosing relational/document/graph; query languages |
| `references/03-storage-and-retrieval.md` | Ch. 3, p. 69 | Indexes, LSM vs. B-tree, OLTP vs. OLAP |
| `references/04-encoding-and-evolution.md` | Ch. 4, p. 111 | Schema/API changes, migrations, serialization |
| `references/05-replication.md` | Ch. 5, p. 151 | Replicas, failover, lag, conflicts |
| `references/06-partitioning.md` | Ch. 6, p. 199 | Sharding, hot keys, secondary indexes, rebalancing |
| `references/07-transactions.md` | Ch. 7, p. 221 | Concurrency bugs, isolation, locking |
| `references/08-distributed-systems-faults.md` | Ch. 8, p. 273 | Timeouts, clocks, pauses, leases, fencing |
| `references/09-consistency-and-consensus.md` | Ch. 9, p. 321 | Linearizability, ordering, 2PC, consensus |
| `references/10-batch-processing.md` | Ch. 10, p. 389 | Jobs, backfills, derived datasets |
| `references/11-stream-processing.md` | Ch. 11, p. 439 | Queues, CDC, event sourcing, windows |
| `references/12-correctness-and-integrity.md` | Ch. 12, p. 489 | End-to-end correctness, auditing, privacy |
| `references/hazard-catalog.md` | Cross-cutting | Every review and verification |

**Adjacent references in `system-design`**, for the questions this skill deliberately does
not answer:

| Reference | Read when |
|---|---|
| `system-design/references/02-estimation.md` | You need the actual numbers — QPS, storage, latency constants, availability math |
| `system-design/references/03-scaling-ladder.md` | Deciding what infrastructure to add next, and what triggers it |
| `system-design/references/04-building-blocks.md` | Choosing a load balancer, cache strategy, queue type, rate limiter, or ID scheme |
| `system-design/references/05-database-selection.md` | Choosing or defending a datastore; B-tree vs. LSM; the RUM conjecture |
| `system-design/references/06-simplicity-and-design-laws.md` | Any design that feels large — the subtraction pass |

---

# 8. PROPORTIONALITY RULE (ANTI-OVER-ENGINEERING)

Rigor must scale with blast radius. Applying the full Design Record to a copy change is
itself a failure mode.

| Change class | Required output |
|---|---|
| Cosmetic / single-file / no persistence, no concurrency | Nothing from this skill |
| New endpoint or job over existing schema | Hazard scan only |
| New table, index, migration, queue, or cache | Hazard scan + Decisions + Invariants |
| New service, replication/partitioning change, or new source of truth | Full Design Record |
| Anything with money, auth, PII, or irreversible side effects | Full Design Record + explicit integrity section |

**Simplicity is a first-class goal.** Accidental complexity — complexity not inherent in
the problem — is the main reason systems become unmaintainable. If a single-node
solution meets the requirements, that *is* the correct answer, and the Design Record
should say so.

---

**Attribution:** concepts, terminology, and page references throughout this skill and its
references are drawn from *Designing Data-Intensive Applications* by Martin Kleppmann
(O'Reilly, 2017). Page numbers refer to that edition.
