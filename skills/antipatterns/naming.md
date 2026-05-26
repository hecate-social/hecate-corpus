---
title: "ANTIPATTERNS: Naming"
layer: skill
audience: [agent, human]
stage: stable
---

# ANTIPATTERNS: Naming — Names That Don't Scream

*Demons about naming violations. Names must scream business intent.*

[Back to Index](INDEX.md)

---

## 🔥 Technical Names Don't Scream

**Date:** 2026-02-04
**Origin:** hecate-daemon apprentice

### The Antipattern

Slice names that describe **HOW** (technical implementation) instead of **WHAT** (business intent):

| Technical Name (BAD) | What It Actually Does |
|---------------------|----------------------|
| `llm_model_poller` | Detects what models Ollama has |
| `llm_rpc_listener` | Handles incoming LLM requests from mesh |
| `*_listener` | Technical concern, not business intent |
| `*_poller` | Technical concern, not business intent |
| `*_handler` | Technical concern, not business intent |
| `*_worker` | Technical concern, not business intent |

These names are **horizontal thinking in disguise**. They describe the mechanism, not the meaning.

### The Rule

> **Slice names must be VERB PHRASES that scream WHAT the slice does, not HOW it does it.**

| ❌ Technical (HOW) | ✅ Screaming (WHAT) |
|-------------------|---------------------|
| `poll_llm_models/` | `detect_llm_models/` |
| `handle_llm_rpc/` | `listen_for_llm_request/` |
| `capability_listener/` | `discover_remote_capability/` |
| `event_handler/` | `on_capability_announced_update_index/` |

### Why It Matters

A stranger reading your folder structure should understand **what your system does**, not **what frameworks you used**.

```
# BAD — I see plumbing
apps/serve_llm/src/
├── poll_llm_models/
├── handle_llm_rpc/
└── emit_llm_events/

# GOOD — I see capabilities
apps/serve_llm/src/
├── detect_llm_models/
├── listen_for_llm_request/
└── announce_llm_availability/
```

### The Test

Read your slice name aloud. If it sounds like infrastructure, rename it.

- "This slice *polls models*" → Infrastructure. ❌
- "This slice *detects available LLM models*" → Business. ✅

---

## 🔥 Ambiguous Query Module Names

**Date:** 2026-02-10
**Origin:** hecate-daemon get_venture/list_ventures rename

### The Antipattern

Query modules with vague names that don't scream their intent or hide scaling dangers.

**Symptoms:**
```
apps/query_ventures/src/
├── get_venture/          # Get by what? ID? Name? Status?
├── list_ventures/        # Returns ALL domains? Unbounded!
└── get_all_divisions/    # "All" is a scaling time bomb
```

**Related violations:**
- `get_{aggregate}` without specifying lookup strategy
- `list_{aggregates}` / `get_all_{aggregates}` returning unbounded result sets
- No pagination in list queries — works in dev, crashes in production

### The Rule

> **1. Single lookups MUST specify the strategy: `get_{aggregate}_by_id`, `get_{aggregate}_by_name`**
> **2. List queries MUST be paged: `get_{aggregates}_page` — NEVER `list_{aggregates}`**

### The Correct Names

| Anti-Pattern | Correct | Why |
|-------------|---------|-----|
| `get_venture` | `get_venture_by_id` | Specifies lookup strategy |
| `list_ventures` | `get_ventures_page` | "page" enforces bounded results |
| `get_all_divisions` | `get_divisions_page` | No unbounded queries |
| `find_ventures` | `search_ventures` or `get_ventures_page` | "find" is vague |

### The Correct Structure

```
apps/query_ventures/src/
├── get_venture_by_id/        # One domain by primary key
├── get_active_venture/       # The currently active domain
├── get_ventures_page/        # Bounded page of domains
└── search_ventures/          # Full-text search (also paged)
```

### Why It Matters

- **Scaling**: `list_ventures` returning 10,000 rows will kill the daemon
- **Screaming architecture**: A stranger knows exactly what each module does
- **Extensibility**: Adding `get_venture_by_name` later doesn't conflict
- **Client expectations**: "page" in the name tells clients to expect pagination metadata

### The Lesson

> **Query names ARE the API contract. If the name doesn't scream "bounded" and "specific", the query is dangerous.**

Reference: `../skills/codegen/erlang/CODEGEN_ERLANG_NAMING.md`

---

## 🔥 Demon 16: Parameterized Phase Lifecycle

**Date exorcised:** 2026-02-12
**Where it appeared:** `guide_division_alc` — `start_phase/pause_phase/resume_phase/complete_phase` parameterized by phase name
**Cost:** Hides business intent behind generic lifecycle management

### The Lie

"Use generic lifecycle commands parameterized by phase name. It's DRY — 4 slices instead of 16."

### What Happened

`guide_division_alc` uses `start_phase(dna)`, `pause_phase(anp)`, etc. — generic commands that route by a `phase` parameter. Reading the slice names tells you nothing about what the system does:

```
src/start_phase/      ← Start what? Which phase? Why?
src/pause_phase/      ← Pause what?
src/complete_phase/   ← Complete what?
```

Compare with `guide_venture_lifecycle`, which has `start_discovery`, `pause_discovery` — those scream.

### Why It's Wrong

1. **Doesn't scream** — a stranger reading folder names has no idea what the system does
2. **Horizontal in disguise** — it's a generic lifecycle manager, not a business action
3. **API noise** — `POST /api/divisions/:id/phases/start?phase=dna` is infrastructure plumbing
4. **Loses the desk metaphor** — "someone opens a design workshop" vs "someone starts a phase"

### The Rule

> **Every long-lived sub-process gets its own screaming lifecycle verbs.**
> Generic `start_phase` / `pause_phase` / `complete_phase` is FORBIDDEN.

### The Correct Pattern

| Generic (BAD) | Screaming (GOOD) |
|--------------|-----------------|
| `start_phase(dna)` | `open_design` |
| `pause_phase(dna)` | `shelve_design` |
| `resume_phase(dna)` | `resume_design` |
| `complete_phase(dna)` | `conclude_design` |
| `start_phase(anp)` | `open_planning` |
| `pause_phase(anp)` | `shelve_planning` |
| `complete_phase(anp)` | `conclude_planning` |

### The Lifecycle Verb Convention

| Step | Verb | Meaning |
|------|------|---------|
| **Start** | `open_{process}` | Open the investigation/workshop/work |
| **Pause** | `shelve_{process}` | Put it on the shelf (with reason) |
| **Resume** | `resume_{process}` | Pick it back up |
| **Complete** | `conclude_{process}` or domain-specific | Hand off to next process |

Use a natural completion verb when one exists:
- `submit_vision` (submitting for review) > `conclude_vision`
- `confirm_deployment` (confirming it's live) > `conclude_operations`

### The Test

> "If I read only the slice directory names, can I understand what this app does?"

### The Lesson

> **DRY is not a virtue when it obscures intent. 16 screaming slices > 4 silent ones.**

See also: [SLICE_AUDIT.md](../SLICE_AUDIT.md) for the full audit workflow.

---

## 🔥 Demon 17: CRUD Verbs in Event-Sourced Commands

**Date exorcised:** 2026-02-12
**Where it appeared:** `guide_node_lifecycle` — `update_identity`, `update_capability`
**Cost:** Names that violate the "no CRUD" rule and hide business intent

### The Lie

"update" is a reasonable verb — it tells you the entity is being changed.

### Why It's Wrong

"Update" is the U in CRUD. It says HOW (a record is being modified) not WHAT (a business action is happening). In event sourcing, every state change is a business event — the verb must capture the business meaning.

### The Rule

> **Replace "update" with the business verb that describes WHAT changed.**

| CRUD (BAD) | Business Verb (GOOD) | When to Use |
|-----------|---------------------|-------------|
| `update_identity` | `amend_identity` | Metadata correction/refinement |
| `update_capability` | `amend_capability` | Details changed after announcement |
| `update_profile` | `enrich_profile` | Adding more information |
| `update_status` | `promote` / `suspend` / `activate` | Status transition |
| `update_address` | `correct_address` | Error correction |
| `update_config` | `reconfigure_{thing}` | Configuration change |

### The Lesson

> **If you catch yourself typing "update_", STOP. Ask: "What is the human at this desk actually doing?"**
> The answer is never "updating a record." It's always something more specific.

---

*Add more demons as we exorcise them.* 🔥🗝️🔥
