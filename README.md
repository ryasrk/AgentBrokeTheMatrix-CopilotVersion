# AgentBrokeTheMatrix — Copilot Version

A collection of **27 specialized agents** and **76 domain skills** for GitHub Copilot Chat in VS Code. Designed for AI-powered software development with maximum parallelism — a planner agent orchestrates sub-agents across AI/ML, Backend, Frontend, DevOps, and more.

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

Once installed, open **Copilot Chat** in VS Code and select an agent from the agent picker (the `@` menu), or let the **planner** agent orchestrate automatically:

- **`@planner`** — Breaks complex tasks into parallel sub-agent dispatches
- **`@code-reviewer`** — Reviews code for quality, security, maintainability
- **`@tdd-guide`** — Enforces test-driven development workflow
- **`@architect`** — System design and scalability decisions
- **`@security-reviewer`** — OWASP Top 10 vulnerability detection

Use the planner for multi-step work — it will automatically route to the right agents in parallel.

---

## Agents (27)

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

---

## Skills (76)

Skills are domain knowledge files that agents load for context. Organized under `.github/skills/`:

<details>
<summary>View all 76 skills</summary>

| Skill | Domain |
|-------|--------|
| accessibility-patterns | Frontend |
| agent-harness-construction | Agent Ops |
| agentic-engineering | Agent Ops |
| ai-first-engineering | Agent Ops |
| ai-ml-patterns | AI/ML |
| api-design | Backend |
| article-writing | Content |
| auth-patterns | Security |
| autonomous-loops | Agent Ops |
| backend-patterns | Backend |
| clickhouse-io | Database |
| coding-standards | Standards |
| computer-vision-patterns | AI/Vision |
| configure-ecc | Tooling |
| content-engine | Content |
| content-hash-cache-pattern | Backend |
| continuous-agent-loop | Agent Ops |
| continuous-learning | Agent Ops |
| continuous-learning-v2 | Agent Ops |
| cost-aware-llm-pipeline | AI/LLM |
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

## How It Works

The **planner** agent is the orchestrator. When you give it a complex task:

1. **Understand** — Explores codebase, loads relevant skills
2. **Plan** — Creates a phased plan with parallel agent assignments
3. **Execute** — Dispatches sub-agents in maximum-parallel batches
4. **Review** — Runs code-reviewer + security-reviewer on all changes

Agents communicate back to the planner (never to the user directly), and the planner synthesizes results and asks for feedback via TaskSync.

---

## License

MIT
