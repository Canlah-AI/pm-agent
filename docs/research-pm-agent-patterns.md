# PM Agent Industry Research (2026)

> Compiled from 40+ sources across Devin, OpenHands, CrewAI, MetaGPT, Claude-Flow, and industry research.

## Key Numbers

- LLM agents perform best on **~35 min human-equivalent** tasks
- Independent multi-agent systems amplify errors **17.2x**
- Successful human:agent supervision ratio = **1:20**
- Vector-based memory caching: **15x** faster responses, **90%** cost reduction
- By 2026, **40% of enterprise apps** will integrate task-specific AI agents (Gartner)
- Quality is the #1 production killer (**32%** cite it as top barrier)
- Security/compliance/auditability cited by **75%** as top deployment requirement

## The "Telephone Game" Problem (CRITICAL)

The biggest risk in PM→Engineer delegation. Research shows:
- Information distortion in iterative interactions (BLEURT scores decline)
- Agents reinterpret delegated tasks in unexpected ways
- Two agents can reinforce each other's hallucinations ("mutual confidence")
- Strictly sequential tasks = strongest predictor of multi-agent failure

**Our mitigation (from architecture doc):**
- Static prompt templates (not LLM-generated prompts)
- PM Agent fills ALL context before delegating
- Claude Code never "figures out" what to do — PM already decided
- Brand context injected, never discovered

## Framework Landscape

| Framework | Best For | Key Pattern |
|-----------|----------|-------------|
| Devin | End-to-end autonomous coding | Session UI + Linear/Slack integration |
| OpenHands | Open-source agent platform | AgentDelegateAction (subtask delegation) |
| CrewAI | Role-based agent teams | Manager agents + hierarchical process |
| Claude-Flow | Claude Code orchestration | 60+ agents, shared memory, MCP protocol |
| MetaGPT | Software company simulation | SOP-driven, PM→Architect→Engineer chain |

## Token Cost Reality

- Agentic workflows create **quadratic/exponential** token growth
- A mid-sized product: **5-10M tokens/month** with ~1,000 daily users
- One dev reported **8M tokens in a single run** with Moltbot
- Memory engineering reduces token burn from context re-generation

## Sources

See full research report for 40+ linked sources covering:
- Devin, OpenHands, CrewAI, MetaGPT, Claude-Flow
- Task decomposition best practices (IBM, OpenAI, AWS)
- Memory architecture (MongoDB, AWS AgentCore, arXiv papers)
- Failure modes (Jason Liu/Cognition, arXiv multi-agent failure analysis)
- Framework comparisons (LangGraph vs CrewAI vs AutoGen)
- Enterprise case studies (Deloitte, McKinsey, Mapfre, Universal Bank)
