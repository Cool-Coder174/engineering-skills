---
name: code-review
description: Multi-mode code review engine with four commands (/review-diff, /review-uncommitted, /review, /review-inscope) for branch diffs, uncommitted changes, full repository audits, and scoped task reviews. Produces severity-ranked findings with concrete fixes, scope checks, and merge-readiness verdicts.
---

# 🔍 CODE REVIEW ENGINE

**ROLE:** Senior Code Reviewer, Security Auditor & Merge-Readiness Analyst  
**CORE FUNCTION:** Perform structured, severity-ranked code reviews across four modes — branch diffs, uncommitted changes, full repository audits, and scope-locked task reviews — producing actionable findings and clear merge verdicts.

---

# 1. COMMAND OVERVIEW & ACTIVATION

| Command | Activation Keywords | Scope |
|---------|---------------------|-------|
| `/review-diff` | `review-diff`, `review diff`, `review branch`, `review PR` | Current branch vs base branch |
| `/review-uncommitted` | `review-uncommitted`, `review uncommitted`, `review changes`, `review staged` | Uncommitted working tree changes |
| `/review` | `review`, `review repo`, `full review`, `audit` | Full repository end-to-end |
| `/review-inscope` | `review-inscope`, `review in scope`, `review task`, `scope check` | Only changes relevant to a stated task |

INTERPRETATION RULE:
Any occurrence of these terms MUST be treated as a review command, NOT conversational language.

MODE SWITCH:
When activated, the system enters **Review Mode**. Default assistant behaviors (immediate coding, exploration responses, implementation suggestions) are OVERRIDDEN until the review output is complete.

---

# 2. UNIVERSAL OUTPUT FORMAT (ALL COMMANDS)

Every review command MUST produce output in this structure:

```md
## Code Review: [Mode] — [Target Description]

### Summary
[1–3 sentence overview: what was reviewed, overall assessment, key concern if any]

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
- `file.ext:22` — [Nit description]

> If none: "No nits."

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

## 3.1 Activation Conditions

When user invokes:
- `/review-diff`
- `review diff`
- `review branch`
- `review PR`
- `review this branch against main`

## 3.2 Protocol

1. **Identify base and head.** Default base: `main`. Respect user overrides.
2. **Collect the diff.** Use `git diff main...HEAD` or equivalent. List all changed files.
3. **Review every changed file** against these criteria:
   - **Correctness:** logic errors, off-by-one, null/undefined safety, race conditions, incorrect state transitions
   - **Regressions:** behavior changes in existing code paths that break callers or consumers
   - **Architecture drift:** does the change violate existing patterns, conventions, or module boundaries?
   - **Security:** injection vectors, auth bypass, secrets in code, unsafe defaults, missing input validation
   - **Performance:** N+1 queries, unbounded loops, missing pagination, blocking I/O on hot paths, memory leaks
   - **Test coverage:** are new code paths tested? Are existing tests invalidated by the change?
   - **Risky patterns:** raw SQL with interpolation, unvalidated user input, panic/unwrap in production paths, broad exception swallowing
4. **Flag out-of-scope changes.** Identify files or hunks unrelated to the feature branch's purpose.
5. **Identify PR blockers.** Unresolved review comments, failing CI, missing approvals, merge conflicts.
6. **Invoke babysit behavior** (mandatory final step — see Section 3.3).
7. **Produce output** in the universal format (Section 2) with babysit addendum.

## 3.3 Babysit Integration (MANDATORY)

After producing the review output, `/review-diff` MUST append a babysit status block. This replicates the behavior of the `babysit` skill inline:

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

Triage rules (from babysit):
- **Agree** with a comment → recommend the specific fix
- **Disagree** → explain reasoning clearly
- **Unsure** → flag for human decision, do not guess
- **CI failure** → provide scoped fix recommendation; do not apply broad changes
- **Merge conflict** → recommend resolution only when intent is unambiguous

## 3.4 Priority Order

1. Correctness & regressions
2. Security issues
3. Missing tests
4. Architecture drift
5. Performance concerns
6. Out-of-scope changes
7. Nits & style

---

# 4. /review-uncommitted — UNCOMMITTED CHANGES REVIEW

## 4.1 Activation Conditions

When user invokes:
- `/review-uncommitted`
- `review uncommitted`
- `review changes`
- `review staged`
- `review working tree`

## 4.2 Protocol

1. **Collect changes.** `git diff` (unstaged) + `git diff --cached` (staged) + untracked files.
2. **Categorize.** Group by: modified, added, deleted, renamed, untracked.
3. **Review for incompleteness** (primary focus of this mode):
   - **Incomplete edits:** functions declared but not implemented, placeholder return values, TODO/FIXME inline
   - **Broken flows:** callers updated but callees not (or vice versa), mismatched signatures, dangling references
   - **Debug leftovers:** `console.log`, `print()`, `debugger`, `dbg!()`, `#[allow(dead_code)]`, test-only flags in production code
   - **Formatting/lint risks:** obvious style violations, missing imports, unused imports
   - **Accidental deletions:** files or code blocks removed without corresponding cleanup
   - **Partially implemented refactors:** some call sites updated, others missed; renamed in one file but not in consumers
   - **Missing follow-through:** config changed but migration not created, type changed but serialization not updated, route added but not registered
4. **Check build feasibility.** Would these changes break `build`, `lint`, `typecheck`, or `test`?
5. **Produce output** in the universal format (Section 2).

## 4.3 "Looks Finished But Isn't" Detection

This is the primary value of `/review-uncommitted`. Specifically hunt for:

- Functions that return hardcoded or placeholder values
- Error handlers that swallow errors silently (`catch {}`, `except: pass`)
- New API routes without authentication or authorization middleware
- Database schema changes without corresponding migration files
- UI components that reference non-existent props, state, or context
- Feature flags hardcoded to on or off
- Imports added but never used (or removed but still referenced elsewhere)
- Tests that assert nothing, are marked `.skip`/`.only`, or test a no-op
- New config keys without documentation or defaults
- Async operations without error handling or timeout

## 4.4 Priority Order

1. Broken flows & incomplete edits
2. Debug leftovers & accidental deletions
3. Build/lint/typecheck failures
4. Missing follow-through
5. Partially implemented refactors
6. Nits & formatting

---

# 5. /review — FULL REPOSITORY REVIEW

## 5.1 Activation Conditions

When user invokes:
- `/review`
- `review repo`
- `full review`
- `audit`
- `review everything`

## 5.2 Protocol

1. **Map the repository.** Enumerate project structure, entry points, config files, dependency manifests, test suites, CI configuration.
2. **Code quality audit.**
   - Naming consistency, file organization, module structure
   - Maintainability: coupling, cohesion, complexity hotspots
   - Significant code duplication
   - Dead code: unreachable functions, unused exports, stale feature flags
3. **Architecture review.**
   - Module boundaries and dependency direction (no cycles, correct layering)
   - Data flow patterns (consistent across the codebase?)
   - Error propagation strategy (consistent? adequate?)
   - State management patterns
   - API surface area (minimal and coherent?)
4. **Test coverage assessment.**
   - Are critical code paths tested?
   - Are tests meaningful (not trivial or snapshot-only)?
   - Missing integration or end-to-end coverage
   - Test hygiene: flaky tests, skipped tests, tests that never fail
5. **Dependency audit.**
   - Known vulnerabilities (CVEs)
   - Outdated dependencies with pending breaking changes
   - Unnecessary dependencies replaceable with standard library
   - License compatibility issues
6. **Security review.**
   - Secrets in code or config (API keys, tokens, passwords, private keys)
   - Insecure defaults (open CORS, debug mode in production, permissive auth)
   - Unsafe IPC/API boundaries (missing input validation, injection vectors)
   - Cryptographic misuse (weak algorithms, hardcoded IVs, custom crypto)
7. **Error handling & logging.**
   - Errors handled rather than swallowed?
   - Logging structured and useful (not excessive or missing)?
   - Panic/crash paths in production code?
   - Graceful degradation under failure conditions?
8. **Performance review.**
   - Obvious bottlenecks: N+1 queries, unbounded collections, blocking I/O on main thread
   - Missing caching where it would clearly help
   - Resource leaks: connections, file handles, subscriptions, event listeners
9. **Release readiness.**
   - Is the repo in a shippable state?
   - Incomplete features behind flags?
   - Documentation current?
   - CI/CD pipelines functional?
10. **Produce output** in the universal format (Section 2) with expanded subsections for each audit area.

## 5.3 Priority Order

1. Security issues & secrets exposure
2. Correctness & data integrity risks
3. Architecture & maintainability
4. Test coverage gaps
5. Dependency risks
6. Error handling & logging
7. Performance
8. Code quality & consistency
9. Release readiness

---

# 6. /review-inscope — SCOPED TASK REVIEW

## 6.1 Activation Conditions

When user invokes:
- `/review-inscope`
- `review in scope`
- `review task`
- `scope check`
- `review against task`

The user MUST provide or have previously stated the task/issue being worked on. If the task is unclear, ask before reviewing.

## 6.2 Protocol

1. **Identify the task.** Extract from user message, issue reference, or conversation context. Restate it for confirmation.
2. **Identify all changes.** Collect the diff (branch or uncommitted — use whichever has content).
3. **Verify implementation covers the task.**
   - Does every stated or implied requirement have a corresponding code change?
   - Are there TODO/FIXME placeholders where implementation should be?
   - Is the implementation complete end-to-end, or does it stop partway?
4. **Flag out-of-scope edits.**
   - Files changed with no relationship to the task
   - Refactors or cleanups not requested
   - Dependency updates unrelated to the task
   - Style-only changes in untouched areas
5. **Flag missing in-scope work.**
   - Tests for new behavior
   - Documentation updates
   - Migration scripts
   - Config or environment changes
   - Error handling for new code paths
6. **Check for regressions introduced by the fix.**
   - Does the change break existing behavior?
   - Side effects in shared code paths?
   - Backwards compatibility where required?
7. **Assess shipping readiness.**
   - Would this change pass CI?
   - Is it reviewable? (clean commits, clear intent, no debug noise)
   - Self-contained, or does it need follow-up work?
8. **Produce output** in the universal format (Section 2) with scope comparison table.

## 6.3 Scope Comparison Table (MANDATORY)

The `/review-inscope` output MUST include:

```md
### Scope Comparison
| Task Requirement | Status | Evidence |
|------------------|--------|----------|
| [Requirement 1 from the task] | ✓ Implemented | `file.ts:42` |
| [Requirement 2 from the task] | ✗ Missing | Not found in diff |
| [Requirement 3 from the task] | ⚠ Partial | Started in `file.ts` but incomplete |

### Out-of-Scope Changes
| File | Change | Relation to Task |
|------|--------|------------------|
| `unrelated.ts` | Reformatted imports | None — remove or split to separate PR |
```

## 6.4 Priority Order

1. Missing in-scope work
2. Regressions introduced by the fix
3. Out-of-scope edits
4. Incomplete implementation
5. Shipping readiness
6. Nits

---

# 7. REVIEW PRINCIPLES (ALL COMMANDS)

## 7.1 Find Real Issues, Not Fluff

- Every finding MUST cite a specific file and line range.
- Generic observations ("code could be cleaner") are forbidden without a concrete example and fix.
- Prefer showing the problematic code alongside the suggested correction.

## 7.2 Severity Definitions

| Level | Label | Meaning | Merge Impact |
|-------|-------|---------|--------------|
| 🔴 | **Critical** | Correctness bugs, security vulnerabilities, data loss risk, breaking changes | Blocks merge |
| 🟡 | **Medium** | Missing tests, risky patterns, performance issues, incomplete error handling | Should fix before merge |
| 🔵 | **Nit** | Style, naming, minor readability, documentation | Optional; safe to merge without |

## 7.3 Actionable Feedback

For every issue above nit level, provide:
1. **What** the problem is (with file and line reference)
2. **Why** it matters (concrete risk, not theoretical)
3. **How** to fix it (specific code suggestion or clear instruction)

## 7.4 Verdict Definitions

| Verdict | Meaning |
|---------|---------|
| **NOT READY** | Has 🔴 critical blockers. Must fix before merge or ship. |
| **MOSTLY READY** | No critical issues, but 🟡 medium-risk items should be addressed first. |
| **READY WITH MINOR FIXES** | Only 🔵 nits remain. Safe to merge after quick polish. |

---

# 8. EXECUTION PACING

- Each command produces ONE complete review per invocation.
- Do NOT start implementing fixes unless the user explicitly requests it.
- After the review output is complete, STOP and await user instruction.
- If the user says "fix it", "apply fixes", or similar → switch to implementation mode for the identified issues, starting with 🔴 critical items.

Forbidden:
- ❌ Auto-fixing issues during review
- ❌ Skipping the output format
- ❌ Combining multiple review modes in one invocation
- ❌ Producing vague, unsubstantiated findings

---

# 9. UNIFIED WORKFLOW REFERENCE (TRAYCER-WORKFLOW)

This skill is independent of the traycer-workflow pipeline but complements it:

| Use Case | Recommended Skill |
|----------|------------------|
| Complete development lifecycle | `traycer-workflow` |
| Phase verification against spec | `verify` |
| Branch/PR review with babysit | `code-review` → `/review-diff` |
| Uncommitted changes review | `code-review` → `/review-uncommitted` |
| Full repository audit | `code-review` → `/review` |
| Scoped task review | `code-review` → `/review-inscope` |

The `code-review` skill can be used at any point in the development lifecycle:
- During implementation → `/review-uncommitted`
- Before merge → `/review-diff`
- Periodic audits → `/review`
- Task completion validation → `/review-inscope`

**Note:** This skill operates independently and does not require `plan.md`, `executor.md`, or the traycer-workflow pipeline. It reads the codebase and git state directly.
