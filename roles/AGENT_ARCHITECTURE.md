---
title: Martha Agent Architecture
layer: role
audience: [agent]
stage: stable
---

# Martha Agent Architecture

*How AI agents collaborate to guide the domain lifecycle.*

**Date:** 2026-03-11
**Status:** Active

---

## Principles

1. **Minimize cost/token consumption** — use the cheapest model tier that can do the job
2. **Human-in-the-loop at defined gates** — the human can always intervene, but 5 mandatory gates require explicit approval
3. **Multi-provider** — agents request a capability tier, not a specific model. `serve_llm` resolves to the best available
4. **All LLM calls through `serve_llm`** — never direct API calls. Usage tracked per agent/domain/division automatically
5. **Self-hosted where possible** — Ollama/local models for trivial tasks (free)

---

## Model Tiers

Agents request a capability tier. The daemon maps it to whatever model is available.

| Tier | Capability | Example Models | Cost |
|------|-----------|---------------|------|
| **T1 — Reasoning** | Deep analysis, DDD, boundary decisions, review | Claude Opus, GPT-4o, Qwen-72B | $$$ |
| **T2 — Competent** | Good code gen, solid reasoning, follows patterns | Claude Sonnet, Qwen-32B, Llama-70B, DeepSeek-V3 | $$ |
| **T3 — Fast** | Template following, simple transforms, coordination | Claude Haiku, Qwen-7B, Llama-8B, Groq-hosted small models | $ |
| **T0 — Local** | Trivial tasks, formatting, JSON extraction | Self-hosted via Ollama | Free |

**Cost optimization rule:** Start at the lowest tier that can do the job. Escalate only on failure or insufficient output quality.

---

## Agent Roster

| # | Role | Phase(s) | Tier | HITL Gate | File |
|---|------|----------|------|-----------|------|
| 1 | **Visionary** | setup_venture | T1 | Vision Gate | `visionary.md` |
| 2 | **Explorer** | discover_divisions | T1 | Boundary Gate | `explorer.md` |
| 3 | **Stormer** | storming | T1→T2 | Design Gate | `stormer.md` |
| 4 | **Architect** | storming→planning | T2 | — | `architect.md` |
| 5 | **Erlang Coder** | crafting | T2 | — | `erlang_coder.md` |
| 6 | **Svelte Coder** | crafting | T2 | — | `svelte_coder.md` |
| 7 | **SQL Coder** | crafting | T3 | — | `sql_coder.md` |
| 8 | **Tester** | crafting | T2 | — | `tester.md` |
| 9 | **Reviewer** | cross-cutting | T1 | Review Gate | `reviewer.md` |
| 10 | **Coordinator** | kanban, all | T3/T0 | — | `coordinator.md` |
| 11 | **Delivery Manager** | crafting (post-review) | T3/T0 | Release Gate | `delivery_manager.md` |
| 12 | **Mentor** | post-mortem, continuous | T1 | — | `mentor.md` |

---

## Human-in-the-Loop Gates

Five mandatory checkpoints where the human must approve before the pipeline advances.

```
┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  Visionary ──► [VISION GATE] ──► Explorer ──► [BOUNDARY GATE]       │
│                                                       │              │
│                    Stormer ◄───────────────────────────┘              │
│                       │                                              │
│               [DESIGN GATE]                                          │
│                       │                                              │
│                   Architect                                          │
│                       │                                              │
│         ┌─────────────┼─────────────┐                                │
│         │             │             │                                │
│    Erlang Coder  Svelte Coder  SQL Coder                            │
│         │             │             │                                │
│         └─────────────┼─────────────┘                                │
│                       │                                              │
│                    Tester                                             │
│                       │                                              │
│                   Reviewer ──► [REVIEW GATE] ──► Delivery Manager   │
│                                                       │              │
│                                               [RELEASE GATE]        │
│                                                       │              │
│                                                   Mentor             │
│                                               (retrospective)        │
│                                                                      │
│                    Coordinator (orchestrates all)                     │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

| Gate | After | Before | Human Approves |
|------|-------|--------|----------------|
| **Vision Gate** | Visionary produces domain brief | Explorer starts discovery | Domain name, vision, brief |
| **Boundary Gate** | Explorer identifies divisions | Stormer begins per-division work | Division names, boundaries, rationale |
| **Design Gate** | Stormer produces EventStorm per division | Architect translates to technical design | Aggregates, events, desk inventory, dependencies |
| **Review Gate** | Reviewer examines generated code | Delivery Manager prepares release | Code quality, anti-pattern compliance |
| **Release Gate** | Delivery Manager has version-bumped + CI green | Publish to appstore mesh | Final go/no-go for shipping |

Between gates, agents work autonomously. The Coordinator manages flow and escalates to human only at gates or on failure.

---

## Role File Structure

Each role file (`roles/{role}.md`) contains:

```markdown
---
id: role_id
name: Display Name
tier: T2
context:
  - philosophy/DDD.md
  - skills/NAMING_CONVENTIONS.md
---

{system prompt: identity + task + behavioral rules + output format}
```

### Context Loading

The `context:` frontmatter lists shared knowledge files the agent needs. The runtime:
1. Loads the role file → system prompt (always small, cheap)
2. Reads `context:` manifest → loads listed files as reference material
3. Context files are loaded ONCE per agent session, not per turn

### Context Budget

| Role | Context Files | Est. Input Tokens |
|------|--------------|-------------------|
| Visionary | 4 files | ~4K |
| Explorer | 5 files | ~6K |
| Stormer | 4 files | ~5K |
| Architect | 4 files | ~5K |
| Erlang Coder | 4 files | ~8K |
| Svelte Coder | 2 files | ~3K |
| SQL Coder | 2 files | ~3K |
| Tester | 2 files | ~4K |
| Reviewer | 10 files | ~12K |
| Coordinator | 1 file | ~1K |
| Delivery Manager | 2 files | ~3K |
| Mentor | 4 files | ~6K |

---

## serve_llm Integration

Every agent call flows through the daemon's LLM service:

```
Agent (Martha plugin)
  → POST /api/llm/chat
    body: {
      model_tier: "T2",
      messages: [...],
      venture_id: "...",
      division_id: "...",
      agent_id: "erlang_coder"
    }
  → serve_llm resolves tier → best available model
  → Provider dispatches (Anthropic/Groq/Ollama/...)
  → Usage tracked automatically per agent/domain/division
```

Agents never know which model they're talking to. They request a tier, the daemon picks.

---

## Relationship to Existing Personas

The old `agents/` directory contains simple system prompts used by the Martha frontend AI Assist panel. These are **chat personas** for human-interactive use.

The new `roles/` directory contains **autonomous agent roles** with context manifests, tier assignments, and behavioral rules. They are used by the agent orchestration pipeline, not by the chat UI.

| Directory | Purpose | Used By |
|-----------|---------|---------|
| `agents/` | Chat personas (human-interactive) | Martha AI Assist panel |
| `roles/` | Autonomous agent roles (pipeline) | Martha agent orchestrator |

The two coexist. Some roles (Visionary, Stormer) are human-interactive AND autonomous.

---

## Learning Loop

The Mentor operates continuously in three modes:

```
┌─────────────────────────────────────────────────────────┐
│                    LIVE (T3 — cheap)                     │
│  Watches every agent's output as it's produced.         │
│  Flags issues BEFORE downstream agents consume them.    │
│  One-line corrections. Pattern matching, not reasoning. │
└───────────────────────────┬─────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────┐
│                 GATE COACHING (T1 — per gate)            │
│  At each HITL gate, briefs the human:                   │
│  "Here's what looks good, here are my concerns."        │
│  Helps the human make better gate decisions.            │
└───────────────────────────┬─────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────┐
│               POST-MORTEM (T1 — per run)                │
│  After RELEASE GATE: full retrospective.                │
│  Amends role files, antipattern docs, tier assignments. │
│  Encodes lessons permanently for the next run.          │
└─────────────────────────────────────────────────────────┘
```

**Why three modes:** A correction after the Stormer costs 1 message. After the Reviewer it costs rework across 5 agents. Catch early, fix cheap.

Every domain the team builds makes the next domain cheaper, faster, and higher quality. The role files are living documents that encode accumulated wisdom.
