# The Design Method

**Sources:** *System Design Interview* Ch. 3 (Xu); *Grokking the System Design Interview*,
"System Design Interviews: A step by step guide".

The two books describe the same process at different granularity. Xu gives four steps with
time allocation; Grokking gives seven. Merged below into seven steps, adapted from the
interview context to real engineering work.

---

## Why a method at all

Design problems are open-ended, ambiguous, and broadly scoped, with no single correct
answer. Without a method, two failure modes dominate:

1. **Answering too fast.** Jumping to a solution before understanding the requirements
   produces a well-built version of the wrong system. In an interview this is the single
   biggest red flag; in real work it is the single biggest source of wasted quarters.
2. **Never converging.** Exploring everything at equal depth, running out of time, and
   producing a diagram with no decisions in it.

The method exists to force scope before solution, and depth where depth matters.

---

## Step 1 — Clarify requirements and establish scope

**Rule: always ask for clarification. Never assume your assumption is correct.**

When you cannot ask (async work, no stakeholder available), write assumptions down
explicitly and label them, so a reviewer can attack the assumption rather than discovering
it three weeks later embedded in the schema.

### The question bank

**Product**
- What specific features are we building? What are we explicitly *not* building?
- Which features are core vs. nice-to-have?
- Web, mobile, or both? Third-party API consumers?
- Is there an offline requirement?

**Scale**
- How many users total? How many daily active?
- What growth is anticipated at 3 months, 6 months, 12 months?
- Read-heavy or write-heavy, and at what ratio?
- What does peak look like versus average? Is load diurnal, bursty, or event-driven?

**Data**
- What entities exist and how big is each?
- What is the retention period?
- Does it include media (images, video), and what fraction?
- Any PII, residency, or deletion obligations?

**Behavior**
- Ordering: chronological, ranked, or something else? (This one question can change the
  entire architecture — ranked feeds need a scoring pipeline.)
- Search required? Notifications? Real-time updates?
- What is the latency expectation, for which operations?
- Can users tolerate stale data, and for how long?

**Context**
- What is the existing technology stack? What can we reuse?
- What is the team size and operational maturity?
- What is the deadline, and what is the budget?

**The stack question is the most underrated.** In real work, "we already run Postgres and
nobody here has operated Cassandra" is a stronger constraint than any theoretical
comparison, and a design that ignores it will not survive contact with the on-call rotation.

### Output
Functional requirements, non-functional requirements **with numbers**, explicit non-goals,
and a labeled assumptions list.

---

## Step 2 — Define the interface

Define the APIs the system exposes. This establishes the exact contract and — more
valuable — **verifies you understood the requirements**. If you cannot write the endpoint
signature, the requirement is still ambiguous.

For each endpoint or event:
- Method, path, parameters, response shape
- Authentication and authorization
- Idempotency: is it required, and what is the key?
- Pagination style (offset vs. cursor/keyset)
- Rate limits
- Error cases and status codes

For event-driven paths: event schema, partition key, ordering guarantee, delivery
semantics (at-least-once + idempotent consumer is the normal answer).

Whether to include API detail at this stage depends on the problem's altitude. For
"design a search engine" it is too low-level to start here; for "design the backend of a
multiplayer game" it is exactly right. Match the altitude to the question.

---

## Step 3 — Back-of-the-envelope estimation

> "Back-of-the-envelope calculations are estimates you create using a combination of
> thought experiments and common performance numbers to get a good feel for which designs
> will meet your requirements." — Jeff Dean

**The process matters more than the result.** The purpose is to know whether you need one
server or a hundred, one database or a sharded fleet — and to have the ammunition to reject
an over-engineered proposal.

Rules:
- **Round aggressively.** "99,987 / 9.1" becomes "100,000 / 10". Precision is not expected
  and pursuing it wastes the time you need for design.
- **Write down assumptions** so they can be referenced and challenged.
- **Label every unit.** "5" is meaningless; "5 MB" is not.
- Estimate at minimum: QPS, peak QPS, storage, bandwidth, cache size, server count.

Formulas and constants: `02-estimation.md`.

---

## Step 4 — Define the data model

Identify the entities, their relationships, and how they interact. Defining the data model
early clarifies how data flows between components, and it directly drives partitioning and
caching decisions later.

For each entity: fields, approximate size, write rate, read rate, and retention.

Then enumerate **the top ~10 access patterns** by frequency and by latency sensitivity.
The access patterns determine the store — not the entity diagram, and not what the team
used last time.

Then choose: relational, document, wide-column, key-value, graph, blob storage, search
index. Selection methodology in `05-database-selection.md`.

---

## Step 5 — High-level design and agreement

Draw a block diagram with roughly **five to eight boxes** representing the core components,
enough to solve the problem end to end. Typical components: clients, DNS, CDN, load
balancer, web/app tier, cache, database, message queue, workers, object storage, derived
stores.

Then:
- **Walk concrete use cases end to end.** At minimum the primary write path and the
  primary read path. This is where the edge cases you have not considered surface.
- **Check the blueprint against the Step 3 numbers.** If the estimate says 200 QPS, a
  design with a Kafka cluster and twelve services needs to justify itself.
- **Get agreement before going deeper.** In an interview this means checking in with the
  interviewer; in real work it means a design review before the deep dive. Deep-diving a
  high-level design nobody agrees with is wasted effort either way.

---

## Step 6 — Deep dive

Identify and prioritize the components that deserve detail. **Do not distribute attention
evenly.** Choose targets by asking which component has:

- the highest QPS or the largest data volume,
- the hardest consistency or latency requirement,
- the worst hot-key or skew problem,
- the least reversible decision,
- the most novelty (nobody on the team has built one before).

For each target, go into: algorithm/data structure, partitioning, replication, concurrency
control, cache strategy, failure behavior, and the trade-off accepted.

**Time management matters here.** It is easy to get lost in detail that demonstrates
nothing — tuning a ranking algorithm's weights when the question is whether the feed
architecture scales. Depth is valuable only where the risk is.

Run the hazard scan from `data-systems-design/references/hazard-catalog.md` on each deep
dive. Design decisions become correctness bugs at exactly this level of detail.

---

## Step 7 — Bottlenecks, failure, and operations

Cover, explicitly:

- **Bottlenecks:** which component saturates first, at what value, with what symptom.
- **Single points of failure:** what is not redundant, and what happens when it dies.
- **Replication and redundancy:** enough copies of data *and* of services to survive the
  failures you claimed to tolerate.
- **Failure behavior:** for each dependency, what the system does when it is down or slow.
  Degrade, shed, queue, or fail — pick one per dependency and say so.
- **Monitoring:** which metrics and logs would catch each failure above, with thresholds
  and alerts. A failure mode with no corresponding signal is undetected by definition.
- **Rollout:** flags, canaries, percentage ramps, and the rollback path.
- **The next scale curve:** if this supports 1M users, what changes at 10M?
- **Refinements:** what you would improve with more time.

**Never say the design is perfect or complete.** There is always something to improve, and
the ability to name it is the point.

---

## Effort allocation

Xu's guidance for a 45-minute interview, and the equivalent proportions for real work:

| Step | Interview (45 min) | Share of effort |
|---|---|---|
| 1. Scope | 3–10 min | 10–20% |
| 2. Interface | (within 1–2) | 5–10% |
| 3. Estimation | (within 2) | 10–15% |
| 4. Data model | (within 2) | 10–15% |
| 5. High-level design | 10–15 min | 15–20% |
| 6. Deep dive | 10–25 min | 25–35% |
| 7. Wrap-up | 3–5 min | 10–15% |

The shape to preserve: **roughly a third on understanding the problem, a third on the deep
dive, and the rest spread across structure and operations.** A design that spends 90% on
the diagram and 10% on requirements has the ratio backwards.

---

## Dos and don'ts

**Do**
- Ask for clarification; never assume the assumption is correct.
- Communicate your thinking — silence hides the reasoning, which is the actual deliverable.
- Suggest multiple approaches and compare them.
- Agree on the blueprint before detailing; design the most critical components first.
- Treat reviewers as collaborators, not examiners.
- Ask for feedback early and often.

**Don't**
- Jump into a solution before clarifying requirements and assumptions.
- Go deep on one component before the high-level design exists.
- Think in silence.
- Over-engineer. Design purity that ignores trade-offs is a defect; over-engineered systems
  carry compounding costs that someone else pays.
- Be stubborn or narrow-minded when given new information — updating on feedback is a
  strength.
- Assume you are done because you produced a design.

---

## Interview context vs. real work

The method transfers, with four differences worth naming:

| | Interview | Real work |
|---|---|---|
| Clarification | Ask the interviewer | Ask stakeholders; where you can't, write labeled assumptions |
| Scale numbers | Given or assumed | **Measure them** — production metrics beat estimates every time |
| Stack choice | Mostly free | Heavily constrained by what the team already runs and can operate |
| Deliverable | A conversation | A design doc that must survive review and still be readable in a year |

The most important carry-over: **the final design matters less than the process that
produced it.** A design without stated requirements, numbers, and rejected alternatives
cannot be evaluated by anyone — including you, six months later.
