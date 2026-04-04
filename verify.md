---
name: verify
description: Phase verification engine that reads executor.md and compares user-implemented code against the phase’s specification. Generates a structured verification report + fix recommendations appended into executor.md.
---

# VERIFY ENGINE SCRIPT

**ROLE:** Deterministic Phase Verifier aligned to executor.md

---

## 0. INPUT FORMAT

User invokes:

* `verify. phase <N>`
* `<N>` = phase number to verify
* Only **one phase per invocation**

---

## 1. VERIFICATION SOURCE

* `executor.md` = single source of truth
* Only the section `## Phase <N>: [Name]` is verified
* Subsections include: architecture, workflow, database, monitoring, alerts, and safety checks

---

## 2. VERIFICATION STEPS

**Locate Phase Section**

* Parse `executor.md` for:

```
## Phase N: [Name]
```

**Extract Specifications**

* Architecture modules/classes/functions
* Workflow steps
* Database tables and columns
* Monitoring/alert thresholds
* Circuit breakers & safety rules

**Compare Implementation**

* Check code in user files (`Python modules`, `app.py`, `monitoring/`) against specs:

  * Missing modules
  * Missing or incorrect classes/methods
  * Wrong method signatures or parameters
  * Workflow deviations
  * Database schema mismatches
  * Monitoring & alert mismatches
  * Integration gaps (connections to `app.py`)

**Generate Verification Report**

* For each discrepancy:

```
### Verification Report
- [ ] <Description of missing/incorrect item>
- [ ] <Next discrepancy>
```

**Example:**

* [ ] Missing method: `RiskManager.validate_pre_trade()`
* [ ] `SlippageValidator` threshold incorrect: 0.7% vs 0.5%
* [ ] `update_position()` not implemented in `PositionTracker`
* [ ] WebSocket `'get_portfolio'` event missing

**Generate Fix Recommendations**

```
### Fix Recommendations
1. Implement missing validate_pre_trade() method in risk_management.py
2. Correct SlippageValidator threshold to 0.5%
3. Implement update_position() method in PositionTracker
4. Add get_portfolio WebSocket event
```

---

## 3. APPENDING RESULTS TO EXECUTOR.MD

* Append **Verification Report** and **Fix Recommendations** directly under the phase’s section:

```
## Phase N: [Name]
...
### Verification Report
...
### Fix Recommendations
...
```

* **Do not create new files**

---

## 4. PHASE-LOCKED BEHAVIOR

* Only verifies the specified phase `<N>`
* Stops after **one phase**
* User must invoke `"verify. phase N+1"` to check the next phase
* Verification is non-invasive: **no other changes are made**

---

## 5. SPECIAL CHECKS

**Python Module Verification**

* Ensure all classes, methods, and function signatures in `executor.md` are implemented

**Database Schema Verification**

* Compare user tables and columns to `executor.md` SQL spec

**Workflow Validation**

* Ensure all workflow steps are implemented in correct order

**Monitoring & Alerts**

* Confirm `system_health.py`, `trading_monitor.py`, `alerts.py` match `executor.md` spec

**Integration Checks**

* Ensure `app.py` uses orchestrator modules correctly (`TradingEngine`, `Portfolio`, `RiskManager`)

---

## 6. OUTPUT FORMAT

```
Current Phase: <N>

Verification Report (missing/incorrect items)

Fix Recommendations

STOP — await next "verify. phase N+1" command
```

---

# 7. UNIFIED WORKFLOW REFERENCE (TRAYCER-WORKFLOW)

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
| Verification only | `verify` |
| Code validation in complex projects | `traycer-workflow` |
| Quick validation tasks | `verify` |

## Invoking traycer-workflow:

```
traycer-workflow: verify F1
```

This continues from implementation and validates the code.

**Note:** This skill remains fully functional and is the verification component of the unified workflow.

---