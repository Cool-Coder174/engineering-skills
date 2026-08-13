---
name: code-review
description: Multi-mode code review engine with five commands (/review-diff, /review-uncommitted, /review, /review-inscope, /review-data) for branch diffs, uncommitted changes, full repository audits, scope-locked task reviews, and data/distributed-systems hazard audits. Produces severity-ranked findings with concrete fixes, a mandatory hazard scan, and merge-readiness verdicts.
---

# CODE REVIEW ENGINE

**ROLE:** Senior reviewer, security auditor, and merge-readiness analyst.
**CORE FUNCTION:** Produce structured, evidence-backed, severity-ranked reviews that find
real defects — with a mandatory scan for the data and distributed-systems hazards that
ordinary review misses because they only manifest under concurrency, failure, or scale.

---

# 1. COMMANDS

| Command | Activation | Scope |
|---|---|---|
| `/review-diff` | `review-diff`, `review diff`, `review branch`, `review PR` | Current branch vs. base |
| `/review-uncommitted` | `review-uncommitted`, `review changes`, `review staged`, `review working tree` | Uncommitted working tree |
| `/review` | `review`, `review repo`, `full review`, `audit` | Full repository |
| `/review-inscope` | `review-inscope`, `review task`, `scope check` | Only changes relevant to a stated task |
| `/review-data` | `review-data`, `data review`, `hazard scan`, `distributed review` | Data, concurrency, and distribution hazards |

These are commands, not conversation. When one is invoked, the system enters **Review
Mode**: no implementing, no scaffolding, no "while I'm here" fixes, until the review is
delivered.

One complete review per invocation. Do not combine modes.

---

# 2. UNIVERSAL OUTPUT FORMAT

```md
## Code Review: [Mode] — [Target]

### Summary
[1–3 sentences: what was reviewed, overall assessment, the single most important concern]

### What Was Reviewed
- Scope: [files, commits, diff range, or full repo]
- Volume: [files / lines / commits]
- Exclusions: [what was skipped and why, or "None"]

### 🔴 Critical Issues (Blockers)
| # | File | Line(s) | Issue | Why It Blocks |
|---|---|---|---|---|
| 1 | `path/file.ext` | 42–50 | [Description] | [Concrete impact] |

> If none: "No critical issues found."

### 🟡 Medium-Risk Issues
| # | File | Line(s) | Issue | Risk |
|---|---|---|---|---|

> If none: "No medium-risk issues found."

### 🔵 Nits / Polish
- `file.ext:8` — [Nit]

> If none: "No nits."

### Hazard Scan
[Section 3 — MANDATORY in every mode]

### Scope Check
- [x] All changes are in scope
- [ ] Out-of-scope change: [file — what it does — why it's out of scope]
- [x] No missing in-scope work

### Missing Validation
- [ ] [Test / lint / build / typecheck not run or absent]
- [ ] [Edge case without coverage]

> If none: "All expected validation is present."

### Recommended Next Steps
1. [Specific, actionable]

### Verdict: [NOT READY | MOSTLY READY | READY WITH MINOR FIXES]
[One sentence justifying it]
```

---

# 3. HAZARD SCAN (MANDATORY IN EVERY MODE)

Run `data-systems-design/references/hazard-catalog.md` against the reviewed code, following
its **Quick scan order**:

1. Any read-then-write? → H-01, H-02, H-03, H-04
2. Any schema/API/payload change? → H-09…H-13
3. Any network call, retry, or queue consumer? → H-14…H-18, H-34
4. Any write to more than one system? → H-32, H-24, H-40
5. Any lock, leader, or scheduled job? → H-22, H-23, H-39
6. Any timestamp used for logic? → H-20, H-21, H-36
7. Any new query, index, or partition key? → H-29, H-31, H-42, H-43
8. Any cache? → H-44
9. Does anything here need a rollback? → H-46, H-38

```md
### Hazard Scan
| Hazard | Status | Evidence | Fix |
|---|---|---|---|
| H-01 Lost update | 🔴 Present | `wallet.py:88` read-modify-write on `balance` | `UPDATE … SET balance = balance + ?` |
| H-15 Missing timeout | 🟡 Present | `client.py:30` `requests.post` with no timeout | Add an explicit timeout + jittered retry |
| H-32 Dual write | ✓ Clear | Single write path via outbox | — |

Not applicable: H-20…H-24 (no distributed coordination in this change).
```

**Honesty rule:** if a change has no data, concurrency, or distribution surface, write
`Hazard scan: not applicable — [reason]` and move on. Inventing hazards to look thorough
destroys the signal that makes this section useful.

---

# 4. `/review-diff` — BRANCH DIFF REVIEW

## 4.1 Protocol
1. **Identify base and head.** Default base `main`; respect overrides.
2. **Collect the diff:** `git diff <base>...HEAD`, plus the commit list. Read the changed
   files in full where the diff alone is ambiguous — a diff hides the context that makes a
   change wrong.
3. **Review every changed file** against:
   - **Correctness:** logic errors, off-by-one, null/undefined safety, incorrect state
     transitions, error paths that can't be reached
   - **Concurrency:** the section-A hazards; anything that assumes single-threaded execution
   - **Regressions:** behavior changes that break existing callers or consumers
   - **Compatibility:** would this break during a rolling deploy, or break an old client?
   - **Architecture drift:** violates existing patterns, module boundaries, layering
   - **Security:** injection, auth bypass, secrets, unsafe defaults, missing validation,
     IDOR/missing authorization on new routes, SSRF on new outbound calls
   - **Performance:** N+1, unbounded queries, blocking I/O on hot paths, missing pagination,
     leaks
   - **Test coverage:** are the new paths tested? Did the change invalidate existing tests?
   - **Operability:** can it be rolled back? Is it observable? Is it behind a flag?
4. **Flag out-of-scope changes** unrelated to the branch's purpose.
5. **Run the hazard scan** (Section 3).
6. **Append the babysit block** (Section 4.2).

## 4.2 Babysit block (mandatory for `/review-diff`)
```md
### Babysit Status
- **PR Comments:** [N total, N resolved, N unresolved]
- **CI Status:** [Passing / Failing — each failure with its root cause]
- **Merge Conflicts:** [None / conflicting files]
- **Unresolved Threads:**
  | # | Author | Comment | Triage |
  |---|---|---|---|
  | 1 | @reviewer | [Summary] | Agree — fix X / Disagree — [reason] / Needs clarification |
- **Merge Readiness:** [Ready / Blocked — reason]
```

Triage rules: **agree** → recommend the specific fix; **disagree** → explain the reasoning;
**unsure** → flag for a human, do not guess; **CI failure** → scoped fix recommendation
only; **merge conflict** → recommend a resolution only when the intent is unambiguous.

## 4.3 Priority order
Correctness & concurrency → security → compatibility/migration safety → missing tests →
architecture drift → performance → out-of-scope → nits.

---

# 5. `/review-uncommitted` — WORKING TREE REVIEW

Primary purpose: catch work that **looks finished but isn't**.

## 5.1 Protocol
1. Collect `git diff`, `git diff --cached`, and untracked files.
2. Categorize: modified / added / deleted / renamed / untracked.
3. Hunt for incompleteness:
   - Functions declared but not implemented; placeholder or hardcoded return values
   - `TODO` / `FIXME` / `XXX` added in this change
   - Broken flows: caller updated but callee not (or vice versa); mismatched signatures;
     dangling references
   - Debug leftovers: `console.log`, `print()`, `debugger`, `dbg!()`, `binding.pry`,
     commented-out code, temporarily disabled tests, hardcoded test credentials
   - Missing imports; unused imports
   - Accidental deletions
   - Partially applied refactors: renamed in one place, not in consumers
   - Missing follow-through: config changed but no migration; type changed but serialization
     not updated; route added but not registered; env var read but not documented or defaulted
   - Errors swallowed silently (`catch {}`, `except: pass`)
   - New routes without auth/authorization middleware
   - Schema change without a migration file
   - Feature flags hardcoded on or off
   - Tests that assert nothing, are `.skip`/`.only`, or assert a mock's own return
4. **Check build feasibility:** would this break build, lint, typecheck, or test? Run them
   if possible and report actual results.
5. Run the hazard scan.

## 5.2 Priority order
Broken flows & incomplete edits → secrets/debug leftovers → build/type failures → missing
follow-through → partial refactors → hazards → nits.

---

# 6. `/review` — FULL REPOSITORY AUDIT

## 6.1 Protocol
1. **Map the repository:** structure, entry points, config, dependency manifests, test
   suites, CI, deploy, migrations.
2. **Code quality:** naming consistency, module structure, coupling/cohesion, complexity
   hotspots, significant duplication, dead code and stale flags.
3. **Architecture:** dependency direction and cycles, layering, data flow consistency, error
   propagation strategy, state management, API surface coherence.
4. **Data architecture** (load `data-systems-design`): sources of truth and derived stores,
   any dual writes, schema evolution practice, index hygiene, transaction boundaries,
   partitioning, caching strategy, background job safety.
5. **Test coverage:** are critical paths tested? Are the tests meaningful? Integration and
   end-to-end gaps; flaky/skipped tests; concurrency tests for concurrent code paths.
6. **Dependencies:** known CVEs, unmaintained packages, duplicate functionality, license
   compatibility, lockfile present and consistent.
7. **Security:** secrets in code or history, insecure defaults (open CORS, debug in prod,
   permissive auth), missing input validation, injection vectors, cryptographic misuse,
   authorization gaps, unsafe deserialization.
8. **Error handling & logging:** errors handled rather than swallowed; structured, useful
   logs without secrets or PII; crash paths; graceful degradation.
9. **Performance:** N+1s, unbounded collections, blocking I/O, missing indexes for hot
   queries, resource leaks (connections, handles, subscriptions, listeners).
10. **Reliability & operability:** timeouts and retries on every external call; bounded
    queues; health checks; monitored invariants; rollback capability; runbooks; backup and
    **tested** restore.
11. **Release readiness:** shippable state, incomplete features behind flags, current
    documentation, functional CI/CD.

## 6.2 Priority order
Security & secrets → correctness and data-integrity risk → reliability gaps (timeouts,
retries, unbounded resources) → architecture & maintainability → test gaps → dependency
risk → error handling → performance → code quality → release readiness.

---

# 7. `/review-inscope` — SCOPED TASK REVIEW

Requires a stated task. If it is unclear, ask before reviewing.

## 7.1 Protocol
1. **Identify and restate the task.**
2. **Collect the changes** (branch or uncommitted, whichever has content).
3. **Verify coverage:** does every stated *and implied* requirement have a corresponding
   change? Implied requirements normally include tests, migrations, docs, error handling,
   authorization, and observability.
4. **Flag out-of-scope edits:** unrelated files, unrequested refactors, unrelated dependency
   bumps, style-only churn.
5. **Flag missing in-scope work.**
6. **Check for regressions introduced by the fix:** broken existing behavior, side effects
   in shared paths, backwards compatibility.
7. **Assess shipping readiness:** would CI pass? Is it reviewable? Self-contained?
8. Run the hazard scan.

## 7.2 Mandatory tables
```md
### Scope Comparison
| Task Requirement | Status | Evidence |
|---|---|---|
| [Requirement 1] | ✓ Implemented | `file.ts:42` |
| [Requirement 2] | ✗ Missing | Not found in diff |
| [Requirement 3] | ⚠ Partial | Started in `file.ts:60`, error path not handled |

### Out-of-Scope Changes
| File | Change | Relation to Task |
|---|---|---|
| `unrelated.ts` | Reformatted imports | None — split into a separate PR |
```

## 7.3 Priority order
Missing in-scope work → regressions → incomplete implementation → out-of-scope edits →
shipping readiness → nits.

---

# 8. `/review-data` — DATA & DISTRIBUTED SYSTEMS AUDIT

A focused deep audit for systems where correctness under concurrency, failure, and scale is
the primary risk. Load `data-systems-design` and its references.

## 8.1 Protocol
1. **Map the data topology:** every datastore, cache, index, queue, and external system.
   For each, identify whether it is a **source of truth** or **derived** — and how derived
   data is kept in sync. Any application-code dual write is a 🔴 finding (H-32).
2. **Schema & evolution:** recent and pending migrations audited against expand→migrate→
   contract; nullability and defaults; blocking DDL; backfill safety; retired-field reuse.
   (`references/04-encoding-and-evolution.md`)
3. **Transactions & concurrency:** enumerate every read-then-write and confirm its
   protection mechanism; identify check-then-act write-skew patterns; verify uniqueness is
   enforced by an index; check for external side effects inside transactions; check for a
   retry loop where serializable isolation is used.
   (`references/07-transactions.md`)
4. **Replication & reads:** identify every replica read and check for read-after-write and
   monotonic-read violations; verify replication lag is monitored.
   (`references/05-replication.md`)
5. **Partitioning:** partition keys checked for hot spots and monotonic keys; scatter/gather
   on hot paths; rebalancing strategy. (`references/06-partitioning.md`)
6. **Distributed calls:** timeouts, backoff+jitter, retry layering, idempotency keys and
   where they are deduplicated, ambiguous-outcome handling, circuit breakers.
   (`references/08-distributed-systems-faults.md`)
7. **Coordination:** locks, leases, leader election, fencing tokens enforced at the resource,
   scheduled-job overlap protection. (`references/09-consistency-and-consensus.md`)
8. **Streams & jobs:** ordering requirements, consumer idempotence, consumer lag and
   retention, dead-letter handling, event-time vs. processing-time windowing, job
   determinism and resumability. (`references/10-batch-processing.md`, `11-stream-processing.md`)
9. **Clocks:** every use of time in logic — durations on monotonic clocks, no cross-node
   wall-clock ordering, no LWW where lost writes are unacceptable.
10. **Integrity:** end-to-end request IDs, dedup at the point of effect, reconciliation and
    invariant monitoring, deletion propagation to derived stores.
    (`references/12-correctness-and-integrity.md`)
11. **Capacity:** percentile-based SLOs (not averages), unbounded queries and queues,
    N+1s, cache invalidation and stampede protection, correlated failure modes.
    (`references/01-reliability-scalability-maintainability.md`)

## 8.2 Additional output sections
```md
### Data Topology
| System | Role | Written by | Kept in sync via | Divergence risk |
|---|---|---|---|---|
| Postgres `orders` | Source of truth | `orders-api` | — | — |
| Elasticsearch `orders` | Derived | `orders-api` (direct) | **Application dual write** | 🔴 Permanent divergence (H-32) |

### Invariants and Their Enforcement
| Invariant | Enforced by | Verified? |
|---|---|---|
| One charge per idempotency key | Unique index `idempotency_keys.key` | ✓ `migrations/0143` |
| No overlapping bookings | **Application `SELECT` check only** | 🔴 Not enforced under concurrency (H-02) |

### Failure Behavior Matrix
| Failure | Current behavior | Acceptable? |
|---|---|---|
| Search cluster down | Order writes fail | ✗ — should degrade, not fail |
| Duplicate webhook delivery | Duplicate refund | ✗ — needs a dedup key |
```

## 8.3 Priority order
Data loss/corruption → integrity violations → dual writes and divergence → concurrency
anomalies → migration/compatibility risk → cascading-failure risk (timeouts, unbounded
resources) → scalability limits → observability gaps.

---

# 9. REVIEW PRINCIPLES (ALL MODES)

## 9.1 Find real issues, not fluff
- Every finding cites a specific file and line range.
- Generic observations ("could be cleaner", "consider adding tests") are forbidden without a
  concrete example and a specific fix.
- Show the problematic code next to the correction.
- **Do not invent findings to fill sections.** "No critical issues found" is a valid,
  valuable result. A review padded with nits trains people to ignore reviews.
- Do not report an issue you cannot substantiate. If you suspect something but could not
  verify it, put it under "Needs human verification" and say what you could not check.

## 9.2 Severity

| Level | Meaning | Merge impact |
|---|---|---|
| 🔴 **Critical** | Correctness bug, security vulnerability, data loss or corruption risk, breaking change, unsafe migration | Blocks |
| 🟡 **Medium** | Missing tests, risky pattern, performance problem, incomplete error handling, missing observability on a new critical path | Should fix first |
| 🔵 **Nit** | Style, naming, minor readability, docs | Optional |

**Hazard-catalog findings inherit the catalog's severity.** Anything that causes silent
data loss or corruption is 🔴 even when it "hasn't happened yet" — these bugs are invisible
until they are expensive.

## 9.3 Actionable feedback
For everything above nit level: **what** (with file:line), **why** (concrete risk, not
theoretical), **how** (specific code or clear instruction).

## 9.4 Verdicts

| Verdict | Meaning |
|---|---|
| **NOT READY** | Has 🔴 blockers |
| **MOSTLY READY** | No 🔴, but 🟡 items should be addressed first |
| **READY WITH MINOR FIXES** | Only 🔵 nits remain |

---

# 10. PACING

- One complete review per invocation.
- **Do not fix during review.** If the user then says "fix it", switch to implementation
  mode starting with the 🔴 items.
- After the output: **STOP**.

**Forbidden:** auto-fixing during review, skipping the output format, combining modes,
vague findings, and omitting the hazard scan.

---

# 11. RELATED SKILLS

| Need | Skill |
|---|---|
| Verify against a written spec | `verify` |
| Hazard catalog and design references | `data-systems-design` |
| Plan the fixes | `planner` |
| Implement the fixes | `implement` |
| Full pipeline | `engineer-workflow` |

This skill is independent: it needs no `plan.md` or `executor.md` and reads the codebase and
git state directly.
