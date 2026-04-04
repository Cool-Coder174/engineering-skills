# Engineer Workflow - AI Development Pipeline

A unified multi-stage development workflow system that combines Planning, Detail_Planning, Implementation, and Verification phases into a cohesive pipeline.

## 🚀 Features

- **Four Integrated Phases**: Planning → Detail_Planning → Implementation → Verification
- **History Tracking**: Auto-saves plans to `History/` subfolder
- **Click-Through Navigation**: Progress through phases sequentially
- **Iterative Refinement**: Can revisit any previous phase
- **Brownfield Support**: Analyze existing plans (TODO.md, plan.md)
- **Cross-Platform**: Works with Web, Mobile, Desktop, Flutter projects
- **Distributed Systems**: Supports microservices architectures

## 📁 File Structure

```
skills/
├── engineer-workflow.md   # Unified workflow (main entry point)
├── planner.md             # Planning phase component
├── detail_planning.md    # Detail planning phase component
├── implement.md           # Implementation phase component
├── verify.md             # Verification phase component
└── code-review.md        # Multi-mode code review engine
```

## 🛠️ Installation

### For Kilo Code / Claude Code Agents

1. Copy the `skills/` folder to your agent's skills directory
2. The skills will be automatically detected

### For Custom Agent Setup

Copy these skill files to your agent's skill folder:
- `skills/engineer-workflow.md` - Main unified workflow
- `skills/planner.md` - Planning component
- `skills/detail_planning.md` - Detail planning component
- `skills/implement.md` - Implementation component
- `skills/verify.md` - Verification component
- `skills/code-review.md` - Code review engine (independent, works with any workflow)

## 📖 Usage

### Quick Start

```
engineer-workflow: I want to build a task management app
```

### Phase Commands

| Command | Description |
|---------|-------------|
| `planner` | Start planning phase - generates structured plan with F1, F2, F3... phases |
| `detail_planning F1` | Expand phase F1 with detailed file paths and implementation steps |
| `implement F1` | Generate code for phase F1 |
| `verify F1` | Validate phase F1 implementation |
| `continue` | Move to next phase |
| `history` | View execution history |
| `/review-diff` | Review current branch diff against main (includes babysit) |
| `/review-uncommitted` | Review all uncommitted changes |
| `/review` | Full end-to-end repository audit |
| `/review-inscope` | Review changes against the stated task scope |

### Complete Workflow Example

```bash
# Step 1: Start with an idea
> engineer-workflow: Build a React todo app with Node.js API

# Output: Generates plan with F1, F2, F3 phases

# Step 2: Expand a phase
> detail_planning F1

# Output: Detailed breakdown with file paths

# Step 3: Generate code
> implement F1

# Output: Production-ready code

# Step 4: Verify implementation
> verify F1

# Output: Verification report with fix recommendations
```

## 📋 Output Formats

### Planning Output
```md
# Plan: Project Title

## Phase F1: Project Setup
- Status: Not Started
- Description: Initialize project

### Breakdown
- Initialize React project
- Configure TypeScript
- Set up state management
```

### Detail Planning Output
```md
## Phase F1: Project Setup - Detailed Breakdown

### Step 1: Initialize Project
- File: d:\Code\myapp\src\App.tsx
- Action: Create React app structure
```

### Implementation Output
```typescript
// File: d:\Code\myapp\src\App.tsx
export function App() {
  return <div>Hello World</div>;
}
```

### Verification Output
```
┌────────────────────────────────────┐
│ VERIFICATION CHECKLIST             │
├────────────────────────────────────┤
│ [✓] Component implemented         │
│ [✗] Missing method: update()      │
└────────────────────────────────────┘
```

## 🔄 Greenfield vs Brownfield

### Greenfield (New Projects)
Simply describe your idea:
```
engineer-workflow: I want to build a trading dashboard
```

### Brownfield (Existing Plans)
Analyze an existing plan file:
```
engineer-workflow: analyze plan.md
```

## 🎯 Use Cases

| Use Case | Recommended Command |
|----------|-------------------|
| New project from scratch | `engineer-workflow: [idea]` |
| Analyze existing TODO.md | `planner: analyze TODO.md` |
| Expand specific phase | `detail_planning F2` |
| Generate code | `implement F2` |
| Validate implementation | `verify F2` |
| Continue workflow | `continue` |
| Review a PR / branch diff | `/review-diff` |
| Check uncommitted work | `/review-uncommitted` |
| Audit the full repository | `/review` |
| Validate task completion | `/review-inscope` |

## 🔧 Configuration

### Project Root Notation
The workflow uses `d:\Code\` notation for file paths. Update paths as needed for your project structure.

### History Files
Plans are automatically saved to `History/` subfolder with timestamps:
- `History/project-name-2026-03-10.md`

## 📦 Dependencies

No external dependencies required. This is a skill framework for AI agents.

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch
3. Submit a pull request

## 📄 License

MIT License

## 🔗 Related

- [Planner Skill](skills/planner.md) - Planning component
- [Detail Planning Skill](skills/detail_planning.md) - Phase expansion
- [Implement Skill](skills/implement.md) - Code generation
- [Verify Skill](skills/verify.md) - Validation
- [Code Review Skill](skills/code-review.md) - Multi-mode code review engine

---

**Note:** Individual skill files (`planner.md`, `detail_planning.md`, `implement.md`, `verify.md`) remain fully functional and can be used independently. The `engineer-workflow.md` provides the unified experience combining all four phases. The `code-review.md` skill is fully independent and can be used at any point in the development lifecycle.
