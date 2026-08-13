# Storage and Retrieval

**Source:** DDIA Ch. 3, p. 69.

You do not need to implement a storage engine, but you must know enough to pick one and to
predict how it behaves under your workload — because the engine's characteristics leak
into your p99.

---

## 1. Indexes are a trade

**Every index speeds reads and slows writes.** An index is additional, derived structure
that must be updated on every write. This is the single most useful fact about indexing:
"just add an index" is never free, and a table with a dozen indexes has a dozen-times write
amplification on inserts.

Rules:
- Add indexes from measured query plans, not from intuition.
- Composite index column order matters: equality predicates first, then range, then sort.
- An index that is never used is pure write-side cost — audit and drop them.
- Covering indexes (including the selected columns) avoid a heap lookup at the cost of size.

---

## 2. Log-structured (LSM-tree) vs. page-oriented (B-tree)

### 2.1 LSM-trees (SSTables + memtable + compaction)
Writes go to an in-memory sorted structure (memtable) plus a write-ahead log, and are
flushed to immutable sorted files (SSTables) that are merged in the background
(compaction). Used by RocksDB, LevelDB, Cassandra, HBase, Lucene.

- **Strength:** high write throughput; sequential writes; better compression; lower write
  amplification in write-heavy workloads.
- **Weakness:** a read may have to check several SSTables (mitigated by Bloom filters);
  **compaction competes with foreground traffic for disk bandwidth**, so p99 latency is
  less predictable. At high write throughput, compaction can fall behind, disk space grows
  unboundedly, and reads slow down — a failure mode you must monitor explicitly.

### 2.2 B-trees
Fixed-size pages, updated in place, kept balanced; the standard for relational databases.
Requires a write-ahead log (redo log) so that a crash mid-page-split is recoverable.

- **Strength:** predictable read latency (each key exists in exactly one place); mature,
  well-understood transactional and locking behavior; range scans in key order.
- **Weakness:** writes hit disk at least twice (WAL + page), sometimes more with page
  splits; pages fragment; concurrency requires latches.

### 2.3 Choosing

| Workload | Prefer |
|---|---|
| Write-heavy ingest, time-series, event logs | LSM |
| Read-heavy, latency-SLO-bound, transactional | B-tree |
| Strong transactional isolation needed | B-tree (locks attach naturally to the tree) |
| Disk space / compression a primary cost | LSM |

**Operational consequence to write into the design:** if you choose an LSM engine, your
monitoring must include compaction backlog and space amplification. If you choose a B-tree,
your monitoring must include page cache hit rate and WAL/checkpoint behavior.

---

## 3. Other index structures

- **Secondary indexes:** keys are not unique; entry points to a list of matching rows, or
  the row ID is appended to the key.
- **Clustered index:** the row is stored inside the index (primary key index in InnoDB).
  Fast reads by PK; a large PK makes every secondary index bigger.
- **Multi-column / concatenated indexes:** left-most prefix rule applies.
- **Multi-dimensional (R-tree)** for geospatial; standard B-trees cannot answer
  "within this bounding box" efficiently with two separate single-column indexes.
- **Full-text / fuzzy:** Lucene-style term dictionaries with edit-distance automata.
- **In-memory stores** (Redis, Memcached, VoltDB): fast not primarily because they avoid
  disk reads (the OS page cache already caches hot data) but because they avoid the
  overhead of encoding data into a disk-writable form, and can offer data models that are
  awkward on disk (sets, sorted sets, priority queues).

---

## 4. OLTP vs. OLAP — a hard boundary

| | OLTP | OLAP |
|---|---|---|
| Read pattern | Small number of records per query, by key | Aggregate over huge numbers of records |
| Write pattern | Random-access, low-latency, user input | Bulk import (ETL) or event stream |
| Used by | End users, via the application | Analysts, dashboards |
| Data size | GB–TB | TB–PB |
| Bottleneck | Disk seek / lock contention | Disk bandwidth / scan throughput |

**The rule:** do not run analytical queries against the transactional store. A single
unbounded analytical scan can evict the OLTP working set from the page cache, hold
long-lived MVCC snapshots that bloat the database, and blow the p99 for user traffic.
Derive a separate analytical copy (ETL, CDC → warehouse; see `11-stream-processing.md`).

**Star schema (dimensional modeling):** a central fact table (one row per event) with
foreign keys to dimension tables (who, what, where, when, how, why). Snowflake schema
further normalizes dimensions. Fact tables get very wide (100+ columns) but queries
typically touch four or five.

---

## 5. Column-oriented storage

Store all values of one column together instead of all values of one row together. Because
analytical queries touch a handful of columns out of a hundred, this is a large win.

- **Column compression:** bitmap encoding with run-length encoding works extremely well on
  low-cardinality columns and turns `WHERE x IN (...)` into cheap bitwise OR/AND.
- **Sort order:** choose a sort key (e.g. date, then product) to aid range filtering and to
  improve compression of the first sort column dramatically. Different replicas can be
  sorted differently and the query picks the best.
- **Vectorized processing** and operating on compressed data directly keep the CPU cache
  hot — often the actual bottleneck once I/O is solved.
- **Writes are hard:** in-place update of a compressed sorted column is impractical.
  LSM-style staging (in-memory store → merged into columnar files) is the standard answer,
  which means a **write→visible-in-analytics delay** you should state explicitly.
- **Materialized views / data cubes** trade write cost and flexibility for read speed. A
  materialized aggregate is derived data and must have a defined refresh/invalidation path.

---

## 6. Review checklist

- [ ] Every added index justified by a query plan; unused indexes identified for removal
- [ ] Write amplification of the index set considered on write-heavy tables
- [ ] Storage engine choice matches read/write ratio, and its characteristic monitoring
      (compaction backlog for LSM, cache hit rate for B-tree) is in place
- [ ] No analytical/reporting queries running against the OLTP store
- [ ] Long-running read transactions bounded (they hold MVCC snapshots)
- [ ] Range/prefix queries have an index whose column order actually supports them
- [ ] Geospatial or full-text needs use an appropriate index type, not a scan
- [ ] Materialized views and cached aggregates have a defined refresh and invalidation path
- [ ] For columnar/analytical stores: the visibility delay from write to analytics is stated
