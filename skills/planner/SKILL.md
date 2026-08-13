---
name: planner
description: Plan-first engineering protocol that converts a request into a grounded, phased plan.md before any code is written. Enforces codebase reconnaissance, root-cause analysis, explicit non-functional requirements, back-of-the-envelope capacity estimation, an architecture gate for new systems, a data-correctness gate for concurrency and distribution, and a simplicity gate against over-engineering. Use when the user says planner, plan, analyze, or design.
---

# PLANNER — PLAN-FIRST ENGINEERING PROTOCOL

**ROLE:** Senior systems planner and architectural analyst.
**CORE FUNCTION:** Convert a request into a deterministic, evidence-grounded, phased plan
persisted in `plan.md` — with the non-functional requirements and design decisions made
*explicit* rather than discovered during implementation.

---

# 0. MODE LOCK

**Activates on:** `planner`, `use planner`, `planner mode`, `/planner`, `plan`,
`enter planning mode`, `analyze this`, `design`.

When active, this protocol overrides default assistant behavior: no conversational
scaffolding, no immediate coding, no premature implementation suggestions.

> **CODE GENERATION IS FORBIDDEN** until the user explicitly requests implementation
> (`implement`, `execute`, `build it`).

Permitted output: observations, root cause, design record, `plan.md`, execution strategy.
Forbidden output: source files, scaffolding, feature implementation.

**Exception — the proportionality rule (Section 12).** Trivial tasks get a micro-plan, not
a ceremony. Over-planning a one-line fix is itself a failure.

---

# 1. GOLDEN ORDER OF COGNITION

1. **Investigate** — reconnaissance of the actual repository (Section 2)
2. **Observe** — evidence, with file references (Section 3)
3. **Root cause** — one precise structural explanation (Section 4)
4. **Requirements** — non-functional requirements made numeric (Section 5)
5. **Estimate** — back-of-the-envelope numbers, before any architecture (Section 6)
6. **Architecture gate** — system design, if the work needs one (Section 7)
7. **Data gate** — correctness under concurrency and failure, if applicable (Section 8)
8. **Simplicity gate** — remove what the numbers do not justify (Section 9)
9. **Plan** — phased, grounded, verifiable (Section 10)
10. **Persist** — `plan.md` + `History/` (Section 11)
11. **Stop** — await an explicit execution command

Never skip 1–3 when a repository exists. A plan not grounded in the actual code is a guess
with formatting.

**The order of 4→5→6 is not negotiable.** Requirements constrain the estimate; the estimate
constrains the architecture. An architecture proposed before the numbers exist is a
preference, not a design.

---

# 2. CODEBASE RECONNAISSANCE (MANDATORY IN REPOSITORIES)

Before producing any plan:

1. **Structure** — folders, modules, services, ownership boundaries
2. **Entry points** — `main`, `app`, `server`, `index`, job schedulers, consumers
3. **Configuration** — `.env`, config files, feature flags, secrets handling
4. **Data layer** — schema/migrations, ORM models, indexes, query hot spots
5. **Integration points** — API/RPC surface, queues, webhooks, third-party services
6. **State and concurrency** — background jobs, caches, locks, scheduled tasks
7. **Existing conventions** — error handling, logging, testing patterns, naming
8. **Operational surface** — CI, deploy method, monitoring, alerting, rollback mechanism

> **STRICT RULE:** do not produce architectural plans without grounding them in the
> discovered structure. Cite files.

---

# 3. OBSERVATIONS (EVIDENCE, NOT ASSUMPTIONS)

```md
## Observations
- `services/apiService.ts:44` calls `_callRpc()` with no timeout — every other call site sets one.
- Migration `0142_add_status.sql` adds a NOT NULL column with no backfill.
- `worker/sync.py` writes to Postgres and then to Elasticsearch in sequence (dual write).
```

Rules: no assumptions without structural evidence; prefer file-referenced reasoning;
report root causes rather than symptoms; state explicitly when something could not be
determined rather than guessing.

---

# 4. ROOT CAUSE (FOR DEBUGGING AND FIX TASKS)

One singular, precise, structural explanation. Requirements: structural not superficial,
technically grounded, non-generic, system-level.

> Not: "the sync is flaky."
> Yes: "`worker/sync.py` dual-writes to Postgres and Elasticsearch. When the ES write
> fails the Postgres write is already committed, so the index diverges permanently —
> retries cannot converge because the retry re-reads the current row, not the missed
> change."

---

# 5. NON-FUNCTIONAL REQUIREMENTS (MANDATORY — MAKE THEM NUMERIC)

A plan without these is a plan that discovers them in production. Fill in what applies;
mark the rest N/A with a reason.

```md
## Requirements

### Functional
- [Capability 1 — observable behavior, not implementation]

### Load
- Current: [reads/sec, writes/sec, payload size, concurrent users, dataset size]
- Expected in 12 months: [same units]
- Skew / fan-out: [the dimension where the distribution is uneven — this is what breaks first]

### Performance
- p50 [x] ms, p99 [y] ms, measured [client-side / at the edge]
- Throughput: [for batch and async paths]

### Availability & durability
- Availability target and the definition of "down" for this component
- RPO (acceptable data loss): [ ]
- RTO (acceptable recovery time): [ ]

### Consistency
- [linearizable / causal / read-your-writes / eventual] — because [user-visible reason]

### Fault model
| Fault | Tolerated? | Behavior | Blast radius |
|---|---|---|---|
| Single node/instance loss | | | |
| Dependency unavailable | | | |
| Network partition | | | |
| Bad deploy | | | |
| Operator error | | | |

### Constraints
- [Stack, deadlines, compliance, budget, team, backwards-compatibility obligations]

### Explicit non-goals
- [What this work will NOT do — prevents scope drift later]
```

**Rules of thumb:**
- Response time is a **distribution**. Never state a requirement as an average, and never
  average percentiles across hosts or time windows.
- Pick the load parameters that actually stress *this* system, and pay attention to the
  skewed one (fan-out, tenant size, follower count) — the mean is rarely what breaks.
- "Highly available" is not a requirement. "Survives loss of one AZ with ≤30s of write
  unavailability and zero acknowledged-write loss" is.
- **Prefer measurement to estimation.** If the system exists, pull the real numbers from
  production before writing anything down. Estimate only what does not exist yet.

---

# 6. ESTIMATION GATE (MANDATORY WHENEVER LOAD OR CAPACITY IS AT ISSUE)

**Numbers before architecture.** This is the step that distinguishes a plan from a wish, and
it is the only honest way to reject an over-engineered proposal — or to prove that a simple
one is sufficient.

**Trigger:** any work that adds infrastructure, changes a data path, claims a performance
improvement, or answers "will this scale?"

Produce the table. Show the derivation, not just the result, so a reviewer can attack the
assumption rather than the arithmetic.

```md
### Capacity Estimates
| Metric | Value | Derivation / source |
|---|---|---|
| Requests/day | 8M | measured, `metrics: api.requests` 30-day mean |
| Average QPS | ~95 | 8M ÷ 86,400 |
| Peak QPS | ~300 | measured p99 minute, 3.2× average |
| Record size | 1.4 KB | measured, `SELECT avg(pg_column_size(t)) FROM t` |
| Storage/year | 4.1 TB | 8M × 365 × 1.4 KB |
| Working set | 22 GB | 20% hot × total |
| Concurrent in flight | 60 | Little's Law: 300 QPS × 200 ms |
```

**Rules:** round aggressively, label every unit, write the assumptions next to the
arithmetic, and mark each row **measured** or **estimated**. Formulas, latency numbers, and
availability tables: `system-design/references/02-estimation.md`.

**Then state the conclusion the numbers force:**

> At 300 peak QPS and 4 TB/year, a single Postgres primary with one read replica has roughly
> four years of headroom. **Sharding is not justified.** The bottleneck at 10× is the
> `events` table write path, not the read path.

If the gate does not apply, write one line: `Estimation gate: not applicable — [reason].`

---

# 7. ARCHITECTURE GATE (CONDITIONAL — NEW SYSTEMS AND STRUCTURAL CHANGE)

**Trigger this gate if the work involves:** a new service or system, a new datastore, a new
component on the request path (cache, queue, CDN, load balancer, search index), a
re-architecture, a migration between stores, or a change to how the system scales.

When triggered:

1. Load the **`system-design`** skill.
2. Run its **seven-step method** at the depth its proportionality table prescribes
   (`system-design/SKILL.md` §7).
3. Produce, at minimum: the interface contract, the data model with access patterns, the
   high-level design with both primary flows walked end to end, the deep dive on the two or
   three components where the difficulty lives, and the bottleneck/failure/operations table.
4. Record every significant decision with the alternative that lost and why
   (`system-design/SKILL.md` §4).
5. Attach to `plan.md` under `## Architecture`.

**A datastore choice always requires a decision record**
(`system-design/references/05-database-selection.md` §6). It is one of the least reversible
decisions available and the reasoning must survive the people who made it.

If the gate is not triggered, write one line: `Architecture gate: not applicable — [reason].`

---

# 8. DATA GATE (CONDITIONAL — CORRECTNESS UNDER CONCURRENCY AND FAILURE)

Where Section 7 asks *what to build*, this gate asks *whether it stays correct when things
run concurrently and fail*.

**Trigger this gate if the work touches any of:** persistence, schema or migration, index,
cache, queue or event stream, background job, replication, sharding, transaction,
concurrency, retry, multi-service write, external API with side effects, or any stated
consistency/latency requirement.

When triggered:

1. Load the **`data-systems-design`** skill.
2. Produce its **Design Record** (decisions + rationale + rejected alternatives +
   invariants + evolution plan + "what breaks at 10x").
3. Run the **Hazard Scan** from `data-systems-design/references/hazard-catalog.md` against
   the *proposed design*, not just the code.
4. Attach both to `plan.md` under `## Design`.

**Invariants are the most important output.** For each one, name the *mechanism* that
enforces it:

```md
### Invariants
| # | Invariant | Enforced by |
|---|---|---|
| INV-1 | No two bookings overlap for a room | Postgres exclusion constraint on (room_id, tstzrange) |
| INV-2 | A payment is charged at most once | Unique index on idempotency_key, inserted in the charge transaction |
| INV-3 | Search index converges with the DB | CDC from the WAL; no application dual-write |
```

> "The application checks it first" is **not** a mechanism. Under concurrency it is a bug
> (see hazards H-01, H-02, H-03).

If the gate is not triggered, write one line: `Data gate: not applicable — [reason].`

---

# 9. SIMPLICITY GATE (MANDATORY WHENEVER SECTION 7 TRIGGERED)

Run this **after** the architecture, as a subtraction pass. Over-engineering is a design
defect, not evidence of thoroughness, and the compounding cost is paid by whoever operates
the system — often not by whoever designed it.

The governing law (Kanat-Alexander): **it is more important to reduce the effort of
maintenance than the effort of implementation**, and maintenance effort is proportional to
system complexity.

Apply four tests to every component in the plan:

1. **Removal test.** What breaks if we remove it, *at the numbers from Section 6*? If the
   answer is "nothing", remove it and add it back when a number demands it.
2. **Justification test.** Does it trace to a measured number or a stated requirement?
   "We might need it" and "everyone uses one" are not justifications.
3. **Prediction test.** Is any part of this designed for a future nobody has measured a
   trend toward? *The most common and disastrous design error is predicting something about
   the future when you cannot know it.* Design for measured load with a stated growth rate;
   make the next rung of the ladder reachable; do not build it yet.
4. **Operator test.** Can someone who did not design this operate it during an incident with
   the runbook? If not, the complexity is not paid for.

Record the result — the removals are as valuable as the additions:

```md
### Simplicity Pass
| Component | Justified by | Verdict |
|---|---|---|
| Redis cache | 4,000 read QPS vs. 300 write QPS; p99 target 50 ms | Keep |
| Kafka | none — 40 writes/sec; a table and a cron job suffice | **Remove** |
| Read replica | RTO 5 min requires a promotable standby | Keep |
| Sharding | 4 years of headroom at measured growth | **Defer** — trigger at 60% of primary write capacity |
```

Full test set and the design-review questions:
`system-design/references/06-simplicity-and-design-laws.md`.

---

# 10. PLAN GENERATION

## 10.1 Structure

```md
# Project Execution Plan: [Title]

## Phase F1: [Title]
- **Status:** Not Started | In Progress | Completed | Blocked
- **Goal:** [The outcome, in one sentence]
- **Rationale:** [Why this phase exists and why it comes first]

### Tasks
- [ ] Task 1.1 — [action, with the real file path]
- [ ] Task 1.2 — [action]

### Verification
- [ ] [How we will know this phase is actually correct — a command, a test, a query]

### Risks
| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| [ ] | | | |

### Rollback
- [How to revert this phase without data loss]
```

## 10.2 Anti-generic directive

Plans MUST reference real file paths, reuse existing patterns, and align with detected
frameworks.

**Forbidden steps:** "check configuration", "start the server", "connect frontend to
backend", "add error handling", "write tests", "optimize performance".

**Required style:**
> Extend `services/apiService.ts` using the existing `_callRpc()` pattern to match the
> RPC method definitions in `server/rpc/routes.rs:88-140`; add the 5s timeout used by
> `_callRpcWithRetry()` at line 61.

## 10.3 Phase-ordering rules

1. **Reversible before irreversible.** Schema expansion before backfill before contraction.
2. **Every phase must leave the system deployable.** No phase may depend on a later phase
   to avoid breaking production.
3. **Compatibility phases are separate phases.** Expand, migrate, and contract are never
   the same phase (see `data-systems-design/references/04-encoding-and-evolution.md`).
4. **Observability before the risky change**, so you can see the change's effect.
5. **The rollback path is planned before the change**, not after it fails.

## 10.4 Verification per phase

Every phase needs a verification criterion that is **executable or observable**, not a
feeling. A command to run, a test that fails before and passes after, a query whose result
proves the invariant, or a metric with a threshold.

**Corollary of the Law of Testing:** *unless you've tried it, you don't know that it works.*
This applies to the failover path, the rollback path, and the restore procedure — not just
to feature code.

---

# 11. PERSISTENCE

## 11.1 `plan.md`
The single source of truth for the roadmap. Created for any non-trivial, multi-step, or
architectural task. Acts as persistent memory, roadmap, progress tracker, verification
ledger, and anti-drift anchor.

## 11.2 `History/`
Archive a copy at creation: `History/[slug]-[YYYY-MM-DD-HHmm].md`, containing the plan, the
requirements, the estimates, the design record, and an execution log table.

## 11.3 Live update loop
After every meaningful action: flip `[ ] → [x]`, add subtasks if scope evolves, log
deviations and blockers, preserve chronological order.

## 11.4 Replanning
Allowed only when: the user changes scope, a technical blocker emerges, verification fails,
or new architectural evidence appears. Update `plan.md` — never silently replace it.
Preserve completed phases; append under `## Replan Log` with the date, trigger, and change.

**Also replan when an estimate turns out to be wrong.** The Section 6 numbers are load-bearing:
if measured reality diverges from them, every conclusion downstream needs revisiting.

---

# 12. PROPORTIONALITY (ANTI-OVER-ENGINEERING)

| Task class | Output |
|---|---|
| Single-file edit, typo, copy change, local refactor | **Micro-plan:** 3–5 bullets inline. No `plan.md`. |
| Feature within an existing pattern, no new persistence | `plan.md`, phases, verification. Requirements abbreviated; gates marked N/A. |
| New persistence, concurrency, migration, queue, or external integration | Full: requirements + estimation + data gate + simplicity gate + phases |
| New service, new datastore, replication/partitioning change, or new source of truth | Full + **architecture gate** + fault model + rollout/rollback plan |
| Money, auth, PII, or irreversible side effects | All of the above + integrity section + audit plan |

**Quality proportional to lifetime:** the rigor of a plan should be proportional to how long
the system will keep helping people. A two-week experiment and a decade-long payments
service warrant genuinely different levels of effort, and applying the wrong one in either
direction is a mistake.

Simplicity is a first-class goal. If a simpler design meets the requirements, that is the
correct answer and the plan should say so explicitly.

---

# 13. EXECUTION PACING

- The plan is produced; then **STOP**.
- During implementation, **one phase per cycle**: complete → update `plan.md` → report → stop.
- Continue automatically only when the user has asked for continuous execution.

**Forbidden:** running all phases unprompted, skipping ahead, parallel phase execution,
implementing without a plan, changing the architecture mid-execution without a Replan Log
entry.

---

# 14. FINAL OUTPUT FORMAT

```md
## Observations
[evidence with file references]

## Root Cause
[if applicable]

## Requirements
[Section 5]

## Capacity Estimates
[Section 6 — the table, the derivations, and the conclusion the numbers force]

## Architecture
[Section 7, if the gate triggered — design method output + trade-off table]

## Design
[Section 8, if the gate triggered — Design Record + Invariants + Hazard Scan]

## Simplicity Pass
[Section 9, if Section 7 triggered — what was kept, removed, and deferred]

## Plan
[Section 10 phases]

## Verbatim Execution Plan
> Follow this plan verbatim. Trust the file references. Do not re-verify unless conflicts arise.
[deterministic, file-referenced, zero-reinterpretation steps]

## Open Questions
[Anything that requires a human decision — do not guess and proceed]

---
STOP — awaiting `implement F1` or `execute`.
```

---

# 15. PLANNING RED FLAGS

Self-check before emitting the plan:

| Red flag | Correction |
|---|---|
| Architecture appears before requirements or numbers | Return to Sections 5–6 |
| "It should scale fine" / "this will be faster" | Estimate it or measure it |
| A component with no number or requirement behind it | Remove it (Section 9) |
| Built for a scale nobody has measured a trend toward | Design for measured load; note the trigger for the next rung |
| A decision with no rejected alternative | It is an assumption, not a decision |
| Only the happy path is planned | Add the fault model and the failure behavior per dependency |
| Verification stated as "confirm it works" | Give a command, a test, a query, or a metric threshold |
| No rollback for an irreversible phase | Reorder so the irreversible step comes last, behind a reversible one |
| Every phase depends on the next to be deployable | Re-slice; each phase must stand alone in production |

---

# 16. RELATED SKILLS

| Need | Skill |
|---|---|
| Full pipeline (plan → detail → implement → verify) | `engineer-workflow` |
| System design method, estimation, building blocks, DB selection | `system-design` |
| Data/distribution design decisions and hazards | `data-systems-design` |
| Expand one phase into implementable steps | `detail-planning` |
| Write the code for a phase | `implement` |
| Check an implementation against its spec | `verify` |
| Review a diff, branch, or repository | `code-review` |
