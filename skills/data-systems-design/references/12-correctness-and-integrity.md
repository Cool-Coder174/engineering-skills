# Correctness, Integrity, and Responsibility

**Source:** DDIA Ch. 12, p. 489.

The closing argument of the book: transactions and consensus are not enough on their own;
correctness must be established **end to end**, and it must be continuously **verified**
rather than assumed.

---

## 1. Data integration: derive, don't dual-write

No single tool does everything. Real systems combine an OLTP database, a cache, a search
index, a warehouse, and ML pipelines. The way to keep them consistent is not to write to
each of them, but to define:

- **One source of truth** (a system of record), and
- **derived data**, produced deterministically from it by a repeatable transformation.

Getting the **ordering** right is the crux: an ordered event log (total order broadcast)
applied deterministically makes all derived systems converge. Application-level dual writes
do not, because concurrent writes can be applied in different orders.

**Batch and stream are the same thing at different windows.** The **lambda architecture**
(parallel batch and stream paths, merged at read time) captured the right insight — derived
views should be recomputable from immutable input — but the cost of maintaining two
implementations of the same logic is high. Prefer a single engine that can both reprocess
history and process new events, and keep the derivation deterministic so reprocessing gives
the same answer.

**Reprocessing is how you evolve derived schemas without a big-bang migration:** maintain
the old and new views side by side, gradually shift a fraction of users to the new one,
compare, and keep the ability to revert.

---

## 2. Unbundling the database

Think of the pieces of a database — the write log, indexes, materialized views, triggers,
replication — as separable components composed at the application level. Two composition
approaches:

- **Federation (unified reads):** a query interface over heterogeneous stores.
- **Unbundling (unified writes):** synchronizing writes across systems via an ordered log.

The design principle to take away: **application code should be a deterministic function
from an input stream to a derived output stream**, with state changes flowing in one
direction. Dataflow-oriented design (event log → processors → derived views) makes
correctness, recovery, and evolution tractable in a way that ad-hoc mutual RPC between
services does not.

**Observing derived state:** the read path is where derived data meets the user. There is
a **spectrum between writing and reading** — how much work you do eagerly on write
(materializing) versus lazily on read (querying). Caches, materialized views, and search
indexes are all points on that spectrum. Choose the point deliberately and state the
staleness the read path can observe.

---

## 3. Aiming for correctness

### 3.1 The end-to-end argument
> A function can be completely and correctly implemented only with the knowledge and help
> of the application standing at the endpoints of the communication system.

Low-level reliability (TCP checksums, database transactions) is valuable but **cannot
substitute for an application-level check**. TCP guarantees the bytes of one connection
arrive; it cannot tell you the user pressed submit twice. A transaction guarantees
atomicity of one database's writes; it cannot tell you the same payment request was
retried after a timeout.

**Practical consequence — the pattern to enforce everywhere:**

> Generate a unique **operation ID / request ID at the true endpoint** (the client, the
> browser, the caller), carry it through every hop unchanged, and **deduplicate at the
> point of effect** using a uniqueness constraint in the same transaction as the effect.

```sql
-- Dedup and effect committed atomically. The unique constraint does the work.
BEGIN;
  INSERT INTO request_ids (request_id, account_id) VALUES (?, ?);  -- fails on duplicate
  UPDATE accounts SET balance = balance + ? WHERE id = ?;
COMMIT;
```

Note the two failure modes this closes: the transaction retried after an ambiguous timeout,
and the user double-clicking. A dedup check performed *before* and *outside* the
transaction closes neither.

### 3.2 Enforcing constraints
- **Uniqueness** requires consensus on the value being claimed. The practical technique:
  **partition by the constrained value** (partition usernames by hash of the username), so
  the constraint is enforceable by a single-node atomic operation. Then use the log:
  request a claim, let the ordered log decide who was first, and consumers apply the same
  deterministic decision.
- **Multi-partition constraints** (transfer money between two accounts in different
  partitions) can be done without an atomic commit across partitions: append a single
  request event with a unique ID to the log, then have each partition apply its part
  idempotently, deriving both effects from the same ordered request. Correctness comes from
  the log's order plus idempotence rather than from 2PC.
- **Constraints that can be enforced by the database should be** — a unique index, a check
  constraint, and a foreign key are more reliable than any amount of application code,
  because they hold under concurrency and under buggy callers.

### 3.3 Timeliness vs. integrity — the distinction to make in every design

| | Definition | Violation | Repairable? |
|---|---|---|---|
| **Timeliness** | Users see the system in an up-to-date state | Reading stale data | **Yes** — it resolves itself, and can be apologized for |
| **Integrity** | No corruption; no lost or duplicated data; derived data matches its source | Permanent inconsistency | **No** — requires explicit repair, and may be undetectable |

**Spend your coordination budget on integrity.** Timeliness violations are usually a
business problem with a business remedy (apologize, refund, retry). Integrity violations
are a data problem forever.

Dataflow systems can maintain integrity **without** distributed transactions by combining:
1. writing the request as a **single atomic message** (an event),
2. deriving all state deterministically from it,
3. passing a **client-generated request ID** end to end,
4. making messages **immutable** and reprocessing **idempotent**.

This achieves integrity with much better performance and availability than coordination.

**And be honest about coordination-avoiding designs:** they let you accept a write that
might later prove invalid (an oversell, an overdraft). That is a legitimate business choice
— many businesses already operate this way (airline overbooking, warehouse stock) and have
a **compensating process** (rebook, apologize, refund). Make the compensating transaction
an explicit, designed part of the system rather than an incident response.

### 3.4 Trust, but verify
Do not assume perfection. Disks corrupt silently, software has bugs, migrations lose rows,
and "this can't happen" happens.

- **Auditing is not optional.** Run continuous checks: reconciliation of derived data
  against its source, invariant checks (balances sum to zero, no overlapping bookings, no
  orphaned rows), row counts, and checksums.
- **Alert on invariant violations**, and treat them as incidents.
- **Design for auditability:** an event-based system with immutable inputs and deterministic
  derivations is checkable — you can recompute and compare. A system of in-place mutations
  is not.
- **Prefer detection you own** over detection by a user or a regulator.
- Do not blindly trust a single tool's guarantees, including the database's. Verify what
  you actually depend on.

---

## 4. Doing the right thing

Technical correctness is not the whole obligation. When designing systems that hold data
about people:

- **Predictive analytics** encodes the past into the future; systematic bias in training
  data produces systematic discrimination at scale, with the appearance of objectivity and
  no route to appeal. Automated decisions that materially affect people need explainability
  and a human appeal path.
- **Data as an asset is also a liability.** Every field you retain is something that can
  leak, be subpoenaed, be sold in a bankruptcy, or be abused by a future owner with
  different values. Collect less.
- **Consent is usually not meaningful** when the service is take-it-or-leave-it and the
  future uses of the data are unspecified. Do not treat a checkbox as ethical cover.
- **Practical engineering obligations:** data minimization, purpose limitation, defined
  retention and deletion (including from derived stores, backups, and immutable logs —
  design for this before you build an event-sourced system), encryption at rest and in
  transit, access control and access **logging**, and pseudonymization where it does not
  break the product.

---

## 5. Review checklist

- [ ] A single source of truth is identified for every dataset; derived stores are derived, not dual-written
- [ ] Derivations are deterministic and reprocessable from immutable input
- [ ] End-to-end request/operation ID generated at the true endpoint and carried through every hop
- [ ] Deduplication happens at the point of effect, in the same transaction as the effect
- [ ] Constraints enforced by the database (unique index, FK, check) wherever possible
- [ ] Multi-partition constraints handled by ordered log + idempotence, not by 2PC
- [ ] Each requirement classified as timeliness (repairable) or integrity (not), and the
      coordination budget spent on integrity
- [ ] Where an invalid write may be accepted, a compensating process is designed and owned
- [ ] Continuous auditing/reconciliation jobs exist and alert on invariant violations
- [ ] Backfills and repairs are possible because inputs are immutable and retained
- [ ] Data minimization: every retained personal field has a stated purpose and retention period
- [ ] Deletion path covers derived stores, caches, search indexes, backups, and event logs
- [ ] Access to sensitive data is controlled **and logged**
- [ ] Automated decisions affecting people have explainability and an appeal path
