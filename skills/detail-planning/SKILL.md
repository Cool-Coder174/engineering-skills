---
name: detail-planning
description: Expands one phase of plan.md into a deterministic, file-referenced implementation spec in executor.md — including failure-mode analysis, idempotency and retry contracts, schema compatibility matrix, isolation levels, observability, and rollback. Use when the user says detail, detail planning, breakdown, or expand phase.
---

# DETAIL PLANNING — PHASE EXPANSION ENGINE

**ROLE:** Staff engineer turning a planned phase into a specification precise enough that
implementation requires no further design decisions.
**CORE FUNCTION:** Expand exactly one phase of `plan.md` into `executor.md`, with every
file, signature, failure mode, and verification named.

---

# 0. MODE LOCK

**Activates on:** `detail-planning`, `detail_planning`, `detail F<N>`, `expand phase F<N>`,
`breakdown F<N>`, `spec out F<N>`.

- Expand **exactly one phase per invocation**.
- **No code is written** — this phase produces a specification, not an implementation.
  Signatures, schemas, and interface definitions are permitted; function bodies are not.
- If `plan.md` does not exist, request `planner` first.

---

# 1. DOCUMENTATION POLICY

`executor.md` is the single documentation artifact for detailed specs. Content that would
otherwise go into a separate design doc, schema doc, or notes file is appended into
`executor.md` under the correct section, annotated:

> *(Redirected from attempted `[filename]`)*

Rationale: scattered planning documents drift out of sync and each one becomes a second
source of truth. `plan.md` is the roadmap authority; `executor.md` is the specification
authority.

> **Scope note:** this policy governs *documentation the agent generates*. It does not
> forbid creating source files, migrations, tests, or config during the `implement` phase,
> and it does not override a repository's existing documentation conventions (ADRs,
> `docs/`) when the user asks for them.

---

# 2. EXPANSION PROTOCOL

1. **Locate the phase** in `plan.md`. Restate its goal.
2. **Load context:** the phase's tasks, the Requirements and Design sections of `plan.md`,
   and the actual files the phase touches — read them, do not assume their contents.
3. **Check the design gate.** If this phase touches persistence, concurrency, distribution,
   or external side effects, load `data-systems-design` and carry its Design Record
   decisions into this spec.
4. **Check the security gate.** If this phase touches identity, authorization, secrets,
   cryptography, untrusted input, or a new reachable surface, load `security-engineering` and
   carry the plan's Security Record into this spec — **each named mechanism becomes a step at a
   named enforcement point**, not a note. A mechanism with no step does not get built.
5. **Decompose into ordered steps**, each mapped to real files.
6. **Analyze failure modes per step** (Section 4) — this is the part that is normally
   skipped and is the reason implementations fail in production.
7. **Define contracts** (Section 5): idempotency, retry, isolation, compatibility, and the
   security contract.
8. **Define observability and verification** (Section 6).
9. **Define rollback** (Section 7).
10. **Append to `executor.md`** under `## Phase F<N>: [Name]`.
11. **STOP.**

---

# 3. OUTPUT FORMAT

```md
## Phase F<N>: [Title] — Specification

### Goal
[One sentence. What is true after this phase that was not true before.]

### Components
| Component | Path | Type | Action |
|---|---|---|---|
| [Name] | `src/services/billing.ts` | class | Modify |
| [Name] | `migrations/0143_add_idempotency_keys.sql` | migration | Create |

### Interfaces
[Exact signatures, types, schemas, API contracts, message payloads — declarations only]

### Steps (ordered)

#### Step 1 — [Title]
- **File:** `path/to/file.ext`
- **Action:** [Precise change]
- **Depends on:** [Prior step, or "none"]
- **Failure modes:** [Section 4 table row references]
- **Done when:** [Observable/executable criterion]

#### Step 2 — [Title]
…

### Failure Mode Analysis
[Section 4 — including the hazard scan, and the vulnerability scan if the security gate applies]

### Contracts
[Section 5 — including the security contract (5.5) if the security gate applies]

### Observability
[Section 6]

### Verification
[Section 6.2]

### Rollback
[Section 7]

### Risks
| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|

### Out of Scope
- [Explicitly deferred items → recorded under `## Future Enhancement Notes`]
```

---

# 4. FAILURE MODE ANALYSIS (MANDATORY FOR NON-TRIVIAL PHASES)

For each step that performs I/O, mutates state, or crosses a process boundary:

```md
| Step | What can fail | Detection | Behavior | Recovery |
|---|---|---|---|---|
| 3 | Payment provider times out (outcome unknown) | 5s deadline exceeded | Do NOT re-charge; mark `pending`, reconcile via provider lookup | Reconciliation job resolves within 15m |
| 4 | Outbox relay crashes mid-batch | Consumer lag alert | Rows stay unsent; at-least-once redelivery | Consumers are idempotent on `event_id` |
| 5 | Backfill interrupted | Checkpoint row stale > 10m | Resume from checkpoint | No manual intervention |
```

Then run the **hazard scan** for this phase against
`data-systems-design/references/hazard-catalog.md`, using its "Quick scan order":

```md
### Hazard Scan (this phase)
| Hazard | Applies | Mitigation in this spec |
|---|---|---|
| H-14 Non-idempotent retry | Yes | Step 3 idempotency key; unique index in migration (Step 1) |
| H-32 Dual write | Yes | Step 4 uses the outbox; no direct index write |
| H-29 Hot partition key | No | Partitioned by `tenant_id`, evenly distributed |
```

Then, if the security gate applies to this phase, run the **vulnerability scan** against
`security-engineering/references/vulnerability-catalog.md`, using its "Quick scan order". The
mitigation column must name a step, so the spec is what gets built:

```md
### Vulnerability Scan (this phase)
| Vulnerability | Applies | Mitigation in this spec |
|---|---|---|
| V-43/V-44 Missing or incomplete mediation | Yes | Step 2 adds the tenant predicate in the repository, not the handler; Step 6 negative test asserts cross-tenant 404 |
| V-25 No replay protection | Yes | Step 4 rejects timestamps older than 5m and caches seen `event_id` for the window |
| V-14 Secret in source | Yes | Step 1 reads the signing key from the secret store; no default value in config |
| V-09 Weak randomness | No | No tokens or identifiers generated in this phase |
```

---

# 5. CONTRACTS (SPECIFY WHAT APPLIES; MARK THE REST N/A)

### 5.1 Idempotency and delivery
- **Idempotency key:** [what it is, who generates it, where it is stored]
- **Dedup point:** [where the duplicate is rejected — must be at the point of effect]
- **Delivery semantics:** at-most-once / at-least-once + idempotent (effectively-once)
- **Retry policy:** max attempts, base delay, backoff factor, jitter, retry budget
- **Terminal behavior:** dead-letter destination, alert, owner, replay procedure

### 5.2 Concurrency and isolation
- **Isolation level per transaction:** [read committed / snapshot / serializable]
- **Protection mechanism for each read-then-write:** atomic operation, unique constraint,
  `SELECT … FOR UPDATE`, CAS with retry, or serializable + retry loop
- **Retry-on-serialization-failure:** [present / N/A]
- **Lock/lease:** scope, TTL, fencing token, behavior on mid-operation expiry
- **Transaction boundaries:** what is inside; **confirm no network calls are inside**

### 5.3 Schema and interface compatibility
```md
| Change | Backward compatible? | Forward compatible? | Phase (expand/migrate/contract) |
|---|---|---|---|
| Add `idempotency_key` nullable | Yes | Yes | Expand |
| Backfill existing rows | N/A | N/A | Migrate |
| Add NOT NULL constraint | No — requires backfill complete | Yes | Contract (later deploy) |
```
Plus: migration lock strategy (`CONCURRENTLY` / online DDL / `lock_timeout`), batch size,
and resume mechanism for backfills.

### 5.4 Data contract
- Partition/shard key and why it distributes
- Indexes added, and the query plan each one serves
- Expected row/volume growth
- Retention and deletion path, including derived stores

### 5.5 Security contract
- **Trust boundary crossed by this phase:** [what is on each side, and what is assumed about
  the caller]
- **Authentication:** how the caller's identity is established, and by which component
- **Authorization:** the exact predicate, and the **enforcement point** — the layer through
  which the operation cannot be reached without it (not the handler if the repository is
  reachable directly)
- **Untrusted inputs:** each one, its canonical form, and the parser or validator that produces
  it before any decision is made on it
- **Secrets and keys:** where each comes from, its lifetime, and what happens on rotation
- **Freshness/replay:** the window, the identifier cache, and the behavior at the boundary
- **Audit events emitted:** which security events, with which fields (see
  `security-engineering/references/10-intrusion-detection-and-audit.md`)
- **Failure posture:** what happens when the auth, policy, or key dependency is unavailable —
  and the justification if it is anything other than deny

Mark N/A per line rather than deleting the lines; an N/A that someone had to write is a
decision, and a missing line is an omission nobody noticed.

---

# 6. OBSERVABILITY AND VERIFICATION

### 6.1 Observability spec (a phase is not done if you cannot see it working)
```md
| Signal | Type | Name | Alert threshold |
|---|---|---|---|
| Charge attempts by outcome | counter | `billing.charge.total{outcome}` | error ratio > 1% for 5m |
| Outbox relay lag | gauge | `outbox.oldest_unsent_age_seconds` | > 60s |
| Duplicate key rejections | counter | `billing.idempotency.duplicate` | informational |
| Invariant: unsent outbox rows older than 1h | check | reconciliation job | any > 0 → page |
```
Include structured log fields (especially the end-to-end request ID) and trace spans.

### 6.2 Verification plan
```md
| # | Check | Method | Passes when |
|---|---|---|---|
| 1 | Duplicate charge impossible | Integration test issuing the same key twice concurrently | Exactly one charge; second returns the first result |
| 2 | Migration is non-blocking | Run against a seeded copy with concurrent writes | No lock wait > 1s |
| 3 | Consumer is idempotent | Replay the last 1,000 events | No change in derived state |
| 4 | Rollback works | Deploy N+1 then revert to N | Application healthy, no data loss |
```

Include the concurrency test explicitly for any hazard in section A of the catalog —
single-threaded tests cannot detect lost updates or write skew.

**Every security mechanism needs a negative test.** A test proving the authorized caller
succeeds says nothing about the unauthorized one; the check may not exist. For each entry in the
vulnerability scan, add a check that the wrong principal, the stale timestamp, the tampered
payload, or the cross-tenant identifier is *refused*.

---

# 7. ROLLBACK PLAN (MANDATORY)

```md
- **Revert method:** [flag flip / redeploy previous / down migration]
- **Data reversibility:** [fully reversible / reversible with loss of X / irreversible after step N]
- **Point of no return:** [the step after which rollback requires a restore]
- **Rollback verification:** [how you confirm the revert worked]
```

If a phase is irreversible past a point, that fact belongs in `plan.md` too, so nobody
discovers it during an incident.

---

# 8. STEP GRANULARITY

- A step should be independently reviewable — roughly one commit's worth.
- If a step needs more than ~5 sub-actions, split it.
- Every step names real paths. No placeholders like `path/to/service`.
- Steps are ordered so that **the system is deployable after each one**.

---

# 9. PACING

- **One phase per invocation.** No lookahead into the next phase.
- Re-running detail-planning on the same phase **overwrites only that phase's section** in
  `executor.md`.
- Out-of-scope ideas go to `## Future Enhancement Notes` — never into the current spec.
- After output: **STOP**, and await `implement F<N>`.

---

# 10. RELATED SKILLS

| Need | Skill |
|---|---|
| Create or update the roadmap | `planner` |
| Implement this spec | `implement` |
| Check the implementation against this spec | `verify` |
| Review the resulting diff | `code-review` |
| Data/distribution decisions and hazard catalog | `data-systems-design` |
| File, process, signal, and thread decisions and the failure catalog | `systems-programming` |
| Threat models, security decisions, and vulnerability catalog | `security-engineering` |
| Full pipeline | `engineer-workflow` |
