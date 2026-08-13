# Stream Processing

**Source:** DDIA Ch. 11, p. 439.

Unbounded input, incrementally processed. The chapter's most valuable contributions to
everyday application design are the **dual-write problem**, the **outbox/CDC pattern**, and
the correct definition of "exactly once".

---

## 1. Transmitting event streams

An **event** is a small, self-contained, **immutable** record of something that happened,
with a timestamp. Producers write once; consumers may be many.

### 1.1 Messaging systems — the two questions
1. **What if producers outpace consumers?** Options: drop messages, buffer in a queue
   (bounded or unbounded — an unbounded queue eventually exhausts memory or disk and the
   behavior at that limit is often untested), or apply **backpressure** (block the
   producer). Choose explicitly. "It'll be fine" is how outages start.
2. **What if a node crashes?** Durability costs writing to disk and/or replication. If
   losing messages is acceptable, say so; you get much higher throughput.

### 1.2 Two families

**Message brokers with per-message acknowledgment (JMS/AMQP style — RabbitMQ, SQS):**
messages are assigned to consumers, deleted on ack, and redelivered on failure. Good for
task queues where processing order doesn't matter and per-message latency does.
- **Load balancing** across consumers **destroys ordering**: two consumers processing
  messages for the same entity can apply them out of order. Redelivery after a crash also
  reorders. If order matters for an entity, do not fan those messages across consumers.

**Partitioned logs (Kafka, Kinesis, Pulsar):** an append-only log, partitioned; consumers
read sequentially and track an **offset**. Messages are **not deleted on consumption** —
they are retained for a period.
- **Order is preserved within a partition** — which is why partitioning by entity key
  (user ID, account ID, order ID) is the standard technique for getting per-entity ordering.
- Parallelism is bounded by the number of partitions, and a slow message blocks its whole
  partition (head-of-line blocking).
- **Replay is possible**: reset the offset and reprocess. This is the log's superpower —
  it makes streams as re-runnable as batch jobs, which makes experimentation and recovery
  from bugs cheap.
- **Consumer lag** is the key metric: monitor it, alert on it. Growing lag is the leading
  indicator of every stream outage.
- If a consumer falls so far behind that messages it needs have been deleted by retention,
  it has **silently lost data**. Size retention against your worst realistic outage, and
  alert before the boundary.

---

## 2. Databases and streams

### 2.1 The dual-write problem — read this twice
Writing the same data from application code to two systems (database **and** search index,
or database **and** cache, or database **and** message queue) is broken in two independent
ways:

1. **Race:** two concurrent writes reach the two systems in different orders; the database
   ends with value B while the index ends with value A. They are **permanently** divergent,
   and no amount of retrying fixes it.
2. **Partial failure:** one write succeeds and the other fails. Now you need an atomic
   commit across heterogeneous systems (2PC — see `09-consistency-and-consensus.md`), which
   you almost certainly do not want.

**The fix: one system is the leader/source of truth, and everything else is derived from
its ordered change log.** Derived data has a clear direction of dataflow and is
deterministic and replayable; dual writes have neither.

### 2.2 Change data capture (CDC)
Observe the database's own replication/write-ahead log and publish the changes as a stream
(Debezium, Kafka Connect, Postgres logical decoding, MySQL binlog). Downstream consumers
apply changes **in the same order**, so derived systems converge.

- **Initial snapshot:** you need a consistent snapshot plus the **log position** it
  corresponds to, so the stream can continue from exactly there (same requirement as
  setting up a replica — `05-replication.md`).
- **Log compaction:** keep only the latest value per key so the log can rebuild derived
  state from scratch without unbounded retention.
- Deletes must be represented explicitly (**tombstones**), or deleted rows resurrect in
  derived systems.

### 2.3 The transactional outbox (the pattern to use in application code)
When you need "update the database **and** publish an event", write both **in the same
local transaction**:

```
BEGIN;
  UPDATE orders SET status = 'paid' WHERE id = 42;
  INSERT INTO outbox (id, aggregate_id, type, payload, created_at)
       VALUES (uuid, 42, 'order.paid', '{…}', now());
COMMIT;
-- A separate relay reads the outbox and publishes, marking rows as sent.
-- Delivery is at-least-once, so consumers MUST be idempotent.
```

This converts an impossible distributed atomic write into a single-node transaction plus an
at-least-once delivery problem, which is solvable with idempotence.

### 2.4 Event sourcing
Store the **immutable log of events** as the source of truth; derive current state by
replaying. Distinctions that matter:
- Events describe **what the user did** (intent) in an immutable form; **commands** may be
  rejected, events may not — validation happens when the command is turned into an event.
- Current state is a **fold** over the log; snapshots are an optimization.
- Enables retroactive derivation of new views, audit for free, and debugging by replay.
- Costs: you must handle schema evolution of events forever (`04-encoding-and-evolution.md`),
  and GDPR-style deletion of immutable data requires real design (crypto-shredding, or
  segregating personal data outside the log).

### 2.5 State, streams, and immutability
`state = fold(events)`. Immutable append-only logs separate **the facts** (what happened)
from **the interpretation** (the derived views). You can add a new view later by replaying
history; you can fix a bug by rebuilding the view. Mutable-state-only systems lose the
history and therefore lose that ability.

Limits: high-churn or high-privacy datasets make an unbounded immutable log impractical —
compaction, GC, and deletion strategies must be designed rather than assumed.

---

## 3. Processing streams

### 3.1 Uses
Complex event processing (pattern matching), streaming analytics (rolling aggregates),
maintaining materialized views/search indexes, and monitoring.

### 3.2 Reasoning about time — the biggest source of subtle bugs

- **Event time vs. processing time.** Use **event time** for windowing; processing time
  produces artifacts (a consumer restart after a lag makes traffic appear to spike when it
  is only catching up).
- **You can never be sure a window is complete.** Straggler events arrive after you have
  declared a window closed. Choose explicitly: ignore stragglers (and track a metric for
  how many you drop), or publish a **correction** that updates the previous output.
- **Whose clock?** The device clock may be wrong or deliberately manipulated. A common
  correction: record three timestamps — event time on the device, send time on the device,
  and receive time on the server — then use (receive − send) to estimate device clock offset.
- **Window types:** tumbling (fixed, non-overlapping), hopping (fixed, overlapping),
  sliding (relative to each event), session (grouped by activity with an inactivity gap).
  Pick the one that matches the question you are answering.

### 3.3 Stream joins
- **Stream-stream** (window join): both sides buffered for a window; e.g. matching a search
  event to a later click. Requires state proportional to the window.
- **Stream-table** (enrichment): join events against a local copy of a table kept up to
  date by CDC. Preferable to querying a remote database per event (which is slow and
  nondeterministic on replay).
- **Table-table** (materialized view maintenance): both sides are change streams.
- **Time dependence:** joins are **nondeterministic if the state changes** — replaying with
  a later version of the table gives a different result. If determinism matters, join
  against a **versioned** state (an identifier for the version of the joined record at
  event time). This is the "slowly changing dimension" problem: a tax rate applied to an
  old order must be the rate at the time of the order, not today's.

### 3.4 Fault tolerance and "exactly-once"
Batch jobs get fault tolerance by re-running failed tasks and discarding the output of
failed ones. Streams are unbounded, so instead:

- **Microbatching** (Spark Streaming) or **checkpointing** (Flink) creates recovery points.
- That alone gives exactly-once *within* the framework — but **any side effect that escapes
  the framework** (a database write, an email, an HTTP call) can happen more than once.
- **"Exactly-once" really means "effectively-once"**: at-least-once delivery plus
  **idempotent** effects (a dedup key at the point of effect, a conditional write, or a
  unique index), and/or an atomic commit of output + offset.
- **Idempotence often needs metadata**: e.g. record the offset of the last-applied message
  with the value, and skip messages at or below it. Fencing tokens apply here too when a
  restarted consumer overlaps with an old one.
- **Rebuilding state after failure:** either keep state in a remote store (network cost per
  access) or keep it local and replicate the changelog (Kafka Streams/Flink). Local +
  changelog is normally the right choice.

---

## 4. Review checklist

- [ ] No dual writes: exactly one source of truth; all other stores derived via CDC or outbox
- [ ] Events published in the same transaction as the state change (outbox), not after commit
- [ ] Every consumer is idempotent, with a stated dedup key and where the dedup state lives
- [ ] Message ordering requirements identified; per-entity ordering achieved by partition key
- [ ] Consumer lag is monitored and alerted before retention expiry
- [ ] Retention window sized against the longest realistic consumer outage
- [ ] Every queue is bounded; overflow behavior (drop / backpressure / shed) is defined
- [ ] Dead-letter queue exists, with an owner, an alert, and a documented replay procedure
- [ ] Poison-message handling: bounded retry then DLQ, never an infinite retry loop
- [ ] Windowing uses event time, with a stated lateness policy for stragglers
- [ ] Untrusted device clocks corrected or not relied upon
- [ ] Stream-table joins use a locally-maintained copy, not a per-event remote query
- [ ] Joins requiring determinism use versioned state, not "current" state
- [ ] Deletes propagate as tombstones so derived stores don't resurrect data
- [ ] Consumer state after restart is rebuildable (changelog or replay), and this is tested
- [ ] Event schema evolution follows `04-encoding-and-evolution.md` — old events stay readable
