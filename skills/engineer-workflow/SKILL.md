---
name: engineer-workflow
description: Unified engineering pipeline orchestrating Planning, Design, Detail Planning, Implementation, Verification, and Review as one navigable workflow with persistent state in plan.md/executor.md and history tracking. Supports greenfield and brownfield projects. Use when the user wants an end-to-end workflow rather than a single phase.
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
┌──────────┐   ┌────────┐   ┌────────────────┐   ┌───────────┐   ┌────────┐   ┌────────┐
│ PLANNING │──▶│ DESIGN │──▶│ DETAIL PLANNING│──▶│ IMPLEMENT │──▶│ VERIFY │──▶│ REVIEW │
│   (F1…)  │   │  gate  │   │   (spec F<N>)  │   │  (F<N>)   │   │ (F<N>) │   │ (diff) │
└──────────┘   └────────┘   └────────────────┘   └───────────┘   └────────┘   └────────┘
      │             │                │                 │              │            │
      ▼             ▼                ▼                 ▼              ▼            ▼
   plan.md    plan.md ##Design   executor.md      source code    executor.md   findings
      └──────────────────────── History/ (archived at plan creation) ───────────────────┘
```

Iteration is allowed from any phase back to any earlier one. Skipping *forward* is not.

## 1.1 Phase routing

| Phase | Triggers | Delegates to |
|---|---|---|
| Planning | `plan`, `planner`, `analyze`, a new idea | `planner` |
| Design gate | automatic (Section 3) | `data-systems-design` |
| Detail planning | `detail F<N>`, `expand F<N>`, `breakdown F<N>` | `detail-planning` |
| Implementation | `implement F<N>`, `execute`, `build`, `continue` | `implement` |
| Verification | `verify F<N>`, `validate` | `verify` |
| Review | `/review-diff`, `/review-uncommitted`, `/review`, `/review-inscope`, `/review-data` | `code-review` |

Load the component skill and follow it. Do not re-derive its rules from memory.

---

# 2. STATE

| Artifact | Owner phase | Authority for |
|---|---|---|
| `plan.md` | Planning | Roadmap, requirements, design record, phase status |
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
| 2026-08-12T11:02Z | Design | Design record + hazard scan | ✓ 2 hazards mitigated |
| 2026-08-12T11:40Z | Detail | F1 specified | ✓ |
| 2026-08-12T12:30Z | Implement | F1 implemented | ⚠ 1 deviation logged |
| 2026-08-12T12:55Z | Verify | F1 verified | ✗ 1 blocking discrepancy |
```

---

# 3. THE DESIGN GATE (WHAT MAKES THIS PIPELINE DIFFERENT)

Between Planning and Detail Planning, the workflow **must** decide whether the design gate
applies. It applies if the work touches any of:

persistence · schema or migration · index · cache · queue or event stream · background or
scheduled job · replication · partitioning/sharding · transaction or concurrency · retry ·
a write to more than one system · an external API with side effects · money, auth, or PII ·
any stated latency, availability, or consistency requirement.

**If it applies:** load `data-systems-design`, produce its Design Record, run the hazard
scan against the *design*, and write both into `plan.md` under `## Design`. Detail planning
then carries those decisions into each phase spec, implementation enforces them as
invariants, and verification checks them. The gate is what makes the later phases
mechanical rather than improvised.

**If it does not apply:** record one line — `Design gate: not applicable — [reason]` — and
continue. Applying data-systems ceremony to a copy change is its own failure mode.

---

# 4. GREENFIELD VS BROWNFIELD

## 4.1 Greenfield (new idea, no existing plan)
1. Clarify the goal and the explicit non-goals.
2. Run Planning → produce `plan.md` with phases F1…Fn.
3. Run the design gate.
4. Archive to `History/`.
5. Present the plan and **stop** for confirmation before any implementation.

## 4.2 Brownfield (existing plan, TODO.md, or existing codebase)
1. **Read the existing artifact** and the code it refers to.
2. **Assess viability** honestly.
3. **Identify gaps** — especially the ones this suite exists to catch: missing
   non-functional requirements, no fault model, no rollback path, no verification criteria,
   unaddressed hazards.
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
| `design` | Run the design gate explicitly |
| `detail F<N>` | Expand phase N into `executor.md` |
| `implement F<N>` | Implement phase N |
| `verify F<N>` | Verify phase N against its spec |
| `/review-diff` etc. | Run a review mode |
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
  specifying without a plan; skipping the design gate when it applies; marking a phase
  complete without its verification criterion being met.

## 6.1 Stop-and-ask conditions
Stop and ask the user, rather than guessing, when:
- a requirement is ambiguous in a way that changes the design;
- the spec conflicts with the code and the correct resolution is not obvious;
- a change is irreversible (data deletion, destructive migration, production action);
- a hazard mitigation would materially change the plan's scope;
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
| `planner` | Reconnaissance, requirements, phased `plan.md` |
| `data-systems-design` | Design records, decision tables, hazard catalog |
| `detail-planning` | Per-phase spec in `executor.md` |
| `implement` | Code, with robustness invariants enforced |
| `verify` | Conformance of code to spec, with evidence |
| `code-review` | Diff, working tree, repository, scope, and data-hazard reviews |

Each also works standalone; this skill exists to run them as one coherent process with
shared state.
