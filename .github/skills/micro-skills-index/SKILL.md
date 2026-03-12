---
name: micro-skills-index
description: Strategy for loading minimal, task-specific sections from large skill files instead of full files — reducing context consumption by 70-90%.
origin: AgentBrokeTheMatrix
---

# Micro-Skills Index

Load only the exact section of a skill file that your current task needs, instead of consuming the entire file. This saves 70-90% of context tokens per skill load.

## When to Activate

- Loading a skill file that is >500 lines
- Agent's task only needs one specific pattern from the skill
- Context window is >50% full
- Dispatching 3+ agents that each need different skill sections

## How It Works

Each skill file in `.github/skills/` contains multiple sections. Instead of loading the whole file, look up the section you need in this index and load only those lines.

## Index: AI/ML Skills

### ai-ml-patterns/SKILL.md
| Section | Lines | Use When |
|---------|-------|----------|
| Training Loop | 30-80 | Writing/reviewing training code |
| Data Pipeline | 82-140 | DataLoader, augmentation, preprocessing |
| Early Stopping | 142-180 | Overfitting prevention |
| Model Export | 182-230 | ONNX, TorchScript, SavedModel conversion |
| Experiment Tracking | 232-280 | MLflow, W&B, TensorBoard logging |
| Distributed Training | 282-340 | Multi-GPU, DDP patterns |

### prompt-engineering/SKILL.md
| Section | Lines | Use When |
|---------|-------|----------|
| Prompt Templates | 30-70 | Designing system/user prompts |
| RAG Pipeline | 72-130 | Retrieval-augmented generation |
| Tool Calling | 132-180 | Function calling, structured output |
| Cost Routing | 182-230 | Model selection by task complexity |
| Evaluation | 232-280 | Prompt testing, A/B testing |
| Security | 282-330 | Injection defense, output filtering |

### computer-vision-patterns/SKILL.md
| Section | Lines | Use When |
|---------|-------|----------|
| YOLO Detection | 30-80 | Object detection pipeline |
| OpenCV Basics | 82-130 | BGR/RGB, resize, preprocessing |
| OCR Pipeline | 132-180 | Text extraction from images |
| Video Processing | 182-230 | Frame extraction, batch processing |
| Model Export | 232-280 | ONNX, TensorRT deployment |
| Letterbox Resize | 282-310 | Maintaining aspect ratio |

## Index: Backend Skills

### auth-patterns/SKILL.md
| Section | Lines | Use When |
|---------|-------|----------|
| JWT Lifecycle | 30-80 | Token creation, refresh, rotation |
| OAuth2 Flows | 82-140 | Authorization code, PKCE, client credentials |
| RBAC | 142-190 | Role/permission models |
| Password Hashing | 192-230 | bcrypt, argon2 configuration |
| Cookie Security | 232-270 | HttpOnly, SameSite, Secure flags |
| Session Management | 272-310 | Server-side sessions, Redis storage |

### performance-optimization/SKILL.md
| Section | Lines | Use When |
|---------|-------|----------|
| Profiling | 30-70 | Finding bottlenecks |
| Caching Layers | 72-130 | Redis, in-memory, HTTP caching |
| Database Indexes | 132-180 | Index strategy, query plans |
| Connection Pooling | 182-220 | Database pool configuration |
| Async I/O | 222-260 | Non-blocking patterns |
| Frontend Perf | 262-310 | Bundle size, lazy loading, images |

### graphql-patterns/SKILL.md
| Section | Lines | Use When |
|---------|-------|----------|
| Schema Design | 30-80 | Type definitions, interfaces |
| DataLoader | 82-130 | N+1 prevention, batching |
| Pagination | 132-170 | Relay cursor, offset pagination |
| Authorization | 172-220 | Field-level auth, directives |
| Error Handling | 222-260 | Error types, partial responses |

### realtime-patterns/SKILL.md
| Section | Lines | Use When |
|---------|-------|----------|
| WebSocket | 30-80 | Bidirectional real-time |
| SSE | 82-120 | Server-sent events, one-way stream |
| Pub/Sub | 122-170 | Event bus, Redis messaging |
| Connection Management | 172-220 | Heartbeats, reconnection, scaling |

## Index: Frontend Skills

### state-management-patterns/SKILL.md
| Section | Lines | Use When |
|---------|-------|----------|
| Zustand | 30-70 | Global state store |
| React Query | 72-120 | Server state, caching |
| Context | 122-160 | Theme, locale, preferences |
| useReducer | 162-200 | Complex local state |
| Selectors | 202-240 | Derived state, memoization |
| Optimistic Updates | 242-280 | Instant UI feedback |

### accessibility-patterns/SKILL.md
| Section | Lines | Use When |
|---------|-------|----------|
| ARIA Roles | 30-70 | Semantic markup |
| Focus Management | 72-120 | Focus traps, skip nav |
| Forms | 122-170 | Labels, errors, descriptions |
| Live Regions | 172-210 | Dynamic content announcements |
| Contrast | 212-240 | Color, text sizing |

### frontend-patterns/SKILL.md
| Section | Lines | Use When |
|---------|-------|----------|
| Component Patterns | 30-80 | Composition, props, children |
| Hooks | 82-140 | Custom hooks, rules of hooks |
| Performance | 142-200 | Memo, callback, lazy, Suspense |
| Next.js App Router | 202-260 | Server components, routing |
| Forms | 262-310 | react-hook-form, validation |
| Error Boundaries | 312-350 | Error UI, recovery |

## Index: DevOps Skills

### devops-patterns/SKILL.md
| Section | Lines | Use When |
|---------|-------|----------|
| GitHub Actions | 30-80 | CI/CD pipeline YAML |
| Docker | 82-140 | Dockerfile, multi-stage builds |
| Docker Compose | 142-190 | Local dev multi-service |
| Monitoring | 192-240 | Prometheus, Grafana, alerting |
| Logging | 242-290 | Structured logging, correlation |
| Deployment | 292-340 | Blue-green, canary, rollback |

## How Planner/Coordinators Use This Index

### Before Dispatching a Sub-Agent

```markdown
# Instead of:
"Load skill: auth-patterns"  (loads 2000 tokens)

# Do this:
"Load skill: auth-patterns, section 'JWT Lifecycle' (lines 30-80)"  (loads 300 tokens)
```

### In Sub-Agent Prompts

```markdown
Task: Implement JWT refresh token rotation
Skill reference: Read .github/skills/auth-patterns/SKILL.md lines 30-80 for JWT patterns
Files: src/auth/jwt.ts
```

### Combining Multiple Micro-Loads

If a task needs patterns from 2 sections in different skills:
```markdown
Skill references:
1. Read .github/skills/auth-patterns/SKILL.md lines 30-80 (JWT Lifecycle)
2. Read .github/skills/performance-optimization/SKILL.md lines 72-130 (Caching for token blacklist)
Total: ~600 tokens instead of 4000 tokens
```

## Maintaining the Index

When adding or modifying a skill file:
1. Count the lines for each major section
2. Add/update the section entry in this index
3. Keep line numbers approximate (±10 lines is fine — agents can expand the range)

**Note**: Line numbers are approximate. Skills may be edited over time. The index provides a starting point — agents should read a few lines before and after the range if the exact content has shifted.

## Token Savings

| Approach | Tokens per Skill | 5 Skills |
|----------|-----------------|----------|
| Full file load | 1500-2500 | 7500-12500 |
| Micro-load (1 section) | 200-400 | 1000-2000 |
| **Savings** | **~80%** | **~80%** |
