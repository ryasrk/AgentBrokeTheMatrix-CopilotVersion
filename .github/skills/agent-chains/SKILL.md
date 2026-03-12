---
name: agent-chains
description: Pre-defined multi-agent chains for common development workflows — new API endpoint, new React component, ML pipeline, full-stack feature. Planner selects the chain and runs it automatically.
---

# Agent Chains — Pre-Built Multi-Agent Workflows

Reusable sequences of agent dispatches for common development tasks. Instead of the planner planning from scratch every time, it picks the matching chain and executes it.

## When to Activate

- Planner receives a task that matches a known pattern
- Task is a common development workflow (new endpoint, new component, etc.)
- Speed matters more than custom planning

## Available Chains

### Chain: New API Endpoint

**Trigger**: "add endpoint", "create route", "new API", "REST endpoint"

```
Phase 1 (parallel):
├── database-reviewer → Design schema/migration if new data needed
└── api-designer → Define endpoint contract (method, path, request/response shapes)

Phase 2:
└── tdd-guide → Write tests first, then implement endpoint

Phase 3 (parallel):
├── code-reviewer → Review implementation
└── security-reviewer → Check auth, input validation, injection

Phase 4 (optional):
└── performance-optimizer → Check query efficiency, add caching if needed
```

**Memory flow:**
- Phase 1 writes schema + contract to `/memories/session/backend-phase-results.md`
- Phase 2 reads it before implementing
- Phase 3 reads implementation for review

---

### Chain: New React Component

**Trigger**: "create component", "new page", "build UI", "new form"

```
Phase 1:
└── frontend-reviewer → Design component architecture (props, state, composition)

Phase 2:
└── tdd-guide → Write component tests, then implement

Phase 3 (parallel):
├── ui-ux-auditor → Accessibility audit (WCAG 2.1 AA)
└── code-reviewer → Code quality review

Phase 4 (optional):
└── e2e-runner → E2E test for the new component flow
```

**Memory flow:**
- Phase 1 writes component design to `/memories/session/frontend-phase-results.md`
- Phase 2-3 read it for context

---

### Chain: ML Pipeline

**Trigger**: "train model", "ML pipeline", "object detection", "inference"

```
Phase 1 (parallel):
├── cv-specialist → Data pipeline design (preprocessing, augmentation)
└── library-docs-checker → Verify framework API versions (ultralytics, torch, etc.)

Phase 2:
└── ai-ml-engineer → Training loop implementation + evaluation

Phase 3:
└── performance-optimizer → Inference optimization (batch, FP16, model export)

Phase 4 (parallel):
├── code-reviewer → Review ML code
└── tdd-guide → Add tests for data pipeline + inference
```

**Memory flow:**
- Phase 1 writes data pipeline design + verified APIs to `/memories/session/ai-phase-results.md`
- Phase 2-3 read it for training and optimization

---

### Chain: Full-Stack Feature

**Trigger**: "full-stack", "end-to-end feature", "complete feature"

```
Phase 0:
└── architect → High-level design, component breakdown

Phase 1 (parallel - use coordinators):
├── backend-coordinator → Schema + API + Auth
└── ai-coordinator → ML components (if AI feature)

Phase 2:
└── frontend-coordinator → UI components consuming backend API

Phase 3 (parallel):
├── code-reviewer → Review all code
├── security-reviewer → Full security audit
└── e2e-runner → E2E tests for the complete flow

Phase 4:
├── devops-engineer → CI/CD + Docker for deployment
└── self-improver → Analyze session for improvements
```

**Memory flow:**
- Phase 0 writes architecture to `/memories/session/project-context.md`
- Phase 1 coordinators write to their domain results
- Phase 2 reads backend results for API contracts
- Phase 3-4 read all results

---

### Chain: Bug Fix

**Trigger**: "fix bug", "debug", "broken", "not working", "error"

```
Phase 1:
└── Explore → Find the root cause (search errors, read related files)

Phase 2:
└── tdd-guide → Write failing test that reproduces the bug, then fix

Phase 3 (parallel):
├── code-reviewer → Verify fix doesn't introduce regressions
└── library-docs-checker → Check if bug is a known library issue

Phase 4:
└── security-reviewer → Check if bug had security implications
```

---

### Chain: Code Refactor

**Trigger**: "refactor", "clean up", "improve code", "simplify"

```
Phase 1:
└── refactor-cleaner → Identify dead code, duplicates, complexity

Phase 2:
└── tdd-guide → Ensure test coverage before refactoring (80%+ required)

Phase 3:
└── architect → Review architectural impact of refactoring

Phase 4 (parallel):
├── code-reviewer → Review refactored code
└── performance-optimizer → Verify no performance regressions
```

---

### Chain: Auth Implementation

**Trigger**: "add auth", "login", "authentication", "authorization", "JWT"

```
Phase 1:
└── auth-specialist → Design auth flow (JWT, OAuth, RBAC, sessions)

Phase 2 (parallel):
├── database-reviewer → User/role/session tables
└── api-designer → Auth endpoints (login, register, refresh, logout)

Phase 3:
└── tdd-guide → Implement with tests

Phase 4 (parallel):
├── security-reviewer → Full auth security audit
└── frontend-reviewer → Auth state management, route guards

Phase 5:
└── e2e-runner → E2E auth flow tests
```

---

### Chain: Database Migration

**Trigger**: "add table", "schema change", "migration", "alter column"

```
Phase 1:
└── database-reviewer → Design schema change + write migration

Phase 2:
└── tdd-guide → Tests for new queries/models

Phase 3 (parallel):
├── performance-optimizer → Index strategy for new schema
└── code-reviewer → Review migration safety (reversibility, zero-downtime)
```

---

### Chain: CI/CD Setup

**Trigger**: "add CI", "GitHub Actions", "deploy", "Docker", "pipeline"

```
Phase 1:
└── devops-engineer → CI/CD pipeline + Dockerfile + docker-compose

Phase 2:
└── security-reviewer → Review pipeline security (secrets, permissions)

Phase 3:
└── code-reviewer → Review infrastructure code
```

## How Planner Uses Chains

1. **Pattern match** — Compare user request against chain triggers
2. **Select chain** — Pick the best matching chain
3. **Customize** — Adjust phases if task has unique requirements
4. **Execute** — Run the chain with memory blackboard protocol
5. **Skip optional phases** — Only run optional phases if the user asks or if issues are detected

## Creating Custom Chains

Add new chains following this template:

```markdown
### Chain: [Name]

**Trigger**: "[keyword1]", "[keyword2]", "[keyword3]"

Phase 1:
└── [agent] → [task description]

Phase 2 (parallel):
├── [agent] → [task description]
└── [agent] → [task description]

Phase 3:
└── [agent] → [task description]
```

Rules:
- Keep chains to 3-5 phases
- Maximize parallelism within each phase
- Always include a review phase (code-reviewer and/or security-reviewer)
- Use memory blackboard for cross-phase context
