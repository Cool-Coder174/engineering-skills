# How to Select a Database

**Sources:** *Database Internals* (Alex Petrov), for the comparison method, the DBMS
architecture, the B-Tree and LSM trade-offs, and the RUM Conjecture. *Grokking the System
Design Interview*, for SQL compared to NoSQL. *System Design Interview*, Chapter 1 (Xu).

Petrov states the reason to take care:

> "Your choice of database system may have long-term consequences. If there's a chance that a
> database is not a good fit because of performance problems, consistency issues, or
> operational challenges, it is better to find out about it earlier in the development cycle,
> since it can be nontrivial to migrate to a different system."

This decision is one of the hardest to reverse.

---

## 1. How not to select a database

Petrov names three methods that fail:

1. **Comparison by component.** Which storage engine does it use? How does it partition and
   replicate?
2. **Comparison by rank.** DB-Engines, popularity lists, and consultancy reports.
3. **Comparison by implementation language.** C++, Java, or Go.

All three "can lead to invalid and premature conclusions". They are coarse signals. They help
only for a comparison as broad as HBase against SQLite.

Four more failures occur inside organizations:

4. **Selection by career interest.** Someone wants to learn the technology.
5. **Selection by anecdote.** "Company X runs it at scale." That company has a different
   scale, a different workload, and an infrastructure team that you do not have.
6. **Selection by vendor benchmark.** A vendor benchmark measures the vendor's workload.
7. **Selection by data shape alone.** The access patterns select the database. The entity
   diagram does not.

**Start every comparison with a stated goal.** A small bias invalidates the whole
investigation.

---

## 2. The evaluation method

### Step 1 — Record the current and expected values

Petrov's list:

- Schema and record sizes
- Number of clients
- Types of queries and access patterns
- Rates of read and write queries
- **Expected changes in any of these values**

The last item matters most, and teams skip it most often. A database that fits today's
numbers and cannot absorb 10 times the load is a migration that you scheduled without saying
so.

### Step 2 — Answer these questions with those values

- Does the database support the queries that we need?
- Can it hold the volume of data that we plan to store?
- How many reads and writes can one node handle?
- How many nodes does the system need?
- How do we grow the cluster at the expected rate?
- What is the maintenance process?

### Step 3 — Run your own workload

Petrov's instruction:

> "The best thing you can do is to simulate these workloads against different database
> systems, measure the performance metrics that are important for you, and compare results."

Two points decide whether this test is useful.

**Run the test for a long time, on a production-like system.** Some problems appear only
after time passes or after the data grows. A five-minute test against an empty database tells
you nothing about compaction, index growth, vacuum pressure, or garbage collection pauses.
Those are the real problems.

**The test also teaches you to operate and debug the database.** It shows you how helpful the
community is. That knowledge is an output of the test, not a side effect. Most databases ship
a stress tool that you can adapt.

### Step 4 — Weigh operability with performance

Petrov states the priority:

> "Performance often turns out **not** to be the most important aspect: it's usually much
> better to use a database that slowly saves the data than one that quickly loses it."

Weigh these items as well:

- The operational knowledge in your team.
- The backup and restore procedure, with **a restore that you tested**.
- The upgrade path.
- The monitoring support.
- The documentation and the community.
- The license.

**A database that nobody on the team can debug during an incident is the wrong database.**
The benchmark does not change that result.

---

## 3. The categories

| Category | Model | Strong at | Weak at | Select it when |
|---|---|---|---|---|
| **Relational (OLTP)** | Tables, rows, SQL, ACID | Transactions across entities. Ad-hoc queries. Constraints. Joins. | Very high write volume on one node. Rigid schema changes. | **This is the default.** The data is structured. You need transactions or strong consistency. The query patterns are not yet fixed. |
| **Document** | JSON-like documents | Schemas that vary and change. Reads of a whole document. | Joins across documents. Consistency across documents. | Each record is complete on its own. The fields vary for each record. |
| **Key-value** | Values addressed by key | Very high throughput. Very low latency. Simple scaling. | Any access that is not by primary key. | Session stores, caches, feature flags, and simple lookups at high volume |
| **Wide-column** | Partition key with clustering columns | Very high write volume. Time series. Queries inside one partition. | Ad-hoc queries. Joins. Transactions across partitions. | The write rate exceeds a relational database, **and** the access patterns are fixed and match the partition key |
| **Graph** | Nodes and edges | Queries that follow many relationships | Bulk analytics. High write volume. | The query follows more than 2 or 3 relationships, and that query is the main one |
| **Search index** | Inverted index | Full-text search, facets, and ranking | Holding the authoritative data | Search is a main feature. Use it as a **derived** store, never as the authoritative one. |
| **Column-oriented (OLAP)** | Columnar storage | Aggregation over large scans. Compression. | Single-row reads. Frequent single-row updates. | Analytics and reporting, separated from the transactional system |
| **Object storage** | Files addressed by key | Cheap. Very large. Durable. | Queries of any kind | Media, backups, and large files. **Store the file here. Store the reference in the database.** |
| **Time-series** | Timestamped points | Queries over time ranges. Downsampling. Retention rules. | General workloads | Metrics and telemetry with a fixed retention period |

**Petrov's three system types.** OLTP systems handle many short transactions for users, with
queries that are mostly known in advance. OLAP systems handle complex aggregations for
analytics and reporting, with long ad-hoc queries. HTAP systems combine both.

**Do not run OLTP and OLAP work on one instance.** This is the most frequent version of the
mistake. The long analytical query that locks a user-facing table is a recurring incident
with a known fix.

---

## 4. SQL compared to NoSQL

**Select a relational database when:**

- You need ACID transactions. Money, inventory, and permissions all need them.
- The data is structured, and the relationships matter.
- You need ad-hoc queries, because you do not yet know the access patterns.
- You need the database to enforce constraints, rather than every client.

**Select a non-relational database when** (Xu's list): you need very low latency, the data is
unstructured, you only serialize and deserialize records, or you store an enormous volume.

**Three corrections to the usual comparison:**

1. **"NoSQL scales and SQL does not" is false as stated.** A partitioned relational database
   handles very large volumes. One modern Postgres or MySQL primary handles far more than
   most teams assume. Estimate the load with `02-estimation.md` before you decide that you
   have exceeded it.

2. **Name the real trade.** You exchange transactions and flexible queries for write
   throughput and simpler operation at scale. Check that trade against your requirements.

3. **Using several databases is normal, and each one costs.** A typical set has four parts:
   - A relational database for the authoritative data.
   - Object storage for files.
   - A search index for search.
   - A cache for hot reads.

   Each extra store creates a synchronization obligation. **If the application writes to two
   stores directly, that is a dual write. It is hazard H-32, and the two stores will
   diverge.** Derive the second store from the change log of the first one.

---

## 5. Storage engine internals that change a decision

A database is an application built on a storage engine. The storage engine decides what the
database is good at.

Engines such as BerkeleyDB, LevelDB, RocksDB, LMDB, and WiredTiger were built separately from
the databases that now contain them. MySQL can run InnoDB, MyISAM, or RocksDB. MongoDB has
run WiredTiger, In-Memory, and MMAPv1.

### B-Tree compared to LSM Tree

| Property | B-Tree | LSM Tree |
|---|---|---|
| Writes | In place. Find the page, then update it, sometimes more than once. | Append to a memory table. Flush it to sorted files. |
| Reads | Optimized. One structure to traverse. | Reads several files. Bloom filters and compaction reduce the cost. |
| Write amplification | From writeback and repeated updates to one page | From compaction, which rewrites data between files |
| Space | Reserved space for future updates and deletes | Duplicate records remain until compaction removes them |
| Databases that use it | Postgres, MySQL with InnoDB, most relational systems | Cassandra, RocksDB, LevelDB, HBase, ScyllaDB |

**Petrov's warning: the two kinds of write amplification have different causes. If you compare
the raw numbers directly, you reach an incorrect conclusion.**

Immutable, log-structured storage has three specific problems:

1. **Read amplification.** A read must address several files.
2. **Write amplification.** Compaction rewrites data continuously.
3. **Space amplification.** Several records for one key remain for a period.

### The RUM Conjecture

The RUM Conjecture is a cost model with three overheads: **R**ead, **U**pdate, and **M**emory.

**The conjecture states that reducing two of these overheads makes the third one worse. An
optimization always costs one of the three.**

- **B-Trees** optimize reads. They pay in write cost and space.
- **LSM Trees** optimize writes. They pay in read cost. Bloom filters, compaction strategies,
  and caches reduce that cost.

The model excludes several important factors: latency, access patterns, implementation
complexity, maintenance work, and hardware. For distributed databases, it also excludes
consistency and replication cost. **Use it as a first approximation, not as a verdict.**

**Why this matters during design.** Suppose the workload is mostly writes, with reads over key
ranges. Then an LSM engine matches it. Suppose the workload is mostly reads, with ad-hoc
queries and single-row lookups. Then a B-Tree engine matches it. If you select against the
workload, you fight the storage engine forever. No amount of tuning corrects a structural
mismatch.

### Two more properties

- **Memory compared to disk.** An in-memory database is much faster. RAM and the durability
  method limit it. "In-memory with persistence" still has a window in which it can lose data.
  Learn the exact size of that window before you store money in it.
- **Rows compared to columns.** Row storage suits reads of whole records, which is OLTP work.
  Column storage suits scans of a few columns across many rows, which is OLAP work. Column
  storage also compresses much better.

---

## 6. Record the decision

Write this record. In six months, nobody will remember the reasons. The reasons are what
permit a future engineer to revisit the decision correctly.

```md
### Database decision: [component]

**Access patterns** (the most frequent ones)
| Pattern | Frequency | Latency need | Consistency need |
|---|---|---|---|

**Values**
- Record size: [ ]
- Total volume now, after 1 year, after 3 years: [ ]
- Read QPS and write QPS, average and peak: [ ]
- Number of clients: [ ]
- Expected change in the values above: [ ]

**We chose:** [database, version, deployment model]

**Why:** [connect the reason to the access patterns and values above, not to preference]

**We rejected:**
| Alternative | Why we rejected it |
|---|---|

**Evidence:** [a load test, production data, or "no evidence. This is an assumption."]

**Operations:** who operates it. The backup and restore procedure. **The date of the last
tested restore.** The upgrade path. The experience in the team.

**Growth plan:** what we do when [value] reaches [number]

**Exit cost:** how hard is a migration away from this database. What makes it harder over
time.
```

---

## 7. Faults to find in a database decision

| Fault | Why it is a problem |
|---|---|
| The team chose before it recorded the access patterns | The access patterns select the database. Any other basis is a guess. |
| The reason is "it scales better", with no numbers | Estimate the load. The current database often has years of capacity. |
| The benchmark ran for 5 minutes on an empty database | Compaction, index growth, vacuum, and garbage collection appear only at volume and over time |
| Nobody on the team has operated it in production | The first incident becomes the training exercise |
| The team chose for a scale that nobody requested | This is a prediction. It is the most frequent and most damaging design error. |
| Files are stored in the database | Object storage costs far less and keeps the database small |
| A search index or a cache holds the authoritative data | A derived store must be rebuildable. If you cannot rebuild it, it is not derived. |
| The application writes to several stores directly | This is a dual write (hazard H-32). Derive the second store from a change log. |
| Analytical queries run on the transactional database | The long scan locks a user-facing table |
| Nobody has tested a restore | You have backups. You do not have recovery. |
