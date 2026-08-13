# Hazard Catalog

A catalog of named, detectable failure modes for data-intensive and distributed systems,
derived from *Designing Data-Intensive Applications* (Kleppmann).

Each entry has:
- **Signature** — what the code, schema, or design looks like. This is what you grep for.
- **Consequence** — what actually goes wrong, and when.
- **Fix** — the required remedy.

**How to use it:** `code-review` runs the applicable sections against a diff; `verify`
runs them against an implemented phase; `planner`/`detail-planning` run them against a
proposed design. Report only applicable hazards. "Not applicable — single-process, no
concurrency, no persistence" is a valid result and is preferable to inventing findings.

**Severity guidance:** hazards marked 🔴 cause data loss or corruption and block merge.
🟡 cause outages, incorrect results under load, or unbounded operational cost. 🔵 are
maintainability and operability risks.

---

## A. Concurrency and transactions (Ch. 7)

### 🔴 H-01 — Read-modify-write lost update
**Signature:** code reads a value, computes a new value in application memory, writes it
back. Any `SELECT` → arithmetic → `UPDATE`; any ORM `obj = fetch(); obj.count += 1; save()`;
any JSON document read, mutated, and re-saved.
**Consequence:** concurrent updates silently overwrite each other. Counters undercount,
balances drift, list edits vanish. Never reproduces in single-threaded tests.
**Fix:** atomic write (`UPDATE t SET n = n + 1 WHERE …`), `SELECT … FOR UPDATE`, or
compare-and-set with an affected-row check and a retry loop.

### 🔴 H-02 — Check-then-act write skew
**Signature:** query a condition, branch on it in application code, then insert or update
based on the branch — inside or outside a transaction.
**Consequence:** two concurrent transactions both observe the condition as satisfied and
both act. Double bookings, two doctors off call, overdrafts, oversold inventory.
**Not prevented by snapshot isolation** in PostgreSQL repeatable read, MySQL/InnoDB
repeatable read, Oracle "serializable", or SQL Server snapshot isolation.
**Fix:** `SERIALIZABLE` isolation with a retry loop; or a database constraint (unique
index / exclusion constraint) that expresses the invariant; or materialize the conflict
and lock it with `FOR UPDATE`.

### 🔴 H-03 — Uniqueness enforced in application code
**Signature:** `SELECT … WHERE email = ?` → `if (empty) INSERT`.
**Consequence:** duplicates under concurrency. A special case of H-02, and extremely common.
**Fix:** a database **unique index**. Handle the constraint-violation error as the normal
"already taken" path.

### 🟡 H-04 — Unknown or assumed isolation level
**Signature:** no statement anywhere of the isolation level; a multi-step read-then-write
that assumes "the transaction protects it".
**Consequence:** the code is correct under an isolation level you are not running.
**Fix:** state the isolation level per critical transaction and set it explicitly;
document which anomaly each critical path is protected against and by what mechanism.

### 🟡 H-05 — Serializable without a retry loop
**Signature:** `SERIALIZABLE` (SSI) set, but no handling of serialization-failure errors
(Postgres `40001`) or deadlock errors (`40P01`).
**Consequence:** user-facing 500s under contention. SSI aborts by design.
**Fix:** a bounded retry loop with backoff around the whole transaction, and a metric on
the retry rate.

### 🔴 H-06 — External side effect inside a transaction
**Signature:** an HTTP call, payment charge, email send, or queue publish between `BEGIN`
and `COMMIT`.
**Consequence:** the transaction rolls back but the side effect already happened; or the
side effect is repeated on retry; or a slow remote call holds locks and stalls the database.
**Fix:** transactional outbox — write the intent in the transaction, deliver
asynchronously with idempotency (`references/11-stream-processing.md` §2.3).

### 🟡 H-07 — Long-running read transaction
**Signature:** an analytical/reporting/export query, or a transaction left open across
user think-time or a network call.
**Consequence:** MVCC snapshot retention causes table/index bloat; vacuum cannot advance;
performance degrades cluster-wide.
**Fix:** bound transaction duration; move analytics off the OLTP store; set
`idle_in_transaction_session_timeout` and `statement_timeout`.

### 🟡 H-08 — Unbounded transaction size
**Signature:** a migration or backfill issuing a single `UPDATE`/`DELETE` over an entire
large table.
**Consequence:** lock escalation and long lock waits; replication lag spike; possible
out-of-disk on WAL; a failure loses all progress.
**Fix:** batch by primary-key range with a committed checkpoint per batch and a pause
between batches.

---

## B. Schema and API evolution (Ch. 4)

### 🔴 H-09 — Breaking change deployed in a single step
**Signature:** one PR that adds the new column/field *and* removes the old one, or renames
a column, or makes a new column `NOT NULL` without a backfill.
**Consequence:** during the rolling deploy both versions run; one of them crashes or
writes data the other cannot read. Also breaks rollback.
**Fix:** expand → migrate → contract across at least two deploys
(`references/04-encoding-and-evolution.md` §4).

### 🔴 H-10 — Required field added without a default
**Signature:** a new non-nullable column, a new required protobuf field, a new mandatory
API parameter.
**Consequence:** old writers produce records the new readers reject (and vice versa).
**Fix:** optional with a default; make it required only after all writers are upgraded.

### 🟡 H-11 — Unknown-field dropping on round-trip
**Signature:** an older service reads a record, decodes into a struct that lacks newer
fields, modifies one field, and writes the whole record back.
**Consequence:** silent, permanent data loss of the newer fields.
**Fix:** preserve unknown fields through the decode/encode cycle, or restrict writes to
targeted field updates rather than whole-record replacement.

### 🟡 H-12 — Blocking DDL
**Signature:** `CREATE INDEX` without `CONCURRENTLY`, `ALTER TABLE` that rewrites, a
migration without `lock_timeout`.
**Consequence:** an exclusive lock queues every subsequent query — a full outage on a busy
table, and the incident often looks like "the database froze".
**Fix:** online/concurrent DDL, short `lock_timeout` with retry, and run migrations
separately from application deploys.

### 🔵 H-13 — Reused or unreserved schema tag / enum value
**Signature:** a removed protobuf/Thrift field tag reused by a new field; an enum value
repurposed.
**Consequence:** old data decodes into the wrong field with the wrong meaning.
**Fix:** never reuse tags; reserve them explicitly. Add new enum values instead of
repurposing, and ensure readers handle unknown enum values.

---

## C. Distributed calls and retries (Ch. 8)

### 🔴 H-14 — Non-idempotent retry
**Signature:** a retry wrapper, SDK auto-retry, or queue redelivery around an operation
with side effects (charge, insert, publish, increment) with no dedup key.
**Consequence:** duplicate charges, duplicate orders, double-counted metrics. Triggered by
a timeout on a request that actually succeeded.
**Fix:** end-to-end idempotency key generated at the true endpoint, deduplicated at the
point of effect in the same transaction (`references/12-correctness-and-integrity.md` §3.1).

### 🟡 H-15 — Missing timeout
**Signature:** any HTTP client, database driver, socket, or lock acquisition with no
explicit timeout.
**Consequence:** threads/connections pile up on a slow dependency until the pool is
exhausted, and the caller fails entirely. The most common cascading-failure mechanism.
**Fix:** an explicit, justified timeout on every outbound call; a connection-pool
acquisition timeout; total request deadline propagated to callees.

### 🟡 H-16 — Retry without backoff or jitter
**Signature:** `for i in range(3): try(); sleep(1)` — a fixed delay, or no delay.
**Consequence:** synchronized retry storms amplify load on a struggling dependency and
prevent recovery.
**Fix:** exponential backoff with **jitter**, a capped attempt count, and a retry budget.

### 🟡 H-17 — Nested retries
**Signature:** retries configured in the SDK **and** in a wrapper **and** in the gateway.
**Consequence:** attempts multiply (3 × 3 × 3 = 27 requests); a partial outage becomes a
total one.
**Fix:** retry at exactly one layer; make the others fail fast.

### 🟡 H-18 — Ambiguous outcome treated as failure
**Signature:** a timeout on a write is handled by immediately re-issuing the write.
**Consequence:** duplicate effect, because a timeout does not mean the operation did not
happen.
**Fix:** idempotency key, or a verify-then-act read, or a state machine that can query the
true outcome.

### 🟡 H-19 — No circuit breaker / bulkhead on a critical dependency
**Signature:** every request calls a failing dependency and waits for its timeout.
**Consequence:** the whole service's latency becomes the dependency's timeout; capacity
collapses.
**Fix:** circuit breaker with a half-open probe; isolate resource pools per dependency;
define a degraded response.

---

## D. Clocks, locks and leadership (Ch. 8, Ch. 9)

### 🔴 H-20 — Wall-clock ordering across nodes
**Signature:** comparing `created_at`/`updated_at` from different machines to decide which
write wins or which event came first; last-write-wins on timestamps.
**Consequence:** clock skew silently discards writes; order is wrong; the bug is invisible
until an audit.
**Fix:** logical clocks / version vectors for ordering; a single-node sequence or an
ordered log for a total order.

### 🟡 H-21 — Elapsed time measured with a wall clock
**Signature:** `end = now(); duration = end - start` using a time-of-day clock, for
timeouts, rate limits, or lease expiry.
**Consequence:** NTP steps and leap-second handling produce negative or huge durations;
timeouts fire immediately or never.
**Fix:** a monotonic clock (`System.nanoTime`, `CLOCK_MONOTONIC`, `time.monotonic()`).

### 🔴 H-22 — Distributed lock without a fencing token
**Signature:** acquire lock → `if (lock.isValid())` → write to the resource. No token is
passed to or checked by the resource.
**Consequence:** a GC pause or VM suspend outlasts the lease; the paused holder wakes and
writes after another holder has taken over. Corruption with two "owners".
**Fix:** monotonically increasing fencing token issued with the lock, sent with every
write, and **enforced by the resource** (reject lower tokens). Client-side checks are not
sufficient.

### 🟡 H-23 — Self-assessed leadership
**Signature:** a node decides it is the leader based on its own state, a config flag, or a
local timer.
**Consequence:** split brain; two leaders accepting writes.
**Fix:** leadership from a consensus service (ZooKeeper/etcd) with sessions, leases and
fencing; a defined behavior for losing leadership mid-operation.

### 🟡 H-24 — Cross-channel race
**Signature:** write to store A, then publish a message/notification; the consumer reads
from store A.
**Consequence:** the consumer arrives before the write is visible (async replication,
eventual-consistency store, cache) and sees nothing or stale data.
**Fix:** include the payload in the message; or read from the leader; or have the consumer
retry until the expected version is visible; or derive the message from the store's own
change log.

---

## E. Replication and partitioning (Ch. 5, Ch. 6)

### 🟡 H-25 — Read-after-write from a replica
**Signature:** write, then immediately read from a read replica or a load-balanced pool, on
a path where the user sees their own change.
**Consequence:** the user's change appears to have vanished; they retry and create duplicates.
**Fix:** read from the leader for a bounded window after a user's write; or pin the session;
or wait for the replica to reach the write's log position.

### 🟡 H-26 — Non-monotonic reads
**Signature:** repeated reads or pagination hitting different replicas.
**Consequence:** results move backwards in time; items appear and disappear between pages.
**Fix:** sticky routing of a user to one replica; or read a consistent snapshot/cursor.

### 🟡 H-27 — Unmonitored replication lag
**Signature:** replicas exist; no lag metric, threshold, or alert.
**Consequence:** the application's consistency assumptions break silently during load or
incidents, exactly when it matters.
**Fix:** monitor lag; alert on a threshold; remove lagging replicas from the read pool
automatically.

### 🟡 H-28 — Silent last-write-wins conflict resolution
**Signature:** multi-leader/leaderless store with default conflict resolution and no
documented policy.
**Consequence:** concurrent writes are silently discarded.
**Fix:** an explicit conflict policy — avoid conflicts by routing, merge with a CRDT,
or surface siblings for application/user resolution. Document it.

### 🟡 H-29 — Hot-spot partition key
**Signature:** partitioning/sharding by timestamp, auto-increment ID, a single tenant, or a
constant.
**Consequence:** all traffic to one node; the cluster runs at one node's speed.
**Fix:** hash or compound key `(entity_id, timestamp)`; bucket-prefix genuinely hot keys and
fan out reads.

### 🟡 H-30 — `hash mod N` partition assignment
**Signature:** `partition = hash(key) % node_count`.
**Consequence:** adding a node reshuffles nearly all keys — a massive, unnecessary
rebalancing.
**Fix:** a fixed large number of partitions mapped to nodes, or consistent hashing, or
dynamic partitioning.

### 🔵 H-31 — Scatter/gather on a latency-critical path
**Signature:** a query that must consult every partition (local secondary index lookup) on
a user-facing hot path.
**Consequence:** p99 becomes the max over all partitions and degrades as you add partitions.
**Fix:** align the access pattern with the partition key, or use a global (term-partitioned)
index and accept its write cost and staleness.

---

## F. Streams, queues, and derived data (Ch. 10, Ch. 11, Ch. 12)

### 🔴 H-32 — Dual write
**Signature:** application code writes the same fact to two systems — database and search
index, database and cache, database and queue — in sequence.
**Consequence:** permanent divergence on partial failure or reordered concurrent writes.
Retrying cannot fix it.
**Fix:** one source of truth; derive everything else from its change log (CDC) or a
transactional outbox.

### 🟡 H-33 — Unbounded queue or buffer
**Signature:** an in-memory list/channel/queue with no capacity limit; no backpressure.
**Consequence:** memory exhaustion and OOM under a burst, or unbounded latency; the
behavior at the limit was never tested.
**Fix:** bound every queue; define the overflow policy (block/backpressure, shed, or drop
with a metric).

### 🟡 H-34 — No dead-letter path
**Signature:** a consumer that retries a failing message forever, or acknowledges and drops
it.
**Consequence:** a poison message blocks the partition permanently, or data is silently lost.
**Fix:** bounded retries → dead-letter queue with an alert, an owner, and a documented
replay procedure.

### 🟡 H-35 — Unmonitored consumer lag / retention loss
**Signature:** a stream consumer with no lag metric; retention shorter than a realistic
outage.
**Consequence:** the consumer falls behind, retention deletes unread messages, data is lost
silently.
**Fix:** lag metric and alert; retention sized against the longest realistic outage with an
alert before the boundary.

### 🟡 H-36 — Processing-time windowing
**Signature:** aggregation windows keyed on `now()` at processing time.
**Consequence:** a restart or a lag catch-up produces phantom spikes and misattributed
data; results are not reproducible on replay.
**Fix:** window on event time, with an explicit lateness policy and a metric for dropped
stragglers.

### 🟡 H-37 — Non-deterministic job or consumer
**Signature:** output depends on `now()`, randomness, map iteration order, or a live remote
lookup.
**Consequence:** re-runs and replays produce different results; recovery and backfill
cannot be trusted; joins against "current" state misprice historical events.
**Fix:** inject time and randomness; join against versioned state; isolate nondeterministic
steps behind an idempotent boundary.

### 🟡 H-38 — Destructive in-place backfill
**Signature:** a script that mutates rows in place across a table with no record of prior
values and no checkpoint.
**Consequence:** a bug destroys data irreversibly; a partial failure leaves an unknown
mixed state.
**Fix:** write to new columns/tables, verify, switch atomically, retain the old data for a
defined period; checkpoint and make it resumable.

### 🟡 H-39 — Overlapping scheduled job runs
**Signature:** a cron/scheduler task whose runtime can exceed its interval, with no lock.
**Consequence:** two instances process the same records concurrently — duplicates and lost
updates.
**Fix:** a lease/advisory lock with a token; skip or queue if the previous run is active;
alert on overrun.

### 🔵 H-40 — Missing tombstones on delete
**Signature:** deletes are applied to the source but not propagated as explicit events to
derived stores.
**Consequence:** deleted records resurrect in the search index, cache, or analytics store —
including records deleted for privacy/legal reasons.
**Fix:** propagate deletes explicitly as tombstone events; verify deletion across all
derived stores.

---

## G. Reliability, capacity and operability (Ch. 1)

### 🟡 H-41 — Performance stated as an average
**Signature:** "average response time under 200ms"; dashboards showing means; averaged
percentiles across hosts.
**Consequence:** the tail — where the pain and the churn are — is invisible. Averaging
percentiles is mathematically meaningless.
**Fix:** p50/p95/p99 from aggregated histograms, measured client-side.

### 🟡 H-42 — Unbounded result set
**Signature:** a query or API with no `LIMIT`, no pagination, or pagination with an
unbounded page size; loading a full table into memory.
**Consequence:** works in dev with 100 rows, exhausts memory and times out in production.
**Fix:** mandatory pagination with a maximum page size; keyset pagination for deep pages;
streaming for large exports.

### 🟡 H-43 — N+1 queries
**Signature:** a loop issuing one query per item; ORM lazy loading inside a serializer.
**Consequence:** latency scales with result size; the database saturates on connection and
round-trip overhead.
**Fix:** batch/eager-load, a join, or a single `IN` query. Assert query counts in tests for
hot endpoints.

### 🟡 H-44 — Cache without an invalidation or staleness contract
**Signature:** a cache added with a TTL and no statement of what stale data is acceptable;
no stampede protection.
**Consequence:** users see stale data in paths that cannot tolerate it; on expiry or a cold
start, a thundering herd hits the origin and takes it down.
**Fix:** state the staleness bound; define the invalidation trigger (preferably derived from
the change log); add request coalescing/soft-TTL; test cold-start capacity.

### 🟡 H-45 — Correlated failure across replicas
**Signature:** all replicas run the same version, on the same input, with the same
dependency, in the same AZ.
**Consequence:** redundancy does not help — one bad input or dependency takes all of them
down simultaneously.
**Fix:** identify the correlated dimension; add input validation and resource limits;
stage rollouts; spread across failure domains; define a degraded mode.

### 🔵 H-46 — No rollback path
**Signature:** a change (schema, data, or config) with no stated way to revert.
**Consequence:** an incident becomes long because the only option is fixing forward under
pressure.
**Fix:** a written rollback plan per change; expand/contract migrations so every
intermediate state is deployable; feature flags for read-path switches.

### 🔵 H-47 — Invariants unmonitored
**Signature:** monitoring covers CPU, memory and error rate, but nothing checks that the
data is correct.
**Consequence:** corruption accumulates for months and is discovered by a customer or an
auditor.
**Fix:** continuous reconciliation and invariant checks with alerts (balances reconcile,
derived counts match source, no orphans, no overlaps).

### 🔵 H-48 — Untested failure path
**Signature:** failover, retry, degradation and recovery code that no test or drill ever
exercises.
**Consequence:** the least-tested code runs during the worst moment and is broken.
**Fix:** fault injection in tests and staging; periodic failover drills; restore-from-backup
drills that actually restore.

---

## Quick scan order

When reviewing a diff, scan in this order — it finds the expensive problems first:

1. **Any read-then-write?** → H-01, H-02, H-03, H-04
2. **Any schema/API/payload change?** → H-09, H-10, H-11, H-12, H-13
3. **Any network call, retry, or queue consumer?** → H-14, H-15, H-16, H-17, H-18, H-34
4. **Any write to more than one system?** → H-32, H-24, H-40
5. **Any lock, leader, or scheduled job?** → H-22, H-23, H-39
6. **Any timestamp used for logic?** → H-20, H-21, H-36
7. **Any new query, index, or partition key?** → H-29, H-31, H-42, H-43
8. **Any cache?** → H-44
9. **Does anything here need a rollback?** → H-46, H-38
