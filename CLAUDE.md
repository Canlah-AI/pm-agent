# CLAUDE.md

## What This Is
Autonomous product iteration engine that operates Claude Code like an expert user. Creates a mission from a goal, builds v1, then enters an infinite improvement loop: find issues -> fix in independent Claude Code sessions -> score with unbiased review -> repeat until quality target is met.

## Tech Stack
- Python (orchestration)
- Claude Code (execution engine for code changes)
- Independent review sessions (bias-free scoring)

## Architecture
```
Goal -> Plan (vision + quality target)
  -> Build v1
  -> Review (independent session) -> Score
  -> Loop: Prioritize issues -> Fix -> Review -> Score
     (until score target met or plateau)
```

## Key Files
- `docs/architecture.md` — System architecture design
- `docs/architecture-v2.md` — V2 architecture iteration
- `docs/research-pm-agent-patterns.md` — Research on PM agent patterns

## Watch Out For
- This is a design/research repo — mostly docs, no runnable code yet
- The concept is a meta-programmer that uses Claude Code as its execution engine
- Each fix runs in an independent Claude Code session to avoid context contamination

## Shared Knowledge Base — agent-kb

Before starting work, check [Canlah-AI/agent-kb](https://github.com/Canlah-AI/agent-kb) — shared memory for all Canlah AI agents:
- `decisions/` — architectural rationale
- `patterns/` — reusable how-to
- `lessons/` — known gotchas — scan before debugging
- `products/` — living product docs (including `products/AMAZON_SUITE.md`)
- `runbooks/` — incident playbooks
- `apis/` — external API quirks

**Read without cloning:**
```bash
gh api repos/Canlah-AI/agent-kb/contents/<path> --jq .content | base64 -d
```

**Read when:** starting any task here, designing a new feature, hitting non-obvious bugs.

**Write when:** you learn something non-obvious — drop a lesson in `lessons/YYYY-MM-DD_<slug>.md` (<200 lines), commit to agent-kb main directly.
