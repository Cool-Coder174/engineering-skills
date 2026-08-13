# Engineering Skills

A suite of Agent Skills that make an AI coding agent behave like a senior engineer: plan
before coding, make design decisions explicitly, implement with robustness invariants,
verify against the spec, and review for the failure modes that only appear under
concurrency, failure, and scale.

The design knowledge is grounded in **[*Designing Data-Intensive Applications*][ddia]** by
Martin Kleppmann (O'Reilly, 2017). References throughout cite its chapters and pages.

---

## Why this exists

Agents are good at producing code that works on the happy path in a single-threaded test.
They are much worse at the things that cause real incidents:

- read-modify-write races that silently lose updates
- check-then-act logic that double-books, double-charges, or oversells
- migrations that break during the rolling deploy window
- retries without idempotency that duplicate side effects
- dual writes that leave two datastores permanently divergent
- missing timeouts that turn one slow dependency into a total outage
- wall-clock timestamps used to order events across machines

This suite turns those into a **named, detectable hazard catalog** that the planning,
implementation, verification, and review skills all check against.

---

## Structure

```
skills/
├── engineer-workflow/SKILL.md      # Orchestrator: routes phases, owns shared state
├── planner/SKILL.md                # Recon → requirements → phased plan.md
├── detail-planning/SKILL.md        # One phase → implementable spec in executor.md
├── implement/SKILL.md              # Spec → code, with robustness invariants
├── verify/SKILL.md                 # Code vs. spec, with evidence
├── code-review/SKILL.md            # 5 review modes incl. /review-data
└── data-systems-design/
    ├── SKILL.md                    # Decision tables, design record, language discipline
    └── references/
        ├── 01-reliability-scalability-maintainability.md
        ├── 02-data-models.md
        ├── 03-storage-and-retrieval.md
        ├── 04-encoding-and-evolution.md
        ├── 05-replication.md
        ├── 06-partitioning.md
        ├── 07-transactions.md
        ├── 08-distributed-systems-faults.md
        ├── 09-consistency-and-consensus.md
        ├── 10-batch-processing.md
        ├── 11-stream-processing.md
        ├── 12-correctness-and-integrity.md
        └── hazard-catalog.md       # 48 named hazards: signature → consequence → fix
```

Each skill is a self-contained folder with a `SKILL.md`, which is the layout Claude Code,
Cursor, and compatible agents expect.

---

## Installation

Copy the skill folders into your agent's skills directory:

```bash
# Claude Code / Cursor (user-level)
cp -r skills/* ~/.claude/skills/
cp -r skills/* ~/.cursor/skills/

# Project-level
cp -r skills/* .claude/skills/
```

On Windows (PowerShell):

```powershell
Copy-Item -Recurse -Force skills\* $HOME\.cursor\skills\
```

Skills are discovered automatically by name. `data-systems-design` must be installed for
the other skills' hazard scans to resolve their references.

---

## The pipeline

```
PLANNING → DESIGN GATE → DETAIL PLANNING → IMPLEMENT → VERIFY → REVIEW
 plan.md    plan.md##Design   executor.md      code      report   findings
```

| Command | What happens |
|---|---|
| `engineer-workflow: [idea]` | Start the full pipeline |
| `planner` | Recon the codebase, quantify requirements, produce phased `plan.md` |
| `design` | Run the data-systems design gate and hazard scan |
| `detail F1` | Expand phase F1 into a spec with contracts, failure modes, rollback |
| `implement F1` | Write the code, enforcing the robustness invariants |
| `verify F1` | Compare code to spec, with file-and-line evidence |
| `/review-diff` | Review the branch vs. main, plus PR/CI babysit status |
| `/review-uncommitted` | Catch work that looks finished but isn't |
| `/review` | Full repository audit |
| `/review-inscope` | Check the change against the stated task |
| `/review-data` | Deep data/concurrency/distribution hazard audit |

Each phase stops when it's done and waits for you. One phase per cycle, by design.

### Example

```
> engineer-workflow: add a wallet with balance top-ups via Stripe

  → Planning: recon, load parameters, fault model, phases F1–F4
  → Design gate triggers (money + persistence + external side effects):
      INV-1  balance never negative  → CHECK constraint + atomic UPDATE
      INV-2  one charge per request  → unique index on idempotency_key
      H-14 non-idempotent retry, H-06 side effect in transaction → mitigated in F2
  → STOP

> detail F2
  → Spec: files, signatures, failure-mode table, idempotency contract,
    isolation level, migration compatibility matrix, observability, rollback
  → STOP

> implement F2
  → Code + concurrency tests, validation run, deviations logged
  → STOP

> verify F2
  → ✗ FAIL: dedup uses a pre-flight SELECT outside the transaction (H-03/H-14)
    with the exact fix
```

---

## What the design gate produces

For anything touching persistence, concurrency, distribution, money, auth, or PII:

- **Requirements** as numbers — load parameters, p50/p99 (never averages, never averaged
  percentiles), RPO/RTO, and a consistency level chosen for a stated user-visible reason
- **A fault model** — which faults are tolerated, the resulting behavior, and blast radius
- **Decisions with rejected alternatives** — data model, storage engine, replication,
  partition key, isolation level, delivery semantics
- **Invariants with enforcement mechanisms** — "the application checks it first" is not a
  mechanism under concurrency
- **An evolution plan** — expand → migrate → contract, with a rollback path
- **"What breaks at 10x"** — the specific resource that saturates first
- **A hazard scan** against the catalog

It also enforces **language discipline**: phrases like "eventually consistent",
"we'll retry", "exactly once", "we'll keep them in sync", "add a cache", and
"use a distributed lock" are rejected unless accompanied by the actual mechanism.

---

## The hazard catalog

48 named hazards, each with a **detection signature** (what to grep for), a
**consequence**, and a **required fix** — grouped as:

| Group | Examples |
|---|---|
| A. Concurrency & transactions | H-01 lost update · H-02 write skew · H-03 uniqueness in app code · H-06 side effect in transaction |
| B. Schema & API evolution | H-09 breaking change in one deploy · H-11 unknown-field dropping · H-12 blocking DDL |
| C. Distributed calls & retries | H-14 non-idempotent retry · H-15 missing timeout · H-17 nested retries |
| D. Clocks, locks, leadership | H-20 wall-clock ordering · H-22 lock without fencing token · H-24 cross-channel race |
| E. Replication & partitioning | H-25 read-after-write from a replica · H-29 hot partition key · H-30 `hash mod N` |
| F. Streams, queues, derived data | H-32 dual write · H-33 unbounded queue · H-36 processing-time windowing |
| G. Reliability & operability | H-41 averages instead of percentiles · H-43 N+1 · H-44 cache without invalidation |

The catalog is deliberately shared: `code-review` scans a diff with it, `verify` scans an
implementation, and `planner` scans a proposed design — so the same defect is caught at
whichever stage it appears.

---

## Design principles

**Proportionality.** Every skill has an explicit anti-over-engineering rule. A copy change
gets a three-bullet micro-plan and no hazard scan; a new source of truth gets the full
design record. Applying distributed-systems ceremony to a single-file fix is itself a
failure mode.

**Evidence over assertion.** Verification and review findings must cite `file:line`.
"No critical issues found" is a valid, valuable result; padding a review with invented nits
trains people to ignore reviews.

**Mechanisms over intentions.** Every invariant names the thing that enforces it. Every
retry names its idempotency key. Every cache names its invalidation trigger.

**Stop and ask.** Ambiguity that changes the design, irreversible operations, and
spec/code conflicts stop the pipeline rather than getting a guess.

---

## Using skills independently

Nothing requires the full pipeline:

- `code-review` reads the codebase and git state directly — no `plan.md` needed.
- `data-systems-design` answers design questions on its own ("should this be
  serializable?", "is this partition key safe?").
- `planner` is useful alone for turning a vague request into a grounded plan.

---

## Migrating from the previous layout

Skills previously lived as flat files at the repository root. They are now folders under
`skills/`, matching the standard agent-skill layout:

| Before | After |
|---|---|
| `engineer-workflow.md` | `skills/engineer-workflow/SKILL.md` |
| `planner.md` | `skills/planner/SKILL.md` |
| `detail_planning.md` | `skills/detail-planning/SKILL.md` |
| `implement.md` | `skills/implement/SKILL.md` |
| `verify.md` | `skills/verify/SKILL.md` |
| `code-review.md` | `skills/code-review/SKILL.md` |
| — | `skills/data-systems-design/` (new) |

If you installed the old flat files, remove them before installing the new folders so the
agent does not load two versions of the same skill.

---

## Dependencies

None. These are Markdown skill definitions for AI agents.

## Contributing

Fork, branch, PR. New hazards are welcome — follow the catalog's format
(signature → consequence → fix) and cite the source of the failure mode.

## License

MIT

## Attribution

Design concepts, terminology, and page references come from
[*Designing Data-Intensive Applications*][ddia] by Martin Kleppmann (O'Reilly, 2017).
This repository contains original prose applying those concepts to agent workflows; it is
not a reproduction of the book, and reading the book is still strongly recommended.

[ddia]: https://dataintensive.net/
