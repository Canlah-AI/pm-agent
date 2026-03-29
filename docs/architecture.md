# PM Agent Architecture Design Document

**Project:** CanMarket PM Agent on OpenClaw
**Author:** Architecture Review (Claude Opus 4.6)
**Date:** 2026-02-10
**Status:** Draft for Review

---

## Table of Contents

1. [System Architecture](#1-system-architecture)
2. [PM Agent Skill Design](#2-pm-agent-skill-design)
3. [Task Decomposition Engine](#3-task-decomposition-engine)
4. [Claude Code Orchestration Layer](#4-claude-code-orchestration-layer)
5. [Brand Memory Integration](#5-brand-memory-integration)
6. [Communication Protocol](#6-communication-protocol)
7. [Security Architecture](#7-security-architecture)
8. [File Structure](#8-file-structure)
9. [Architecture Decision Records](#9-architecture-decision-records)

---

## 1. System Architecture

### 1.1 High-Level Architecture Diagram

```
                           USER CHANNELS
                    ┌──────────┬──────────┐
                    │  Slack   │ WhatsApp  │  Web Chat
                    └────┬─────┴────┬─────┘
                         │          │
                    ┌────▼──────────▼────┐
                    │                    │
                    │     OPENCLAW       │
                    │   Runtime Layer    │
                    │                    │
                    │  ┌──────────────┐  │
                    │  │  PM Agent    │  │
                    │  │  Skill       │  │
                    │  │              │  │
                    │  │ ┌──────────┐ │  │
                    │  │ │ Router   │ │  │
                    │  │ │ Engine   │ │  │
                    │  │ └────┬─────┘ │  │
                    │  │      │       │  │
                    │  │ ┌────▼─────┐ │  │
                    │  │ │ Task     │ │  │
                    │  │ │ Decomp   │ │  │
                    │  │ └────┬─────┘ │  │
                    │  │      │       │  │
                    │  │ ┌────▼─────┐ │  │
                    │  │ │ Claude   │ │  │
                    │  │ │ Code     │ │  │
                    │  │ │ Orch.    │ │  │
                    │  │ └────┬─────┘ │  │
                    │  └──────┼───────┘  │
                    │         │          │
                    │  ┌──────▼───────┐  │
                    │  │  State Store │  │
                    │  │  (SQLite +   │  │
                    │  │   MEMORY.md) │  │
                    │  └──────────────┘  │
                    └─────────┬──────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
     ┌────────▼──────┐  ┌────▼─────┐  ┌──────▼───────┐
     │  Claude Code  │  │ Claude   │  │  Claude Code  │
     │  Instance A   │  │ Code     │  │  Instance C   │
     │  (Single Task)│  │ Team     │  │  (Single Task)│
     └────────┬──────┘  │ (Multi)  │  └──────┬───────┘
              │         └────┬─────┘         │
              │              │               │
     ┌────────▼──────────────▼───────────────▼────────┐
     │                                                 │
     │            CANMARKET SERVICE MESH               │
     │                                                 │
     │  ┌─────────────┐  ┌──────────────┐  ┌────────┐│
     │  │brand-strat-cc│  │ content-agent│  │ market ││
     │  │Style Genome™ │  │ multi-plat   │  │backend ││
     │  └─────────────┘  └──────────────┘  └────────┘│
     │  ┌─────────────┐  ┌──────────────┐  ┌────────┐│
     │  │consultation │  │ campaign-gen │  │style-  ││
     │  │brand analy. │  │ calendar     │  │analyzer││
     │  └─────────────┘  └──────────────┘  └────────┘│
     │  ┌─────────────┐  ┌──────────────┐  ┌────────┐│
     │  │tvc-generator│  │video-auto-ed │  │social- ││
     │  │script→video │  │transcr/edit  │  │posting ││
     │  └─────────────┘  └──────────────┘  └────────┘│
     │  ┌─────────────┐                              │
     │  │social-media │                              │
     │  │style 175K DB│                              │
     │  └─────────────┘                              │
     └─────────────────────────────────────────────────┘
```

### 1.2 Data Flow Architecture

```
User Message
    │
    ▼
┌─────────────────────────────────────────────────┐
│  INGRESS LAYER                                  │
│                                                 │
│  Slack/WhatsApp → OpenClaw Webhook → Normalize  │
│  → Extract: text, files, brand_id, user_id      │
└────────────────────┬────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────┐
│  INTENT + CONTEXT LAYER                         │
│                                                 │
│  1. Load brand memory (Style Genome + history)  │
│  2. Classify intent (see Section 3)             │
│  3. Check if clarification needed               │
│     → YES: reply to user, STOP                  │
│     → NO: proceed to decomposition              │
└────────────────────┬────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────┐
│  TASK DECOMPOSITION LAYER                       │
│                                                 │
│  1. Map intent → task template                  │
│  2. Fill template with brand context            │
│  3. Resolve dependencies (DAG)                  │
│  4. Estimate cost + time                        │
│  5. If cost > $2 or time > 10min: confirm       │
└────────────────────┬────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────┐
│  EXECUTION LAYER                                │
│                                                 │
│  Route each task to Claude Code:                │
│  - Simple (1 service) → claude -p               │
│  - Complex (2-3 services) → claude -p serial    │
│  - Major (4+ services) → Agent Team             │
│                                                 │
│  Each Claude Code instance:                     │
│  1. Reads CLAUDE.md (has brand context)         │
│  2. Calls CanMarket services via API/CLI        │
│  3. Returns structured result                   │
└────────────────────┬────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────┐
│  ASSEMBLY + QA LAYER                            │
│                                                 │
│  1. Collect all task outputs                    │
│  2. Brand consistency check (Style Genome™)     │
│  3. Compose user-facing response                │
│  4. Log to state store                          │
│  5. Reply to user channel                       │
└─────────────────────────────────────────────────┘
```

### 1.3 Key Architectural Decisions

**Decision: PM Agent is a single OpenClaw skill, not a multi-skill orchestrator.**

The PM Agent skill owns the full lifecycle from user message to response. It does not delegate to other OpenClaw skills. Instead, it delegates engineering work to Claude Code instances that call CanMarket services directly. This avoids skill-to-skill coordination complexity and keeps the PM Agent as the single source of truth for task state.

**Decision: Claude Code is the universal execution substrate.**

Every engineering action goes through Claude Code CLI. CanMarket services are never called directly from the PM Agent skill. This means the PM Agent only needs to know how to construct good prompts and CLI flags for Claude Code, not how to make HTTP requests to 10+ different services.

**Decision: Stateless PM Agent, stateful storage.**

Each PM Agent invocation reads state from SQLite + MEMORY.md, processes the request, writes state back, and exits. No long-running daemon. OpenClaw handles message queueing and retry.

---

## 2. PM Agent Skill Design

### 2.1 SKILL.md Specification

```yaml
---
name: pm-agent
description: >
  CanMarket Product Manager Agent. Receives user requests about brand marketing
  (content creation, campaign planning, video production, brand analysis, social
  posting) and orchestrates Claude Code agent teams to execute them using
  CanMarket's service mesh. Expert at task decomposition, Claude Code CLI
  orchestration, and brand consistency enforcement.
metadata:
  openclaw:
    requires:
      bins:
        - python3
        - claude
        - curl
        - jq
      env:
        - ANTHROPIC_API_KEY
        - OPENAI_API_KEY
        - GOOGLE_GENAI_API_KEY
        - FAL_API_KEY
        - CANMARKET_SUPABASE_URL
        - CANMARKET_SUPABASE_KEY
    install:
      darwin:
        command: |
          pip3 install --user supabase httpx pydantic
          claude --version || echo "Claude Code CLI required"
    permissions:
      tools:
        - Read
        - Write
        - Bash
        - Task
      permission_mode: acceptEdits
    cost:
      max_budget_per_invocation_usd: 10.0
      alert_threshold_usd: 5.0
---
```

### 2.2 Skill Entry Point Behavior

The skill is invoked by OpenClaw with `$ARGUMENTS` containing the user message and metadata. The entry point script (`run.py`) executes the following pipeline:

1. Parse `$ARGUMENTS` as JSON (contains `user_id`, `brand_id`, `channel`, `message`, `attachments`)
2. Load brand context from state store
3. Load conversation history (last 10 messages for this user+brand)
4. Classify intent and determine if clarification is needed
5. If no clarification needed, decompose into tasks
6. Execute tasks via Claude Code
7. Assemble response
8. Write state (task log, updated memory)
9. Return response to OpenClaw for delivery

### 2.3 State Storage Design

**Three tiers of state:**

| Tier | Storage | Purpose | Retention |
|------|---------|---------|-----------|
| Hot | `state/active_tasks.json` | Currently executing tasks | Until completion |
| Warm | `state/pm_agent.db` (SQLite) | Task history, conversation log, brand cache | 90 days |
| Cold | `MEMORY.md` + daily logs | Learnings, patterns, preferences per brand | Permanent |

**SQLite Schema (pm_agent.db):**

```sql
CREATE TABLE conversations (
  id            TEXT PRIMARY KEY,
  user_id       TEXT NOT NULL,
  brand_id      TEXT,
  channel       TEXT,
  message       TEXT NOT NULL,
  role          TEXT NOT NULL,
  intent        TEXT,
  created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE tasks (
  id            TEXT PRIMARY KEY,
  conversation_id TEXT REFERENCES conversations(id),
  brand_id      TEXT,
  type          TEXT NOT NULL,
  status        TEXT DEFAULT 'pending',
  input_json    TEXT,
  output_json   TEXT,
  claude_session TEXT,
  cost_usd      REAL DEFAULT 0,
  duration_sec  INTEGER,
  created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  completed_at  TIMESTAMP
);

CREATE TABLE brand_cache (
  brand_id      TEXT PRIMARY KEY,
  style_genome  BLOB,
  brand_dna     TEXT,
  updated_at    TIMESTAMP
);
```

---

## 3. Task Decomposition Engine

### 3.1 Intent Classification

```
INTENT TAXONOMY
===============

BRAND_SETUP
├── brand_onboarding       "Help me set up my brand"
├── style_genome_extract   "Analyze my brand's style"
├── brand_update           "Update my brand guidelines"
└── competitor_analysis    "Analyze competitor X's style"

CONTENT_CREATION
├── single_post            "Write a post about X"
├── multi_platform_post    "Create content for all platforms"
├── blog_article           "Write an article about X"
└── content_repurpose      "Turn this video into posts"

CAMPAIGN
├── campaign_plan          "Plan a Q2 campaign for X"
├── campaign_calendar      "Generate a content calendar"
├── campaign_brief         "Write a brief for X campaign"
└── campaign_execute       "Execute the approved campaign"

VIDEO
├── tvc_generate           "Create a product video for X"
├── video_edit             "Edit this video: add subtitles"
├── video_highlights       "Extract highlights from this video"
└── video_reframe          "Reframe this video for Instagram Reels"

SOCIAL
├── post_publish           "Publish this to Twitter and LinkedIn"
├── style_search           "Find images similar to this style"
├── engagement_reply       "Draft replies for these comments"
└── social_analytics       "How did last week's posts perform?"

ADMIN
├── status_check           "What's the status of my campaign?"
├── cost_report            "How much have I spent this month?"
├── feedback               "The last post was too casual"
└── clarification_response  (user is answering our question)
```

### 3.2 Request-to-Service Routing

```
ROUTING TABLE
=============

Intent                    Primary Service           Supporting Services
──────────────────────────────────────────────────────────────────────
brand_onboarding         consultation :8004        brand-strategist-cc
style_genome_extract     brand-strategist-cc       style-analyzer :8006
brand_update             consultation :8004        —
competitor_analysis      social-media-style :8000  brand-strategist-cc

single_post              content-agent             —
multi_platform_post      content-agent             social-posting-api
blog_article             content-agent             —
content_repurpose        video-auto-edit :7860     content-agent

campaign_plan            campaign-gen :8007        consultation :8004
campaign_calendar        campaign-gen :8007        —
campaign_brief           consultation :8004        —
campaign_execute         campaign-gen :8007        content-agent, social-posting-api

tvc_generate             tvc-generator CLI         style-analyzer :8006
video_edit               video-auto-edit :7860     —
video_highlights         video-auto-edit :7860     —
video_reframe            video-auto-edit :7860     —

post_publish             social-posting-api        —
style_search             social-media-style :8000  —
engagement_reply         content-agent             —
social_analytics         market backend :8000      —

status_check             (local state store)       —
cost_report              (local state store)       —
feedback                 (update brand memory)     —
```

### 3.3 Task Dependency Graph Patterns

**Pattern A: Linear Chain** (most common, ~60% of requests)

```
Task 1: Load brand context
    │
    ▼
Task 2: Generate content (service call)
    │
    ▼
Task 3: Brand consistency check
    │
    ▼
Task 4: Return to user for approval
```

**Pattern B: Fan-Out / Fan-In** (~25% of requests)

```
Task 1: Load brand context
    │
    ├──────────┬──────────┬──────────┐
    ▼          ▼          ▼          ▼
Task 2a:   Task 2b:   Task 2c:   Task 2d:
Twitter    LinkedIn   小红书      Reddit
    │          │          │          │
    └──────────┴──────────┴──────────┘
                    │
                    ▼
               Task 3: Brand consistency check (all 4)
                    │
                    ▼
               Task 4: Return to user
```

**Pattern C: Conditional Branch** (~10% of requests)

```
Task 1: Brand onboarding
    │
    ▼
Task 2: Style Genome extraction
    │
    ├── IF success ─→ Task 3: Campaign generation
    │
    └── IF failed  ─→ Task 3b: Ask user for more materials
```

### 3.4 Clarification Decision Logic

```
CLARIFICATION REQUIRED IF:
- No brand_id and user has multiple brands
- Content creation without platform specified AND no default
- Video request without product image or reference
- Campaign request without time range
- Ambiguous scope

NEVER ASK IF:
- Brand is obvious (user has only one brand)
- Platform preference exists in brand memory
- Request is a direct follow-up
- Request is admin query
```

---

## 4. Claude Code Orchestration Layer

### 4.1 Execution Mode Decision Matrix

```
                        Single Service    2-3 Services    4+ Services
                        ──────────────    ────────────    ───────────
Independent tasks       claude -p         claude -p       Agent Team
                        (one per task)    (serial)        (parallel)

Dependent chain         claude -p         claude -p       Agent Team
                        (one shot)        (one session)   (team lead)

Needs iteration         claude -c         claude -c       Agent Team
                        (resume)          (resume)        (SendMessage)
```

### 4.2 Claude Code Invocation Patterns

**Pattern 1: One-Shot Task**

```bash
claude -p "{TASK_PROMPT}" \
  --output-format json \
  --model sonnet \
  --max-turns 10 \
  --max-budget-usd 2.00 \
  --allowedTools "Read,Bash(curl *),Bash(python3 *)" \
  --permission-mode acceptEdits
```

**Pattern 2: Multi-Step Session**

```bash
claude -p "{STEP_1}" --output-format json --session-name "task-{ID}"
claude -c -p "{STEP_2}" --output-format json --session-name "task-{ID}"
```

**Pattern 3: Agent Team**

```bash
claude -p "{TEAM_LEAD_PROMPT}" \
  --model opus --max-budget-usd 8.00 \
  --agents '{
    "teammates": [
      {"name": "content-writer", "model": "sonnet", ...},
      {"name": "brand-checker", "model": "haiku", ...},
      {"name": "publisher", "model": "haiku", ...}
    ]
  }'
```

### 4.3 Prompt Construction Strategy

```
## Role
You are a CanMarket engineering agent.

## Brand Context
Brand: {brand_name} (ID: {brand_id})
Voice: {brand_voice_summary}
Colors: {primary_color}, {secondary_color}
Style Rules: {brand_rules}

## Task
{specific_actionable_instruction}

## Service to Call
Endpoint: {service_endpoint}
Payload: {payload_template}

## Success Criteria
{numbered_outcomes}

## Output Format
Return ONLY valid JSON matching schema.
```

**Key principle:** PM thinks, Claude Code does. PM Agent never asks Claude Code to "figure out" which service to use.

### 4.4 Cost Optimization

| Tier | Model | Cost/task | Use for |
|------|-------|-----------|---------|
| 1 | Haiku | $0.01-0.05 | Brand checks, status lookups, publishing |
| 2 | Sonnet | $0.10-0.50 | Content creation, campaign gen, scripts |
| 3 | Opus | $0.50-2.00 | Agent Team lead, brand DNA extraction |

Budget rules:
- Default max per invocation: $2.00
- Agent Team max: $8.00 (requires confirmation)
- Monthly budget per brand: $50.00 (configurable)
- Alert at 80%, hard stop at 100%
- ALWAYS use `--max-budget-usd`

---

## 5. Brand Memory Integration

### 5.1 Brand Memory Architecture

```
Layer 1: Style Genome™ (768-dim CLIP vector)
  → Similarity matching, style search
  → Stored as BLOB in brand_cache

Layer 2: Brand DNA (structured JSON)
  → Colors, fonts, voice, tone, rules
  → Injected into every Claude Code prompt

Layer 3: Operational Memory (MEMORY.md)
  → User preferences, failure patterns, feedback history
  → Grows over time, curated by PM Agent
```

### 5.2 Brand Consistency Enforcement

After every content-producing task:

1. **FORBIDDEN WORDS** - Scan against `brand_dna.rules.never`
2. **REQUIRED ELEMENTS** - Verify `brand_dna.rules.always`
3. **TONE CHECK** - LLM check (haiku) against brand voice
4. **VISUAL CHECK** - Style vector cosine similarity >= 0.75

---

## 6. Communication Protocol

### 6.1 Message Types

| Type | Timing | Example |
|------|--------|---------|
| Acknowledgment | <2 sec | "Got it. Working on your campaign. ~3 min." |
| Progress | Every 60s | "Step 2/4: Generating content..." |
| Clarification | Before work | "Which platforms? Twitter, LinkedIn, or all?" |
| Result | On completion | Formatted output + [Approve] [Revise] [Publish] |
| Cost Alert | Before expensive work | "This will cost ~$5. Proceed?" |
| Error | On failure | "Issue with video service. Retrying..." |

### 6.2 Error Handling

| Level | Type | Strategy |
|-------|------|----------|
| 1 | Transient (timeout, 503, 429) | Retry 3x with backoff (2s, 4s, 8s) |
| 2 | Input error (400) | Fix input, retry once |
| 3 | Service error (500, auth) | Log, notify user, suggest workaround |
| 4 | Budget exceeded | Hard stop, report cost summary |

Circuit breaker: 3 failures in 10min → mark service degraded → auto-retry in 30min.

---

## 7. Security Architecture

### 7.1 API Key Management

- All keys in OpenClaw encrypted vault
- Injected as env vars at runtime
- NEVER in SKILL.md, MEMORY.md, logs, or CLI args
- Rotated every 90 days

### 7.2 Permission Boundaries

PM Agent CAN: read/write own state, execute `claude` CLI, read brand data
PM Agent CANNOT: access other skills' state, modify system files, run arbitrary commands

Claude Code permissions (set by PM Agent):
- Content tasks: `Read, Bash(curl *), Bash(python3 *)`
- Publish tasks: `Bash(python3 *), Bash(curl *)`
- Analysis tasks: `Read, Bash(curl --max-time 10 *)`
- NEVER: `bypassPermissions`, `Bash(rm *)`, `Bash(sudo *)`

### 7.3 Audit Logging

Every action logged with: timestamp, user_id, brand_id, task_id, action, input/output hash, cost, status. Retention: 180 days.

---

## 8. File Structure

```
pm-agent/
├── SKILL.md                          # OpenClaw skill definition
├── CLAUDE.md                         # Claude Code project instructions
├── MEMORY.md                         # Persistent learnings
│
├── run.py                            # Entry point (~100 lines)
├── config.py                         # Constants, endpoints, limits
│
├── core/
│   ├── intent.py                     # Intent classification
│   ├── decomposer.py                 # Task decomposition (DAG)
│   ├── router.py                     # Intent → service routing
│   └── templates.py                  # Task + prompt templates
│
├── orchestration/
│   ├── claude_code.py                # Claude Code CLI wrapper
│   ├── team_builder.py               # Agent Team JSON construction
│   ├── cost_tracker.py               # Budget tracking
│   └── session_manager.py            # Session lifecycle
│
├── brand/
│   ├── memory.py                     # Brand memory CRUD
│   ├── context_builder.py            # Brand context for prompts
│   ├── consistency_checker.py        # 4-check brand validation
│   └── style_genome.py              # Style Genome™ vector ops
│
├── state/
│   ├── db.py                         # SQLite operations
│   └── pm_agent.db                   # Database (runtime)
│
├── channels/
│   ├── formatter.py                  # Response formatting
│   ├── slack.py                      # Slack messages
│   └── whatsapp.py                   # WhatsApp messages
│
├── security/
│   ├── validator.py                  # Input validation
│   ├── audit.py                      # Audit logging
│   └── permissions.py                # Tool policies
│
├── services/
│   ├── registry.py                   # Health + circuit breaker
│   ├── schemas/                      # JSON schemas per service
│   └── payloads/                     # Payload templates
│
├── prompts/                          # Claude Code prompt templates
│   ├── base_system.md
│   ├── content_creation.md
│   ├── campaign_planning.md
│   ├── video_production.md
│   ├── brand_analysis.md
│   └── brand_check.md
│
├── tests/
│   ├── test_intent.py
│   ├── test_decomposer.py
│   ├── test_router.py
│   └── fixtures/
│
├── .claude/rules/
│   ├── brand-enforcement.md
│   ├── cost-limits.md
│   └── service-endpoints.md
│
└── docs/
    ├── architecture.md               # This document
    └── adr/
        ├── ADR-001-claude-code-as-substrate.md
        ├── ADR-002-single-skill-not-multi.md
        ├── ADR-003-sqlite-state-store.md
        └── ADR-004-prompt-templates-not-dynamic.md
```

**Estimated total: 2,000-2,500 lines**

---

## 9. Architecture Decision Records

### ADR-001: Claude Code as Universal Execution Substrate
PM Agent constructs prompts, Claude Code executes. No direct HTTP calls from PM Agent to services. Adding a new service = writing a new prompt template.

### ADR-002: Single Skill Architecture
One monolithic OpenClaw skill owns the full lifecycle. No inter-skill coordination. Simpler debugging and deployment.

### ADR-003: SQLite for State Storage
Zero dependencies, ~1ms reads, sufficient for single-agent workload. Migration path to Supabase when concurrent instances needed.

### ADR-004: Static Prompt Templates
Variable interpolation over LLM-generated prompts. Deterministic, auditable, cheap. Generic fallback template for unknown intents (<5% of requests).

---

## 10. Design Principles

1. **PM thinks, Claude Code does.** All reasoning in PM Agent, all execution in Claude Code.
2. **Brand context is injected, never discovered.** Every invocation gets full brand context.
3. **Templates over generation.** Static prompts with variables, not LLM-generated prompts.
4. **Ask once, ask early, ask everything.** Single clarification message, never piecemeal.
5. **Budget is first-class.** Every invocation has `--max-budget-usd`.
6. **Fail fast, retry smart.** Transient=retry, service=circuit-breaker, budget=hard-stop.
7. **State is local, memory is persistent.** SQLite for structure, MEMORY.md for learnings.
8. **Security by default.** Keys in vault, minimal permissions, all actions audited.
