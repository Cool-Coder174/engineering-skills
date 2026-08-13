---
name: system-design
description: A method to design a system in seven steps. The steps clarify the requirements, define the interface, estimate the load, and define the data model. Then they make a high-level design, examine two or three components in detail, and find the bottlenecks. Use this skill when you design a new system or a new service. Use it when you estimate capacity. Use it when you select a database, a cache, a queue, or an architecture. Use it when you review a design that exists. The content comes from four books. They are System Design Interview (Alex Xu), Grokking the System Design Interview, Database Internals (Petrov), and Code Simplicity (Kanat-Alexander).
---

# SYSTEM DESIGN

**ROLE:** You are a staff engineer. You make a system design.

**CORE FUNCTION:** You receive a request that is unclear and open-ended. You produce an
architecture that you can defend.

A design that you can defend has six parts:

1. The scope is written down.
2. The load has numbers.
3. Each component has a stated reason.
4. The bottlenecks have names.
5. The trade-offs are written down.
6. The failure behavior is written down.

## How this skill relates to `data-systems-design`

The two skills answer different questions. Use both when the work needs both.

| Skill | Question it answers |
|---|---|
| `system-design` (this skill) | What must we build? |
| `data-systems-design` | Does the design stay correct? |

Use this skill first. It decides the shape of the system. Then use `data-systems-design`.
It checks that the shape stays correct during concurrent operation and during failures.

---

# 1. WHEN TO USE THIS SKILL

Use this skill for these tasks:

- You design a new system, a new service, or a large feature.
- You estimate capacity or load.
- You answer this question: does the design scale?
- You select a database, a cache, a queue, or a communication protocol.
- You review an architecture.
- You plan a migration or a new architecture.
- You answer this question: what must change to support 10 times the load?

Do not use this skill for these tasks:

- A bug fix.
- A text change.
- A refactor inside one module.
- A feature that fits a pattern that exists, and that adds no infrastructure.

Section 7 gives the full rule for how much output each task needs.

---

# 2. THE SEVEN-STEP METHOD

This method comes from two books. Alex Xu gives four steps. Grokking gives seven steps.
The table below merges them.

Do the steps in order. Each step limits the next step. **Most bad designs come from a step
that the engineer skipped.**

| Step | Action | Output | Share of effort |
|---|---|---|---|
| 1 | Clarify the requirements and the scope | Functional requirements. Non-functional requirements. Non-goals. | 10–20% |
| 2 | Define the interface | API contract or event schema | 5–10% |
| 3 | Estimate the load | QPS, storage, bandwidth, memory, server count | 10–15% |
| 4 | Define the data model | Entities, access patterns, database choice | 10–15% |
| 5 | Make a high-level design | A diagram of 5 to 8 components. The request flows. | 15–20% |
| 6 | Examine 2 or 3 components in detail | A detailed design for each one | 25–35% |
| 7 | Find the bottlenecks and the failure modes | Bottleneck table. Failure table. Operations plan. | 10–15% |

**Two rules control this method:**

**Rule 1. Do not propose a solution before you write down the requirements.** A fast answer
without clarification is a fault, not a strength.

**Rule 2. Do not add a component that no number requires.** Engineers who like design purity
often ignore trade-offs. They do not see the cost of an architecture that is too large.
Someone else pays that cost every day.

## Step 1 — Clarify the requirements and the scope

**Never skip this step. Never assume that your assumption is correct.**

If you cannot ask a person, write your assumptions down. Mark each one as an assumption.
Then a reviewer can challenge it.

Ask these questions:

- **Features.** What do we build? What do we not build?
- **Users.** How many users in total? How many users each day? What growth do we expect
  after 3 months, 6 months, and 12 months?
- **Traffic.** Are there more reads than writes? What is the ratio? What is the peak? Is the
  traffic steady, or does it change through the day?
- **Data.** What do we store? How large is each item? How long do we keep it?
- **Latency.** What is the p99 target? Which operations does that target cover?
- **Consistency.** Can a user see old data? For how long? Which operations must show a
  change to the person who made that change?
- **Availability.** What is the target? What does the word "down" mean for this system?
- **Existing systems.** What do we run today? What can we reuse? *This limit is often more
  important than any theoretical best choice.*
- **Clients.** Web, mobile, or a third-party API? Do we support offline use? Do we send push
  messages?
- **Compliance.** Does the system hold personal data? Are there rules for data location,
  retention, audit, or deletion?

Record the answers in this format:

```md
### Requirements
**Functional**
- FR-1 [a capability, described as behavior that a user can see]

**Non-functional**
- Load: [daily active users, QPS, growth]
- Latency: p50 [x] ms, p99 [y] ms, for [operations]
- Availability: [target]. "Down" means [definition].
- Consistency: [level] for [operations], because [reason that a user can see]
- Durability: RPO [ ], RTO [ ]

**Not in scope**
- [What we do not build. Record it now, not during the review.]

**Assumptions** (nobody has confirmed these. Challenge them.)
- [The assumption. What breaks if it is wrong.]
```

## Step 2 — Define the interface

Write the API before you design the architecture. Two reasons make this order correct:

1. The API makes each requirement concrete.
2. The API shows a misunderstanding at once. If you cannot write the endpoint signature, you
   do not yet understand the requirement.

```md
### API
POST /v1/posts                 { content, media_ids[], idempotency_key } -> { post_id }
GET  /v1/feed?cursor=&limit=   -> { items[], next_cursor }
```

For each endpoint, record these items: authentication, idempotency, pagination style, rate
limit, and error cases.

For an event, record these items instead: the event schema, the partition key, and the
delivery guarantee.

## Step 3 — Estimate the load

**This step changes a diagram into a design.**

Numbers decide how many servers you need. Numbers decide whether you need one database or
many. Numbers are also the only honest way to reject a design that is too large.

Calculate at least these values: average QPS, peak QPS, storage each day, storage at the
retention limit, bandwidth, cache size, and server count.

Three rules control this step:

1. **Round the numbers.** Precision is not the goal.
2. **Write the unit after every number.**
3. **Write each assumption next to its arithmetic.** Then a reviewer can attack the
   assumption instead of the result.

For formulas, latency numbers, availability tables, and worked examples, read
`references/02-estimation.md`.

## Step 4 — Define the data model

Record the entities, the relationships, and the counts. Then record the 10 most frequent
access patterns. Mark which ones need low latency.

**The access patterns select the database. The entity diagram does not.**

Select the database. Record why you selected it. Record which alternative you rejected.

For the selection method, read `references/05-database-selection.md`.

## Step 5 — Make a high-level design

Draw 5 to 8 components. Draw the request flows between them.

Add no component that the Step 3 numbers do not require.

A typical set of components is: clients, DNS, CDN, load balancer, stateless application
servers, cache, primary database, message queue, asynchronous workers, derived stores such
as a search index, and object storage.

Then **examine the flows from start to end**. Examine the write path. Examine the read path.
This examination finds the edge cases that a component diagram hides.

For the standard order in which to add infrastructure, read `references/03-scaling-ladder.md`.
That reference also gives the signal that triggers each addition.

For how to select each component, read `references/04-building-blocks.md`.

## Step 6 — Examine 2 or 3 components in detail

Select the **two or three components that hold the difficulty**. Design those components
fully.

Do not give equal attention to every component. Equal attention produces a design that is
shallow everywhere.

Select each component with these questions:

- Which component has the highest QPS?
- Which component has the strictest consistency requirement?
- Which component has the worst hot-key problem?
- Which decision is the hardest to reverse?

For each selected component, record these items: the algorithm or data structure, the
partition plan, the replication plan, the concurrency control, the failure behavior, and the
trade-off that you accepted.

**Run the hazard scan on every detailed component.** The hazard list is in
`data-systems-design/references/hazard-catalog.md`. At this level of detail, a design error
becomes a correctness defect.

## Step 7 — Find the bottlenecks, the failure modes, and the operations plan

Most designs omit this step. Reviewers care about it most.

```md
### Bottlenecks
| Component | Single point of failure? | Saturates at | Symptom | Mitigation |
|---|---|---|---|---|
| Primary database writes | Yes | ~8k writes/s | Write latency rises. Replication lag grows. | Partition by user_id. Move low-priority writes to a queue. |
| Feed fan-out worker | No | ~50k fan-outs/s | Feeds become old | Use push for normal accounts and pull for large accounts |

### Failure Modes
| Failure | Behavior | Degraded mode | Recovery |
|---|---|---|---|
| Cache cluster stops | Load on the database rises 20 times | Read from the database. Merge identical requests. | Fill the cache in stages |
| Region stops | The region cannot accept writes | Serve read-only traffic from a replica region | Promote a replica. Then repair the data. |

### Operations
- Monitoring: [the signal that detects each failure above, with a threshold]
- Rollout: [flag, canary, or a staged percentage]
- Rollback: [the method, and the point after which rollback is not possible]

### The next scale step
At 10 times the load: [which value saturates first, and the next architecture]
```

**Never state that a design is finished or optimal.** Record what you would improve with
more time. That list is part of the output.

---

# 3. THE DESIGN DOCUMENT

This is the output artifact. Match its size to the decision. Section 7 gives the rule.

```md
# Design: [System]

## 1. Problem and scope
[What we build. Why now. What we do not build.]

## 2. Requirements
[Functional. Non-functional, with numbers. Assumptions.]

## 3. API
[Endpoints or event schemas, with idempotency, pagination, and errors]

## 4. Capacity estimates
| Metric | Value | How we calculated it |
|---|---|---|
| Daily active users | 10M | given |
| Write QPS (average / peak) | 3.5k / 7k | 10M x 2 posts / 86,400. Peak = 2 x average. |
| Storage each day | 30 TB | 10M x 2 x 10% media x 1 MB |
| Storage after 5 years | ~55 PB | 30 TB x 365 x 5 |
| Cache (20% hot) | 240 GB | hot item count x item size |
| Application servers | ~14 | peak QPS / 500 per server |

## 5. Data model
[Entities. Relationships. Access patterns. Database choice. Rejected alternative.]

## 6. High-level architecture
[Diagram. The write path from start to end. The read path from start to end.]

## 7. Detailed components
[2 or 3 components, each with its hazard scan]

## 8. Bottlenecks, failure modes, and operations
[Use the format in Section 2, Step 7]

## 9. Trade-offs and rejected alternatives
| Decision | We chose | Alternative | Why we rejected it |
|---|---|---|---|

## 10. Open questions and risks
[What needs a decision from a person. What we are unsure about.]
```

---

# 4. HOW TO RECORD A TRADE-OFF

**There is no correct answer. There is no best answer.**

A design for a startup with 1,000 users is different from a design for 100 million users.
Both can be correct. A design that fails at a large scale can be correct today.

Record every important decision in this format:

| Decision | We chose | Alternative | Why we rejected it |
|---|---|---|---|
| Feed generation | Push for small accounts, pull for large accounts | Push for all accounts | An account with many followers makes the write cost unbounded |
| Database | Postgres | Cassandra | We need transactions across rows. One node holds the data for more than 2 years. |
| Client protocol | WebSocket | Long polling | We must send data in both directions in less than 100 ms |

If you cannot name a real alternative, you did not make a decision. You made an assumption.

---

# 5. SIMPLICITY IS A REQUIREMENT

An architecture that is too large is a **defect**. It is not evidence of care.

Kanat-Alexander gives the laws that control this. Five of them apply here:

- **Software exists to help people.** A design that serves elegance instead of users has
  failed. Its technical properties do not change that result.

- **The equation of software design.** How much we want a change equals (value now + future
  value) divided by (effort to build + effort to maintain). Over time, one conclusion
  remains: **effort to maintain matters more than effort to build.** A design that is fast to
  build and expensive to operate is a bad design.

- **The law of simplicity.** Maintenance effort is proportional to the complexity of each
  part. Every component that you add is a permanent cost.

- **Do not predict the future.** The most frequent and most damaging error is a prediction
  that you cannot make. Design from what you measure now. Make the design only as general as
  you need it to be now.

- **The best design permits the largest change in the environment with the smallest change in
  the software.** That is not the same as a design that covers every possible future
  requirement. It is the opposite.

**The test:** for each component in the diagram, ask this question. *What breaks if we remove
this component, at the load we measure today?* If the answer is "nothing", remove it. Add it
again when a number requires it.

For the full set of tests, read `references/06-simplicity-and-design-laws.md`.

---

# 6. FAULTS TO FIND IN A DESIGN

| Fault | How it looks | Correction |
|---|---|---|
| **Solution before scope** | The architecture appears before the requirements | Return to Step 1 |
| **Architecture too large** | Many services and a message broker for 100 QPS | Justify each component from the Step 3 numbers |
| **No numbers** | The text says "it will scale" | Estimate the load |
| **Equal attention** | Every component has the same shallow description | Select 2 or 3 components. Design those fully. |
| **No failure plan** | The design shows only the path where nothing fails | Complete Step 7 |
| **Vague terms** | The text says "eventually consistent" or "we will cache it" with no mechanism | Read the language rules in `data-systems-design` |
| **Claims you cannot test** | The text says "this design scales to any load" | Name the value, the current number, and the saturation point |
| **No alternatives** | One option appears as the only option | Name the alternative and why it lost |
| **Prediction** | The design serves a load that nobody measured | Design for the measured load. Record the next scale step. |
| **Never finished** | The text says "the design is complete and optimal" | Record what you would improve with more time |

---

# 7. HOW MUCH OUTPUT EACH TASK NEEDS

| Type of change | Output |
|---|---|
| Fits a pattern that exists. Adds no infrastructure. | Nothing from this skill |
| A new endpoint or job on infrastructure that exists | Steps 1 and 2 only, in a few sentences |
| A new feature that adds storage, a cache, or a queue | Steps 1 to 5, plus the hazard scan. A short design document. |
| A new service, or a large change to scale or architecture | All 7 steps. A full design document. |
| Money, authentication, personal data, or a migration you cannot reverse | All 7 steps, plus a section on data integrity and a section on rollback |

A design document for a change that does not need one is waste. It costs review time. Then
it becomes wrong as the system changes. Match the document to the decision.

---

# 8. REFERENCE INDEX

| Reference | Content | Read it when |
|---|---|---|
| `references/01-design-method.md` | The 7 steps in full. The questions to ask. How to divide the effort. | You run a design from start to end |
| `references/02-estimation.md` | Powers of two. Latency numbers. Availability tables. Formulas for QPS, storage, bandwidth, cache, and server count. Worked examples. | Step 3, or any question about scale |
| `references/03-scaling-ladder.md` | The order in which to add infrastructure, from one server to a partitioned database. The signal that triggers each step. | You decide what to add next |
| `references/04-building-blocks.md` | Load balancers, caches, CDNs, proxies, indexes, queues, consistent hashing, rate limiting, ID generation, and client protocols | You select a component |
| `references/05-database-selection.md` | How to evaluate a database. SQL compared to NoSQL. Storage engine trade-offs. How to run an honest benchmark. | You select or defend a database |
| `references/06-simplicity-and-design-laws.md` | The equation of software design. The six laws. The three flaws. Tests for an architecture that is too large. | Any design that feels large. Any review. |
| `references/07-reference-architectures.md` | Worked patterns: URL shortener, feed fan-out, chat, notifications, crawler, file sync, autocomplete, and video | You identify which known pattern applies |

**Related skill: `data-systems-design`.** It covers correctness during concurrent operation
and during failures. It covers isolation levels, replication anomalies, and a list of 48
named hazards. Run its hazard scan on every component that you design in detail.

---

**Sources.** The method and the numbers come from these books:

- *System Design Interview: An Insider's Guide*, Alex Xu, 2020
- *Grokking the System Design Interview*, Design Gurus
- *Database Internals*, Alex Petrov, O'Reilly, 2019
- *Code Simplicity*, Max Kanat-Alexander, O'Reilly, 2012
