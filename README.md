# PM Agent

Autonomous product iteration engine that operates Claude Code like an expert user.

## What This Does

PM Agent is a meta-programmer that continuously iterates on a product until it meets quality targets. It creates a mission from a goal, builds an initial version, then enters an infinite improvement loop: find highest-priority issues, execute fixes in independent Claude Code sessions, and evaluate results with unbiased reviews. The loop continues until the quality score plateaus or the target is met.

## Tech Stack

- Python (orchestration)
- Claude Code (execution engine)
- Independent review sessions (bias-free scoring)

## Architecture

```
Goal → Plan (vision + quality target)
  → Build v1
  → Review (independent session) → Score
  → Loop:
      Prioritize issues → Fix → Review → Score
      (until score target met or no issues found)
```

## Setup

```bash
git clone https://github.com/Canlah-AI/pm-agent.git
cd pm-agent
```

See `docs/` for architecture documentation and research notes.

## Related

Part of the [CanMarket](https://github.com/Canlah-AI/market) ecosystem.
