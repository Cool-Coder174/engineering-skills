# Engineering Skills

A suite of Agent Skills that make an AI coding agent behave like a senior engineer: clarify
the problem before designing, size it in numbers before choosing an architecture, make
design decisions explicitly, implement with robustness invariants, verify against the spec,
and review for the failure modes that only appear under concurrency, failure, and scale.

Grounded in five books:

| Book | What it contributes |
|---|---|
| **[*Designing Data-Intensive Applications*][ddia]** — Kleppmann | Correctness under concurrency, replication, and failure; the 48-hazard catalog |
| **[*System Design Interview: An Insider's Guide*][xu]** — Alex Xu | The design method, back-of-the-envelope estimation, the scaling ladder |
| **[*Grokking the System Design Interview*][grok]** — Design Gurus | Building-block selection; the seven-step design process |
| **[*Database Internals*][dbi]** — Alex Petrov | Database selection methodology, storage engine trade-offs, the RUM conjecture |
| **[*Code Simplicity*][cs]** — Max Kanat-Alexander | The laws of software design; the anti-over-engineering discipline |

---

## Why this exists

Agents are good at producing code that works on the happy path in a single-threaded test.
They are much worse at two things that cause real problems.

**They skip the design process.** They answer before clarifying, propose architectures with
no numbers behind them, and reach for Kafka at 40 writes per second. The estimation gate and
the simplicity pass exist to stop both.

**They miss the failure modes that only appear under concurrency and scale:**

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
├── planner/SKILL.md                # Recon → requirements → estimates → gates → phased plan.md
├── detail-planning/SKILL.md        # One phase → implementable spec in executor.md
├── implement/SKILL.md              # Spec → code, with robustness invariants
├── verify/SKILL.md                 # Code vs. spec, with evidence
├── code-review/SKILL.md            # 5 review modes incl. /review-data
├── system-design/                  # WHAT to build
│   ├── SKILL.md                    # 7-step method, design doc, red flags, proportionality
│   └── references/
│       ├── 01-design-method.md             # The steps in depth + question banks
│       ├── 02-estimation.md                # Powers of two, latency numbers, nines, formulas
│       ├── 03-scaling-ladder.md            # Single server → sharding, with triggers
│       ├── 04-building-blocks.md           # LB, cache, CDN, queues, rate limiting, IDs, protocols
│       ├── 05-database-selection.md        # Evaluation method, B-tree vs LSM, RUM conjecture
│       ├── 06-simplicity-and-design-laws.md # Six laws, three flaws, over-engineering tests
│       └── 07-reference-architectures.md   # Fan-out, chat, crawler, file sync, autocomplete…
└── data-systems-design/            # Whether it stays CORRECT
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

**The two design skills split the problem deliberately.** `system-design` decides *what to
build* — scope, capacity, components, trade-offs. `data-systems-design` decides *whether it
stays correct* — isolation, replication, ordering, hazards. A new service runs both; adding
a retry to an existing call runs only the second.

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
PLANNING → ARCHITECTURE GATE → DESIGN GATE → DETAIL PLANNING → IMPLEMENT → VERIFY → REVIEW
 plan.md   plan.md##Architecture  plan.md##Design  executor.md     code      report   findings
```

| Command | What happens |
|---|---|
| `engineer-workflow: [idea]` | Start the full pipeline |
| `planner` | Recon the codebase, quantify requirements, estimate capacity, produce phased `plan.md` |
| `design` / `system design` | Run the 7-step design method, estimation, and simplicity pass |
| `estimate` | Just the back-of-the-envelope numbers |
| `data design` | Run the data-systems design gate and hazard scan |
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

  → Planning: recon, requirements, explicit non-goals, phases F1–F4
  → Estimation: 40 top-ups/min peak, 2 KB/row, 1.2 GB/yr
      ⇒ single Postgres primary has years of headroom; no queue, no sharding
  → Architecture gate: API contract, data model, write path walked end to end,
      trade-off table (ledger table vs. balance column → ledger, for auditability)
      Simplicity pass: Kafka removed (no number justifies it), Redis kept (p99 target)
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

## What the architecture gate produces

For a new system, a new datastore, a new component on the request path, or a scaling change:

- **Requirements** clarified before any solution is proposed, with explicit non-goals and
  labeled assumptions — because answering fast without clarifying is how you build the wrong
  system correctly
- **Capacity estimates** with derivations shown, each row marked measured or estimated, and
  the conclusion the numbers force ("sharding is not justified; the bottleneck at 10× is the
  `events` write path")
- **An interface contract** written before the architecture, which is what makes the
  requirements concrete
- **A data model driven by access patterns**, with a datastore decision record listing the
  rejected alternative, the operability story, and the exit cost
- **Deep dives on the two or three components where the difficulty actually lives** — not
  even attention across every box
- **A bottleneck, failure-mode, and operations table**, including what saturates first, the
  degraded mode per dependency, and the next scale curve
- **A simplicity pass** that removes every component the numbers do not justify

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
failure mode. Kanat-Alexander's version: the quality level of a design should be
proportional to how long the system will keep helping people.

**Numbers before architecture.** No component enters a design without an estimate or a
requirement behind it. "It should scale fine" is not a design, and neither is a diagram with
a queue nobody sized.

**Don't design for a future you can't measure.** The most common and disastrous design error
is predicting something about the future when you cannot know it. Design for measured load
with a stated growth rate; make the next rung of the scaling ladder reachable; don't build
it until a number says so.

**Maintenance cost outweighs implementation cost.** A design that's fast to build and
expensive to operate is a bad design. Every component added is a permanent tax on whoever
is on call.

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
- `system-design` runs standalone for design work: "design a notification service", "will
  this scale?", "should we use Postgres or Cassandra here?", "estimate the storage for this".
  It's also useful as a review lens on an architecture someone else wrote.
- `data-systems-design` answers correctness questions on its own ("should this be
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
| — | `skills/system-design/` (new) |

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

Concepts, terminology, methodology, and reference numbers come from:

- [*Designing Data-Intensive Applications*][ddia] — Martin Kleppmann (O'Reilly, 2017).
  Page references throughout `data-systems-design` refer to that edition.
- [*System Design Interview: An Insider's Guide*][xu] — Alex Xu (2020). The design
  framework, back-of-the-envelope estimation, and the scaling progression.
- [*Grokking the System Design Interview*][grok] — Design Gurus. The seven-step process and
  the building-block catalog.
- [*Database Internals*][dbi] — Alex Petrov (O'Reilly, 2019). Database evaluation
  methodology, storage engine trade-offs, and the RUM conjecture.
- [*Code Simplicity: The Science of Software Development*][cs] — Max Kanat-Alexander
  (O'Reilly, 2012). The laws of software design and the anti-over-engineering discipline.

This repository contains original prose applying those concepts to agent workflows. It is
not a reproduction of any of the books, and reading them is still strongly recommended.

[ddia]: https://dataintensive.net/
[xu]: https://www.systemdesigninsider.com/
[grok]: https://www.designgurus.io/course/grokking-the-system-design-interview
[dbi]: https://www.databass.dev/
[cs]: https://www.codesimplicity.com/
