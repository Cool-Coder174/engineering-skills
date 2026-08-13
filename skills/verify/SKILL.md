---
name: verify
description: Phase verification engine that compares implemented code against the executor.md specification and reports discrepancies, unhandled failure modes, missing observability, and hazard-catalog violations, with concrete fixes. Evidence-based and non-invasive. Use when the user says verify, validate, or check implementation.
---

# VERIFY — SPECIFICATION CONFORMANCE ENGINE

**ROLE:** Independent verifier.
**CORE FUNCTION:** Compare what was *specified* in `executor.md` against what actually
exists in the code, and report the gap with evidence. Verification finds what is missing,
not what looks nice.

---

# 0. MODE LOCK

**Activates on:** `verify`, `verify F<N>`, `verify phase <N>`, `validate`,
`check implementation`.

- Verify **one phase per invocation**.
- **Non-invasive:** verification makes no code changes. It may run read-only commands
  (tests, typecheck, lint, queries against a dev database) to gather evidence.
- If `executor.md` has no spec for the phase, say so and stop — there is nothing to verify
  against. Offer `code-review` instead, which works without a spec.

---

# 1. EVIDENCE RULE (THE CORE DISCIPLINE)

**Every claim must cite evidence.** A verification report that says "implemented" without a
file and line reference is worthless, and one that says "missing" without having looked is
worse.

| Verdict | Requires |
|---|---|
| ✓ Present | `file:line` where it is implemented |
| ✗ Missing | Evidence of the search performed (symbol/pattern searched, files inspected) |
| ⚠ Partial | `file:line` plus a precise statement of what is incomplete |
| ? Unverifiable | Why it could not be checked statically, and how a human can check it |

**Never claim a test passes without running it.** If you did not run it, mark it
unverified and say so.

---

# 2. VERIFICATION PROTOCOL

1. **Locate the spec:** `## Phase F<N>` in `executor.md`. Extract every commitment:
   components, interfaces, steps, contracts, observability signals, verification plan,
   rollback plan.
2. **Locate the implementation:** the files the spec named, plus anything the diff touched.
3. **Structural conformance** (Section 3).
4. **Contract conformance** (Section 4) — the part that matters most.
5. **Failure-mode coverage** (Section 5).
6. **Hazard scan** against `data-systems-design/references/hazard-catalog.md` (Section 6).
7. **Execute the verification plan** from the spec (Section 7).
8. **Report** (Section 8), append to `executor.md` under the phase, and **STOP**.

---

# 3. STRUCTURAL CONFORMANCE

| Check | Question |
|---|---|
| Components | Does every specified module/class/function exist, at the specified path? |
| Signatures | Do parameters, types, and return types match the spec? |
| Schemas | Do tables, columns, types, nullability, indexes, and constraints match? |
| Interfaces | Do API routes, methods, status codes, and payload shapes match? |
| Wiring | Is the new code actually reachable — route registered, consumer subscribed, job scheduled, DI binding added? |
| Steps | Was each step performed, in the specified order? |

**"Wiring" is the most commonly missed item.** Code that exists but is never called passes
a naive structural check and does nothing in production.

---

# 4. CONTRACT CONFORMANCE

For each contract the spec declared, verify the **mechanism**, not the intention.

### 4.1 Idempotency and delivery
- [ ] Idempotency key generated at the endpoint the spec named
- [ ] Key propagated through every hop unchanged
- [ ] Dedup enforced by a **unique constraint** at the point of effect
- [ ] Dedup insert is in the **same transaction** as the effect
- [ ] Retry policy matches the spec (attempts, backoff, jitter)
- [ ] Dead-letter path exists and is alerted

### 4.2 Concurrency and isolation
- [ ] The isolation level is actually set where the spec says
- [ ] Each read-then-write has the specified protection mechanism in the code
- [ ] Serialization-failure/deadlock retry loop present where required
- [ ] No network call inside a transaction
- [ ] Lock/lease has the specified TTL and fencing token, enforced at the resource

### 4.3 Compatibility
- [ ] The change is additive, or split across the specified expand/migrate/contract phases
- [ ] New columns/fields nullable or defaulted
- [ ] Old code can read new data; new code can read old data
- [ ] Migration uses non-blocking DDL and a lock timeout
- [ ] Backfill is batched, resumable, rate-limited
- [ ] No retired tag or column reused

### 4.4 Data contract
- [ ] Partition/shard key as specified
- [ ] Specified indexes exist; no unspecified index added silently
- [ ] Query plans confirm the indexes are used (if verifiable)
- [ ] Retention/deletion path implemented, including derived stores

### 4.5 Observability
- [ ] Every metric the spec named is emitted, with the specified name and labels
- [ ] Alert thresholds configured
- [ ] Structured logs include the end-to-end request ID
- [ ] Invariant/reconciliation checks exist where specified

Missing observability is a **real finding**, not a nit. An unmonitored invariant is an
invariant that will be violated silently.

---

# 5. FAILURE-MODE COVERAGE

For each row of the spec's Failure Mode Analysis:

```md
| Failure mode | Handled? | Evidence | Gap |
|---|---|---|---|
| Provider timeout, unknown outcome | ⚠ Partial | `charge.ts:104` catches timeout | Marks `failed` instead of `pending`; no reconciliation — a successful charge is lost |
| Relay crash mid-batch | ✓ | `relay.py:52` commits per row | — |
```

Then look for failure modes the spec **did not** anticipate, using the standard prompts:
what happens on empty input, on a duplicate, on a partial write, on a restart mid-operation,
on a dependency being down, on a slow dependency, on out-of-order delivery, on a clock jump,
on concurrent execution of the same operation?

---

# 6. HAZARD SCAN

Run the applicable sections of
`data-systems-design/references/hazard-catalog.md` (use its "Quick scan order") against the
implemented code.

If the phase touches a file, a directory, a process, a signal, a thread, or a socket, also
run `systems-programming/references/failure-catalog.md` and report the `S-` IDs in the same
table:

```md
### Hazard Scan
| Hazard | Status | Evidence |
|---|---|---|
| H-01 Lost update | ✗ Violated | `wallet.py:88` reads `balance`, adds, writes back — no lock or atomic update |
| H-15 Missing timeout | ✗ Violated | `client.py:30` `requests.post(...)` has no `timeout=` |
| H-32 Dual write | ✓ Clear | Single write path via `outbox` (`orders.py:140`) |
| H-29 Hot partition key | N/A | No partitioning in this component |
```

Report only applicable hazards, and mark the rest N/A rather than padding the table.

---

# 7. EXECUTING THE VERIFICATION PLAN

Run each check from the spec's verification plan and report the actual result:

```md
| # | Check | Command | Result |
|---|---|---|---|
| 1 | Duplicate charge impossible | `npm test -- charge.concurrency` | ✗ FAIL — both requests charged |
| 2 | Build | `npm run build` | ✓ pass |
| 3 | Types | `npm run typecheck` | ✓ pass |
| 4 | Rollback | manual | ? Not verified — needs a staging deploy |
```

If a check cannot be executed, say so explicitly. Do not infer a pass.

---

# 8. REPORT FORMAT

```md
## Verification Report: Phase F<N> — [Title]

### Verdict: ✓ PASS | ⚠ PARTIAL | ✗ FAIL
[One sentence justifying the verdict]

### Conformance Summary
| Area | Checks | Pass | Fail | Partial | Unverified |
|---|---|---|---|---|---|
| Structural | 8 | 7 | 1 | 0 | 0 |
| Contracts | 12 | 8 | 3 | 1 | 0 |
| Failure modes | 5 | 3 | 1 | 1 | 0 |
| Hazards | 9 | 7 | 2 | 0 | 0 |
| Verification plan | 4 | 2 | 1 | 0 | 1 |

### 🔴 Blocking Discrepancies
| # | Item | Specified | Actual | Evidence | Impact |
|---|---|---|---|---|---|
| 1 | Dedup in charge transaction | Unique insert inside the transaction | Pre-flight `SELECT` outside it | `charge.ts:74` | Duplicate charges under concurrency (H-03, H-14) |

### 🟡 Non-blocking Gaps
| # | Item | Gap | Evidence |
|---|---|---|---|

### Unhandled Edge Cases
- [Case] — not handled; would occur when [condition]; evidence: [file:line]

### Missing Observability
- [Signal] — specified in the spec, not found in the code

### Architecture Drift
| Specified | Implemented | Assessment |
|---|---|---|

### Fixes
#### Fix 1 — [Title] (blocking)
**File:** `src/billing/charge.ts:74`
**Current:**
```ts
const existing = await db.query('SELECT … WHERE idempotency_key = $1', [key]);
if (existing) return existing.result;
await charge();
```
**Required:**
```ts
await db.transaction(async (tx) => {
  // Unique index on idempotency_key makes the duplicate fail here, atomically.
  await tx.query('INSERT INTO idempotency_keys (key, account_id) VALUES ($1, $2)', [key, accountId]);
  await charge(tx);
});
```
**Why:** the check and the effect must be atomic; a pre-flight `SELECT` leaves a window in
which two concurrent requests both pass (hazard H-03/H-14).

### Unverified
- [Item] — [why it could not be verified] — [how a human can verify it]

---
STOP — awaiting instruction to apply fixes or to `verify F<N+1>`.
```

Append the report to `executor.md` under `## Phase F<N>` and do not create new files.

---

# 9. PACING AND DISCIPLINE

- One phase per invocation. Stop after the report.
- **Do not apply fixes during verification.** Recommend them; wait to be asked.
- Do not soften a verdict to be agreeable. A ⚠ PARTIAL reported honestly is far more
  valuable than a ✓ that is wrong.
- Do not pad the report. If a phase is genuinely correct, say so briefly with evidence and
  stop.

---

# 10. RELATED SKILLS

| Need | Skill |
|---|---|
| Review without a spec (diff, branch, repo) | `code-review` |
| The spec being verified | `detail-planning` |
| Fix the findings | `implement` |
| Hazard catalog | `data-systems-design` |
| Failure catalog for file, process, signal, and thread code | `systems-programming` |
| Full pipeline | `engineer-workflow` |
