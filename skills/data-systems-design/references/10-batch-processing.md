# Batch Processing

**Source:** DDIA Ch. 10, p. 389.

Batch processing means bounded input, no user waiting, and — the property that matters
most — **the ability to re-run**. Most of the value in this chapter is a design discipline
that applies to any job, script, backfill, or cron task, not just Hadoop.

---

## 1. Three kinds of system

| Type | Input | Output | Primary measure |
|---|---|---|---|
| **Services (online)** | Requests | Responses | Response time |
| **Batch (offline)** | Bounded, known-size dataset | Derived dataset | Throughput |
| **Stream (near-real-time)** | Unbounded event stream | Derived output | Latency + throughput |

---

## 2. The Unix philosophy, and why it still applies

Unix tools compose because of four properties, and they are exactly the properties a good
job should have:

1. **Each program does one thing well.**
2. **Output of one is input to another** — a uniform interface (a file of lines).
3. **Design and build to be tried early** — iterate quickly.
4. **Automate; prefer tools over unskilled repetitive help.**

Two consequences worth internalizing:

- **Separation of logic and wiring.** A program that reads stdin and writes stdout can be
  redirected anywhere. A program that hardcodes its input and output paths cannot be
  tested, reused, or re-run in a different context. This is the same principle as dependency
  injection, and it is the single most useful thing to copy into application jobs.
- **Immutable input + no side effects (except the output).** Because input files are never
  modified, you can re-run with different arguments as many times as you like. This is what
  makes experimentation and recovery cheap.

---

## 3. MapReduce and distributed filesystems

MapReduce is "Unix tools at scale": read files from a distributed filesystem (HDFS/S3),
map (extract key-value pairs), sort by key (the expensive part, distributed across the
shuffle), reduce (iterate over values sharing a key).

Details that matter even if you never write MapReduce:

- **Putting the computation near the data** avoids moving large datasets over the network.
- **Skew / hot keys** ("linchpin objects") make one reducer far slower than the rest, and
  the job finishes when the slowest task finishes. Mitigations: sample first and spread hot
  keys across multiple reducers, or handle them separately. **This is the same hot-key
  problem as in `06-partitioning.md`, and it dominates job runtime.**
- **Reduce-side joins (sort-merge)** work for any input but require the shuffle.
  **Map-side joins** (broadcast hash join for a small side; partitioned hash join when both
  sides are partitioned the same way) avoid the shuffle and are much faster — but require
  assumptions about the input that you must state.
- **Fault tolerance by re-running tasks** relies on the code being **deterministic**.
  Nondeterminism (iterating a hash map, using random numbers, reading the current time,
  calling an external service) breaks it in subtle ways: a re-run produces different output
  than the original, and downstream consumers may have seen both.

---

## 4. The output of batch workflows, and why immutability wins

The critical design property: **the job writes its output to a new location and does not
modify its input.** Consequences you get for free:

- **Recovery from bugs is simply re-running the job.** Fix the code, re-run over the
  unchanged input, replace the output. This is "human fault tolerance" — the ability to
  undo the effects of a bad deploy, and it is what allows fast, low-risk iteration.
- Output is switched over **atomically** (build the new index/table, then flip a pointer).
  Readers see either the entire old version or the entire new one.
- **Rollback is trivial:** flip the pointer back.
- Reasoning is easy because there is no partially-applied state.

**Anti-pattern this replaces:** jobs that mutate rows in place, incrementally, with no
record of what they changed. When such a job has a bug, there is no way back.

**Practical rule for any backfill or data-fixing script:**
1. Read from immutable input (or a snapshot).
2. Write to a **new** table/prefix/version.
3. Verify (counts, checksums, spot checks, a diff against the old output).
4. Atomically switch readers.
5. Keep the old output for a defined retention period so rollback is possible.

---

## 5. Comparing Hadoop-style to distributed databases

- **Diversity of storage:** dump data indiscriminately and interpret later ("sushi
  principle: raw data is better") vs. careful modeling up front. Dumping shifts the burden
  to the consumer, but makes data available at all — often the right first step.
- **Diversity of processing:** general-purpose code beats SQL-only for machine learning,
  recommendation, and format conversion.
- **Frequent faults are designed for:** MapReduce tolerates task failure and restarts at
  task granularity, and is willing to write intermediate state to disk. It was designed for
  an environment with **preemptible/spot resources and overcommitted clusters**, where
  tasks get killed routinely. If you run on spot instances, this design is directly relevant
  to you; if you run on stable resources, keeping everything in memory (Spark, Flink, Tez)
  is faster.

---

## 6. Beyond MapReduce

- **Materializing intermediate state to files** (MapReduce) vs. **pipelining between
  operators** (Spark/Tez/Flink dataflow engines): pipelining avoids writing intermediate
  results to disk, avoids unnecessary sorts, and starts operators as soon as input is
  available. Cost: on failure, more work must be recomputed — mitigated by keeping the
  lineage and recomputing deterministically, which again **requires determinism**.
- **Graph/iterative processing** (Pregel/bulk synchronous parallel): a vertex sends
  messages to other vertices each iteration; the framework handles fault tolerance and
  message batching. Application in ordinary systems: any iterate-until-fixpoint computation
  (transitive permissions, ranking, dependency resolution).
- **High-level APIs** (Hive, Spark SQL, DataFrames, declarative operators) let the engine
  choose join strategies and use vectorized/compiled execution. Prefer them; drop to
  hand-written map/reduce only where necessary.

---

## 7. Applying this to ordinary application jobs

Even a 40-line cron script should have these properties:

| Property | Why |
|---|---|
| **Idempotent** | Re-running after a partial failure must not double-apply |
| **Resumable** | Track a checkpoint/cursor; don't restart a 6-hour job from zero |
| **Batched with bounded memory** | No `SELECT *` of a table into a list |
| **Rate-limited** | A backfill must not saturate the database that serves users |
| **Observable** | Progress, rate, errors, ETA — otherwise you cannot tell hung from slow |
| **Deterministic** | Same input → same output, so re-runs are safe |
| **Non-destructive** | Write new output; keep the old for rollback |
| **Singly-scheduled** | A lock or lease prevents two overlapping runs (a slow run overlapping the next tick is the classic cron bug) |
| **Bounded runtime** | With a kill/alert threshold |

---

## 8. Review checklist

- [ ] Job input is immutable or a snapshot; the job does not mutate its own input
- [ ] Output written to a new location, then atomically switched; old output retained for rollback
- [ ] Job is deterministic (no ambient time, randomness, map iteration order, or external calls
      affecting output) — or nondeterminism is isolated and recorded
- [ ] Job is idempotent and safe to re-run after partial failure
- [ ] Checkpoint/cursor allows resume; progress is logged
- [ ] Processing is batched with bounded memory and a bounded transaction size
- [ ] Rate limiting protects the online datastore from the batch workload
- [ ] Overlap protection (lock/lease) prevents concurrent runs of the same job
- [ ] Skew/hot keys identified; one slow partition does not define the job's runtime
- [ ] Verification step (counts, checksums, diff vs. previous output) before switchover
- [ ] Failure alerting exists — silent failure of a nightly job is the default otherwise
- [ ] Logic separated from wiring (paths/connections injected, not hardcoded)
