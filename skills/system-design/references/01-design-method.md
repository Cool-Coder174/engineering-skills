# The Design Method

**Sources:** *System Design Interview*, Chapter 3 (Xu). *Grokking the System Design
Interview*, "System Design Interviews: A step by step guide".

Both books describe the same process. Xu gives four steps and a time budget. Grokking gives
seven steps. This reference merges them into seven steps. It adapts the steps from interview
use to engineering work.

---

## Why a method is necessary

Design problems are unclear and open-ended. They have no single correct answer. Without a
method, two faults occur.

**Fault 1. The engineer answers too fast.** The engineer proposes a solution before he
understands the requirements. The result is a correct version of the wrong system. In an
interview this is the largest single fault. In real work it wastes months.

**Fault 2. The engineer never reaches a decision.** The engineer examines everything at equal
depth. Time runs out. The result is a diagram that contains no decisions.

The method prevents both faults. It puts scope before solution. It puts depth only where the
risk is.

---

## Step 1 — Clarify the requirements and the scope

**Rule: always ask for clarification. Never assume that your assumption is correct.**

Sometimes you cannot ask. You work asynchronously, or no stakeholder is available. Then
write each assumption down and mark it. A reviewer can then challenge the assumption. If you
hide the assumption, someone finds it three weeks later inside the schema.

### The questions to ask

**Product**
- What features do we build? What features do we not build?
- Which features are necessary? Which features are optional?
- Do we support web, mobile, or both? Do third parties call our API?
- Must the system work offline?

**Scale**
- How many users in total? How many users each day?
- What growth do we expect after 3 months, 6 months, and 12 months?
- Are there more reads than writes? What is the ratio?
- What is the peak load compared to the average load? Does the load change through the day?
  Does it arrive in bursts? Do events cause it?

**Data**
- Which entities exist? How large is each one?
- How long do we keep the data?
- Does the data include images or video? What fraction?
- Does the data include personal data? Are there rules for location, retention, or deletion?

**Behavior**
- What order do we show items in? Newest first, or ranked? *This one question can change the
  whole architecture. A ranked feed needs a scoring pipeline.*
- Do we support search? Notifications? Live updates?
- What latency do users expect? For which operations?
- Can users see old data? For how long?

**Context**
- What technology do we run today? What can we reuse?
- How large is the team? How much operational experience does it have?
- What is the deadline? What is the budget?

**The question about existing technology is the most valuable one.** In real work, this
sentence is a stronger limit than any theoretical comparison: "We already run Postgres, and
nobody here has operated Cassandra." A design that ignores that sentence fails when the
on-call engineers receive it.

### Output
Record the functional requirements. Record the non-functional requirements **with numbers**.
Record what you do not build. Record each assumption and mark it as an assumption.

---

## Step 2 — Define the interface

Define the APIs that the system exposes. This action does two things:

1. It states the exact contract.
2. It proves that you understood the requirements. If you cannot write the endpoint
   signature, the requirement is still unclear.

For each endpoint, record these items:
- Method, path, parameters, response shape
- Authentication and authorization
- Idempotency. Is it necessary? What is the key?
- Pagination style. Offset, or cursor?
- Rate limits
- Error cases and status codes

For an event, record these items instead: the event schema, the partition key, the order
guarantee, and the delivery guarantee. The normal answer for delivery is at-least-once, with
a consumer that is idempotent.

**Match this step to the altitude of the problem.** For "design a search engine", API detail
is too low a level to start with. For "design the backend of a multiplayer game", API detail
is the correct place to start.

---

## Step 3 — Estimate the load

Jeff Dean defines the purpose of this step:

> "Back-of-the-envelope calculations are estimates you create using a combination of thought
> experiments and common performance numbers to get a good feel for which designs will meet
> your requirements."

**The process matters more than the result.** The purpose is to learn two things. First,
whether you need one server or one hundred. Second, whether you need one database or many.
The numbers also give you the evidence to reject a design that is too large.

Four rules control this step:

1. **Round the numbers.** Change "99,987 / 9.1" to "100,000 / 10". Nobody expects precision.
   Precision costs the time that you need for design.
2. **Write each assumption down.** A reviewer must be able to find it and challenge it.
3. **Write the unit after every number.** The value "5" means nothing. The value "5 MB" is
   clear.
4. **Calculate at least these values:** average QPS, peak QPS, storage, bandwidth, cache
   size, and server count.

For the formulas and the constants, read `02-estimation.md`.

---

## Step 4 — Define the data model

Record the entities, the relationships, and how the entities interact. This model shows how
data moves between components. It also controls the partition plan and the cache plan.

For each entity, record these items: the fields, the approximate size, the write rate, the
read rate, and the retention period.

Then record **the 10 most frequent access patterns**. Mark which patterns need low latency.

**The access patterns select the database.** The entity diagram does not select it. The
technology that the team used last time does not select it.

Then select one of these: a relational database, a document database, a wide-column
database, a key-value store, a graph database, object storage, or a search index.

For the selection method, read `05-database-selection.md`.

---

## Step 5 — Make a high-level design and get agreement

Draw a diagram with **5 to 8 components**. Include enough components to solve the whole
problem. A typical set is: clients, DNS, CDN, load balancer, application servers, cache,
database, message queue, workers, object storage, and derived stores.

Then do three things:

1. **Examine the flows from start to end.** Examine the write path. Examine the read path.
   This examination finds the edge cases that you have not considered.
2. **Compare the design against the Step 3 numbers.** Suppose the estimate is 200 QPS. Then
   a design with a message broker and twelve services must justify itself.
3. **Get agreement before you go deeper.** In an interview, ask the interviewer. In real
   work, hold a design review. A detailed design that nobody agrees with is wasted work.

---

## Step 6 — Examine 2 or 3 components in detail

Select the components that need detail. **Do not give equal attention to every component.**

Select each component with these questions:

- Which component has the highest QPS or the most data?
- Which component has the strictest consistency or latency requirement?
- Which component has the worst hot-key problem or the worst uneven load?
- Which decision is the hardest to reverse?
- Which component is new to the team?

For each selected component, record these seven items:

1. The algorithm or the data structure.
2. The partition plan.
3. The replication plan.
4. The concurrency control.
5. The cache plan.
6. The failure behavior.
7. The trade-off that you accepted.

**Control your time in this step.** It is easy to spend time on detail that proves nothing.
An example: you tune the weights of a ranking algorithm, but the real question is whether
the feed architecture scales. Depth is valuable only where the risk is.

Run the hazard scan on each detailed component. The hazard list is in
`data-systems-design/references/hazard-catalog.md`. At this level of detail, a design error
becomes a correctness defect.

---

## Step 7 — Find the bottlenecks, the failure modes, and the operations plan

Record these items:

- **Bottlenecks.** Which component saturates first? At what value? With what symptom?
- **Single points of failure.** Which component has no redundancy? What happens when it
  stops?
- **Replication and redundancy.** Do you have enough copies of the data? Do you have enough
  copies of each service? The copies must match the failures that you claim to tolerate.
- **Failure behavior.** For each dependency, record what the system does when that dependency
  is down or slow. Select one behavior for each dependency: degrade, reject, queue, or fail.
- **Monitoring.** Record which metric or log detects each failure above. Record the threshold
  and the alert. **A failure mode with no signal is a failure mode that nobody detects.**
- **Rollout.** Record the flags, the canary plan, the staged percentages, and the rollback
  path.
- **The next scale step.** The design supports 1 million users. What changes at 10 million?
- **Improvements.** Record what you would improve with more time.

**Never state that the design is perfect or complete.** You can always improve something.
Your ability to name that improvement is the purpose of this step.

---

## How to divide the effort

Xu gives a time budget for a 45-minute interview. The right column converts it to a share of
effort for real work.

| Step | Interview time | Share of effort |
|---|---|---|
| 1. Scope | 3–10 min | 10–20% |
| 2. Interface | inside steps 1 and 2 | 5–10% |
| 3. Estimation | inside step 2 | 10–15% |
| 4. Data model | inside step 2 | 10–15% |
| 5. High-level design | 10–15 min | 15–20% |
| 6. Detailed components | 10–25 min | 25–35% |
| 7. Bottlenecks and operations | 3–5 min | 10–15% |

Keep these proportions. Spend about one third on understanding the problem. Spend about one
third on the detailed components. Spend the rest on structure and operations.

A design that spends 90% on the diagram and 10% on the requirements has the proportions
reversed.

---

## What to do and what not to do

**Do these things**
- Ask for clarification. Never assume that your assumption is correct.
- Say what you are thinking. Silence hides your reasoning, and your reasoning is the output.
- Propose more than one approach. Compare them.
- Agree on the high-level design before you add detail. Design the most important components
  first.
- Treat reviewers as colleagues, not as examiners.
- Ask for feedback early and often.

**Do not do these things**
- Do not propose a solution before you clarify the requirements and the assumptions.
- Do not add detail to one component before the high-level design exists.
- Do not think in silence.
- Do not make the architecture too large. Design purity that ignores trade-offs is a defect.
  An architecture that is too large costs more every year, and someone else pays that cost.
- Do not refuse new information. When you receive a correction, use it. That is a strength.
- Do not assume that you are finished because you produced a design.

---

## Differences between interview use and real work

The method transfers to real work. Four differences matter.

| Item | Interview | Real work |
|---|---|---|
| Clarification | Ask the interviewer | Ask stakeholders. If you cannot, record marked assumptions. |
| Scale numbers | Given, or assumed | **Measure them.** Production metrics are better than estimates. |
| Technology choice | Almost free | Limited by what the team runs today and can operate |
| Output | A conversation | A design document that must survive review and stay readable in a year |

One rule transfers above all others: **the process matters more than the final design.** A
design without requirements, numbers, and rejected alternatives is a design that nobody can
evaluate. That includes you, six months later.
