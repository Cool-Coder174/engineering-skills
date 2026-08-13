# Transactions and Isolation

**Source:** DDIA Ch. 7, p. 221.

This is where most application-level data corruption originates, because the failure mode
is a **race condition that passes every test** — tests are single-threaded, production is
not.

---

## 1. What ACID actually guarantees

The word ACID is marketing as often as it is a specification ("ACID compliant" tells you
almost nothing). Precisely:

- **Atomicity:** *abortability*. If a multi-write transaction fails partway, everything is
  discarded and the client can safely retry. It is **not** about concurrency.
- **Consistency:** an application-level notion — your invariants hold. The database can
  enforce some of it (FKs, uniqueness, CHECK), but "the balance never goes negative across
  two tables" is **your** responsibility. The C is, properly, the application's letter.
- **Isolation:** concurrently executing transactions do not interfere. The textbook meaning
  is serializability; **almost nobody runs at that level by default**.
- **Durability:** committed data is not lost. In practice this means WAL + replication +
  backups; note that "written to disk" is meaningless if the disk is the only copy, and
  that fsync/write-cache behavior varies.

**BASE** ("Basically Available, Soft state, Eventual consistency") means little beyond
"not ACID". Do not accept it as a design statement.

---

## 2. Single-object vs. multi-object

- **Single-object writes** are atomic and isolated in essentially every store (via WAL and
  per-object locks). Also common: atomic increment, and compare-and-set.
- **Multi-object transactions** are what you need when several rows/documents must change
  together: foreign key integrity, denormalized/duplicated fields, secondary indexes.
  Many distributed stores do not support them across partitions.

**Handling errors and aborts:** the whole point of aborts is safe retry — but retry is not
free of hazards:
- The transaction may have succeeded and the *acknowledgment* was lost → retrying performs
  it twice unless you have an application-level **deduplication key**.
- If the error is due to overload, retrying makes it worse → back off, and cap attempts.
- Retrying is pointless for permanent errors (constraint violation).
- If the transaction had **side effects outside the database** (sent an email, charged a
  card, published a message), retrying repeats them. Use two-phase design or an outbox.

---

## 3. Weak isolation levels — the anomaly/level matrix

| Anomaly | Read committed | Snapshot isolation (repeatable read) | Serializable |
|---|---|---|---|
| Dirty read | prevented | prevented | prevented |
| Dirty write | prevented | prevented | prevented |
| Read skew (non-repeatable read) | **possible** | prevented | prevented |
| Lost update | **possible** | **possible** (some engines auto-detect) | prevented |
| Write skew | **possible** | **possible** | prevented |
| Phantoms | **possible** | **possible** (prevented for reads only) | prevented |

### 3.1 Read committed (the common default)
Guarantees only: no dirty reads (you only see committed data) and no dirty writes (you
only overwrite committed data). Implemented with row-level write locks and, for reads, by
remembering the old committed value while a write is uncommitted.

Not sufficient for anything that reads a value and then acts on it.

### 3.2 Snapshot isolation / repeatable read
Each transaction reads from a **consistent snapshot** taken at its start — a key
principle: *readers never block writers, and writers never block readers*. Implemented with
MVCC: multiple committed versions per object, visibility determined by transaction IDs.

Ideal for backups, analytics, and integrity checks — any long-running read query. Note the
naming chaos: the same idea is "repeatable read" in PostgreSQL/MySQL, "serializable" in
Oracle, "snapshot isolation" in SQL Server. **Never reason from the level's name; verify
what your engine actually provides.**

**Operational note:** long-running snapshots force the database to keep old versions
around, causing bloat. Bound the runtime of analytical transactions.

### 3.3 Lost updates
Two transactions do read-modify-write concurrently; one overwrites the other's result
without incorporating it. Classic cases: incrementing a counter, updating a JSON document
in application code, editing a wiki page concurrently.

Fixes, in order of preference:
1. **Atomic write operation:** `UPDATE counters SET value = value + 1 WHERE key = 'x'`.
   Correct, simple, and it takes an exclusive lock on the row. **Beware ORMs that fetch,
   mutate in memory, and write back — they silently defeat this.**
2. **Explicit locking:** `SELECT … FOR UPDATE` on the rows the decision depends on.
3. **Compare-and-set:** `UPDATE … SET content = 'new' WHERE id = 1 AND content = 'old'`;
   check the affected-row count and retry on 0. Caution: if the `WHERE` clause reads from
   an old snapshot, CAS may not be effective — verify your engine's behavior.
4. **Automatic detection:** PostgreSQL repeatable read, Oracle serializable, and SQL Server
   snapshot isolation detect lost updates and abort; **MySQL/InnoDB repeatable read does
   not**. This is a big engine difference; know yours.
5. **Replicated data:** locks and CAS assume a single copy. In multi-leader/leaderless
   systems, use commutative operations (counters that merge), or siblings + application
   merge. **Last-write-wins is prone to lost updates and is the default in many replicated
   databases** — check and change it.

### 3.4 Write skew and phantoms — the ones to hunt for
**Definition:** two transactions read the same set of objects, then update **different**
objects, each invalidating the premise the other read. Generalizes lost update (where they
update the *same* object).

**Detection signature in code:**
```
BEGIN;
SELECT count(*) / SELECT … WHERE <condition>;   -- check a premise
if (application-level check passes) {
    INSERT / UPDATE something;                   -- act on the premise
}
COMMIT;
```
If two transactions run this concurrently, both see the premise satisfied and both act.

Real examples: two on-call doctors both go off-call; two bookings for the same meeting
room; two players moving to the same board square; two users claiming the same username;
a spending limit exceeded by two concurrent claims against separate rows.

**Why it is hard:**
- Atomic single-object operations don't help — multiple objects are involved.
- Automatic lost-update detection doesn't help. Write skew is **not** detected by
  PostgreSQL repeatable read, MySQL/InnoDB repeatable read, Oracle serializable, or SQL
  Server snapshot isolation.
- A database constraint would fix it, but the constraint spans multiple rows/objects,
  which most databases don't support directly (triggers or materialized views may work).

**The phantom:** a write in one transaction changes the result of a search in another. It
is what makes write skew possible. Because the row does not yet exist, there is nothing to
lock — you cannot `SELECT … FOR UPDATE` a row that isn't there.

**Fixes, in order of preference:**
1. **Serializable isolation.** The only general answer.
2. **A database constraint** if the invariant can be expressed on one object — a unique
   index is the correct fix for "claim a username" and for exactly-once semantics.
3. **Materializing conflicts:** create rows representing the lockable resource in advance
   (e.g. a row per room per 15-minute slot) and lock those. Ugly, leaks a concurrency
   control mechanism into the data model, and a last resort.
4. **Explicit `SELECT … FOR UPDATE`** on the rows the decision depends on — works when the
   rows exist, fails against phantoms.

---

## 4. Serializability

The only isolation level that guarantees the result is the same as *some* serial execution.
Three implementations:

### 4.1 Actual serial execution
Run one transaction at a time on a single thread (VoltDB/H-Store, Redis, Datomic). Viable
now that RAM is cheap and OLTP transactions are short. Requirements:
- Transactions must be **small and fast** — one slow transaction stalls everything.
- Use **stored procedures** rather than interactive multi-statement transactions, so no
  network round trip happens inside the transaction.
- The active dataset should fit in memory.
- Write throughput must fit on a single CPU core, or the data must be partitioned so that
  each partition has its own thread — and **cross-partition transactions are much slower**.

### 4.2 Two-phase locking (2PL)
Writers block readers *and* readers block writers (unlike snapshot isolation). Shared locks
for readers, exclusive for writers, all held until commit. **Predicate locks / index-range
locks** are what solve phantoms — locking a *condition* rather than existing rows.

Cost: much worse and much less predictable latency (a single slow transaction blocks
everything), and **deadlocks** occur frequently and must be detected and retried.

### 4.3 Serializable snapshot isolation (SSI)
Optimistic: run on a snapshot, track whether the premise a transaction read has since
changed, and abort at commit if it has. Detects stale MVCC reads and writes that affect a
prior read. Available as PostgreSQL `SERIALIZABLE` and in FoundationDB.

- **Much better performance than 2PL** when contention is low, because reads never block.
- Aborts under high contention → your application **must** implement a retry loop with
  bounded attempts and backoff. Without the retry loop, SSI just moves the bug from data
  corruption to user-visible errors.
- Not limited to one CPU core; scales across partitions.

---

## 5. Practical policy for the skill suite

1. **Every read-then-write sequence must declare its isolation level and its protection
   mechanism.** Options: atomic operation, unique constraint, `FOR UPDATE`, CAS with retry,
   or `SERIALIZABLE` with a retry loop.
2. **A unique index is the strongest, cheapest tool you have.** Use it for idempotency keys,
   claim/reservation semantics, and dedup. It survives concurrency without a lock.
3. **If you choose `SERIALIZABLE`, you owe a retry loop.**
4. **If you choose anything weaker, name the anomaly you are accepting** and why it is
   tolerable.
5. **Never put a network call, an email send, or a payment charge inside a transaction.**
   Use the outbox pattern (write intent transactionally; deliver asynchronously with
   idempotency).

---

## 6. Review checklist

- [ ] Isolation level of the connection/framework is known (not assumed) and documented
- [ ] Every read-modify-write uses an atomic op, a lock, a CAS, or serializable + retry
- [ ] ORM fetch-mutate-save patterns on counters/balances identified and replaced
- [ ] Check-then-act patterns audited for write skew; fixed by constraint or serializable
- [ ] Uniqueness enforced by a database unique index, not an application `SELECT` check
- [ ] Phantom-prone operations (booking, claiming, allocating) reviewed explicitly
- [ ] Serializable transactions have a bounded retry loop with backoff
- [ ] Deadlock retry present where 2PL is in use
- [ ] Long-running read transactions bounded (snapshot bloat)
- [ ] Multi-object invariants enforced by a mechanism, not by application ordering
- [ ] No external side effects (email, HTTP, payment) inside a database transaction
- [ ] Retry after an ambiguous failure is protected by a dedup/idempotency key
- [ ] LWW conflict resolution not used where lost updates are unacceptable
