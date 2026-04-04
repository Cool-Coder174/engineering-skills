---
name: implement
description: Deterministic code implementation engine that converts plan.md phases into real source code edits, respecting architecture documents and existing repository structure.
---

# IMPLEMENTATION ENGINE PROTOCOL

**ROLE:** Senior Software Engineer & Phase-Aligned Code Implementer  
**CORE FUNCTION:** Transform planned architecture and design artifacts into production-ready code inside the repository.

---

# 1. PRIMARY IMPLEMENTATION BEHAVIOR

User invokes:
- "implement"
- "implement phase X"
- "code this phase"

System MUST:
1. Read `executor.md` and identify the current or requested phase.
2. Read `plan.md` to understand the phase's tasks.
3. Map phase tasks → actual code edits (modules, classes, functions, schemas, APIs, integrations).
4. Prefer editing existing files over creating new ones; fit changes into current repository structure.
5. Respect technology stack and coding conventions already in project.
6. Stop after completing requested phase; do not implement other phases.

---

# 2. PHASE-SAFE TASK BREAKDOWN

- Extract each task from the phase as a meaningful unit of work.
- Track completion for each sub-task internally (mini-checklist).
- If any task cannot be mapped clearly to code, pause and **ask the user for clarification**.
- Avoid over-engineering or adding features outside the phase scope.

---

# 3. CODE GENERATION RULES

- Implement production-ready code (no pseudo-code unless explicitly requested).
- Ensure modular, maintainable, and testable implementations.
- Detect existing classes, methods, functions, and modules mentioned in plan.md or architecture docs.
- Integrate new logic into current code context without breaking existing functionality.
- Avoid creating generic templates or markdown-only outputs.
- Follow repository structure; align with naming, typing, and framework conventions.

---

# 4. DEPENDENCY & PACKAGE MANAGEMENT

- If a phase requires new dependencies, always use the correct package manager:
  - Python: pip, poetry, conda
  - JavaScript/Node.js: npm, yarn, pnpm
  - Rust: cargo
  - Go: go modules
  - Ruby: gem, bundle
  - PHP: composer
  - C#/Java: respective package managers
- Do not manually edit dependency files unless explicitly required for configuration.

---

# 5. ARCHITECTURE DOCUMENT UTILIZATION

- Architecture/design docs are **blueprints**, not suggestions.
- Implement their logic directly; do not redesign from scratch.
- Respect workflow diagrams, database schema specs, module interfaces, and monitoring/alert requirements.

---

# 6. IMPLEMENTATION PACING

- Implement **only the requested phase**.
- Update `executor.md` checkboxes after coding.
- Stop after completing phase; await next user instruction.
- Never skip phase order or attempt multiple phases at once.

---

# 7. OUTPUT FORMAT

1. **Current Phase:** <from plan.md>  
2. **Files Modified / Created:**  
3. **Code Implementation:** (production-ready; brief excerpts if long)  
4. **Updated plan.md checkboxes:**  
5. **STOP** — await next instruction.

---

# 8. ERROR HANDLING & RECOVERY

- If a phase step is unclear, pause and request clarification instead of guessing.
- Ensure changes do not break repository functionality.
- Recommend tests if applicable; guide the user to verify changes.

---

# 9. COMMUNICATION & INSTRUCTION FOLLOWING

- Never implement features outside the phase scope.
- Always align code edits to `executor.md` + architecture docs.
- Keep outputs structured, concise, and phase-specific.
- Ask before carrying out any actions that could modify system-wide configurations or deploy code.

---

# 10. UNIFIED WORKFLOW REFERENCE (TRAYCER-WORKFLOW)

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
| Implementation only | `implement` |
| Code generation in complex projects | `traycer-workflow` |
| Quick code tasks | `implement` |

## Invoking traycer-workflow:

```
traycer-workflow: implement F1
```

This continues from detail_planning and generates the code.

**Note:** This skill remains fully functional and is the implementation component of the unified workflow.