# Choosing a Database

**Sources:** *Database Internals* (Alex Petrov) — "Comparing Databases", DBMS architecture,
B-Tree vs. LSM trade-offs, RUM Conjecture; *Grokking the System Design Interview* —
SQL vs. NoSQL; *System Design Interview* Ch. 1 (Xu).

> "Your choice of database system may have long-term consequences. If there's a chance that
> a database is not a good fit because of performance problems, consistency issues, or
> operational challenges, it is better to find out about it earlier in the development
> cycle, since it can be nontrivial to migrate to a different system."

This is one of the least reversible decisions in a design. Treat it accordingly.

---

## 1. How NOT to choose

Petrov is explicit about the failure modes:

Comparing databases by **their components** (which storage engine, how data is sharded or
replicated), by **their rank** (DB-Engines, consultancy popularity lists), or by
**implementation language** (C++ vs. Java vs. Go) "can lead to invalid and premature
conclusions." These are coarse signals, useful only for distinguishing something as
different as HBase from SQLite.

Add the common organizational failure modes:

- **Choosing by résumé.** Picking the technology someone wants to learn.
- **Choosing by anecdote.** "Company X uses it at scale" — at a scale and workload you do
  not have, with an infrastructure team you do not have.
- **Choosing by benchmark blog post.** Vendor benchmarks measure the vendor's workload.
- **Choosing by the shape of the data alone.** Access patterns select the store; the entity
  diagram does not.

**Every comparison should start by clearly defining the goal**, because even a slight bias
invalidates the entire investigation.

---

## 2. The evaluation method

### Step 1 — Define the current and anticipated variables

Petrov's list:

- **Schema and record sizes**
- **Number of clients**
- **Types of queries and access patterns**
- **Rates of read and write queries**
- **Expected changes in any of these variables**

That last item matters most and is most often skipped. A store that fits today's numbers
and cannot absorb 10× growth is a migration you have scheduled without admitting it.

### Step 2 — Answer these questions with those variables

- Does the database **support the required queries**?
- Can it **handle the amount of data** we plan to store?
- **How many read and write operations can a single node handle?**
- **How many nodes should the system have?**
- **How do we expand the cluster** given the expected growth rate?
- **What is the maintenance process?**

### Step 3 — Simulate the actual workload

"The best thing you can do is to simulate these workloads against different database
systems, measure the performance metrics that are important for you, and compare results."

Two points that make or break this:

- **Some issues only appear over time or as capacity grows.** Run long tests in an
  environment simulating production as closely as possible. A benchmark that runs for five
  minutes against an empty database tells you nothing about compaction behavior, index
  bloat, vacuum pressure, or GC pauses — which is where the real problems are.
- **Simulating real workloads also teaches you to operate, debug, and evaluate the
  community.** That is a deliverable of the exercise, not a side effect. Most stress tools
  shipped with databases can be adapted to your workload.

### Step 4 — Weigh operability alongside performance

> "Performance often turns out **not** to be the most important aspect: it's usually much
> better to use a database that slowly saves the data than one that quickly loses it."

Also weigh: your team's existing operational knowledge, backup and restore procedures
(**tested restores**, not backups), upgrade paths, observability, documentation quality,
community responsiveness, and licensing.

**A database nobody on the team can debug during an incident is the wrong database**,
whatever the benchmark says.

---

## 3. Categories and what they are for

| Category | Model | Strong at | Weak at | Choose when |
|---|---|---|---|---|
| **Relational (OLTP)** | Tables, rows, SQL, ACID | Multi-entity transactions, ad-hoc queries, integrity constraints, joins | Extreme write volume on a single node; rigid schema migrations | **The default.** Data is structured and relational, you need transactions or strong consistency, or query patterns are not yet stable |
| **Document** | JSON-like documents | Flexible/evolving schemas, whole-document reads, aggregate storage | Cross-document joins; consistency across documents | Data is naturally a self-contained aggregate; schema varies per record |
| **Key-value** | Opaque values by key | Very high throughput, very low latency, trivial scaling | Any access not by primary key | Session stores, caches, feature flags, simple lookups at huge volume |
| **Wide-column** | Partition key + clustering columns | Massive write volume, time-series, predictable partition-scoped queries | Ad-hoc queries, joins, multi-partition transactions | Write rate exceeds relational capacity **and** access patterns are known, fixed, and partition-aligned |
| **Graph** | Nodes and edges | Multi-hop traversal, relationship queries | Bulk analytics, high write throughput | Traversal depth exceeds 2–3 joins and is the primary query |
| **Search index** | Inverted index | Full-text, faceting, relevance ranking | Being a system of record | Search is a first-class feature — as a **derived** store, never the source of truth |
| **Column-oriented (OLAP)** | Columnar | Aggregations over huge scans, compression | Point reads, high-frequency single-row updates | Analytics and warehousing, separated from the OLTP path |
| **Object storage** | Blobs by key | Cheap, effectively unbounded, durable | Queries of any kind | Media, backups, large files. **Store the blob here and the reference in the database** |
| **Time-series** | Timestamped points | Time-range queries, downsampling, retention policies | General-purpose workloads | Metrics and telemetry with defined retention |

**OLTP vs. OLAP vs. HTAP** (Petrov's grouping): OLTP handles many short, predefined,
user-facing transactions; OLAP handles complex, long-running aggregations for analytics and
warehousing; HTAP combines both. **Mixing OLTP and OLAP on one instance is the most common
version of this mistake** — the analytical query that locks the user-facing table is a
recurring incident with a well-known fix.

---

## 4. SQL vs. NoSQL, stated honestly

**Reasons to use a relational database:**
- You need **ACID transactions** — anything touching money, inventory, or entitlements
- Data is **structured and unchanging**, with relationships that matter
- You need **ad-hoc queries** and do not yet know the access patterns
- You need **constraints enforced by the store** rather than by every client

**Reasons to use a non-relational database** (Xu's list): super-low latency requirements,
unstructured or non-relational data, a need only to serialize and deserialize data, or a
need to store a massive amount of data.

**Three corrections to the usual framing:**

1. **"NoSQL scales, SQL doesn't" is false as stated.** A well-partitioned relational
   database handles enormous volume, and the practical ceiling for a single modern
   Postgres/MySQL primary is far higher than most teams assume. Do the estimation
   (`02-estimation.md`) before concluding you have outgrown it.
2. **The real trade is transactions and query flexibility for write throughput and
   operational simplicity at scale.** Name that trade explicitly and check it against your
   actual requirements.
3. **Polyglot persistence is normal, and each store is a cost.** Relational as the system of
   record, object storage for blobs, a search index for search, a cache for hot reads. Each
   additional store adds a synchronization obligation — and if it is written directly
   alongside the primary, that is a **dual write** (hazard H-32) and it will diverge.
   Derive secondary stores from the primary's change log instead.

---

## 5. Storage engine internals that change design decisions

A database is an application built on a storage engine; the storage engine defines what the
database is actually good at. Engines like BerkeleyDB, LevelDB/RocksDB, LMDB, and WiredTiger
were developed independently of the systems that embed them — MySQL can run InnoDB, MyISAM,
or RocksDB; MongoDB has run WiredTiger, In-Memory, and MMAPv1.

### B-Tree vs. LSM Tree

| | B-Tree | LSM Tree |
|---|---|---|
| Writes | In-place; locate the page, then update, possibly repeatedly | Sequential appends to a memtable, flushed to sorted files |
| Reads | Read-optimized — a single structure to traverse | Must consult multiple tables; mitigated by Bloom filters and compaction |
| Write amplification | From writeback and repeated updates to the same page | From compaction rewriting data between files |
| Space | Extra reserved space for future updates/deletes | Redundant records retained until compaction |
| Used by | Postgres, MySQL/InnoDB, most relational systems | Cassandra, RocksDB, LevelDB, HBase, ScyllaDB |

Petrov's warning: **the sources of write amplification differ between the two, so comparing
the raw numbers directly leads to incorrect conclusions.**

Immutable, log-structured storage faces three specific problems: **read amplification**
(addressing multiple tables to retrieve data), **write amplification** (continuous rewrites
during compaction), and **space amplification** (multiple records per key preserved for a
time).

### The RUM Conjecture

A cost model over three overheads — **R**ead, **U**pdate, **M**emory. It states that
**reducing two of these inevitably worsens the third**; optimizations come only at the
expense of one of the three.

- **B-Trees** are read-optimized, paying in write and space overhead.
- **LSM Trees** are write-optimized, paying in read cost (mitigated by Bloom filters,
  compaction strategies, and caching).

The model deliberately excludes latency, access patterns, implementation complexity,
maintenance overhead, hardware specifics, and — for distributed systems — consistency and
replication overhead. **Use it as a first approximation, not a verdict.**

**Why this matters at design time:** if your workload is write-dominated with mostly
key-range reads, an LSM engine matches it. If it is read-dominated with ad-hoc queries and
point lookups, a B-Tree engine matches it. Choosing against the grain means fighting the
storage engine forever, and no amount of tuning fixes a structural mismatch.

### Other axes

- **Memory- vs. disk-based:** in-memory stores are dramatically faster and bounded by RAM
  and durability strategy. "In-memory with persistence" still has a durability window —
  know exactly what it is before putting money in it.
- **Row- vs. column-oriented:** row layout suits fetching whole records (OLTP); column
  layout suits scanning few columns across many rows (OLAP), and compresses far better.

---

## 6. The decision record

Write this down. Six months from now nobody will remember why, and the reasoning is what
lets a future engineer revisit the decision correctly.

```md
### Datastore Decision: [component]

**Access patterns** (top N by frequency)
| Pattern | Frequency | Latency need | Consistency need |
|---|---|---|---|

**Variables**
- Record size: [ ]      - Total volume now / at 1 yr / at 3 yr: [ ]
- Read QPS / Write QPS (avg, peak): [ ]
- Clients: [ ]          - Expected change in the above: [ ]

**Chosen:** [store] — [version, deployment model]

**Why:** [tied to the access patterns and variables above, not to preference]

**Rejected:**
| Alternative | Why not |
|---|---|

**Verified by:** [load test, existing production evidence, or "unverified — assumption"]

**Operability:** who operates it, backup/restore procedure, **last tested restore**,
upgrade path, existing team experience

**Scaling path:** what we do when [variable] reaches [value]

**Exit cost:** how hard is migrating off this — and what makes it harder over time
```

---

## 7. Red flags in a database choice

| Red flag | Why it's a problem |
|---|---|
| Chosen before access patterns were written down | The patterns select the store; anything else is a guess |
| "It scales better" with no numbers | Do the estimation; the current store often has years of headroom |
| Benchmarked for 5 minutes on an empty dataset | Compaction, bloat, vacuum, and GC problems only appear at volume and over time |
| Nobody on the team has operated it in production | The incident will be your training exercise |
| Chosen for a scale nobody has requested | Designing for an unknown future — the most common and disastrous design error |
| Blobs stored in the database | Object storage is orders of magnitude cheaper and keeps the database small |
| Search index or cache treated as a system of record | Derived stores are rebuildable by definition; if it cannot be rebuilt, it is not derived |
| Multiple stores written directly by the application | Dual write (H-32). Derive from a change log instead |
| Analytical queries on the OLTP primary | The long scan that locks the user-facing table |
| No tested restore | You have backups; you do not have recovery |
