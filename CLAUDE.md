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
