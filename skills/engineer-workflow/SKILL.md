---
name: engineer-workflow
description: Unified engineering pipeline orchestrating Planning, Design, Security, Detail Planning, Implementation, Verification, and Review as one navigable workflow with persistent state in plan.md/executor.md and history tracking. Supports greenfield and brownfield projects. Use when the user wants an end-to-end workflow rather than a single phase.
---

# ENGINEER WORKFLOW — UNIFIED PIPELINE

**ROLE:** Development orchestrator.
**CORE FUNCTION:** Run the full lifecycle — idea → plan → design → spec → code → verify →
review — with context preserved across phases and no phase silently skipped.

This skill is a **router and state machine**. Each phase's rules live in its own skill;
this file decides which phase runs, enforces the gates between them, and owns the shared
state. It deliberately does not duplicate the component skills' contents.

---

# 1. PIPELINE

```
┌──────────┐  ┌────────────┐  ┌────────┐  ┌──────────┐  ┌────────────────┐  ┌───────────┐  ┌────────┐  ┌────────┐
│ PLANNING │─▶│ ARCHITECT  │─▶│ DESIGN │─▶│ SECURITY │─▶│ DETAIL PLANNING│─▶│ IMPLEMENT │─▶│ VERIFY │─▶│ REVIEW │
│  (F1…)   │  │    gate    │  │  gate  │  │   gate   │  │   (spec F<N>)  │  │  (F<N>)   │  │ (F<N>) │  │ (diff) │
└──────────┘  └────────────┘  └────────┘  └──────────┘  └────────────────┘  └───────────┘  └────────┘  └────────┘
      │             │              │            │               │                 │             │           │
      ▼             ▼              ▼            ▼               ▼                 ▼             ▼           ▼
   plan.md   plan.md          plan.md      plan.md         executor.md      source code   executor.md  findings
             ##Architecture   ##Design     ##Security
      └────────────────────────── History/ (archived at plan creation) ───────────────────────────────────┘
```

Iteration is allowed from any phase back to any earlier one. Skipping *forward* is not.

## 1.1 Phase routing

| Phase | Triggers | Delegates to |
|---|---|---|
| Planning | `plan`, `planner`, `analyze`, a new idea | `planner` |
| Architecture gate | automatic (Section 3.1), or `design`, `system design` | `system-design` |
| Design gate | automatic (Section 3.2) | `data-systems-design` |
| Systems gate | automatic (Section 3.3), or `systems`, `low level` | `systems-programming` |
| Security gate | automatic (Section 3.4), or `security design` | `security-engineering` |
| Detail planning | `detail F<N>`, `expand F<N>`, `breakdown F<N>` | `detail-planning` |
| Implementation | `implement F<N>`, `execute`, `build`, `continue` | `implement` |
| Verification | `verify F<N>`, `validate` | `verify` |
| Review | `/review-diff`, `/review-uncommitted`, `/review`, `/review-inscope`, `/review-data`, `/review-security` | `code-review` |

Load the component skill and follow it. Do not re-derive its rules from memory.

---

# 2. STATE

| Artifact | Owner phase | Authority for |
|---|---|---|
| `plan.md` | Planning | Roadmap, requirements, capacity estimates, architecture, design record, security record, phase status |
| `executor.md` | Detail planning | Per-phase specs, verification reports |
| `History/[slug]-[timestamp].md` | Planning | Immutable archive of the plan at creation, plus the execution log |
| Source code | Implementation | Actual behavior |

Rules:
- `plan.md` is the roadmap authority; `executor.md` is the specification authority; the code
  is the truth. When they disagree, that gap **is** the finding.
- Each phase reads the previous phase's artifact rather than reconstructing context.
- Every phase transition updates status in `plan.md`.
- The `History/` copy is written once at plan creation and never rewritten; the execution
  log table inside it is appended to.

### 2.1 Execution log (in the History file)
```md
## Execution Log
| Timestamp | Phase | Action | Result |
|---|---|---|---|
| 2026-08-12T10:14Z | Planning | Plan created (F1–F4) | ✓ |
| 2026-08-12T10:40Z | Architecture | 7-step design + estimates + simplicity pass | ✓ Kafka removed, cache kept |
| 2026-08-12T11:02Z | Design | Design record + hazard scan | ✓ 2 hazards mitigated |
| 2026-08-12T11:20Z | Security | Threat model + vulnerability scan | ✓ 3 findings designed out, 1 accepted |
| 2026-08-12T11:40Z | Detail | F1 specified | ✓ |
| 2026-08-12T12:30Z | Implement | F1 implemented | ⚠ 1 deviation logged |
| 2026-08-12T12:55Z | Verify | F1 verified | ✗ 1 blocking discrepancy |
```

---

# 3. THE GATES (WHAT MAKES THIS PIPELINE DIFFERENT)

Between Planning and Detail Planning sit three gates. They ask different questions and are
triggered independently:

- **Architecture gate — *what should we build?*** Structure, components, capacity, trade-offs.
- **Design gate — *will it stay correct?*** Concurrency, failure, consistency, hazards.
- **Security gate — *what does an attacker get?*** Adversaries, trust boundaries, vulnerabilities.

A new public service triggers all three. Adding a retry to an existing call triggers only the
second. Adding an authenticated endpoint triggers only the third.

**The design gate and the security gate are not the same question.** One assumes faults are
random; the other assumes they are chosen. A queue consumer that is idempotent under duplicate
delivery can still be a replay vulnerability, because idempotency answers "did this happen
twice" and not "did the right party ask for it."

## 3.1 Architecture gate

**Applies when the work involves:** a new service or system · a new datastore · a new
component on the request path (cache, queue, CDN, load balancer, search index) ·
a re-architecture · a migration between stores · a change to how the system scales ·
a stated capacity or scale target.

**If it applies:** load `system-design` and run its seven-step method at the depth its
proportionality table prescribes. Produce the capacity estimates, the interface contract,
the data model with access patterns, the high-level design with both primary flows walked
end to end, the deep dives, the bottleneck/failure/operations table, and the trade-off
table with rejected alternatives. Then run the **simplicity pass** and remove what the
numbers do not justify. Write it into `plan.md` under `## Architecture`.

**The ordering rule:** estimates before architecture. A component chosen before the numbers
exist is a preference, not a decision — and this gate is the cheapest place in the pipeline
to delete something nobody needs.

**If it does not apply:** record `Architecture gate: not applicable — [reason]`.

## 3.2 Design gate

**Applies when the work touches any of:** persistence · schema or migration · index · cache ·
queue or event stream · background or scheduled job · replication · partitioning/sharding ·
transaction or concurrency · retry · a write to more than one system · an external API with
side effects · money, auth, or PII · any stated latency, availability, or consistency
requirement.

**If it applies:** load `data-systems-design`, produce its Design Record, run the hazard
scan against the *design*, and write both into `plan.md` under `## Design`. Detail planning
then carries those decisions into each phase spec, implementation enforces them as
invariants, and verification checks them. The gate is what makes the later phases
mechanical rather than improvised.

**If it does not apply:** record one line — `Design gate: not applicable — [reason]` — and
continue. Applying data-systems ceremony to a copy change is its own failure mode.

## 3.3 Systems gate

**Applies when the work touches any of:** a file read, write, copy, move, or delete · a
directory walk or a document ingest · a durability or crash-safety requirement · a process
that the code starts · a signal handler · a daemon or long-lived worker · a thread or shared
memory · a pipe, a FIFO, or a socket · a build, link, or load failure.

**If it applies:** load `systems-programming`. Apply the five rules of the system call
boundary and the file management recipes. Run its failure catalog against the design and
record the `S-` IDs that apply.

This gate catches what the design gate cannot see. The design gate asks whether the *design*
stays correct across machines. This gate asks whether the *code* stays correct against one
kernel. A durable write that omits `fsync` on the directory passes every design review and
still loses the file.

**If it does not apply:** record one line — `Systems gate: not applicable — [reason]`.
## 3.4 Security gate

**Applies when the work touches any of:** authentication · authorization · session or token
handling · a secret, key, or credential · cryptography of any kind · a new endpoint or
externally reachable surface · untrusted input reaching an interpreter or memory operation ·
PII, PHI, or payment data · a new trust boundary or inter-service call · file upload or
download · an outbound request to a caller-influenced destination · logging of sensitive
fields · a dependency on an external identity provider.

**If it applies:** load `security-engineering`, produce its Security Record at the depth its
proportionality table prescribes, run the vulnerability scan against the *design*, and write
both into `plan.md` under `## Security`. Name the adversary and the capability assumed before
naming a mechanism, and state the residual risk being accepted.

Detail planning then carries each named mechanism into the phase spec at a specific enforcement
point, implementation builds it there, and verification checks that the mechanism exists where
the record claimed. **A security decision that is not carried into a phase spec does not get
built** — this is the most common way a threat model becomes decoration.

**If it does not apply:** record one line — `Security gate: not applicable — [reason]`.
Demanding a threat model for a CSS change trains people to skip the gate when it matters.

---

# 4. GREENFIELD VS BROWNFIELD

## 4.1 Greenfield (new idea, no existing plan)
1. Clarify the goal and the explicit non-goals. **Ask before assuming** — for a new system,
   run the requirements question bank in `system-design/references/01-design-method.md`.
2. Run Planning → produce `plan.md` with phases F1…Fn.
3. Run the architecture gate, then the design gate, then the security gate.
4. Archive to `History/`.
5. Present the plan and **stop** for confirmation before any implementation.

For a genuinely new system, most of the work lives in step 3, not step 2 — the phases
follow from the architecture, not the other way around.

## 4.2 Brownfield (existing plan, TODO.md, or existing codebase)
1. **Read the existing artifact** and the code it refers to.
2. **Assess viability** honestly.
3. **Identify gaps** — especially the ones this suite exists to catch: missing
   non-functional requirements, no fault model, no rollback path, no verification criteria,
   unaddressed hazards, no stated adversary.
4. **Restructure into the plan format**, preserving completed work.
5. Recommend which phase to expand first.

```md
## Existing Plan Analysis: [Name]

### Viability: ✓ VIABLE | ⚠ NEEDS REVISION | ✗ NOT VIABLE

### Strengths
- [ ]

### Gaps
| Gap | Impact | Fix |
|---|---|---|
| No rollback for the migration in step 3 | An incident becomes unrecoverable without a restore | Split into expand/migrate/contract |
| Load never quantified | Cannot tell whether the design is adequate | Add load parameters and percentile SLOs |
| Kafka + 3 services for 40 writes/sec | Permanent operational cost with no number behind it | Run the simplicity pass; collapse to a table and a cron job |

### Proposed Phase Structure
| Phase | Current | Proposed | Why |
|---|---|---|---|
```

---

# 5. NAVIGATION

| Command | Action |
|---|---|
| `engineer-workflow: [idea]` | Start the pipeline at Planning |
| `plan` / `planner` | Create or update `plan.md` |
| `design` / `system design` | Run the architecture gate explicitly |
| `estimate` | Run only the capacity estimation step |
| `data design` | Run the design gate explicitly |
| `security design` | Run the security gate explicitly (threat model + vulnerability scan of the *design*) |
| `detail F<N>` | Expand phase N into `executor.md` |
| `implement F<N>` | Implement phase N |
| `verify F<N>` | Verify phase N against its spec |
| `/review-diff` etc. | Run a review mode |
| `/review-security` | Adversarial review of code that exists (distinct from the gate, which reviews the design) |
| `continue` | Advance to the next incomplete step |
| `back` | Return to the previous phase |
| `status` | Show phase status from `plan.md` |
| `history` | Show the execution log |

Iteration is expected: re-plan when scope changes, re-spec a phase, re-implement after
fixes, re-verify after changes. Every iteration is logged.

---

# 6. PACING

- **One phase per cycle.** Complete → update `plan.md` → report → **STOP**.
- Continuous execution happens only when the user explicitly asks for it (`run all phases`,
  `don't stop`), and even then verification runs after each phase.
- **Forbidden:** running the whole pipeline unprompted; implementing without a spec;
  specifying without a plan; proposing an architecture before the estimates exist; skipping
  any gate when it applies; marking a phase complete without its verification criterion
  being met.

## 6.1 Stop-and-ask conditions
Stop and ask the user, rather than guessing, when:
- a requirement is ambiguous in a way that changes the design;
- a scale or capacity assumption cannot be measured and materially changes the architecture;
- the spec conflicts with the code and the correct resolution is not obvious;
- a change is irreversible (data deletion, destructive migration, production action);
- a hazard mitigation would materially change the plan's scope;
- a security finding requires accepting residual risk that a human should own;
- verification fails in a way that suggests the plan, not the code, is wrong.

---

# 7. MULTI-COMPONENT AND DISTRIBUTED WORK

- **Multi-platform (web / mobile / desktop):** one phase per platform for the same feature,
  with the **shared contract specified once** and referenced by each — that contract is
  where compatibility bugs live.
- **Microservices:** one phase per service, plus an explicit phase for the contract change
  itself, ordered so that every intermediate deploy state is valid (consumers before
  producers for additive reads; producers before consumers for new fields).
- **High complexity:** split into sub-phases (F1.1, F1.2) rather than one large phase.
  Add explicit infrastructure phases for observability, rollout, and rollback.

---

# 8. COMPONENT SKILLS

| Skill | Role |
|---|---|
| `planner` | Reconnaissance, requirements, estimation, phased `plan.md` |
| `system-design` | Design method, capacity estimation, building blocks, database selection, simplicity laws |
| `data-systems-design` | Design records, decision tables, hazard catalog |
| `systems-programming` | System call rules, file management recipes, failure catalog |
| `security-engineering` | Threat models, security records, vulnerability catalog |
| `detail-planning` | Per-phase spec in `executor.md` |
| `implement` | Code, with robustness invariants enforced |
| `verify` | Conformance of code to spec, with evidence |
| `code-review` | Diff, working tree, repository, scope, data-hazard, and security reviews |

Each also works standalone; this skill exists to run them as one coherent process with
shared state.
