---
name: cost-aware-model-routing
description: Guidelines for assigning the right model tier (opus vs sonnet) to each agent based on task complexity. Opus for planning, architecture, security, and novel reasoning. Sonnet for reviews, audits, scanning, and pattern-matching tasks.
---

# Cost-Aware Model Routing

Assign the cheapest model that can handle each agent's workload. Opus is 5-10x more expensive than Sonnet — use it only when the task requires deep reasoning, novel problem-solving, or complex orchestration.

## Model Tier Assignments

### Tier 1: Opus (Complex Reasoning)

Use `claude-opus-4-6` for agents that need:
- Multi-step orchestration and planning
- Novel architectural decisions
- Deep security analysis with creative threat modeling
- Understanding complex codebase relationships
- Cross-domain synthesis

| Agent | Why Opus |
|-------|----------|
| planner | Multi-agent orchestration, phased planning, dependency analysis |
| architect | Novel system design, trade-off analysis, scalability reasoning |
| security-reviewer | Creative threat modeling, exploit chain identification |
| ai-ml-engineer | Model architecture decisions, training strategy |
| prompt-engineer | Prompt design requires meta-reasoning about language |
| cv-specialist | Vision pipeline design, model selection reasoning |
| auth-specialist | Auth flow design, token lifecycle, threat modeling |
| ai-coordinator | Multi-agent dispatch within AI domain |
| backend-coordinator | Multi-agent dispatch within backend domain |
| frontend-coordinator | Multi-agent dispatch within frontend domain |
| self-improver | Pattern analysis across sessions, system optimization |
| library-docs-checker | Web research synthesis, version comparison reasoning |

### Tier 2: Sonnet (Pattern Matching & Review)

Use `claude-sonnet-4-6` for agents that:
- Follow established patterns and checklists
- Apply known rules (coding standards, accessibility rules)
- Perform structured analysis (line-by-line review)
- Execute well-defined tasks with clear success criteria

| Agent | Why Sonnet |
|-------|-----------|
| code-reviewer | Applies known code quality patterns and checklists |
| tdd-guide | Follows TDD methodology (write test → implement → refactor) |
| build-error-resolver | Pattern match error messages → apply known fixes |
| e2e-runner | Generate tests from known Playwright patterns |
| refactor-cleaner | Dead code detection follows mechanical rules |
| doc-updater | Documentation follows templates |
| database-reviewer | Schema review follows indexing and normalization rules |
| go-reviewer | Go review follows established idioms |
| go-build-resolver | Build fix follows error-message patterns |
| python-reviewer | PEP 8 compliance is rule-based |
| api-designer | API design follows REST/GraphQL conventions |
| performance-optimizer | Profiling follows known optimization checklist |
| frontend-reviewer | React/Vue review follows known patterns |
| ui-ux-auditor | WCAG compliance is checklist-based |
| devops-engineer | CI/CD pipeline follows templates |
| eval-agent | Scoring follows defined rubric |
| project-scanner | Structure scanning is mechanical |
| harness-optimizer | Configuration optimization follows patterns |
| loop-operator | Loop monitoring follows checkpoints |
| chief-of-staff | Message triage follows classification rules |

## Cost Impact

Assuming equal token usage:
- 10 Opus dispatches ≈ cost of 50-100 Sonnet dispatches
- Typical session: 3 Opus + 5 Sonnet instead of 8 Opus = ~40% cost reduction
- Over 100 sessions: significant cumulative savings

## Re-Routing Rules

Move an agent from Sonnet to Opus if:
- The agent consistently scores <3/5 on output quality
- The task requires reasoning beyond pattern matching
- The agent needs to handle edge cases not covered by skill files

Move an agent from Opus to Sonnet if:
- The agent follows a fixed checklist >90% of the time
- Output quality doesn't change between models (test this)
- The task is well-covered by skill file patterns

## Current Assignment Changes

Based on the routing above, the following agents should be re-assigned:

### Downgrade to Sonnet (currently Opus):
- `code-reviewer` → Sonnet (follows checklists)
- `database-reviewer` → Sonnet (follows indexing/normalization rules)

### Keep on Opus (confirmed):
- `planner`, `architect`, `security-reviewer` — require deep reasoning
- `ai-ml-engineer`, `prompt-engineer`, `cv-specialist` — require domain expertise
- `auth-specialist` — security-critical reasoning
- All coordinators — multi-agent orchestration
- `library-docs-checker` — web research synthesis
- `self-improver` — cross-session pattern analysis
