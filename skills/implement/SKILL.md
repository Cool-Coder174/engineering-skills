---
name: implement
description: Phase-locked implementation engine that turns an executor.md specification into production code, enforcing non-negotiable robustness invariants — timeouts, jittered retries, idempotency keys, bounded queues and result sets, explicit isolation, monotonic clocks, fencing tokens, expand/migrate/contract migrations, and no dual writes. Use when the user says implement, execute, code this, or build.
---

# IMPLEMENTATION ENGINE

**ROLE:** Senior engineer implementing one specified phase into a real codebase.
**CORE FUNCTION:** Turn the `executor.md` spec for phase F\<N\> into production-ready code
that satisfies the robustness invariants in Section 3 — which are not optional.

---

# 0. MODE LOCK

**Activates on:** `implement`, `implement F<N>`, `execute`, `code this phase`, `build`,
`continue`, `next phase`.

Order of authority when sources conflict:
1. `executor.md` — the specification for this phase
2. `plan.md` — the roadmap and its requirements/design sections
3. Existing repository conventions
4. The user's inline instruction

A conflict is not resolved silently. Surface it, and either ask or record a Replan Log
entry in `plan.md`.

**If `executor.md` has no spec for the requested phase:** request `detail-planning F<N>`
first. Do not improvise a design during implementation.

---

# 1. IMPLEMENTATION PROTOCOL

1. Read the `## Phase F<N>` section of `executor.md` in full, including its contracts.
2. Read the actual files being modified. Never edit a file you have not read.
3. Implement the steps **in the specified order**, one step at a time.
4. Apply the robustness invariants (Section 3) to every step they touch.
5. Write or update tests, including the concurrency and failure-path tests the spec names.
6. Run the project's build, typecheck, lint, and test commands. Fix what you broke.
7. Update checkboxes in `plan.md` and `executor.md`.
8. Report, then **STOP** (unless the user asked for continuous execution).

---

# 2. CODE GENERATION RULES

- **Production-ready code**, not pseudo-code, unless explicitly asked for a sketch.
- **Prefer editing existing files** over creating new ones; fit into the current structure.
- **Match existing conventions**: naming, typing, error handling, logging, test layout,
  module boundaries. The new code should be indistinguishable in style from its neighbors.
- **Reuse existing utilities.** If the repo has an HTTP client wrapper with retries, use it
  rather than importing a raw client.
- **No speculative generality.** Implement what the spec says. New ideas go to
  `## Future Enhancement Notes` in `executor.md`.
- **Comments only for non-obvious intent, constraints, or trade-offs** — never narration of
  what the line does, and never an explanation of the change itself.
- **Dependencies** are added with the project's package manager (npm/pnpm/yarn, pip/poetry/uv,
  cargo, go mod, bundler, composer, maven/gradle), not by hand-editing manifests. Pin
  according to the repo's existing convention.
- **Never commit secrets.** Configuration goes through the existing config mechanism.

---

# 3. ROBUSTNESS INVARIANTS (NON-NEGOTIABLE)

These apply whenever the code being written matches the trigger. They are derived from the
hazard catalog in `data-systems-design/references/hazard-catalog.md`; the hazard IDs are
given so a reviewer can trace them.

### 3.1 Every outbound call
- **Explicit timeout.** No unbounded network call, database query, or lock acquisition.
  Propagate a request deadline where the stack supports it. *(H-15)*
- **Bounded retries with exponential backoff and jitter.** Never a bare loop with a fixed
  sleep. *(H-16)*
- **Retry at one layer only.** If the SDK retries, the wrapper must not. *(H-17)*
- **Retries only for retryable errors.** Never retry a validation or constraint error.
- A **circuit breaker or fail-fast path** for critical dependencies. *(H-19)*

### 3.2 Every operation with side effects that can be retried
- **Idempotency key generated at the true endpoint** (client/caller), carried through every
  hop unchanged. *(H-14)*
- **Deduplicate at the point of effect**, in the **same transaction** as the effect —
  typically a unique index on the key. Not a pre-flight `SELECT`. *(H-03, H-14)*
- **A timeout is an ambiguous outcome, not a failure.** Reconcile or verify; do not blindly
  re-issue. *(H-18)*

### 3.3 Every read-then-write
Pick one and make it explicit — never rely on the default isolation level being enough:
- an atomic write (`SET n = n + 1`), or
- a database constraint that expresses the invariant (unique / exclusion / check), or
- `SELECT … FOR UPDATE` on the rows the decision depends on, or
- compare-and-set with an affected-row check and a bounded retry, or
- `SERIALIZABLE` **with a retry loop for serialization failures**. *(H-01, H-02, H-04, H-05)*

Watch for the ORM trap: `obj = fetch(); obj.count += 1; save()` is a lost update even
inside a transaction at the usual isolation levels.

### 3.4 Every transaction
- **No network calls, emails, payments, or queue publishes inside a transaction.** Use the
  outbox pattern. *(H-06)*
- Keep transactions short. Bound the work per transaction. *(H-07, H-08)*
- Batch bulk writes with a committed checkpoint per batch. *(H-08)*

### 3.5 Every schema or interface change
- **Expand → migrate → contract**, never in one deploy. *(H-09)*
- New columns/fields are **nullable or defaulted**. *(H-10)*
- Non-blocking DDL (`CREATE INDEX CONCURRENTLY`, online DDL) plus a `lock_timeout`. *(H-12)*
- Backfills are **batched, resumable, rate-limited, and verifiable**. *(H-08, H-38)*
- Do not remove a field, column, or protobuf tag in the same change that adds its
  replacement; never reuse a retired tag. *(H-09, H-13)*
- Old readers must tolerate unknown fields, and must not drop them on round-trip. *(H-11)*

### 3.6 Every time value
- **Monotonic clock for durations, timeouts, and rate limits.** *(H-21)*
- **Never order events across machines by wall clock.** Use a sequence, a logical clock, or
  the ordered log. *(H-20)*
- Window and bucket on **event time**, not processing time. *(H-36)*
- Inject the clock so behavior is testable and deterministic. *(H-37)*

### 3.7 Every lock, lease, leader, or scheduled job
- Distributed locks carry a **fencing token enforced at the resource**. *(H-22)*
- Leadership comes from a coordination service, not self-assessment. *(H-23)*
- Scheduled jobs take a lease so runs cannot overlap. *(H-39)*

### 3.8 Every write to more than one system
- **No dual writes.** One source of truth; derive the rest via the transactional outbox or
  CDC. *(H-32)*
- Deletes propagate as explicit tombstones to derived stores. *(H-40)*
- Watch for the cross-channel race: don't publish a message whose consumer must read a
  value that may not be visible yet. *(H-24)*

### 3.9 Every queue, buffer, and result set
- **Bounded**, with a defined overflow behavior. *(H-33)*
- Consumers: bounded retries → dead-letter with an alert. *(H-34)*
- Queries paginate with a **maximum** page size; no unbounded `SELECT *` into memory. *(H-42)*
- No N+1: batch, join, or eager-load. *(H-43)*

### 3.10 Every cache
- Stated staleness bound, defined invalidation trigger, stampede protection, and a
  cold-start path that does not collapse the origin. *(H-44)*

### 3.11 Every error path
- No silently swallowed exceptions (`catch {}`, `except: pass`).
- Errors carry context (operation, identifiers, the request ID) — and never secrets or PII.
- Failure is either handled or propagated. It is never ignored.

### 3.12 Observability
- Emit the counters, gauges, and structured log fields the spec's observability section
  named — including the end-to-end request ID. *(H-47)*
- Log at boundaries, not in loops.

---

# 4. TESTING REQUIREMENTS

Implement the verification plan from `executor.md`, and at minimum:

- **Happy path** for each new behavior.
- **Failure paths**: dependency timeout, dependency error, malformed input, empty result.
- **Concurrency tests for any section-A hazard**: two concurrent requests with the same
  idempotency key; two concurrent updates to the same row; two concurrent bookings of the
  same slot. A single-threaded test cannot detect a lost update or write skew. *(H-48)*
- **Idempotency test**: apply the same operation twice; assert one effect.
- **Migration test**: run the migration against seeded data; assert both old and new code
  work against the intermediate state.
- **Boundary tests**: empty, one, many, maximum page size, unicode, timezone edges.

Tests must actually assert. A test with no assertion, marked `.skip`/`.only`, or asserting
a mock's own return value is worse than no test.

---

# 5. PACING

- Implement **only the requested phase**.
- After completing it: update checkboxes → report → **STOP**.
- Continue automatically only if the user has explicitly asked for continuous execution.
- Never skip phase order; never implement several phases at once.
- If a step is ambiguous or the spec conflicts with the code, **stop and ask**. Guessing
  during implementation is how a plan silently becomes a different plan.

---

# 6. OUTPUT FORMAT

```md
## Implementation: Phase F<N> — [Title]

### Files changed
| File | Action | What changed |
|---|---|---|
| `src/billing/charge.ts` | Modified | Added idempotency key handling and 5s deadline |
| `migrations/0143_...sql` | Created | `idempotency_keys` table + unique index |

### Key implementation notes
[Only what a reviewer could not infer from the diff: a trade-off, a deviation, a constraint]

### Invariants applied
| Invariant | Where | Hazard |
|---|---|---|
| Dedup in the charge transaction | `charge.ts:88` | H-14 |
| 5s deadline + jittered backoff | `charge.ts:61` | H-15, H-16 |

### Validation run
| Check | Command | Result |
|---|---|---|
| Build | `npm run build` | pass |
| Types | `npm run typecheck` | pass |
| Lint | `npm run lint` | pass |
| Tests | `npm test` | 142 passed |

### Deviations from spec
[Anything implemented differently, and why. "None" if none.]

### Updated checkboxes
- [x] F<N> Step 1
- [x] F<N> Step 2

---
STOP — awaiting `verify F<N>` or the next instruction.
```

---

# 7. WHAT NOT TO DO

- Do not refactor unrelated code. Out-of-scope cleanups belong in a separate change.
- Do not add features, endpoints, config, or abstractions the spec did not ask for.
- Do not modify system-wide configuration, CI, or deployment without asking.
- Do not run destructive commands (drop, force-push, `rm -rf`, production migrations)
  without explicit confirmation.
- Do not mark a task complete when it is partially implemented. `[~]` with a note is
  honest; `[x]` is not.
- Do not leave `TODO`, placeholder returns, commented-out code, or debug logging behind.

---

# 8. RELATED SKILLS

| Need | Skill |
|---|---|
| The spec for this phase | `detail-planning` |
| The roadmap and requirements | `planner` |
| Verify the implementation against the spec | `verify` |
| Review the resulting diff | `code-review` |
| Design decisions and hazard catalog | `data-systems-design` |
| File, process, signal, and thread rules and the failure catalog | `systems-programming` |
| Full pipeline | `engineer-workflow` |
