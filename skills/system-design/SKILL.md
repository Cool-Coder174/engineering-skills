---
name: system-design
description: Industry-standard system design methodology — requirements clarification, API contract definition, back-of-the-envelope capacity estimation, data model, high-level architecture, deep dive, and bottleneck analysis. Grounded in Alex Xu's System Design Interview, Grokking the System Design Interview, Database Internals (Petrov), and Code Simplicity (Kanat-Alexander). Use when designing a new system or service, sizing capacity, choosing a database or architecture, or reviewing an existing design.
---

# SYSTEM DESIGN

**ROLE:** Staff engineer running a system design.
**CORE FUNCTION:** Take an open-ended, ambiguous request and produce a defensible
architecture — with the scope clarified, the scale estimated in numbers, the components
justified, the bottlenecks named, and the trade-offs stated out loud.

This skill is the **design methodology**. It pairs with `data-systems-design`, which is the
**correctness knowledge base** (hazards, isolation, replication, consistency). Use this one
to decide *what to build*; use that one to make sure it *stays correct under concurrency
and failure*.

---

# 1. ACTIVATION

Use this skill when the task is:

- designing a new system, service, or major feature
- sizing capacity, estimating load, or answering "will this scale?"
- choosing a database, cache, queue, or communication protocol
- reviewing or critiquing an existing architecture
- planning a migration or a re-architecture
- answering "how would we support 10x?"

**Do not use it** for a bug fix, a copy change, a refactor inside one module, or a feature
that fits an existing pattern with no new infrastructure. See Section 7.

---

# 2. THE SEVEN-STEP METHOD

The method below merges Alex Xu's four-step framework with Grokking's seven steps.
The order matters: each step constrains the next, and skipping ahead is the single most
common cause of designing the wrong system.

> **The discipline this enforces:** do not propose a solution before the requirements are
> written down. Answering fast, without clarifying, is a red flag — not a strength. The
> other red flag is **over-engineering**: delighting in design purity while ignoring
> trade-offs and the compounding cost of the system you just committed someone to
> operating.

| # | Step | Output | Effort share |
|---|---|---|---|
| 1 | Clarify requirements and scope | Functional + non-functional requirements, explicit non-goals | 10–20% |
| 2 | Define the interface | API contract / event schema | 5–10% |
| 3 | Estimate scale | QPS, storage, bandwidth, memory, server count | 10–15% |
| 4 | Define the data model | Entities, relationships, access patterns, store choice | 10–15% |
| 5 | High-level design | 5–8 box diagram, request flows, get agreement | 15–20% |
| 6 | Deep dive | 2–3 critical components in detail | 25–35% |
| 7 | Bottlenecks, failure, operations | SPOFs, failure modes, monitoring, rollout, next scale curve | 10–15% |

## Step 1 — Clarify requirements and establish scope

**Never skip this, and never assume your assumption is correct.** If you cannot ask a
human, write your assumptions down explicitly and mark them as assumptions so they can be
challenged later.

The question bank:

- **Features:** what exactly are we building? What is explicitly *not* in scope?
- **Users:** how many total? How many daily active? What is the growth expectation at
  3 months, 6 months, a year?
- **Traffic shape:** read-heavy or write-heavy, and by what ratio? Peak vs. average? Is
  traffic bursty, diurnal, or event-driven?
- **Data:** what is stored, how big is each item, how long is it retained?
- **Latency:** what is the p99 target, and for which operations specifically?
- **Consistency:** can a user tolerate seeing stale data? For how long? Which operations
  must be immediately visible to the person who performed them?
- **Availability:** what is the target, and what does "down" mean here?
- **Existing stack:** what do we already run that we can use? *This constraint usually
  matters more than any theoretical best choice.*
- **Client types:** web, mobile, third-party API? Offline support? Push required?
- **Compliance:** PII, residency, retention, audit, deletion obligations?

Output:

```md
### Requirements
**Functional**
- FR-1 [capability, observable behavior]

**Non-functional**
- Scale: [DAU, QPS, growth]
- Latency: p50 [x] ms, p99 [y] ms, for [operations]
- Availability: [target] — "down" means [definition]
- Consistency: [level] for [operations], because [user-visible reason]
- Durability: RPO [ ], RTO [ ]

**Explicitly out of scope**
- [Thing we are not building — say so now, not in review]

**Assumptions** (unverified — challenge these)
- [Assumption, and what breaks if it's wrong]
```

## Step 2 — Define the interface

Write the API before the architecture. It forces the requirements to be concrete and
surfaces misunderstandings immediately — if you cannot write the endpoint signature, you
do not yet understand the requirement.

```md
### API
POST /v1/posts            { content, media_ids[], idempotency_key } → { post_id }
GET  /v1/feed?cursor=&limit=   → { items[], next_cursor }
```

For each endpoint state: auth, idempotency, pagination style, rate limit, and error cases.
For event-driven paths, define the event schema, the partition key, and the delivery
semantics instead.

## Step 3 — Estimate scale (back-of-the-envelope)

**This step is what separates a design from a diagram.** Numbers determine whether you
need one server or a hundred, one database or a sharded cluster — and they are the only
honest way to reject an over-engineered proposal.

Compute, at minimum: **QPS (average and peak), storage per day and at retention horizon,
bandwidth, cache size, and server count.**

Rules: round aggressively (precision is not the point), **label every unit**, and write
the assumptions next to the arithmetic so a reviewer can attack the assumption rather than
the result.

Full formulas, latency numbers, availability tables, and worked examples:
`references/02-estimation.md`

## Step 4 — Define the data model

Entities, relationships, cardinalities, and the top ~10 access patterns by frequency and
latency sensitivity. **The access patterns choose the store, not the entity diagram.**

Then choose the store and say why, including the rejected alternative.
Selection methodology: `references/05-database-selection.md`

## Step 5 — High-level design

Draw 5–8 boxes and the request flows between them. Resist adding a component you cannot
justify from the numbers in Step 3.

Typical boxes: clients → DNS/CDN → load balancer → stateless app tier → cache → primary
datastore → message queue → async workers → derived stores (search, analytics) → object
storage.

Then **walk the concrete flows end to end** — the write path and the read path, at
minimum. Walking a real use case is how you find the edge cases the box diagram hides.

The standard progression from one server to millions of users, and what triggers each
step: `references/03-scaling-ladder.md`

Component selection — load balancers, caches, CDN, queues, consistent hashing, rate
limiting, ID generation, client-server protocols: `references/04-building-blocks.md`

## Step 6 — Deep dive

Pick the **two or three components where the difficulty actually lives** and design them
properly. Do not spread attention evenly; that produces a shallow design everywhere.

Choose the deep-dive targets by asking: which component has the highest QPS, the hardest
consistency requirement, the worst hot-key problem, or the least reversible decision?

For each, cover: the algorithm or data structure, the partition/replication strategy, the
concurrency control, the failure behavior, and the specific trade-off you accepted.

**Every deep dive must run the hazard scan** from
`data-systems-design/references/hazard-catalog.md`. This is where design errors become
correctness bugs.

## Step 7 — Bottlenecks, failure, and operations

The step most designs omit, and the one reviewers care about most.

```md
### Bottleneck & Failure Analysis
| Component | SPOF? | Saturates at | Symptom | Mitigation |
|---|---|---|---|---|
| Primary DB writes | Yes | ~8k writes/s | Write latency climbs, replication lag grows | Shard by user_id; queue non-critical writes |
| Feed fan-out worker | No | ~50k fan-outs/s | Feed staleness grows | Hybrid push/pull for high-follower accounts |

### Failure Modes
| Failure | Behavior | Degraded mode | Recovery |
|---|---|---|---|
| Cache cluster down | Origin load ×20 | Serve from DB with request coalescing | Warm cache progressively |
| Region loss | Writes unavailable in region | Read-only from replica region | Promote, then reconcile |

### Operations
- Monitoring: [the signals that would catch each failure above, with thresholds]
- Rollout: [flag / canary / percentage ramp]
- Rollback: [how, and the point of no return]

### The Next Scale Curve
At 10x: [which parameter saturates first, and the next architecture]
```

**Never claim a design is finished or optimal.** State what you would improve with more
time — that list is part of the deliverable.

---

# 3. THE DESIGN DOCUMENT

The output artifact. Sized to the decision (see Section 7).

```md
# Design: [System]

## 1. Problem & Scope
[What we're building, why now, explicit non-goals]

## 2. Requirements
[Functional, non-functional with numbers, assumptions]

## 3. API / Interface
[Endpoints or event schemas with idempotency, pagination, errors]

## 4. Capacity Estimates
| Metric | Value | Derivation |
|---|---|---|
| DAU | 10M | given |
| Write QPS (avg / peak) | 3.5k / 7k | 10M × 2 posts / 86,400; peak = 2× |
| Storage / day | 30 TB | 10M × 2 × 10% media × 1 MB |
| Storage @ 5 yr | ~55 PB | 30 TB × 365 × 5 |
| Cache (20% hot) | 240 GB | working set × item size |
| App servers | ~14 | peak QPS / 500 per server |

## 5. Data Model
[Entities, relationships, access patterns, store choice + rejected alternative]

## 6. High-Level Architecture
[Diagram + write path + read path walked end to end]

## 7. Deep Dives
[2–3 components in detail, each with its hazard scan]

## 8. Bottlenecks, Failure Modes, Operations
[Section 2, Step 7]

## 9. Trade-offs & Alternatives Considered
| Decision | Chosen | Alternative | Why rejected |
|---|---|---|---|

## 10. Open Questions & Risks
[What needs a human decision; what we're unsure about]
```

---

# 4. TRADE-OFF DISCIPLINE

**There is no right answer, and there is no best answer.** A design for a startup with
1,000 users is *correctly different* from a design for 100 million. A design that would be
wrong at scale can be exactly right today.

Every significant decision states the alternative and why it lost:

| Decision | Chosen | Alternative | Why rejected |
|---|---|---|---|
| Feed generation | Hybrid push/pull | Pure fan-out-on-write | Celebrity accounts make write fan-out unbounded |
| Store | Postgres | Cassandra | Need multi-row transactions; volume fits one shard for 2+ years |
| Comms | WebSocket | Long polling | Bidirectional, <100ms delivery required |

If you cannot name a real alternative, you have not made a decision — you have made an
assumption.

---

# 5. SIMPLICITY AS A DESIGN CONSTRAINT

Over-engineering is a design *defect*, not a sign of thoroughness. The counterweight, from
Kanat-Alexander's laws of software design:

- **The purpose of software is to help people.** A design that serves elegance rather than
  users has failed regardless of its properties.
- **The Equation of Software Design:** desirability = (value now + future value) ÷
  (effort of implementation + effort of maintenance). Over time this reduces to a single
  conclusion: **reducing the effort of maintenance matters more than reducing the effort of
  implementation.** A design that is quick to build and expensive to operate is a bad
  design.
- **The Law of Simplicity:** ease of maintenance is proportional to the simplicity of the
  individual pieces. Maintenance effort is proportional to system complexity — so every
  component you add is a permanent tax.
- **Do not predict the future.** The most common and disastrous error is designing for a
  future you cannot know. Design from what is known *now*. Be only as generic as you know
  you need to be right now.
- **The best design allows the most change in the environment with the least change in the
  software** — which is different from, and much more valuable than, building for every
  hypothetical requirement.

**The concrete test:** for every component in the diagram, ask "what happens if we remove
it?" If the answer is "nothing, at our current numbers", remove it. Add it back when the
numbers say so.

Deep dive: `references/06-simplicity-and-design-laws.md`

---

# 6. RED FLAGS IN A DESIGN

| Red flag | What it looks like | Correction |
|---|---|---|
| **Solution before scope** | Architecture proposed before requirements are written | Go back to Step 1 |
| **Over-engineering** | Microservices, Kafka, and a service mesh for 100 QPS | Justify each component from the Step 3 numbers |
| **Numberless design** | "It should scale fine" | Do the estimation |
| **Even attention** | Every component described at the same shallow depth | Pick 2–3 for the deep dive |
| **No failure story** | Only the happy path is drawn | Complete Step 7 |
| **Buzzword substitution** | "eventually consistent", "we'll cache it" with no mechanism | See the language-discipline table in `data-systems-design` |
| **Unfalsifiable claims** | "This is web scale" | State the parameter, the value, and the saturation point |
| **No alternatives** | One option presented as inevitable | Name what lost and why |
| **Prediction** | Built for a scale nobody has asked for | Design for known load; note the next curve |
| **Never finished** | "The design is complete and optimal" | List what you'd improve with more time |

---

# 7. PROPORTIONALITY

| Change class | Output |
|---|---|
| Fits an existing pattern, no new infrastructure | Nothing from this skill |
| New endpoint or job on existing infrastructure | Steps 1–2 inline (requirements + interface), a few sentences |
| New feature with new storage, cache, or queue | Steps 1–5 + hazard scan; short design doc |
| New service, or a significant scale/architecture change | All 7 steps; full design doc |
| Money, auth, PII, or an irreversible migration | All 7 steps + explicit integrity and rollback sections |

A design document for a change that does not need one is itself waste — it costs review
time and then goes stale. Match the artifact to the decision.

---

# 8. REFERENCE INDEX

| Reference | Covers | Read when |
|---|---|---|
| `references/01-design-method.md` | The 7 steps in depth; question banks; effort allocation; interview vs. real-world differences | Running a design end to end |
| `references/02-estimation.md` | Powers of two, latency numbers, availability nines, QPS/storage/bandwidth/cache/server formulas, worked examples | Step 3, or any "will it scale" question |
| `references/03-scaling-ladder.md` | Single server → LB → replication → cache → CDN → stateless tier → multi-DC → queue → sharding → services, with the trigger for each step | Deciding what to add next, or reviewing a scaling plan |
| `references/04-building-blocks.md` | Load balancers, caching, CDN, proxies, indexes, queues, consistent hashing, rate limiting, unique IDs, polling/WebSocket/SSE | Choosing a component |
| `references/05-database-selection.md` | Evaluation methodology, SQL vs. NoSQL, storage engine internals, benchmarking honestly | Choosing or defending a datastore |
| `references/06-simplicity-and-design-laws.md` | Equation of software design, six laws, three flaws, over-engineering detection | Any design that feels large; any review |
| `references/07-reference-architectures.md` | Worked patterns: URL shortener, feed fan-out, chat, notifications, crawler, file sync, autocomplete, video | Recognizing which known pattern applies |

Companion skill: **`data-systems-design`** — correctness under concurrency and failure
(isolation levels, replication anomalies, the 48-hazard catalog). Every deep dive should
run its hazard scan.

---

**Attribution:** methodology and numbers are drawn from *System Design Interview: An
Insider's Guide* (Alex Xu, 2020), *Grokking the System Design Interview* (Design Gurus),
*Database Internals* (Alex Petrov, O'Reilly 2019), and *Code Simplicity* (Max
Kanat-Alexander, O'Reilly 2012).
