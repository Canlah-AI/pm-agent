# PM Agent + Claude Code Workflow Integration

> How the PM Agent leverages Claude Code's full toolkit for autonomous development.
> Updated: 2026-04-12 (based on Claude Code v2.1.104 capabilities)

---

## Overview

The PM Agent is no longer just an OpenClaw skill — it's a **Claude Code orchestration layer** that uses every available primitive:

```
/schedule → PM Agent (P9) → Agent Teams → P7 sub-agents (worktrees)
                ↓
        GitHub Issues (task DB)
                ↓
        /review → /ship → /land-and-deploy
```

---

## Available Primitives (April 2026)

| Primitive | What It Does | PM Agent Use |
|-----------|-------------|--------------|
| Agent Teams | Multi-session shared task list | Dispatch P7 workers |
| `/schedule` | Cloud cron (survives reboot) | Sprint planning, daily reviews |
| `/loop` | In-session recurring | Monitor CI, check team status |
| Remote Control | Mobile/web access | Approve PRs from phone |
| claude-peers | Inter-session messaging | P7 reports back to PM |
| Agent SDK | Programmatic Python API | Automated pipeline |
| Worktrees | Git isolation per agent | No merge conflicts |
| `/review` | Automated code review | Quality gate |
| `/ship` | PR creation | Automated shipping |

---

## Implementation Plan

### Phase 1: Scheduled Routines (No Code Needed)

Just configure /schedule:

```
/schedule "0 9 * * 1" "Read open GitHub issues for Canlah-AI repos. Prioritize by labels. Create sprint plan as GitHub Project board update."

/schedule "0 18 * * *" "List all open PRs with `gh pr list`. Run /review on each. Comment findings. Auto-approve if zero issues."

/schedule "0 17 * * 5" "Run /retro. Summarize this week's commits across all repos. Post summary."
```

### Phase 2: Agent Definition

```json
{
  "pm": {
    "description": "P9 Tech Lead PM. Plans sprints, dispatches tasks, reviews output.",
    "prompt": "You are the PM for Canlah AI. Your job: 1) Read GitHub Issues and prioritize. 2) Decompose features into parallel tasks. 3) Dispatch to sub-agents via Agent Teams (each gets own worktree). 4) Run /review on every PR before merge. 5) Track velocity and report blockers. Never write code yourself — you write Task Prompts.",
    "allowedTools": ["Bash(gh:*)", "Bash(git:*)", "Read", "Glob", "Grep", "Agent", "WebSearch"]
  }
}
```

### Phase 3: Sprint Automation Loop

```bash
# Start PM agent
claude --agent pm -n "pm-sprint-12"

# PM agent internally:
# 1. gh issue list --repo Canlah-AI/market --state open
# 2. Classify by priority/effort
# 3. Create sprint plan
# 4. For each task:
#    - Spawn P7 agent in worktree
#    - P7 implements, commits, pushes
#    - PM reviews via /review
#    - PM ships via /ship if clean
```

### Phase 4: Agent SDK Pipeline (Python)

```python
"""pm_pipeline.py — Autonomous sprint execution"""
from claude_agent_sdk import Agent, Session
import subprocess

# PM Agent
pm = Agent(
    model="claude-opus-4-6",
    system_prompt=open("PM_SYSTEM_PROMPT.md").read(),
    allowed_tools=["Bash(gh:*)", "Bash(git:*)", "Read", "Agent"]
)

# Get sprint tasks
session = pm.create_session(working_directory="/path/to/market")
plan = session.send("Plan this sprint. Return JSON: [{task, assignee, priority}]")

# Dispatch each task to a P7 worker
for task in plan.tasks:
    worker = Agent(
        model="claude-sonnet-4-6",  # Cheaper for execution
        system_prompt=f"Implement: {task.description}. Commit when done.",
        allowed_tools=["Bash", "Edit", "Read", "Write"]
    )
    worker_session = worker.create_session(
        working_directory="/path/to/market",
        worktree=f"sprint-12-{task.id}"
    )
    worker_session.send(task.prompt)

# PM reviews all PRs
session.send("Review all open PRs from this sprint. Approve or request changes.")
```

---

## Integration with Existing Setup

### What We Already Have
- `claude-peers` MCP (localhost:7899) → inter-session messaging
- `pua:p9` skill → Tech Lead persona with Task Prompt format
- `pua:p7` skill → P7 Senior Engineer execution mode
- GitHub MCP → issue/PR management
- `/review` + `/ship` skills → quality gate + shipping

### What to Add
1. **CCPM skill** — GitHub Issues as task database
   ```bash
   # Adds: /sprint-plan, /task-assign, /task-status
   ```
2. **Agent Teams env var**
   ```bash
   claude settings set env.CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS 1
   ```
3. **PM schedule config** — see Phase 1 above

### Migration from Current Architecture (docs/architecture.md)

The existing OpenClaw-based design (Router → Task Decomp → Claude Code Bridge) maps cleanly:

| Old Concept | New Primitive |
|-------------|--------------|
| Router Engine | PM agent --agent definition |
| Task Decomposition | Agent Teams + worktrees |
| Claude Code Bridge | Agent SDK direct calls |
| Communication Protocol | claude-peers MCP |
| Brand Memory Integration | CLAUDE.md + .claude/rules/ |

---

## Next Steps

1. [ ] Enable Agent Teams: `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`
2. [ ] Configure /schedule for daily PR review
3. [ ] Test PM agent definition with one sprint
4. [ ] Install CCPM for GitHub Issues integration
5. [ ] Build pm_pipeline.py for full automation
6. [ ] Connect to佰潮's workflow via claude-peers

---

*This doc supersedes docs/architecture.md for the Claude Code integration layer.*
*The OpenClaw routing layer (for Slack/WhatsApp channels) remains valid.*
