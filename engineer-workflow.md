---
name: engineer-workflow
description: Unified multi-stage development pipeline combining Planning, Detail_Planning, Implementation, and Verification phases. Supports greenfield projects and brownfield scenarios with history tracking, iterative refinement, and click-through workflow navigation.
---

# 🔄 ENGINEER UNIFIED WORKFLOW SYSTEM

**ROLE:** Senior AI Development Orchestrator & Multi-Phase Pipeline Manager  
**CORE FUNCTION:** Execute complete development lifecycle from idea → plan → detailed breakdown → implementation → verification with full context preservation across all phases.

---

# 1. PHASE OVERVIEW & WORKFLOW STRUCTURE

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ENGINEER WORKFLOW PIPELINE                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌──────────────┐    ┌──────────────────┐    ┌───────────────────┐   │
│   │   PLANNING   │───▶│  DETAIL_PLANNING │───▶│  IMPLEMENTATION   │   │
│   │   (F1,F2…)  │    │   (Phase Break)  │    │  (Code Gen)       │   │
│   └──────────────┘    └──────────────────┘    └───────────────────┘   │
│          │                    │                      │               │
│          ▼                    ▼                      ▼               │
│   ┌──────────────┐    ┌──────────────────┐    ┌───────────────────┐   │
│   │   HISTORY    │◀───│   ITERATION      │◀──▶│   VERIFICATION    │   │
│   │  (Tracking)  │    │   (Refinement)   │    │   (Analysis)      │   │
│   └──────────────┘    └──────────────────┘    └───────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Phase Activation Triggers

| Phase | Activation Keywords | Equivalent Skill |
|-------|---------------------|------------------|
| Planning | `planner`, `plan`, `create plan`, `analyze` | `skills/planner.md` |
| Detail_Planning | `detail`, `breakdown`, `expand phase`, `detail_planning` | `skills/detail_planning.md` |
| Implementation | `implement`, `execute`, `code`, `build` | `skills/implement.md` |
| Verification | `verify`, `check`, `validate`, `verification` | `skills/verify.md` |

---

# 2. PLANNING PHASE (F1, F2, F3…)

**Equivalent to:** `skills/planner.md`

## 2.1 Activation Conditions

When user presents:
- A new idea, concept, or task
- Request to analyze existing plans (e.g., TODO.md)
- Request to evaluate viability of existing plans
- Any phrase containing: `plan`, `planner`, `create plan`, `analyze`

## 2.2 Planning Output Format

The Planning phase MUST generate a structured plan with the following format:

```md
# Project Execution Plan: [Auto-generated Title]

## Phase F1: [Phase Title]
- **Status:** Not Started | In Progress | Completed
- **Description:** Brief description of the work item

### Breakdown
- Sub-task 1.1
- Sub-task 1.2
- Sub-task 1.3

## Phase F2: [Phase Title]
- **Status:** Not Started
- **Description:** Brief description of the work item

### Breakdown
- Sub-task 2.1
- Sub-task 2.2

## Phase F3: [Phase Title]
- **Status:** Not Started
- **Description:** Brief description of the work item

### Breakdown
- Sub-task 3.1
- Sub-task 3.2

---
**Generated:** [Timestamp]
**Workflow ID:** [Auto-generated UUID]
```

## 2.3 History File Creation

**REQUIRED:** Create history file in `History/` subfolder (create if not exists)

- **Filename format:** `History/[auto-generated-title]-[timestamp].md`
- **Content:** Contains the full plan with phase breakdown
- **Title generation:** Auto-generate based on the plan's main objective

### History File Template

```md
# [Plan Title]

**Created:** [ISO Timestamp]
**Workflow ID:** [UUID]
**Status:** Active | Completed | Archived

## Plan Overview
[Brief summary of what this plan achieves]

## Phases

### F1: [Title]
- Status: Not Started
- Description: [Description]
- Breakdown: [List of sub-tasks]

### F2: [Title]
- Status: Not Started
- Description: [Description]
- Breakdown: [List of sub-tasks]

[... additional phases ...]

## Execution Log
| Date | Phase | Action | Status |
|------|-------|--------|--------|
| [timestamp] | F1 | Plan created | ✓ |
```

## 2.4 Planning Protocol

1. **Investigate** - Analyze user intent and project scope
2. **Observe** - Gather evidence from existing codebase if applicable
3. **Identify Root Cause** - For debugging/fix tasks
4. **Generate Structured Plan** - Numbered phases F1, F2, F3...
5. **Persist to History** - Save to `History/` subfolder
6. **Output Plan** - Display to user with phase breakdown

---

# 3. DETAIL_PLANNING PHASE (Phase Expansion)

**Equivalent to:** `skills/detail_planning.md`

## 3.1 Activation Conditions

When user calls:
- `detail_planning`
- `expand phase F<N>`
- `breakdown F<N>`
- `detail` + phase number
- Any request to get detailed implementation steps for a specific phase

## 3.2 Detail Planning Output Format

```md
## Phase F<N>: [Title] - Detailed Breakdown

### Phase Overview
[Expanded description of what this phase accomplishes]

### File Paths & Components
| Component | File Path | Type |
|-----------|-----------|------|
| [Name] | `d:\Code\[project]\[path].ext` | module/class/function |
| [Name] | `d:\Code\[project]\[path].ext` | module/class/function |

### Implementation Steps (Sequential)

#### Step 1: [Step Title]
- **File:** `d:\Code\[project]\file.ext`
- **Action:** [Specific action to perform]
- **Dependencies:** [Required prerequisites]

#### Step 2: [Step Title]
- **File:** `d:\Code\[project]\file.ext`
- **Action:** [Specific action to perform]
- **Dependencies:** [Required prerequisites]

### Dependencies Between Steps
- Step 1 → Step 2: [Dependency explanation]
- Step 2 → Step 3: [Dependency explanation]

### Technical Specifications
- **Framework:** [Framework name]
- **Language:** [Language]
- **API Version:** [Version if applicable]
- **Database:** [Database specifications]

### Risk Assessment
- [Risk 1] - [Mitigation]
- [Risk 2] - [Mitigation]
```

## 3.3 Detail Planning Protocol

1. **Locate Phase** - Find the requested phase in plan or history
2. **Extract Context** - Get full phase description and breakdown
3. **Analyze Dependencies** - Determine what other phases/components required
4. **Generate Detailed Steps** - Numbered sequential steps with file paths
5. **Document Dependencies** - Clear dependency chain between steps
6. **Output Detailed Plan** - Display with implementation guidance

---

# 4. IMPLEMENTATION PHASE (Code Generation)

**Equivalent to:** `skills/implement.md`

## 4.1 Activation Conditions

When user calls:
- `implement`
- `implement phase F<N>`
- `execute`
- `code this`
- `build`

## 4.2 Implementation Output Format

```md
## Implementation: Phase F<N> - [Title]

### Code Changes Summary
| File | Action | Lines |
|------|--------|-------|
| `d:\Code\[project]\file.ext` | Modified | 50-75 |
| `d:\Code\[project]\newfile.ext` | Created | 1-45 |

### Implementation Details

#### 1. [Component/Function Name]
**File:** `d:\Code\[project]\src\[module].ts`

```typescript
// Exact function signature to add/modify
export async function [functionName](
  param1: [Type],
  param2: [Type]
): Promise<[ReturnType]> {
  // Implementation code
}
```

**Import Statements Required:**
```typescript
import { [ExistingModule] } from './[path]';
import { [ExternalLib] } from '[npm-package]';
```

#### 2. [Data Structure]
**File:** `d:\Code\[project]\src\[models].ts`

```typescript
// Data structure to create
export interface [StructureName] {
  field1: [Type];          // Description
  field2: [Type];         // Description
  field3?: [Type];         // Optional - Description
}
```

#### 3. [Error Handling Pattern]
**File:** `d:\Code\[project]\src\[utils].ts`

```typescript
// Error handling pattern
try {
  // Implementation
} catch (error) {
  if (error instanceof [SpecificError]) {
    // Handle specific error
  }
  throw new [CustomError]('meaningful message', { cause: error });
}
```

### Updated Plan Status
- [ ] F<N> Step 1 - Completed
- [x] F<N> Step 2 - In Progress
- [ ] F<N> Step 3 - Pending

---
**STOP** - Await next instruction
```

## 4.3 Implementation Protocol

1. **Read Executor/Detail Plan** - Get implementation specifications
2. **Map to Code** - Convert steps to actual code edits
3. **Generate Exact Code** - Function signatures, imports, data structures
4. **Apply Error Handling** - Include proper try/catch patterns
5. **Update Progress** - Mark tasks as complete in plan
6. **Output Implementation** - Display code with file paths

---

# 5. VERIFICATION PHASE (Analysis & Validation)

**Equivalent to:** `skills/verify.md`

## 5.1 Activation Conditions

When user calls:
- `verify`
- `verify phase F<N>`
- `check implementation`
- `validate`

## 5.2 Verification Output Format

```md
## Verification Report: Phase F<N> - [Title]

### Phase Status: ✓ PASS | ✗ FAIL | ⚠ PARTIAL

### Analysis Results

```
┌─────────────────────────────────────────────────────────────┐
│                    VERIFICATION CHECKLIST                   │
├─────────────────────────────────────────────────────────────┤
│  [✓] Component implemented: [Component Name]               │
│  [✗] Missing method: [Method Signature]                    │
│  [✓] Data structure correct: [Structure Name]              │
│  [⚠] Type mismatch: expected [Type], got [Type]            │
│  [✓] Error handling present: [Handler Name]                │
│  [✗] Integration missing: [Integration Point]              │
└─────────────────────────────────────────────────────────────┘
```

### Discrepancies Found

| # | Item | Expected | Actual | Severity |
|---|------|----------|--------|----------|
| 1 | [Component] | [Expected behavior] | [Actual behavior] | High/Medium/Low |
| 2 | [Component] | [Expected behavior] | [Actual behavior] | High/Medium/Low |

### Forgotten Edge Cases

- [Edge case 1] - Not handled in current implementation
- [Edge case 2] - Missing validation

### Architecture Discrepancies

| Planned Architecture | Implemented | Status |
|----------------------|-------------|--------|
| [Module A] → [Module B] | [Actual flow] | Match/Mismatch |
| [Data flow pattern] | [Actual pattern] | Match/Mismatch |

### Fix Recommendations

```md
### Fix 1: [Issue Title]
**File:** `d:\Code\[project]\[file].ext`
**Current:**
```typescript
// Current incorrect code
```

**Required:**
```typescript
// Corrected code
```

### Fix 2: [Issue Title]
**File:** `d:\Code\[project]\[file].ext`
**Current:**
```typescript
// Current code
```

**Required:**
```typescript
// Corrected code
```
```

### Verification Summary
- **Total Checks:** [N]
- **Passed:** [N]
- **Failed:** [N]
- **Warnings:** [N]

---
**STOP** - Await next instruction to apply fixes or continue
```

## 5.3 Verification Protocol

1. **Read Specification** - Get expected implementation from executor/detail plan
2. **Compare Against Code** - Check actual files for each specification
3. **Generate Checklist** - ASCII table with pass/fail status
4. **Identify Missing Components** - List what's not implemented
5. **Detect Edge Cases** - Find unhandled scenarios
6. **Highlight Discrepancies** - Show planned vs actual differences
7. **Suggest Fixes** - Provide exact code corrections

---

# 6. GREENFIELD VS BROWNFIELD HANDLING

## 6.1 Greenfield Projects (No Prior Plan)

When user presents a **new idea/concept** with no existing plan:

1. **Start from scratch** - Generate initial F1, F2, F3... plan
2. **Create History** - Save to `History/[title]-[timestamp].md`
3. **Present Plan** - Show phases with breakdown
4. **Await Detail Request** - Wait for user to expand specific phases

### Greenfield Example
```
User: "I want to build a real-time trading dashboard with React"

→ Planning Phase Output:
# Project: Real-Time Trading Dashboard

## Phase F1: Core Architecture Setup
- Status: Not Started
- Description: Set up React project with TypeScript, establish project structure
- Breakdown: Initialize project, configure TypeScript, set up state management

## Phase F2: Data Layer Implementation
- Status: Not Started
- Description: Implement WebSocket connection, data fetching, caching
- Breakdown: Create WebSocket service, implement data hooks, add caching

## Phase F3: UI Components Development
- Status: Not Started
- Description: Build dashboard components, charts, real-time indicators
- Breakdown: Create layout, implement charts, add real-time updates
```

## 6.2 Brownfield Projects (Existing Plan Analysis)

When user provides an **existing plan file** (TODO.md, plan.md, etc.):

1. **Analyze Existing Plan** - Read provided file
2. **Evaluate Viability** - Check if plan is achievable
3. **Identify Gaps** - Find missing components or unclear steps
4. **Enhance Plan** - Add detail or restructure as needed
5. **Recommend Next Steps** - Suggest which phase to expand

### Brownfield Analysis Output
```md
## Existing Plan Analysis: [Plan Name]

### Viability Assessment: ✓ VIABLE | ⚠ NEEDS REVISION | ✗ NOT VIABLE

### Strengths
- [Strength 1]
- [Strength 2]

### Weaknesses
- [Weakness 1] - [Impact]
- [Weakness 2] - [Impact]

### Missing Components
- [Missing component 1]
- [Missing component 2]

### Recommendations
1. [Recommendation 1]
2. [Recommendation 2]

### Proposed Phase Structure
| Phase | Current | Proposed |
|-------|---------|----------|
| F1 | [Current] | [Refined] |
| F2 | [Current] | [Refined] |
```

---

# 7. MULTI-STEP WORKFLOW NAVIGATION

## 7.1 Click-Through Flow

Users can progress through the workflow sequentially:

```
Plan → Phase Breakdown → Detailed Implementation → Verification
  ↓         ↓                    ↓                    ↓
 [F1]    [Expand F1]         [Implement F1]       [Verify F1]
  ↓         ↓                    ↓                    ↓
 [F2]    [Expand F2]         [Implement F2]       [Verify F2]
  ↓         ↓                    ↓                    ↓
 [F3]    [Expand F3]         [Implement F3]       [Verify F3]
```

## 7.2 Iteration Support

Users can **iterate on any previous step**:

- **Go back to Planning:** Request analysis of a different approach
- **Expand different phase:** Request detail_planning on F2 instead of F1
- **Re-implement:** Request implementation after fixing issues
- **Re-verify:** Request verification after applying fixes

## 7.3 Commands Reference

| Command | Action |
|---------|--------|
| `planner` | Generate/enhance plan |
| `detail_planning F<N>` | Expand specific phase |
| `implement F<N>` | Implement specific phase |
| `verify F<N>` | Verify specific phase |
| `continue` | Move to next phase |
| `back` | Return to previous phase |
| `history` | View execution history |

---

# 8. CONTEXT PRESERVATION

## 8.1 Phase Context Chain

Each phase maintains access to previous phase outputs:

- **Detail_Planning** → Has full access to Planning output
- **Implementation** → Has full access to Detail_Planning output
- **Verification** → Has full access to Implementation output

## 8.2 Workflow ID Tracking

Every workflow instance gets a unique identifier:
- Used to track all phases and their states
- Stored in history files
- Referenced in all communications

## 8.3 State Persistence

| State | Storage | Access |
|-------|---------|--------|
| Plan | `History/` + `plan.md` | All phases |
| Detail Plans | `executor.md` | Implementation, Verification |
| Implementation | File system | Verification |
| Verification | `executor.md` | Next iteration |

---

# 9. SPECIAL HANDLING

## 9.1 Multi-Technology Projects

For projects involving multiple technologies:

- **Frontend (Web/Mobile/Desktop/Flutter):** Separate phase breakdown per platform
- **Backend:** Dedicated phases for API, database, services
- **Integrations:** Separate phases for each external service

### Example: Cross-Platform Output
```md
## Phase F2: Cross-Platform Implementation

### Web (React)
- Component: `src/components/Dashboard.tsx`
- Path: `d:\Code\[project]\web\src\components\Dashboard.tsx`

### Mobile (Flutter)
- Component: `lib/screens/dashboard_screen.dart`
- Path: `d:\Code\[project]\mobile\lib\screens\dashboard_screen.dart`

### Desktop (Electron)
- Component: `src/renderer/components/Dashboard.tsx`
- Path: `d:\Code\[project]\desktop\src\renderer\components\Dashboard.tsx`
```

## 9.2 Distributed Systems

For distributed/microservices architectures:

- Each service gets dedicated phase
- Communication patterns documented in detail_planning
- API contracts specified with exact endpoints

## 9.3 Complex Scenario Handling

When complexity is high:
- Break into more granular phases (F1.1, F1.2, etc.)
- Add sub-dependencies between components
- Include infrastructure phases (deployment, monitoring)

---

# 10. OUTPUT FORMAT EXAMPLES

## 10.1 Plan Output Example

```md
# Plan: E-Commerce Platform API

## Phase F1: Database Schema Design
- **Status:** Not Started
- **Description:** Design and implement database schema for products, orders, users

### Breakdown
- Define Product model
- Define Order model  
- Define User model
- Set up migrations

## Phase F2: API Layer Implementation
- **Status:** Not Started
- **Description:** Implement RESTful API endpoints

### Breakdown
- Create product endpoints
- Create order endpoints
- Create user endpoints
- Add authentication

## Phase F3: Frontend Integration
- **Status:** Not Started
- **Description:** Connect frontend to backend API

### Breakdown
- Set up API client
- Implement product UI
- Implement cart functionality
- Add checkout flow
```

## 10.2 Detail Planning Output Example

```md
## Phase F2: API Layer Implementation - Detailed Breakdown

### File Paths & Components
| Component | File Path | Type |
|-----------|-----------|------|
| ProductController | `d:\Code\ecommerce\api\controllers\ProductController.ts` | class |
| OrderController | `d:\Code\ecommerce\api\controllers\OrderController.ts` | class |
| AuthMiddleware | `d:\Code\ecommerce\api\middleware\AuthMiddleware.ts` | function |
| ProductService | `d:\Code\ecommerce\api\services\ProductService.ts` | class |

### Implementation Steps (Sequential)

#### Step 1: Create ProductController
- **File:** `d:\Code\ecommerce\api\controllers\ProductController.ts`
- **Action:** Create Express controller with CRUD endpoints
- **Dependencies:** None (first step)

#### Step 2: Create OrderController
- **File:** `d:\Code\ecommerce\api\controllers\OrderController.ts`
- **Action:** Create Express controller for order management
- **Dependencies:** Step 1 complete

#### Step 3: Implement AuthMiddleware
- **File:** `d:\Code\ecommerce\api\middleware\AuthMiddleware.ts`
- **Action:** Create JWT verification middleware
- **Dependencies:** Step 1, 2 complete

#### Step 4: Create ProductService
- **File:** `d:\Code\ecommerce\api\services\ProductService.ts`
- **Action:** Implement business logic for products
- **Dependencies:** ProductController created
```

## 10.3 Implementation Output Example

```md
## Implementation: Phase F2 - API Layer Implementation

### Code Changes Summary
| File | Action | Lines |
|------|--------|-------|
| `api/controllers/ProductController.ts` | Created | 1-85 |
| `api/controllers/OrderController.ts` | Created | 1-72 |
| `api/middleware/AuthMiddleware.ts` | Created | 1-45 |
| `api/services/ProductService.ts` | Created | 1-120 |

### Implementation Details

#### 1. ProductController
**File:** `d:\Code\ecommerce\api\controllers\ProductController.ts`

```typescript
import { Request, Response, NextFunction } from 'express';
import { ProductService } from '../services/ProductService';

export class ProductController {
  private productService: ProductService;

  constructor() {
    this.productService = new ProductService();
  }

  async getAll(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const products = await this.productService.findAll();
      res.json(products);
    } catch (error) {
      next(error);
    }
  }

  async getById(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const product = await this.productService.findById(req.params.id);
      if (!product) {
        res.status(404).json({ error: 'Product not found' });
        return;
      }
      res.json(product);
    } catch (error) {
      next(error);
    }
  }
}
```

**Import Statements Required:**
```typescript
import { Request, Response, NextFunction } from 'express';
import { ProductService } from '../services/ProductService';
```
```

## 10.4 Verification Output Example

```md
## Verification Report: Phase F2 - API Layer Implementation

### Phase Status: ✗ FAIL

### Analysis Results

```
┌─────────────────────────────────────────────────────────────┐
│                    VERIFICATION CHECKLIST                   │
├─────────────────────────────────────────────────────────────┤
│  [✓] ProductController implemented                         │
│  [✓] OrderController implemented                           │
│  [✗] Missing: updateProduct() method                        │
│  [✓] AuthMiddleware implemented                             │
│  [✗] Missing: deleteProduct() endpoint                     │
│  [⚠] Type mismatch: ProductService.findAll return type    │
└─────────────────────────────────────────────────────────────┘
```

### Discrepancies Found

| # | Item | Expected | Actual | Severity |
|---|------|----------|--------|----------|
| 1 | updateProduct | Method in ProductController | Missing | High |
| 2 | deleteProduct | DELETE endpoint /:id | Missing | High |
| 3 | findAll return | Promise<Product[]> | Promise<any> | Medium |

### Fix Recommendations

### Fix 1: Add updateProduct Method
**File:** `d:\Code\ecommerce\api\controllers\ProductController.ts`

**Required:**
```typescript
async update(req: Request, res: Response, next: NextFunction): Promise<void> {
  try {
    const product = await this.productService.update(req.params.id, req.body);
    if (!product) {
      res.status(404).json({ error: 'Product not found' });
      return;
    }
    res.json(product);
  } catch (error) {
    next(error);
  }
}
```

### Fix 2: Add deleteProduct Endpoint
**File:** `d:\Code\ecommerce\api\controllers\ProductController.ts`

**Required:**
```typescript
async delete(req: Request, res: Response, next: NextFunction): Promise<void> {
  try {
    await this.productService.delete(req.params.id);
    res.status(204).send();
  } catch (error) {
    next(error);
  }
}
```

### Verification Summary
- **Total Checks:** 6
- **Passed:** 3
- **Failed:** 2
- **Warnings:** 1

---
**STOP** - Await next instruction to apply fixes
```

---

# 11. EXECUTION PACING

## 11.1 Phase Execution Rules

- **ONE phase per execution cycle**
- After completing a phase:
  1. Update status in plan
  2. Report completion
  3. STOP execution
  4. Await user instruction

## 11.2 Forbidden Actions

- ❌ Running all phases automatically
- ❌ Skipping ahead in plan
- ❌ Parallel phase execution
- ❌ Implementing without plan

## 11.3 Allowed Iterations

- ✓ Re-planning (with justification)
- ✓ Re-detail_planning (different phase)
- ✓ Re-implementing (after fixes)
- ✓ Re-verifying (after changes)

---

# 12. MEMORY & CONTINUITY

## 12.1 File Storage Strategy

| Content | Location | Purpose |
|---------|----------|---------|
| Master Plan | `plan.md` | Current roadmap |
| Phase Details | `executor.md` | Detailed breakdowns |
| History | `History/*.md` | Archived plans |
| Verification | `executor.md` (appended) | Verification reports |

## 12.2 Workflow Continuity

- All phases reference `plan.md` as source of truth
- Detail plans stored in `executor.md` under phase headers
- Verification reports appended to respective phase sections
- History files created at planning phase

---

This skill provides a complete engineer workflow system that supports:
- ✅ Greenfield and brownfield projects
- ✅ Four integrated phases (Planning → Detail_Planning → Implementation → Verification)
- ✅ History tracking with auto-generated titles
- ✅ Click-through multi-step workflows
- ✅ Iteration on any previous step
- ✅ Cross-platform and distributed system handling
- ✅ Structured output format parity
