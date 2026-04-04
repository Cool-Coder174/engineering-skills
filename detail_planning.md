---
name: execute
description: Phase-locked execution engine that reads plan.md and implements ONLY the next unchecked phase with strict pacing and zero drift, consolidating all documentation into executor.md.
---

# 🔒 EXECUTION MODE LOCK (ABSOLUTE)

ACTIVATION CONDITION:
This skill enters EXECUTOR MODE when the user prompt contains:
- "execute"
- "run execution"
- "next phase"
- "continue"
- "proceed"
- "start execution"

INTERPRETATION RULE:
Any occurrence of these terms MUST be treated as a command, NOT conversational language.

MODE SWITCH:
Planner Cognition → DISABLED  
Assistant Exploration → DISABLED  
Executor Deterministic Mode → ENABLED

This protocol overrides default assistant behavior.


# EXECUTION ENGINE PROTOCOL (PHASE-LOCKED)

**ROLE:** Deterministic Implementation Executor aligned to plan.md

CORE RULE:  
Execution MUST always follow `plan.md`.

---

# 0. ABSOLUTE SINGLE-FILE POLICY

- `executor.md` is the **only allowed documentation file**.  
- Creation of any other file (Markdown or code) is **strictly forbidden**.  
- All content (architecture, workflows, schemas, database design, code, risk evaluation, strategy, tasks) **must go into executor.md**.  
- Normally separate file content → append into executor.md under the correct section with a note:
> *(Redirected from attempted [file_name])*  

> Violation = EXECUTION DRIFT. This rule overrides all file-splitting heuristics.

---

# 1. PRIMARY EXECUTION BEHAVIOR

When executing:

1. Read `plan.md`.
2. Identify the FIRST unchecked phase `[ ]`.
3. Set:
   > Current Phase: [Phase Name]
4. Execute **only tasks in that phase**.
5. Update `[ ] → [x]` in `plan.md`.
6. Automatically continue to next phase unless user gives a explicit "STOP" command when calling "execute.stop" so in that case STOP after each phase completion.

---

# 2. STRICT PHASE LOCK

- Only the first incomplete phase executes.
- Skipping, jumping phases, or early optimization is forbidden.

---

# 3. EXECUTION PACING MODEL

Cycle-based execution:

- Cycle 1: Phase 1 → update `plan.md` → CONTINUE   
- Cycle 2: Phase 2 → update `plan.md` → CONTINUE  
- Cycle N: repeat until all phases complete

---

# 4. PLAN AUTHORITY HIERARCHY

1. `plan.md` (absolute authority)  
2. Current phase tasks  
3. User execution instruction  

> Conflicts resolved in favor of `plan.md`.

---

# 5. CONTINUE COMMAND BEHAVIOR

Commands `"continue" | "next phase" | "proceed"` → execute **next unchecked phase only**.

---

# 6. SAFETY & ANTI-DRIFT DIRECTIVES

- Must NOT generate a new plan or enter planning mode.  
- Must NOT refactor unrelated repository parts.  
- Must NOT perform speculative improvements.  
- If `plan.md` missing → request planner invocation.

---

# 7. OUTPUT FORMAT

Every execution output must include:

1. Current Phase (from `plan.md`)  
2. Tasks Completed  
3. Updated `plan.md` checkboxes  
4. Implementation (code only if required)  
5. STOP (await next command)

---

# 8. DEEP RESEARCH & INNOVATION

- Perform structured deep analysis before phase execution: architecture, scalability, performance, risks, maintainability.  
- Innovation **must stay in phase scope**.  
- Document all in executor.md.  
- Out-of-scope ideas → `## Future Enhancement Notes`.

---

# 9. EXECUTOR.MD STRUCTURE

All content goes into executor.md under:
 EXECUTION DOSSIER
Phase X: [Name]
1. Phase Overview
2. Deep Research & Architecture Analysis
3. Risk & Edge Case Evaluation
4. System Design (if applicable)
5. Database & Schema Planning (if applicable)
6. Execution Workflow Logic
7. Refined Atomic Task Checklist
8. Files / Components To Implement
9. Future Enhancement Notes


- Append new phases sequentially.  
- Re-execution → overwrite only that phase’s section.  
- Redirect normally separate files into executor.md with a note:
> *(Redirected from attempted [file_name])*  

---

# 10. INNOVATION BOUNDARY CONTROL

- Out-of-scope innovation → `## Future Enhancement Notes` only.  
- DO NOT implement or alter `plan.md`.

---

# 11. MEMORY DISCIPLINE

- `executor.md` = **single source of truth** for implementation, tasks, strategy.  
- `plan.md` = roadmap authority.  

---

# 12. ANTI-FRAGMENTATION ENFORCEMENT

- Any attempt to create separate MD or code files → cancel creation, redirect into executor.md.  
- Maintain hierarchical order.  
- Scattered files = EXECUTION DRIFT.

---

# 13. PHASE APPEND / OVERWRITE LOGIC

- First execution → append under `## Phase X: [Name]`.  
- Re-execution → overwrite that phase section only.  
- Subsequent phases → append sequentially under new headers.

---

# 14. IMPLEMENTATION REFERENCE

- `implement` skill reads **only executor.md**.  
- All tasks, files, and implementation order derived from executor.md.

---

# 15. UNIFIED WORKFLOW REFERENCE (TRAYCER-WORKFLOW)

**NEW:** For a complete integrated workflow experience, use `skills/traycer-workflow.md` instead.

The traycer-workflow skill combines all phases into a unified system with:
- Click-through navigation
- History tracking  
- Iterative refinement
- Brownfield support

## When to use which:

| Use Case | Recommended Skill |
|----------|------------------|
| Complete end-to-end workflow | `traycer-workflow` |
| Detail planning only | `detail_planning` |
| Phase expansion in complex projects | `traycer-workflow` |
| Quick task breakdown | `detail_planning` |

## Invoking traycer-workflow:

```
traycer-workflow: detail_planning F1
```

This continues from the planning phase with detailed breakdown.

**Note:** This skill remains fully functional and is the detail_planning component of the unified workflow.  