# AgentBrokeTheMatrix — Copilot Version

A collection of **33 specialized agents** (including 3 domain coordinators) and **83 domain skills** for GitHub Copilot Chat in VS Code. Designed for AI-powered software development with maximum parallelism — a planner agent orchestrates sub-agents across AI/ML, Backend, Frontend, DevOps, and more.

---

## Prerequisites

Install the **TaskSync** extension for VS Code — it enables the interactive feedback loop (`ask_user`) used by the planner and other agents:

1. Open VS Code
2. Go to **Extensions** (`Ctrl+Shift+X`)
3. Search for **TaskSync**
4. Click **Install**

> The extension ID is `4regab.tasksync-chat`. Without it, agents cannot ask follow-up questions during execution.

---

## Installation

### Option A: Clone and work inside the repo

```bash
git clone https://github.com/ryasrk/AgentBrokeTheMatrix-CopilotVersion.git
cd AgentBrokeTheMatrix-CopilotVersion
```

Open the folder in VS Code and start building your project here. The `.github/` directory is automatically picked up by Copilot Chat.

### Option B: Copy `.github/` to your existing workspace

```bash
# Clone the repo somewhere temporary
git clone https://github.com/ryasrk/AgentBrokeTheMatrix-CopilotVersion.git /tmp/abtm

# Copy agents and skills into your project
cp -r /tmp/abtm/.github /path/to/your/project/
```

On Windows (PowerShell):

```powershell
git clone https://github.com/ryasrk/AgentBrokeTheMatrix-CopilotVersion.git C:\temp\abtm
Copy-Item -Recurse C:\temp\abtm\.github C:\path\to\your\project\.github
```

### Option C: Copy to global VS Code settings (all workspaces)

Copy the `.github/` folder to your VS Code user-level configuration so every workspace can access the agents and skills:

```powershell
# Windows
Copy-Item -Recurse .github "$env:APPDATA\Code\User\.github"
```

```bash
# macOS / Linux
cp -r .github ~/.config/Code/User/.github
```

---

## Usage

Once installed, open **Copilot Chat** in VS Code and select an agent from the agent picker (the `@` menu).

**For maximum power, always use the `@planner` agent.** It is the orchestrating brain that commands all other agents.

---

## The Planner Agent — Your Most Powerful Tool

The **`@planner`** agent is the central orchestrator and the recommended entry point for **every task** — from simple bug fixes to complex multi-service features. Instead of manually picking individual agents, give your task to the planner and let it handle everything.

### Why Planner is the Most Powerful

| Capability | What It Does |
|-----------|--------------|
| **Parallel Dispatch** | Launches multiple sub-agents simultaneously — architect + tdd-guide + database-reviewer can all run at once |
| **Automatic Routing** | Detects the domain (AI, backend, frontend, security, etc.) and routes to the right specialist agent |
| **Skill Loading** | Loads relevant domain knowledge files before dispatching, so agents have full context |
| **Review Gates** | Automatically runs code-reviewer + security-reviewer after every code-writing wave |
| **Interactive Loop** | Uses TaskSync to keep you in the loop — asks for feedback after each phase, never ends without your permission |
| **Phased Execution** | Breaks large features into independently deliverable phases with dependency tracking |

### How to Use the Planner

1. **Select `@planner`** from the agent picker in Copilot Chat
2. **Describe your task** in natural language — be as specific or vague as you want
3. **The planner will**:
   - Explore your codebase to understand the structure
   - Load relevant skills for the detected domains
   - Create a phased plan with parallel agent assignments
   - Ask for your approval before executing
   - Dispatch agents in maximum-parallel batches
   - Report results and ask for feedback after each phase
4. **Give feedback** — the loop continues until you say "done" or "stop"

### Example Prompts for the Planner

```
# Simple — planner routes to the right agent automatically
"Fix the authentication bug in the login endpoint"

# Medium — planner creates a 2-phase plan
"Add rate limiting to all API endpoints with Redis caching"

# Complex — planner creates multi-phase plan with 5+ parallel agents
"Build a complete user dashboard with authentication, real-time notifications,
 REST API, database schema, E2E tests, and deploy it with Docker"

# AI/ML — planner routes to ai-ml-engineer + cv-specialist
"Train a YOLO model for license plate detection with data augmentation
 and export to ONNX for production inference"

# Full stack — planner dispatches frontend + backend + database + devops
"Create a new microservice for payment processing with Stripe integration,
 PostgreSQL schema, React checkout form, and CI/CD pipeline"
```

### Planner's Agent Routing Table

When you give the planner a task, it automatically routes subtasks to these specialists:

| Task Domain | Dispatched Agent |
|------------|-----------------|
| Architecture / system design | `architect` |
| Write tests first (TDD) | `tdd-guide` |
| Code review | `code-reviewer` |
| Security audit | `security-reviewer` |
| Database schema / queries | `database-reviewer` |
| E2E / Playwright tests | `e2e-runner` |
| Dead code cleanup | `refactor-cleaner` |
| Documentation | `doc-updater` |
| Build / type errors | `build-error-resolver` |
| Go code | `go-reviewer` |
| Go build errors | `go-build-resolver` |
| Python code | `python-reviewer` |
| ML training / inference | `ai-ml-engineer` |
| LLM / RAG / prompts | `prompt-engineer` |
| Computer vision / YOLO | `cv-specialist` |
| API design / OpenAPI | `api-designer` |
| Auth / OAuth / JWT | `auth-specialist` |
| Performance / profiling | `performance-optimizer` |
| React / Next.js / Vue | `frontend-reviewer` |
| Accessibility / WCAG | `ui-ux-auditor` |
| CI/CD / Docker / IaC | `devops-engineer` |
| Library version checks | `library-docs-checker` |

### Planner Workflow Phases

```
Phase 1: UNDERSTAND
├── Explore codebase structure
├── Detect domains (AI? Backend? Frontend?)
└── Load relevant skill files for context

Phase 2: PLAN
├── Draft phased plan with parallel groups
├── Assign agents to each step
├── Identify dependencies between steps
└── Ask user for approval

Phase 3: EXECUTE (repeats per batch)
├── Dispatch independent agents in PARALLEL
├── Collect results from all agents
├── Run review gates (code-reviewer + security-reviewer)
├── Update progress, summarize results
└── Ask user for feedback before next batch

Phase 4: CLOSE OUT
├── Final review wave on all changes
├── Summarize everything done
└── Only end when user says "done"
```

### Tips for Getting the Most Out of Planner

- **Be specific about requirements** — "Add user auth with JWT, refresh tokens, and RBAC for admin/user roles" gets better results than "add auth"
- **Let it plan first** — Don't skip the planning phase. Review the plan and give feedback before execution starts
- **Use it for everything** — Even simple tasks benefit from automatic review gates
- **Trust the parallelism** — The planner will run 3-5 agents simultaneously when possible, dramatically speeding up complex work
- **Give feedback at checkpoints** — The planner asks after each phase. Use this to course-correct early

---

## All Agents (33)

### Domain Coordinators (context multipliers)

| Agent | Manages | When to Use |
|-------|---------|-------------|
| ai-coordinator | ai-ml-engineer, prompt-engineer, cv-specialist | Tasks spanning 2+ AI sub-domains |
| backend-coordinator | api-designer, auth-specialist, performance-optimizer, database-reviewer | Tasks spanning 2+ backend sub-domains |
| frontend-coordinator | frontend-reviewer, ui-ux-auditor | Tasks spanning 2+ frontend sub-domains |

Each coordinator gets its own **full context window (~200K tokens)**, so dispatching 3 coordinators in parallel gives you ~600K effective tokens.

### Specialist Agents

| Agent | Domain | Purpose |
|-------|--------|---------|
| planner | Orchestration | Breaks work into parallel sub-agent dispatches |
| architect | Architecture | System design, scalability, technical decisions |
| code-reviewer | Quality | Code review for quality, security, maintainability |
| tdd-guide | Testing | Test-driven development enforcement |
| security-reviewer | Security | Vulnerability detection, OWASP Top 10 |
| build-error-resolver | Build | Fix build/type errors quickly |
| e2e-runner | Testing | Playwright E2E testing |
| refactor-cleaner | Maintenance | Dead code cleanup, consolidation |
| doc-updater | Documentation | Codemaps and docs generation |
| database-reviewer | Database | PostgreSQL/Supabase specialist |
| go-reviewer | Go | Go code review |
| go-build-resolver | Go | Go build error resolution |
| python-reviewer | Python | Python code review (PEP 8, type hints) |
| ai-ml-engineer | AI/ML | Training, inference, data pipelines, MLOps |
| prompt-engineer | AI/LLM | Prompt design, RAG, tool calling, cost routing |
| cv-specialist | AI/Vision | Object detection, OpenCV, OCR, YOLO |
| api-designer | Backend | REST/GraphQL API contracts, OpenAPI specs |
| auth-specialist | Backend | OAuth2, JWT, RBAC, session management |
| performance-optimizer | Backend | Profiling, caching, query tuning |
| frontend-reviewer | Frontend | React/Next.js/Vue code review |
| ui-ux-auditor | Frontend | WCAG accessibility, design systems |
| devops-engineer | DevOps | CI/CD, Docker, monitoring, IaC |
| library-docs-checker | Research | Fetches live docs to verify API syntax and versions |
| chief-of-staff | Communication | Email/Slack triage and draft replies |
| harness-optimizer | Agent Ops | Optimize agent harness configuration |
| loop-operator | Agent Ops | Monitor autonomous agent loops |
| tasksync | Interaction | Interactive feedback loop protocol |
| self-improver | System Ops | Analyzes sessions, refines skills and routing rules |
| eval-agent | System Ops | Scores dispatch quality, tracks orchestration metrics |
| project-scanner | System Ops | Auto-generates project snapshot for instant context |

---

## Skills (83)

Skills are domain knowledge files that agents load for context. Organized under `.github/skills/`:

<details>
<summary>View all 83 skills</summary>

| Skill | Domain |
|-------|--------|
| accessibility-patterns | Frontend |
| agent-chains | Orchestration |
| agent-harness-construction | Agent Ops |
| agentic-engineering | Agent Ops |
| ai-first-engineering | Agent Ops |
| ai-ml-patterns | AI/ML |
| api-design | Backend |
| article-writing | Content |
| auth-patterns | Security |
| autonomous-loops | Agent Ops |
| backend-patterns | Backend |
| benchmarking-dashboard | System Ops |
| clickhouse-io | Database |
| coding-standards | Standards |
| computer-vision-patterns | AI/Vision |
| configure-ecc | Tooling |
| content-engine | Content |
| content-hash-cache-pattern | Backend |
| context-efficient-dispatch | Agent Ops |
| continuous-agent-loop | Agent Ops |
| continuous-learning | Agent Ops |
| continuous-learning-v2 | Agent Ops |
| cost-aware-llm-pipeline | AI/LLM |
| cost-aware-model-routing | System Ops |
| cpp-coding-standards | C++ |
| cpp-testing | C++ |
| database-migrations | Database |
| deployment-patterns | DevOps |
| devops-patterns | DevOps |
| django-patterns | Python |
| django-security | Python |
| django-tdd | Python |
| django-verification | Python |
| docker-patterns | DevOps |
| e2e-testing | Testing |
| enterprise-agent-ops | Agent Ops |
| eval-harness | Agent Ops |
| foundation-models-on-device | AI/Mobile |
| frontend-patterns | Frontend |
| frontend-slides | Frontend |
| golang-patterns | Go |
| golang-testing | Go |
| graphql-patterns | Backend |
| investor-materials | Business |
| investor-outreach | Business |
| iterative-retrieval | Agent Ops |
| java-coding-standards | Java |
| jpa-patterns | Java |
| library-docs-verification | Research |
| liquid-glass-design | iOS |
| market-research | Business |
| memory-blackboard | Agent Ops |
| micro-skills-index | Agent Ops |
| nanoclaw-repl | Tooling |
| nutrient-document-processing | Tooling |
| performance-optimization | Backend |
| plankton-code-quality | Quality |
| postgres-patterns | Database |
| project-guidelines-example | Templates |
| prompt-engineering | AI/LLM |
| python-patterns | Python |
| python-testing | Python |
| ralphinho-rfc-pipeline | Agent Ops |
| realtime-patterns | Backend |
| regex-vs-llm-structured-text | AI/Parsing |
| search-first | Workflow |
| security-review | Security |
| security-scan | Security |
| skill-stocktake | Tooling |
| springboot-patterns | Java |
| springboot-security | Java |
| springboot-tdd | Java |
| springboot-verification | Java |
| state-management-patterns | Frontend |
| strategic-compact | Agent Ops |
| swift-actor-persistence | Swift |
| swift-concurrency-6-2 | Swift |
| swift-protocol-di-testing | Swift |
| swiftui-patterns | Swift |
| tdd-workflow | Testing |
| verification-loop | Quality |
| visa-doc-translate | Tooling |

</details>

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│                   YOU (User)                     │
│         "Build a payment microservice"           │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│              @planner (Orchestrator)              │
│  • Understands request                           │
│  • Loads skills (auth-patterns, api-design, ...) │
│  • Creates phased plan                           │
│  • Dispatches agents in parallel                 │
│  • Reports back via TaskSync                     │
└──┬──────────┬──────────┬──────────┬─────────────┘
   │          │          │          │
   ▼          ▼          ▼          ▼
┌──────┐ ┌──────┐ ┌──────┐ ┌──────────────┐
│ api- │ │ auth │ │ tdd- │ │  database-   │
│design│ │ spec │ │guide │ │  reviewer    │
│  er  │ │ialist│ │      │ │              │
└──┬───┘ └──┬───┘ └──┬───┘ └──────┬───────┘
   │        │        │            │
   └────────┴────────┴────────────┘
                    │
                    ▼
        ┌───────────────────────┐
        │  Review Gate          │
        │  code-reviewer +      │
        │  security-reviewer    │
        └───────────┬───────────┘
                    │
                    ▼
              Results → You
```

The planner is the **only agent that talks to you**. Sub-agents report back to the planner, which synthesizes results and asks for your feedback. This means you get a single, coherent conversation — not 5 agents talking over each other.

---

## Context Optimization Strategies

These built-in strategies maximize agent power within limited context windows:

### 1. Hierarchical Dispatch (Context Multiplier)

```
❌ Flat dispatch (1 context window):
Planner → [agent1, agent2, agent3, agent4, agent5, agent6]
           ↑ all compete for ~200K tokens

✅ Hierarchical dispatch (3 context windows):
Planner → ai-coordinator → [ai-ml-engineer, cv-specialist]     ← 200K
        → backend-coordinator → [api-designer, auth-specialist] ← 200K
        → frontend-coordinator → [frontend-reviewer, ui-ux-auditor] ← 200K
                                                        Total: ~600K effective
```

### 2. Memory Blackboard (Cross-Agent Context Sharing)

Agents write findings to `/memories/session/`, later agents read instead of re-exploring:

```
Agent A explores codebase → writes to memory (5K tokens)
Agent B reads memory → saves 4.8K tokens of redundant exploration
Agent C reads memory → saves 4.8K tokens of redundant exploration
Net savings: ~65% per subsequent agent
```

### 3. Context Packets (10x Compression)

Instead of sending raw files to sub-agents, the planner sends compressed summaries:

```
Raw file: ~3000 tokens
Context packet (signatures + task + constraints): ~300 tokens
Compression: 10x with zero information loss for the task
```

### 4. Micro-Skills Loading

Load only the specific section of a skill file needed for the task:

```
Full skill load: 1500-2500 tokens
Micro-load (1 section): 200-400 tokens
Savings: ~80% per skill load
```

### 5. Result Caching Between Phases

Phase 1 results are cached in session memory so Phase 2 agents don't redo exploration work.

See the `context-efficient-dispatch`, `memory-blackboard`, and `micro-skills-index` skills for full details.

---

## Advanced Features

### Self-Improving System

After each multi-agent session, the planner automatically dispatches:
1. **`eval-agent`** — Scores routing accuracy, context efficiency, parallelism, and output quality (A-F grade)
2. **`self-improver`** — Analyzes the eval report, identifies inefficiencies, updates skill files and micro-skills index

Over time, the system learns which routing decisions work best and which skills need updating.

### Pre-Built Agent Chains

Common workflows are pre-defined in the `agent-chains` skill:

| Chain | Trigger | Agents (in order) |
|-------|---------|-------------------|
| New API Endpoint | "add endpoint" | database-reviewer → api-designer → tdd-guide → code-reviewer → security-reviewer |
| New React Component | "create component" | frontend-reviewer → tdd-guide → ui-ux-auditor → code-reviewer |
| ML Pipeline | "train model" | cv-specialist → ai-ml-engineer → performance-optimizer → code-reviewer |
| Full-Stack Feature | "full-stack" | architect → [backend-coordinator + ai-coordinator] → frontend-coordinator → reviews |
| Bug Fix | "fix bug" | Explore → tdd-guide → code-reviewer → library-docs-checker |
| Auth Implementation | "add auth" | auth-specialist → [database-reviewer + api-designer] → security-reviewer |

The planner selects the matching chain and runs it automatically.

### Cost-Aware Model Routing

Not every agent needs the most expensive model:
- **Opus** (complex reasoning): planner, architect, security-reviewer, coordinators, AI specialists
- **Sonnet** (pattern matching): code-reviewer, tdd-guide, build-error-resolver, devops-engineer, scanners

This reduces cost by ~40% with no quality loss for pattern-matching tasks.

### Project Snapshots

The `project-scanner` agent auto-generates a ~500-token project summary (tech stack, structure, key files, entry points). This loads instantly in every session, replacing 5-10 codebase exploration calls that waste ~5000 tokens.

### Benchmarking Dashboard

Persistent tracking of agent metrics across sessions:
- Agent utilization (which agents are used vs unused)
- Routing accuracy trends
- Context efficiency improvements
- Session scores over time

---

## License

MIT
