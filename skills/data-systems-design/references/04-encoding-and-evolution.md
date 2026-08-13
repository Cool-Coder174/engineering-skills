# Encoding and Evolution

**Source:** DDIA Ch. 4, p. 111.

This chapter is the practical heart of "can we deploy on Friday". Most production
incidents attributed to "a bad migration" are really compatibility failures during the
window when two versions of the code are running at once.

---

## 1. The two directions of compatibility

| Term | Meaning | Broken by |
|---|---|---|
| **Backward compatibility** | New code can read data written by old code | Removing a default; assuming a new field exists |
| **Forward compatibility** | Old code can read data written by new code | Adding a required field; changing a field's type or meaning; strict/exhaustive parsing |

Backward compatibility is usually easy (you know the old format). **Forward compatibility
is the hard one**, because old code must ignore what it does not understand — and must
*preserve* it if it round-trips the data.

**Why both are always required:** server-side rolling upgrades and client-side apps that
users don't update mean old and new code run **simultaneously**, in both directions, for
an unbounded period.

---

## 2. Encoding formats

### 2.1 Language-specific serialization (Java Serializable, Python pickle, Ruby Marshal)
**Do not use for anything persisted or sent over the network.** They tie you to one
language, they instantiate arbitrary classes on decode (remote code execution risk), they
usually neglect versioning, and their efficiency is poor.

### 2.2 JSON, XML, CSV
Ubiquitous and human-readable; good defaults for external APIs. Known hazards to handle
explicitly:

- Number handling: no distinction between integers and floats; **IEEE 754 doubles lose
  precision above 2^53** — large IDs (Twitter/snowflake-style) must be sent as strings.
- No binary type; binary must be Base64-encoded (~33% size increase).
- Optional schema languages (JSON Schema) exist and are worth using at trust boundaries.
- CSV has no schema at all and ambiguous escaping — acceptable only for interchange.

### 2.3 Binary schema-driven formats

**Thrift / Protocol Buffers:** each field has a **tag number** and a type annotation.

Evolution rules:
- Field **tag numbers can never change**; names can.
- New fields must be `optional` (or have a default). Adding a required field breaks
  backward compatibility with existing data.
- You can only remove an **optional** field, and you must **never reuse its tag number** —
  reserve retired tags explicitly.
- Type changes are lossy/risky (e.g. 32-bit → 64-bit int is backward compatible for new
  readers but old readers truncate).
- Repeated fields: a single-valued field can be evolved into `repeated` in protobuf
  (old readers see the last element); Thrift lists cannot.

**Avro:** no tag numbers. Uses a **writer's schema** and a **reader's schema**, resolved
at decode time by matching field *names*, with defaults filling in missing fields.

- Adding/removing a field is compatible **only if it has a default value**.
- Field names can be evolved with aliases; there is no tag number to preserve.
- Excellent for large files and dynamically generated schemas (e.g. schema derived from a
  database table), because the writer's schema is stored once per file/connection rather
  than per record.

### 2.4 The merits of schemas
Binary schema-driven encodings are compact, the schema is **enforced documentation that
cannot go stale**, compatibility can be **checked in CI before deploy**, and code
generation gives type-checked access in statically typed languages.

**Actionable:** put a schema-compatibility check in CI (Protobuf/Avro/JSON-Schema
linters, or the Confluent Schema Registry's compatibility modes). Compatibility that is
only enforced by code review is not enforced.

---

## 3. Modes of dataflow

### 3.1 Through databases
The database is a message from your past self to your future self. Consequences:

- The process writing may be a **newer** version than the process reading — forward
  compatibility is mandatory.
- **Data outlives code.** Rewriting all data on every schema change is expensive; most
  databases allow adding a nullable column without rewriting, and old rows keep returning
  `NULL`. Your code must handle rows written years ago.
- **The unknown-field preservation trap:** an old version of the application reads a
  record, decodes it, modifies one field, and writes it back — silently **dropping**
  fields it did not understand. Fix: preserve unknown fields explicitly, or never
  round-trip whole records through a version that may be older than the writer.

### 3.2 Through services (REST and RPC)
Servers and clients are updated independently; the API is a compatibility contract that
lasts as long as the oldest client you support (for public APIs and mobile apps: forever).

Version via URL path, header, or content negotiation — pick one and be consistent.

**RPC is not a local function call.** Do not let the abstraction fool you:
- A local call is predictable; a network call fails for reasons unrelated to your code.
- A local call returns, throws, or never returns. A network call can also **time out with
  an unknown outcome** — you do not know whether the request was executed.
- Retrying a timed-out request can execute it twice unless the operation is **idempotent**
  or protected by a **deduplication key**. This is the single most important consequence.
- Latency is wildly variable; arguments must be serialized (no pointers to local memory).

### 3.3 Message-passing dataflow (queues, brokers)
Asynchronous, one-way, buffered. Benefits: broker acts as a buffer during downstream
outages, redelivery prevents message loss, decouples sender from recipient, supports
fan-out to multiple consumers.

Compatibility consequence: messages sit in the queue across deploys, so **both
compatibility directions apply to message payloads** exactly as they do to database rows.
Consumers must tolerate unknown fields, and producers must not remove fields consumers
still read.

---

## 4. The mandatory migration pattern: expand → migrate → contract

Any change that is not purely additive must be split across **at least two deploys**.

**Phase 1 — Expand (deploy N):**
- Add the new column/field/table/topic as **nullable or with a default**.
- Write to **both** old and new. Read from **old**.
- No consumer is required to know about the new shape.

**Phase 2 — Migrate (deploy N, backfill job):**
- Backfill historical rows in **bounded batches** with progress tracking and a resume
  point — never one unbounded `UPDATE` that locks the table.
- Verify: row counts match, spot-check values, run a reconciliation query.
- Switch reads to the new field, behind a flag so it can be reverted instantly.

**Phase 3 — Contract (deploy N+1 or later):**
- Stop writing the old field.
- Only after all readers are confirmed upgraded, drop the old column/field.
- Never reuse a retired protobuf/Thrift tag number.

**Rules that follow:**
- Renaming a column is **never** a rename — it is add + backfill + switch + drop.
- Changing a column's type is add-new-column + backfill + switch + drop.
- Making a nullable column `NOT NULL` requires the backfill to complete *and* all writers
  to be upgraded first.
- A single deploy that both adds the new thing and removes the old thing is a guaranteed
  outage during the rolling window.
- Every migration needs a stated **rollback plan** that does not lose data. If rollback
  requires restoring a backup, say so out loud before shipping.

### 4.1 Locking hazards on relational migrations
- Adding a column with a non-constant default, or changing a type, can rewrite the whole
  table under an exclusive lock.
- Adding an index without `CONCURRENTLY` (Postgres) / online DDL (MySQL) blocks writes.
- A migration waiting on a lock queues **everything behind it** — set a short
  `lock_timeout` and retry rather than stalling the application.

---

## 5. Review checklist

- [ ] Every schema/API/message change classified: additive, or requires expand→contract
- [ ] Backward compatibility verified (new code reads old data)
- [ ] Forward compatibility verified (old code reads new data, ignores unknown fields)
- [ ] No required field added without a default
- [ ] No protobuf/Thrift tag number reused; retired tags reserved
- [ ] No round-trip path where an old reader can drop unknown fields on write-back
- [ ] Backfill is batched, resumable, and has a progress/verification query
- [ ] Migration uses non-blocking DDL (`CONCURRENTLY` / online DDL) with a lock timeout
- [ ] Reads switched behind a flag; rollback path stated and does not lose data
- [ ] Drop of the old field is in a **separate, later** deploy
- [ ] Queued/in-flight messages of the old shape are still decodable by new consumers
- [ ] Large integer IDs are not sent as JSON numbers
- [ ] Schema compatibility is checked in CI, not just in review
