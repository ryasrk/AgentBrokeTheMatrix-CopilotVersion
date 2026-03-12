---
name: benchmarking-dashboard
description: Track agent dispatch metrics — token usage, dispatch counts, routing accuracy, and session scores — in a persistent markdown file that accumulates data over time. Identifies high-value vs underutilized agents.
---

# Agent Benchmarking Dashboard

Track and visualize agent performance metrics across sessions. Maintained as a persistent markdown file that the eval-agent and self-improver update after each session.

## When to Activate

- After every multi-agent session (auto-triggered by planner in Phase 4)
- When analyzing agent utilization patterns
- When deciding which agents need improvement or removal
- Monthly system health review

## Dashboard Location

`/memories/repo/agent-benchmarks.md` — persists across sessions in repo memory.

## Dashboard Format

```markdown
# Agent Benchmarks
Last updated: [date]
Total sessions tracked: [N]

## Session Summary (Last 10)

| Session | Date | Task | Agents | Score | Tokens Est. |
|---------|------|------|--------|-------|-------------|
| #10 | 2025-01-15 | Full-stack dashboard | 8 | A (92) | ~50K |
| #9 | 2025-01-14 | Bug fix auth | 3 | B (78) | ~15K |
| ... | ... | ... | ... | ... | ... |

## Agent Utilization

| Agent | Dispatches | Avg Score | Last Used | Status |
|-------|-----------|-----------|-----------|--------|
| code-reviewer | 45 | 4.2/5 | Today | Active |
| tdd-guide | 38 | 4.5/5 | Today | Active |
| security-reviewer | 30 | 4.0/5 | Yesterday | Active |
| architect | 12 | 4.8/5 | 3 days ago | Active |
| cv-specialist | 5 | 3.8/5 | 1 week ago | Low use |
| go-reviewer | 0 | N/A | Never | Unused |
| go-build-resolver | 0 | N/A | Never | Unused |

## Routing Accuracy Trend

| Period | Correct | Wrong | Accuracy |
|--------|---------|-------|----------|
| This week | 42 | 3 | 93% |
| Last week | 35 | 5 | 87% |
| 2 weeks ago | 28 | 8 | 78% |

## Context Efficiency Trend

| Period | Avg Prompt Size | Blackboard Usage | Micro-Skills Usage |
|--------|----------------|------------------|--------------------|
| This week | 450 tokens | 90% | 75% |
| Last week | 800 tokens | 60% | 40% |
| 2 weeks ago | 1200 tokens | 20% | 0% |

## Top Issues (recurring)

| Issue | Count | Impact | Status |
|-------|-------|--------|--------|
| context-packets not used for Python agents | 5 | Medium | Open |
| library-docs-checker timeout on slow connections | 3 | Low | Resolved |

## Agent Performance Rankings

### Top Performers (highest avg score)
1. architect — 4.8/5 (12 dispatches)
2. tdd-guide — 4.5/5 (38 dispatches)
3. code-reviewer — 4.2/5 (45 dispatches)

### Needs Improvement (lowest avg score)
1. [agent] — [score]
2. [agent] — [score]

### Underutilized (active project, never dispatched)
- [agent]: Consider removing or merging
```

## Metrics Collection Protocol

### After Each Session

The eval-agent writes scores to `/memories/session/eval-report.md`. The self-improver reads it and updates the benchmark dashboard:

1. Read `/memories/session/eval-report.md`
2. Read existing `/memories/repo/agent-benchmarks.md`
3. Append session entry to session summary table
4. Update agent utilization counts and scores (running average)
5. Update routing accuracy trend
6. Update context efficiency trend
7. Identify new recurring issues
8. Write updated dashboard

### Calculation Methods

**Running average score:**
```
new_avg = (old_avg × old_count + new_score) / (old_count + 1)
```

**Token estimation:**
```
Rough estimate based on:
- Each agent dispatch ≈ 500-2000 tokens (prompt)
- Each tool call result ≈ 500-5000 tokens
- Each agent output ≈ 1000-3000 tokens
Count dispatches × 5000 for rough total
```

**Status classification:**
- Active: Used in last 7 days
- Low use: Used in last 30 days but not last 7
- Unused: Never dispatched in tracked sessions
- Consider removing: Unused + not relevant to project's tech stack

## Insights to Look For

### System Health Indicators

| Indicator | Healthy | Warning | Critical |
|-----------|---------|---------|----------|
| Routing accuracy | >90% | 75-90% | <75% |
| Context efficiency (avg prompt) | <500 tokens | 500-1000 | >1000 |
| Blackboard usage | >80% | 50-80% | <50% |
| Session score | >80 | 60-80 | <60 |
| Redundant dispatches | <5% | 5-15% | >15% |

### Actionable Signals

1. **Agent scores declining** → Check if skill files are outdated
2. **Routing accuracy dropping** → Planner routing table may need updating
3. **Context efficiency low** → Agents not using context packets
4. **Unused agents** → Project has shifted away from that tech stack
5. **High redundancy** → Memory blackboard not being read properly

## Dashboard Initialization

If `/memories/repo/agent-benchmarks.md` doesn't exist, create it with empty tables and start tracking from the current session.
