---
name: code-review
description: Multi-mode code review engine with six commands (/review-diff, /review-uncommitted, /review, /review-inscope, /review-data, /review-security) for branch diffs, uncommitted changes, full repository audits, scoped task reviews, data/distributed-systems hazard audits, and adversarial security review. Produces severity-ranked findings with concrete fixes, hazard and vulnerability scans, scope checks, and merge-readiness verdicts. Use when reviewing code, PRs, diffs, uncommitted changes, auditing a repo, or assessing security risk.
---

# CODE REVIEW ENGINE

**ROLE:** Senior code reviewer, security auditor, and merge-readiness analyst.
**CORE FUNCTION:** Perform structured, severity-ranked reviews across six modes, producing
findings that name a file, a line, a consequence, and a fix — and a verdict someone can act
on.

**Companion knowledge skills.** This skill decides *what to look for*; two gate skills supply
the named failure modes:

| Gate | Skill | Catalog | Applies when the diff touches |
|---|---|---|---|
| Data hazards | `data-systems-design` | `references/hazard-catalog.md` (H-01 …) | Persistence, concurrency, retries, queues, replication, caching |
| Security vulnerabilities | `security-engineering` | `references/vulnerability-catalog.md` (V-01 …) | Secrets, crypto, identity, authorization, untrusted input, availability |

Load the relevant catalog before reviewing. Cite findings by ID (`H-14`, `V-32`) so that
`planner` and `verify` can trace them.

The two deep modes mirror each other: `/review-data` asks *will this stay correct?*,
`/review-security` asks *will this stay correct when someone is actively trying to break it?*
A change to a payment path, an auth flow, or a multi-tenant query usually deserves both.

---

# 1. COMMAND OVERVIEW & ACTIVATION

| Command | Activation keywords | Scope |
|---|---|---|
| `/review-diff` | `review-diff`, `review diff`, `review branch`, `review PR` | Current branch vs base branch |
| `/review-uncommitted` | `review-uncommitted`, `review uncommitted`, `review changes`, `review staged` | Uncommitted working tree changes |
| `/review` | `review`, `review repo`, `full review`, `audit` | Full repository, end to end |
| `/review-inscope` | `review-inscope`, `review in scope`, `review task`, `scope check` | Only changes relevant to a stated task |
| `/review-data` | `review-data`, `data review`, `hazard scan`, `distributed review` | Data, concurrency, and distribution hazards |
| `/review-security` | `review-security`, `security review`, `threat model`, `pentest this`, `is this secure` | Adversarial security review of a diff, component, or repo |

**Interpretation rule.** Any occurrence of these terms MUST be treated as a review command,
not conversational language.

**Mode switch.** When activated, the system enters **Review Mode**. Default assistant
behaviors — immediate coding, exploration narration, implementation suggestions — are
OVERRIDDEN until the review output is complete.

**Choosing a mode.** One command per invocation. If the user asks for a general review of a
change that touches authentication, cryptography, or untrusted input, run the general mode and
attach the Vulnerability Scan (Section 9.2) — or say that `/review-security` is warranted and
why. Same rule for data hazards and `/review-data`.

---

# 2. UNIVERSAL OUTPUT FORMAT (ALL COMMANDS)

Every review command MUST produce output in this structure:

```md
## Code Review: [Mode] — [Target Description]

### Summary
[1–3 sentence overview: what was reviewed, overall assessment, the single biggest concern]

### What Was Reviewed
- Scope: [files, commits, diff range, or full repo]
- Volume: [number of files / lines / commits]
- Exclusions: [anything skipped and why, or "None"]

### 🔴 Critical Issues (Blockers)
| # | File | Line(s) | Issue | Why It Blocks |
|---|------|---------|-------|---------------|
| 1 | `path/file.ext` | 42–50 | [Description] | [Impact] |

> If none: "No critical issues found."

### 🟡 Medium-Risk Issues
| # | File | Line(s) | Issue | Risk |
|---|------|---------|-------|------|
| 1 | `path/file.ext` | 12 | [Description] | [Risk explanation] |

> If none: "No medium-risk issues found."

### 🔵 Nits / Polish
- `file.ext:8` — [Nit description]

> If none: "No nits."

### Hazard, Failure & Vulnerability Scan
[Section 9 — only the applicable gates, only the applicable entries]

### Scope Check
- [x] All changes are in scope
- [ ] Out-of-scope change: [file — what it does — why it's out of scope]
- [x] No missing in-scope work

### Missing Validation
- [ ] [Test / lint / build / typecheck step not run or absent]
- [ ] [Edge case without coverage]

> If none: "All expected validation is present."

### Recommended Next Steps
1. [Specific, actionable fix or follow-up]
2. [Specific, actionable fix or follow-up]

### Verdict: [NOT READY | MOSTLY READY | READY WITH MINOR FIXES]
[One-sentence justification for the verdict]
```

---

# 3. /review-diff — BRANCH DIFF REVIEW

## 3.1 Activation

`/review-diff`, `review diff`, `review branch`, `review PR`, `review this branch against main`

## 3.2 Protocol

1. **Identify base and head.** Default base: `main`. Respect user overrides.
2. **Collect the diff.** `git diff main...HEAD`. List all changed files.
3. **Review every changed file** against these criteria:
   - **Correctness:** logic errors, off-by-one, null/undefined safety, race conditions, incorrect state transitions
   - **Regressions:** behavior changes in existing code paths that break callers or consumers
   - **Architecture drift:** does the change violate existing patterns, conventions, or module boundaries?
    - **Security:** run the Section 9.2 trigger check; if any trigger fires, load the vulnerability catalog
    - **Data hazards:** run the Section 9.1 trigger check; if any fires, load the hazard catalog
    - **Systems failures:** run the Section 9.3 trigger check; if any fires, load the failure catalog
   - **Performance:** N+1 queries, unbounded loops, missing pagination, blocking I/O on hot paths, leaks
   - **Test coverage:** are new code paths tested? Are existing tests invalidated by the change?
   - **Risky patterns:** raw SQL with interpolation, unvalidated input, panic/unwrap in production paths, broad exception swallowing
4. **Flag out-of-scope changes.** Files or hunks unrelated to the branch's purpose.
5. **Identify PR blockers.** Unresolved review comments, failing CI, missing approvals, merge conflicts.
6. **Invoke babysit behavior** (mandatory final step — Section 3.3).
7. **Produce output** in the universal format with the babysit addendum.

## 3.3 Babysit Integration (MANDATORY)

After the review output, `/review-diff` MUST append:

```md
### Babysit Status
- **PR Comments:** [N total, N resolved, N unresolved — triage below]
- **CI Status:** [Passing / Failing — list failures with root cause]
- **Merge Conflicts:** [None / list conflicting files]
- **Unresolved Review Threads:**
  | # | Author | Comment | Triage |
  |---|--------|---------|--------|
  | 1 | @reviewer | [Summary] | Agree — fix recommended / Disagree — [reason] / Needs clarification |
- **Merge Readiness:** [Ready / Blocked — reason]
```

Triage rules: **agree** → recommend the specific fix; **disagree** → explain the reasoning;
**unsure** → flag for human decision, do not guess. For a **CI failure**, give a scoped fix
recommendation, not a broad change. For a **merge conflict**, recommend resolution only when
the intent is unambiguous.

## 3.4 Priority Order

1. Security vulnerabilities that are exploitable as written
2. Correctness and regressions
3. Data-integrity hazards
4. Missing tests
5. Architecture drift
6. Performance
7. Out-of-scope changes
8. Nits and style

---

# 4. /review-uncommitted — UNCOMMITTED CHANGES REVIEW

## 4.1 Activation

`/review-uncommitted`, `review uncommitted`, `review changes`, `review staged`,
`review working tree`

## 4.2 Protocol

1. **Collect changes.** `git diff` (unstaged) + `git diff --cached` (staged) + untracked files.
2. **Categorize.** Modified, added, deleted, renamed, untracked.
3. **Review for incompleteness** (the primary focus of this mode):
   - **Incomplete edits:** functions declared but not implemented, placeholder returns, inline TODO/FIXME
   - **Broken flows:** callers updated but callees not (or vice versa), mismatched signatures, dangling references
   - **Debug leftovers:** `console.log`, `print()`, `debugger`, `dbg!()`, test-only flags in production code
   - **Security leftovers:** a commented-out auth check, `verify=False` added "temporarily", a hardcoded test token, an `if (isDev)` bypass — see **V-60** and **V-32**
   - **Secrets:** a key, token, or password added to a tracked file, including untracked-but-about-to-be-committed `.env` — see **V-14**
   - **Formatting/lint risks:** obvious style violations, missing imports, unused imports
   - **Accidental deletions:** files or code blocks removed without corresponding cleanup
   - **Partially implemented refactors:** some call sites updated, others missed
   - **Missing follow-through:** config changed but migration not created, type changed but serialization not updated, route added but not registered
4. **Check build feasibility.** Would these changes break `build`, `lint`, `typecheck`, or `test`?
5. **Produce output** in the universal format.

## 4.3 "Looks Finished But Isn't" Detection

This is the primary value of `/review-uncommitted`. Hunt specifically for:

- Functions that return hardcoded or placeholder values
- Error handlers that swallow errors silently (`catch {}`, `except: pass`)
- **New API routes without authentication or authorization middleware** (V-43)
- **A security check wrapped in a `try` that continues on failure** (V-64)
- Database schema changes without corresponding migration files
- UI components that reference non-existent props, state, or context
- Feature flags hardcoded to on or off
- Imports added but never used (or removed but still referenced elsewhere)
- Tests that assert nothing, are marked `.skip`/`.only`, or test a no-op
- New config keys without documentation or defaults
- Async operations without error handling or timeout

## 4.4 Priority Order

1. Secrets and security bypasses left in the working tree
2. Broken flows and incomplete edits
3. Debug leftovers and accidental deletions
4. Build/lint/typecheck failures
5. Missing follow-through
6. Partially implemented refactors
7. Nits and formatting

---

# 5. /review — FULL REPOSITORY REVIEW

## 5.1 Activation

`/review`, `review repo`, `full review`, `audit`, `review everything`

## 5.2 Protocol

1. **Map the repository.** Structure, entry points, config files, dependency manifests, test
   suites, CI configuration.
2. **Code quality audit.** Naming consistency, file organization, module structure;
   maintainability (coupling, cohesion, complexity hotspots); significant duplication; dead
   code.
3. **Architecture review.** Module boundaries and dependency direction; data flow patterns;
   error propagation strategy; state management; API surface area.
4. **Test coverage assessment.** Are critical paths tested? Are tests meaningful? Missing
   integration/E2E coverage? Test hygiene — flaky, skipped, or never-failing tests.
5. **Dependency audit.** Known vulnerabilities (CVEs); outdated dependencies with pending
   breaking changes; unnecessary dependencies; license compatibility. Also check how
   dependencies are pinned (**V-61**).
6. **Security review.** Run the full `/review-security` protocol (Section 8) across the
   repository, and report its Vulnerability Scan as a subsection here. Do not substitute a
   shallow secrets grep for it.
7. **Data hazard review.** Run the `/review-data` protocol (Section 7) against the persistence,
   concurrency, and messaging layers, and report its Data Topology and Hazard Scan here.
8. **Error handling and logging.** Errors handled rather than swallowed? Logging structured
   and useful, and free of secrets (**V-58**)? Panic/crash paths in production code? Graceful
   degradation, and does it fail closed (**V-64**)?
9. **Performance review.** N+1 queries, unbounded collections, blocking I/O on the main
   thread, missing caching, resource leaks.
10. **Release readiness.** Shippable state? Incomplete features behind flags? Documentation
    current? CI/CD functional?
11. **Produce output** in the universal format with an expanded subsection per audit area.

## 5.3 Priority Order

1. Exploitable security vulnerabilities and exposed secrets
2. Correctness and data-integrity risks
3. Architecture and maintainability
4. Test coverage gaps
5. Dependency risks
6. Error handling and logging
7. Performance
8. Code quality and consistency
9. Release readiness

---

# 6. /review-inscope — SCOPED TASK REVIEW

## 6.1 Activation

`/review-inscope`, `review in scope`, `review task`, `scope check`, `review against task`

The user MUST provide, or have previously stated, the task being worked on. If the task is
unclear, ask before reviewing.

## 6.2 Protocol

1. **Identify the task.** Extract from the user message, issue reference, or conversation.
   Restate it for confirmation.
2. **Identify all changes.** Collect the diff (branch or uncommitted — whichever has content).
3. **Verify the implementation covers the task.** Does every stated or implied requirement have
   a corresponding change? Are there placeholders where implementation should be? Is it
   complete end to end?
4. **Flag out-of-scope edits.** Unrelated files, unrequested refactors, unrelated dependency
   updates, style-only changes in untouched areas.
5. **Flag missing in-scope work.** Tests, documentation, migrations, config, error handling —
   and **security work implied by the task**. A task that adds an endpoint implies an
   authorization check; a task that stores a credential implies hashing and a rotation story.
   Missing implied security work is a **missing requirement**, not a nit.
6. **Check for regressions introduced by the fix.** Broken existing behavior, side effects in
   shared paths, backwards compatibility.
7. **Assess shipping readiness.** Would it pass CI? Is it reviewable? Self-contained?
8. **Produce output** in the universal format with the scope comparison table.

## 6.3 Scope Comparison Table (MANDATORY)

```md
### Scope Comparison
| Task Requirement | Status | Evidence |
|------------------|--------|----------|
| [Requirement 1 from the task] | ✓ Implemented | `file.ts:42` |
| [Requirement 2 from the task] | ✗ Missing | Not found in diff |
| [Requirement 3 — implied: authorize the new route] | ⚠ Partial | Handler authenticates but never authorizes (V-43) |

### Out-of-Scope Changes
| File | Change | Relation to Task |
|------|--------|------------------|
| `unrelated.ts` | Reformatted imports | None — remove or split to a separate PR |
```

## 6.4 Priority Order

1. Missing in-scope work (including implied security work)
2. Regressions introduced by the fix
3. Out-of-scope edits
4. Incomplete implementation
5. Shipping readiness
6. Nits

---

# 7. /review-data — DATA & DISTRIBUTED SYSTEMS AUDIT

A focused deep audit for changes where correctness under concurrency, failure, and scale is
the primary risk. Load `data-systems-design` and its references.

## 7.1 Activation

`/review-data`, `data review`, `hazard scan`, `distributed review`

## 7.2 Protocol

1. **Map the data topology:** every datastore, cache, index, queue, and external system. For
   each, identify whether it is a **source of truth** or **derived** — and how derived data is
   kept in sync. Any application-code dual write is a 🔴 finding (H-32).
2. **Schema & evolution:** recent and pending migrations audited against expand→migrate→
   contract; nullability and defaults; blocking DDL; backfill safety; retired-field reuse.
   (`references/04-encoding-and-evolution.md`)
3. **Transactions & concurrency:** enumerate every read-then-write and confirm its protection
   mechanism; identify check-then-act write-skew patterns; verify uniqueness is enforced by an
   index; check for external side effects inside transactions; check for a retry loop where
   serializable isolation is used. (`references/07-transactions.md`)
4. **Replication & reads:** identify every replica read and check for read-after-write and
   monotonic-read violations; verify replication lag is monitored.
   (`references/05-replication.md`)
5. **Partitioning:** partition keys checked for hot spots and monotonic keys; scatter/gather on
   hot paths; rebalancing strategy. (`references/06-partitioning.md`)
6. **Distributed calls:** timeouts, backoff+jitter, retry layering, idempotency keys and where
   they are deduplicated, ambiguous-outcome handling, circuit breakers.
   (`references/08-distributed-systems-faults.md`)
7. **Coordination:** locks, leases, leader election, fencing tokens enforced at the resource,
   scheduled-job overlap protection. (`references/09-consistency-and-consensus.md`)
8. **Streams & jobs:** ordering requirements, consumer idempotence, consumer lag and retention,
   dead-letter handling, event-time vs. processing-time windowing, job determinism and
   resumability. (`references/10-batch-processing.md`, `11-stream-processing.md`)
9. **Clocks:** every use of time in logic — durations on monotonic clocks, no cross-node
   wall-clock ordering, no LWW where lost writes are unacceptable.
10. **Integrity:** end-to-end request IDs, dedup at the point of effect, reconciliation and
    invariant monitoring, deletion propagation to derived stores.
    (`references/12-correctness-and-integrity.md`)
11. **Capacity:** percentile-based SLOs (not averages), unbounded queries and queues, N+1s,
    cache invalidation and stampede protection, correlated failure modes.
    (`references/01-reliability-scalability-maintainability.md`)

## 7.3 Additional Output Sections

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

## 7.4 Priority Order

1. Data loss or corruption
2. Integrity violations
3. Dual writes and divergence
4. Concurrency anomalies
5. Migration and compatibility risk
6. Cascading-failure risk (timeouts, unbounded resources)
7. Scalability limits
8. Observability gaps

**Hand off to `/review-security` when** the data path also carries a trust boundary: a
tenant-scoped query, a token or credential store, an externally triggered webhook, or a queue
that any client can write to. Divergence between two stores is a data hazard; divergence
between an authorization decision and the data it guards is a vulnerability.

---

# 8. /review-security — ADVERSARIAL SECURITY REVIEW

## 8.1 Activation

`/review-security`, `security review`, `security audit`, `threat model`, `is this secure`,
`pentest this`, `review for vulnerabilities`

Also invoke automatically as a subsection of `/review` (Section 5.2 step 6), and recommend it
from any other mode when a Section 9.2 trigger fires on a high-value path (authentication,
payments, PII, key handling).

## 8.2 Stance

This mode is different from the others in kind, not just in topic.

- **Adopt the adversary's goal, not the author's.** The other modes ask "does this work?" This
  one asks "what does an attacker get, and how?" Read every input as attacker-controlled and
  every check as bypassable until you find the thing that prevents it.
- **Absence of a mechanism is a finding.** In correctness review, missing code is usually a
  nit. Here, a missing authorization check, missing replay protection, or missing integrity
  check is the vulnerability itself. Look for what is *not* in the diff.
- **A claimed guarantee with no mechanism is a finding.** Ask which security service each
  mechanism is meant to provide, then verify it actually provides that one — "it's encrypted"
  does not provide integrity (V-15), and a valid signature does not provide freshness (V-25).
- **Report exploitability, not category.** "Uses MD5" is not a finding. "Uses MD5 to derive the
  password-reset token at `auth/reset.py:44`, so an attacker who knows the user's email and
  the request second can compute the token" is.
- **Do not invent findings.** Coverage is judged by the honesty of the scan, not the length of
  the table. "No applicable vulnerabilities — this change adds a read-only query behind
  existing authorization" is a good result.

## 8.3 Protocol

1. **Establish the target and the adversary.**

```md
### Threat Frame
- Target: [diff / component / repository]
- Assets: [what is worth attacking here]
- Adversary: [unauthenticated internet / authenticated user / other tenant / insider / compromised dependency]
- Adversary capability: [observe / modify / replay / impersonate / execute]
- Trust boundaries crossed: [list them — each one is where validation and authorization belong]
- Out of scope: [what is not being defended against here, and why]
```

2. **Map the attack surface.** Every entry point the change adds or touches: routes, RPC
   methods, queue consumers, webhooks, file uploads, CLI arguments, environment variables,
   deserializers, template renders, subprocess invocations, SQL builders.

```md
### Attack Surface
| Entry point | Reachable by | Authenticated? | Authorized by | Input validated where | Rate limited |
|---|---|---|---|---|---|
```

3. **Trace the data.** For each untrusted input, follow it to every sink — interpreter, buffer,
   file path, URL, log, template, subprocess. For each secret, follow it from generation to
   storage to use to disposal.

4. **Run the vulnerability catalog.** Load `security-engineering/references/vulnerability-catalog.md`
   and work the **Quick scan order** at the end of it. Use the catalog's section structure:

   | Section | Covers | Deep dive, when a finding needs the argument behind it |
   |---|---|---|
   | A | Cipher selection and modes | `02-cryptographic-primitives-and-modes.md` |
   | B | Randomness, keys, and secrets | `03-randomness-and-key-management.md` |
   | C | Integrity and message authentication | `05-integrity-hashes-and-macs.md` |
   | D | Authentication protocols and freshness | `06-signatures-and-authentication-protocols.md` |
   | E | Identity, certificates, and trust distribution | `04-public-key-and-key-exchange.md`, `07-identity-certificates-and-pki.md` |
   | F | Channel and transport security | `08-transport-and-channel-security.md` |
   | G | Access control and privilege | `09-authentication-and-access-control.md`, `12-perimeter-and-trusted-systems.md` |
   | H | Passwords and credential storage | `09-authentication-and-access-control.md` |
   | I | Audit, detection, and logging | `10-intrusion-detection-and-audit.md` |
   | J | Untrusted input, malicious code, and availability | `11-malicious-software-and-availability.md` |

   Read a deep dive when you need to *justify* a finding to someone who will push back, or when
   the correct fix is not obvious from the catalog entry. Do not load all twelve to scan a diff.

5. **Check the services actually delivered.** For each security-relevant flow, fill this in.
   A "claimed" column with an empty "mechanism" column is the finding.

```md
### Security Services
| Flow | Confidentiality | Origin auth | Integrity | Freshness | Access control | Availability |
|---|---|---|---|---|---|---|
| [flow] | [mechanism or ✗] | | | | | |
```

6. **Check detectability.** If this attack succeeded, would anyone know? Verify a security
   event is emitted, that it reaches a sink the attacker cannot edit, and that someone owns
   the alert (V-55, V-56, V-57).

7. **Check the failure path.** For every security dependency — policy service, token
   validator, KMS, certificate store — determine whether failure fails open or closed
   (V-64).

8. **Produce output** in the universal format, with the Threat Frame, Attack Surface, Security
   Services, and Vulnerability Scan sections, plus the exploitability addendum below.

## 8.4 Finding Format (MANDATORY in this mode)

Every 🔴 and 🟡 security finding must be reported with all six fields. A finding missing
"Attack" or "Fix" is not actionable and should not be filed.

```md
#### 🔴 V-32 — Certificate verification disabled
- **Location:** `internal/client/http.go:88`
- **Signature:** `TLSClientConfig{InsecureSkipVerify: true}` on the payments client
- **Attack:** Anyone able to intercept the route to the payment API — a compromised network
  hop, a hostile DNS response, an attacker on the pod network — presents any certificate and
  the client accepts it. The attacker then reads and rewrites payment requests in transit.
- **Service broken:** Peer entity authentication; integrity and confidentiality follow it.
- **Reference:** `security-engineering/references/07-identity-certificates-and-pki.md`;
  Stallings §14.2, p. 419
- **Fix:** Remove the flag. For the internal CA, load it explicitly into a dedicated cert pool
  rather than disabling verification. If a test needs a self-signed cert, inject the pool in
  the test only, and fail startup if `InsecureSkipVerify` is set outside tests.
```

## 8.5 Exploitability Addendum

```md
### Exploitability Assessment
| # | Finding | Preconditions | Adversary needed | Effort | Detectable? |
|---|---|---|---|---|---|
| 1 | V-32 | Network position between service and API | Network-adjacent | Low | No — passive attacks leave no trace |

### Attack Chains
[Where two medium findings compose into a critical one. State the chain explicitly:
"V-53 (unthrottled reset endpoint) + V-09 (predictable token) = account takeover with no
credentials." Chains are ranked at the severity of the outcome, not the components.]

### What I Could Not Determine
[Runtime config, deployment topology, WAF rules, or infrastructure not visible in the
repository — state the assumption made and what would change the finding. Do not silently
assume a control exists.]
```

## 8.6 Priority Order

1. Unauthenticated remote path to code execution, data disclosure, or authentication bypass
2. Authenticated path to privilege escalation or cross-tenant access
3. Exposed secrets and key material
4. Broken cryptography — confidentiality or integrity not actually provided
5. Missing freshness, replay, and session-lifecycle defects
6. Missing detection for any of the above
7. Availability and resource exhaustion
8. Defense-in-depth and hardening gaps

---

# 9. GATE INTEGRATION

## 9.1 Data hazard gate

**Trigger if the diff touches** any of: persistence, schema or migration, index, cache, queue
or event stream, background job, replication, sharding, transaction, concurrency, retry,
multi-service write, or an external API with side effects.

When triggered, load `data-systems-design/references/hazard-catalog.md` and report:

```md
### Hazard Scan
| ID | Hazard | Present | Evidence | Required Fix |
|---|---|---|---|---|
| H-01 | Read-modify-write lost update | Yes | `wallet.py:88` reads balance, adds, writes | Atomic `UPDATE … SET balance = balance + ?` |
```

## 9.2 Security vulnerability gate

**Trigger if the diff touches** any of:

| Trigger | Examples in a diff |
|---|---|
| Secrets or key material | new env var, `.env`, key file, KMS call, token constant |
| Cryptography | encrypt/decrypt, hash, sign, verify, TLS config, cipher or mode name |
| Randomness | `random`, `uuid`, token/salt/nonce generation |
| Identity | login, signup, session, JWT, OAuth, MFA, password reset, impersonation |
| Authorization | new route or RPC, role/permission logic, tenant scoping, admin action |
| Untrusted input | request parsing, upload, deserialization, query building, subprocess, template |
| Transport | new outbound client, webhook, certificate handling, proxy config |
| Dependencies | new dependency, changed pin, new install script |
| Detection | changes to logging, audit records, or alerting |

When triggered, load `security-engineering/references/vulnerability-catalog.md` and report:

```md
### Vulnerability Scan
| ID | Vulnerability | Present | Evidence | Required Fix |
|---|---|---|---|---|
| V-09 | Non-cryptographic PRNG for a security value | Yes | `auth/token.py:31` uses `random.choices` | `secrets.token_urlsafe(32)` |
| V-25 | No replay protection | No | Handler dedupes on event ID in the same transaction | — |
```

In `/review-diff`, `/review-uncommitted`, and `/review-inscope`, the scan covers only the
catalog sections the trigger table points to. In `/review` and `/review-security`, work the
full Quick scan order.

## 9.3 Systems failure gate

**Trigger if the diff touches** any of: a file read, write, copy, move, or delete, a directory
walk or document ingest, a durability or crash-safety requirement, a process that the code
starts, a signal handler, a daemon or long-lived worker, a thread or shared memory, a pipe, a
FIFO, or a socket.

When triggered, load `systems-programming/references/failure-catalog.md` and report:

```md
### Failure Scan
| ID | Failure | Present | Evidence | Required Fix |
|---|---|---|---|---|
| S-11 | Write over a file in place | Yes | `index.py:64` opens the target with `O_TRUNC` | Write a temp file in the same dir, fsync, rename, fsync the dir |
| S-07 | Durable write with no fsync | Yes | `index.py:71` reports success after `close` | `os.fsync(fd)` before reporting success |
| S-01 | Unchecked short write | No | Uses `writelines` on a buffered file object | — |
```

This gate catches what the hazard gate cannot see. The hazard gate asks whether the design
stays correct across machines. This gate asks whether the code stays correct against one
kernel. **It applies in every language, not only C** — no runtime makes two calls atomic, and
no runtime calls `fsync` for you.

## 9.4 Gate honesty

If a gate does not trigger, say so in one line:

```md
### Hazard, Failure & Vulnerability Scan
- Data hazard gate: not applicable — no persistence, concurrency, or messaging in this diff.
- Systems failure gate: not applicable — no file, process, signal, or socket surface.
- Security gate: not applicable — documentation and test-fixture changes only.
```

This is a valid and preferred outcome. Padding a scan with inapplicable entries makes the
gate worthless, because reviewers learn to skim it.

---

# 10. REVIEW PRINCIPLES (ALL COMMANDS)

## 10.1 Find real issues, not fluff

- Every finding MUST cite a specific file and line range.
- Generic observations ("code could be cleaner") are forbidden without a concrete example and
  a fix.
- Prefer showing the problematic code alongside the suggested correction.
- Never report a finding you cannot state a consequence for.

## 10.2 Severity definitions

| Level | Label | Meaning | Merge impact |
|---|---|---|---|
| 🔴 | **Critical** | Correctness bugs, exploitable vulnerabilities, data loss risk, breaking changes | Blocks merge |
| 🟡 | **Medium** | Missing tests, risky patterns, performance issues, defense-in-depth gaps, incomplete error handling | Should fix before merge |
| 🔵 | **Nit** | Style, naming, minor readability, documentation | Optional; safe to merge without |

**Security severity is decided by exploitability, not by category.** A weak algorithm on a
path an attacker cannot reach is 🟡 or 🔵. A missing authorization check on a public endpoint
is 🔴 even though the change is one line. Two 🟡 findings that chain into an account takeover
are reported as 🔴 with the chain shown (Section 8.5).

## 10.3 Actionable feedback

For every issue above nit level, provide:
1. **What** the problem is (with file and line reference)
2. **Why** it matters (concrete consequence, not theoretical)
3. **How** to fix it (specific code suggestion or clear instruction)

## 10.4 Verdict definitions

| Verdict | Meaning |
|---|---|
| **NOT READY** | Has 🔴 critical blockers. Must fix before merge or ship. |
| **MOSTLY READY** | No critical issues, but 🟡 items should be addressed first. |
| **READY WITH MINOR FIXES** | Only 🔵 nits remain. Safe to merge after quick polish. |

A diff with an unresolved 🔴 security finding is **NOT READY**, regardless of how small the
change is or how much of the rest is correct.

---

# 11. EXECUTION PACING

- Each command produces ONE complete review per invocation.
- Do NOT start implementing fixes unless the user explicitly requests it.
- After the review output is complete, STOP and await instruction.
- If the user says "fix it", "apply fixes", or similar → switch to implementation mode for the
  identified issues, starting with 🔴 critical items.

Forbidden:
- ❌ Auto-fixing issues during review
- ❌ Skipping the output format
- ❌ Combining multiple review modes in one invocation (the exception is `/review`, which runs
  the Section 7 and Section 8 protocols as subsections by design)
- ❌ Producing vague, unsubstantiated findings
- ❌ Reporting a gate as "clean" without having loaded the catalog
- ❌ Filling a Vulnerability Scan with inapplicable entries to appear thorough

---

# 12. RELATED SKILLS

| Need | Skill |
|---|---|
| Full pipeline (plan → detail → implement → verify) | `engineer-workflow` |
| Plan a change before writing it | `planner` |
| Expand one phase into implementable steps | `detail-planning` |
| Write the code for a phase | `implement` |
| Check an implementation against its spec | `verify` |
| Data and distribution design decisions and hazards | `data-systems-design` |
| File, process, signal, and thread rules and the failure catalog | `systems-programming` |
| Security design decisions, threat models, and the vulnerability catalog | `security-engineering` |

This skill operates independently and does not require `plan.md` or the `engineer-workflow`
pipeline. It reads the codebase and git state directly.

Use it at any point in the lifecycle: `/review-uncommitted` during implementation,
`/review-diff` before merge, `/review-inscope` to validate task completion, `/review` for
periodic audits, `/review-data` before shipping anything that writes to more than one place,
and `/review-security` before shipping anything that handles credentials, money, personal
data, or untrusted input.
