---
name: context-efficient-dispatch
description: Patterns for maximizing agent effectiveness within limited context windows — context compression, focused prompts, result caching, and hierarchical dispatch.
origin: AgentBrokeTheMatrix
---

# Context-Efficient Agent Dispatch

Patterns for getting maximum output from agents operating under context window constraints. Every token matters — waste nothing, compress everything, cache aggressively.

## When to Activate

- Dispatching sub-agents from the planner or domain coordinators
- Context window is filling up (>60% usage)
- Multi-phase tasks where later phases need earlier results
- Tasks spanning 5+ files or 3+ agents
- Any time you're writing a sub-agent prompt

## Core Principle: Token Budget

Every sub-agent dispatch has a **token budget**. Treat it like a financial budget:

```
Total sub-agent context: ~200K tokens
- System prompt + agent definition: ~2K tokens
- Your dispatch prompt: 500-2000 tokens (AIM FOR LOW END)
- Agent's tool calls (reads, searches): ~50-100K tokens
- Agent's reasoning + output: ~50K tokens
- Safety buffer: ~48K tokens

YOUR PROMPT IS THE ONLY THING YOU CONTROL.
KEEP IT UNDER 1000 TOKENS WHEN POSSIBLE.
```

## Pattern 1: Context Compression

### The Context Packet Format

Never send raw file contents. Send a **context packet**:

```markdown
## Context Packet

### Task
[1-2 sentences. What exactly to do.]

### Target Files
- `src/auth/jwt.ts` (L45-L80) — Token refresh logic, currently broken
- `src/middleware/auth.ts` (L12-L25) — Where refresh is called

### Signatures (don't send full code)
- `function refreshToken(token: string): Promise<TokenPair>`
- `interface TokenPair { access: string; refresh: string; expiresIn: number }`
- `const AUTH_CONFIG = { accessTTL: 900, refreshTTL: 604800 }`

### Current Issue
Refresh tokens are not rotated on use — same refresh token works indefinitely.

### Constraints
- Must be backward-compatible with existing mobile clients
- Refresh rotation must invalidate the old token within 30 seconds

### Success Criteria
- [ ] Refresh tokens are single-use
- [ ] Old tokens rejected after rotation
- [ ] Tests cover the rotation flow
```

**Size comparison:**
- Raw files: ~3000 tokens
- Context packet: ~300 tokens
- **10x compression with zero information loss for the task**

### When to Include Full Code

Only include full code (not just signatures) when:
- The bug is in the code's logic flow (not just API usage)
- The agent needs to see surrounding code to understand context
- The code is <30 lines

## Pattern 2: Minimal Sub-Agent Prompts

### Bad Prompt (wastes ~2000 tokens)

```
You are working on a project that uses React with TypeScript for the frontend
and Express with PostgreSQL for the backend. The project structure is...
[500 words of background]
We need to add input validation to the user registration form.
The form is in src/components/RegisterForm.tsx and uses react-hook-form
with zod for validation. Here is the current code:
[200 lines of code]
Please add email format validation, password strength requirements,
and make sure the error messages are accessible.
```

### Good Prompt (achieves the same in ~300 tokens)

```
Task: Add validation to user registration form.
File: src/components/RegisterForm.tsx
Stack: React + react-hook-form + zod (schemas in src/lib/schemas.ts)
Add:
1. Email format validation (zod .email())
2. Password: min 8 chars, 1 uppercase, 1 number
3. Accessible error messages (aria-describedby linking)
Constraints: DO NOT call ask_user. Return results in final message.
Read /memories/session/ for prior phase context.
```

## Pattern 3: Result Caching via Session Memory

### Write After Every Agent Completes

```markdown
# /memories/session/phase1-auth-results.md

## What Was Done
- JWT refresh rotation implemented in src/auth/jwt.ts
- Old tokens invalidated via Redis blacklist (src/auth/blacklist.ts)
- Tests added: src/auth/__tests__/refresh.test.ts (5 tests, all pass)

## Key Decisions
- Used Redis SET with TTL for token blacklist (auto-cleanup)
- Rotation window: 30 seconds (old token valid during window)

## Files Modified
- src/auth/jwt.ts (L45-L95): refreshToken() now rotates
- src/auth/blacklist.ts (NEW): Redis-backed token blacklist
- src/middleware/auth.ts (L12-L25): Checks blacklist before accepting token

## Open Questions
- Should we log rotation events for security audit?
```

### Read Before Every Agent Dispatches

Before sending any sub-agent prompt, check:
```
Read /memories/session/ — check if prior phases have relevant context.
Include the relevant parts (not all of it) in the context packet.
```

## Pattern 4: Hierarchical Dispatch

### Flat (Bad for Context)
```
Planner → [ai-ml-engineer, cv-specialist, prompt-engineer, api-designer, auth-specialist, database-reviewer]
// Planner must understand all 6 domains
// Planner's context fills up managing 6 parallel results
```

### Hierarchical (Good for Context)
```
Planner → [ai-coordinator, backend-coordinator]
// Planner only manages 2 results
// Each coordinator gets its own fresh context window
// Each coordinator dispatches 2-3 specialists with focused context
// Effective context: Planner(200K) + AI-Coord(200K) + Backend-Coord(200K) = 600K
```

### When to Use Hierarchical vs Flat
- **Flat**: Task needs ≤3 agents in the same domain
- **Hierarchical**: Task needs ≥4 agents OR spans ≥2 domains

## Pattern 5: Progressive Context Building

Don't front-load context. Let agents discover what they need:

```
Round 1: Send task + file paths only
         Agent reads files, identifies what it needs

Round 2: Agent reports back with questions
         Coordinator sends targeted context

Round 3: Agent completes work with full understanding
```

This uses 40% less context than sending everything upfront, because agents typically only need 30% of what you'd pre-emptively send.

## Pattern 6: Skill File Micro-Loading

Instead of loading a full 2000-line skill file:

```markdown
# Full skill load (expensive)
Read: .github/skills/auth-patterns/SKILL.md  → 2000 tokens

# Micro-load (cheap)
Read: .github/skills/auth-patterns/SKILL.md lines 45-90  → 300 tokens
(Just the JWT rotation section you actually need)
```

The planner should tell coordinators WHICH SECTION of a skill to load:
```
Load skill: auth-patterns, section "JWT Lifecycle" (lines 45-90)
```

## Anti-Patterns

| Anti-Pattern | Fix |
|-------------|-----|
| Sending entire files to sub-agents | Send context packets with signatures only |
| Including project background in every prompt | Write once to session memory, reference it |
| Loading full skill files | Micro-load specific sections |
| Flat dispatch of 5+ agents | Use hierarchical coordinators |
| Repeating context across agents | Write to memory blackboard, agents read it |
| Verbose prompts with examples | Task + files + constraints + criteria, nothing more |
