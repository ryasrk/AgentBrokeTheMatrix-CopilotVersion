---
name: memory-blackboard
description: Shared memory protocol for multi-agent workflows — session memory as a blackboard for inter-agent communication, result caching, and progressive context building.
origin: AgentBrokeTheMatrix
---

# Memory Blackboard Pattern

Use session memory (`/memories/session/`) as a **shared blackboard** where agents write findings and later agents read them — eliminating redundant codebase exploration and enabling cross-agent context sharing.

## When to Activate

- Multi-phase tasks where Phase 2 depends on Phase 1 results
- Multiple agents need to understand the same part of the codebase
- Cross-domain work (AI → Backend → Frontend) where each domain builds on the previous
- Any task dispatching 3+ agents

## The Problem

Without a blackboard:
- Agent A explores codebase, finds auth uses JWT → spends 5K tokens
- Agent B explores same codebase, finds auth uses JWT → spends 5K tokens
- Agent C explores same codebase, finds auth uses JWT → spends 5K tokens
- **Total waste: 10K tokens on redundant exploration**

With a blackboard:
- Agent A explores, writes findings to memory → 5K tokens + 200 token write
- Agent B reads memory → 200 token read (saves 4.8K tokens)
- Agent C reads memory → 200 token read (saves 4.8K tokens)
- **Total savings: ~9.6K tokens**

## Blackboard Structure

```
/memories/session/
├── project-context.md        # Planner writes: project overview, tech stack, structure
├── ai-phase-results.md       # AI Coordinator writes: model decisions, training results
├── backend-phase-results.md  # Backend Coordinator writes: API, schema, auth decisions
├── frontend-phase-results.md # Frontend Coordinator writes: components, state, a11y
├── review-findings.md        # Code reviewer writes: issues found, fixes needed
└── security-findings.md      # Security reviewer writes: vulnerabilities, mitigations
```

## Protocol

### Step 1: Planner Writes Project Context (Once)

At the start of any multi-agent task, the planner writes:

```markdown
# /memories/session/project-context.md

## Tech Stack
- Language: Python 3.11
- Framework: FastAPI
- Database: PostgreSQL 15 + SQLAlchemy
- Frontend: React 18 + TypeScript + Tailwind
- Testing: pytest + Playwright
- CI/CD: GitHub Actions

## Project Structure
- src/api/ — FastAPI routes
- src/models/ — SQLAlchemy models
- src/services/ — Business logic
- frontend/src/ — React app
- tests/ — Python tests

## Current Task
Build user dashboard with real-time notifications

## Relevant Files
- src/api/users.py — User CRUD endpoints
- src/models/user.py — User model
- frontend/src/pages/Dashboard.tsx — Current (empty) dashboard
```

### Step 2: Coordinator Writes Phase Results

After each coordinator completes its phase:

```markdown
# /memories/session/backend-phase-results.md

## Phase: Backend API Implementation
## Status: Complete
## Timestamp: Phase 2

### Endpoints Created
- GET /api/v1/dashboard — Returns user stats + recent activity
- GET /api/v1/notifications — SSE stream for real-time notifications
- PUT /api/v1/notifications/:id/read — Mark notification as read

### Schema Changes
- Added: notifications table (id, user_id, type, message, read, created_at)
- Added: index on (user_id, read, created_at) for unread query
- Migration: src/migrations/003_add_notifications.py

### Auth Decisions
- Dashboard endpoints require Bearer JWT
- SSE stream uses token in query param (SSE can't set headers)
- Added 1-hour SSE token with narrow scope

### Key Files
- src/api/dashboard.py (NEW) — Dashboard endpoints
- src/api/notifications.py (NEW) — SSE stream + CRUD
- src/models/notification.py (NEW) — Notification model
- src/services/notification_service.py (NEW) — Business logic

### For Frontend Team
- SSE endpoint: GET /api/v1/notifications?token=<sse_token>
- Event format: { type: "notification", data: { id, type, message, created_at } }
- Dashboard data shape: { stats: { total_users, active_today }, recent: Activity[] }
```

### Step 3: Next Agent Reads Before Working

Every agent prompt should include:
```
Memory: Read /memories/session/ for context from prior phases.
```

The agent reads the blackboard, gets:
- What's already been built (no need to re-explore)
- API contracts (frontend knows exact endpoints)
- Schema decisions (no conflicting designs)
- Auth patterns (consistent security approach)

## File Naming Conventions

| File | Who Writes | When |
|------|-----------|------|
| `project-context.md` | Planner | Start of task |
| `ai-phase-results.md` | AI Coordinator | After AI phase completes |
| `backend-phase-results.md` | Backend Coordinator | After backend phase completes |
| `frontend-phase-results.md` | Frontend Coordinator | After frontend phase completes |
| `review-findings.md` | code-reviewer | After review gate |
| `security-findings.md` | security-reviewer | After security audit |
| `phase-N-summary.md` | Planner | After each major phase |

## Reading Protocol

Before starting any work, agents should:

1. Check if `/memories/session/` has relevant files
2. Read files that match their domain
3. **Do NOT read files from unrelated domains** (AI coordinator doesn't need frontend results)
4. Extract only the parts relevant to their current task

### Smart Reading

```
# Backend coordinator starting Phase 2
# Read these:
✅ /memories/session/project-context.md (always)
✅ /memories/session/ai-phase-results.md (if backend depends on AI decisions)

# Don't read these:
❌ /memories/session/frontend-phase-results.md (not written yet)
❌ /memories/session/review-findings.md (not relevant to new work)
```

## Writing Protocol

### Rules for Writing
1. **Be concise** — aim for 200-500 tokens per file
2. **Include file paths** — other agents need to know exactly where things are
3. **Include decisions with rationale** — prevents later agents from making conflicting choices
4. **Include interface contracts** — data shapes, endpoints, function signatures
5. **Overwrite, don't append** — each phase update replaces the previous content

### What NOT to Write
- Full source code (write file paths instead)
- Detailed implementation notes (only decisions and interfaces)
- Debugging logs or error traces
- Personal observations about code quality (save for review-findings)

## Context Savings Calculator

| Scenario | Without Blackboard | With Blackboard | Savings |
|----------|-------------------|-----------------|---------|
| 3 agents exploring same codebase | 15K tokens | 5.4K tokens | 64% |
| Backend → Frontend handoff | 8K tokens | 1K tokens | 87% |
| 5-phase project | 50K tokens | 12K tokens | 76% |
| Review after implementation | 10K tokens | 2K tokens | 80% |

## Anti-Patterns

| Anti-Pattern | Fix |
|-------------|-----|
| Not writing to memory after phase | Always write, even if results seem obvious |
| Writing raw code to memory | Write summaries, file paths, and decisions |
| Reading all memory files | Only read files relevant to current task |
| Appending to files across phases | Overwrite with current state |
| Skipping memory read before dispatching | Always check memory before every dispatch |
