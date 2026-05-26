---
title: The Domain Lifecycle
layer: philosophy
audience: [agent, human]
stage: stable
---

# The Domain Lifecycle — Process-Centric Architecture

_How Hecate models software development as a set of first-class processes._

**Date:** 2026-02-10 (updated 2026-03-10)
**Status:** Active — supersedes parent-child aggregate pattern
**Origin:** DnA conversation on process-centric vs data-centric architecture

---

## The Insight

Traditional software tools model development as **data management**: you create projects, update records, delete tasks. The verbs are CRUD, the nouns are passive containers.

Hecate models development as a **set of processes**: each phase of building software is its own first-class citizen with its own lifecycle, its own state, and its own dossier. The verbs are business actions, the nouns are active processes.

**The test:** imagine a human sitting at a desk. What lands on their desk? What do they do with it? What do they pass to the next desk?

If the answer is "they manage a database record" — the model is wrong.
If the answer is "they investigate, decide, produce, and hand off" — the model is right.

---

## Terminology

### Hierarchy

```
Domain (1)
  └── Division (N)        — one per bounded context
       └── Department (3)  — CMD, PRJ, QRY
            └── Desk (N)   — individual capability
```

### Domain Concepts

| Term | What It Is | Old Term |
|------|-----------|----------|
| **Domain** | The overall business endeavor — a conglomerate of divisions | Venture, Torch |
| **Division** | A specialist firm within the domain, responsible for one bounded context | Cartwheel / Company |
| **Department** | CMD, PRJ, or QRY within a division | Department |
| **Desk** | A single capability within a department (where work gets done) | Spoke |
| **Dossier** | The aggregate — the folder of event slips passing through desks | Dossier |

### Divisions Are Virtual

A division is a **virtual umbrella** — a logical grouping of apps that share a business context. In practice, it maps to either:

1. **Multiple apps within a shared umbrella** (e.g. hecate-daemon)
2. **A separate repo** within the organization

Each division produces N apps following the department pattern:

| Department | Naming | Nature |
|------------|--------|--------|
| CMD | The **process name** itself | Process-centric (verbs) |
| PRJ | `project_{read_model}` | Data-centric (projections) |
| QRY | `query_{read_model}` | Data-centric (queries) |

---

## Three Lifecycle Types

Hecate manages three fundamentally different lifecycle types. Each has its own guide process and its own rhythm.

### 1. Domain Lifecycle (`guide_venture_lifecycle`)

**Scope:** Per domain. **Duration:** Short inception, long-lived discovery.

The domain has two processes:
- `setup_venture` — short-lived, fire-and-done. Birth of the endeavor.
- `discover_divisions` — long-lived, with `open/shelve/resume/conclude` lifecycle. Identifies bounded contexts.

The domain orchestrates divisions. Once divisions are discovered, each follows its own ALC independently.

**Apps:**
- CMD: `guide_venture_lifecycle`
- PRJ: `project_ventures`
- QRY: `query_ventures`

### 2. Division ALC — 2 Processes

**Scope:** Per division. **Duration:** Long-lived, sequential.

The Application Lifecycle has **2 process-centric phases**, each with its own dossier (aggregate), its own event stream, and its own CMD/PRJ/QRY app trio:

| # | Process | Dossier | Purpose |
|---|---------|---------|---------|
| 1 | **Planning** | `division_planning` | Event storming, aggregate design, desk inventory, dependencies |
| 2 | **Crafting** | `division_crafting` | Code generation, testing, release delivery |

Each process is a first-class aggregate with its own event stream, its own desks, and its own read models. Process managers chain them: when planning concludes, crafting initiates automatically.

**Apps per process:**

| Process | CMD App | PRJ App | QRY App |
|---------|---------|---------|---------|
| Planning | `guide_division_planning` | `project_division_plannings` | `query_division_plannings` |
| Crafting | `guide_division_crafting` | `project_division_craftings` | `query_division_craftings` |

**Planning dossier desks:**
- `initiate_planning` — birth (auto-triggered by PM on `division_identified_v1`)
- `archive_planning` — soft delete
- `open_planning` / `shelve_planning` / `resume_planning` / `conclude_planning` — lifecycle
- `design_aggregate` / `design_event` — architecture
- `plan_desk` / `plan_dependency` — desk inventory

**Crafting dossier desks:**
- `initiate_crafting` — birth (auto-triggered by PM on `planning_concluded_v1`)
- `archive_crafting` — soft delete
- `open_crafting` / `shelve_crafting` / `resume_crafting` / `conclude_crafting` — lifecycle
- `generate_module` / `generate_test` — code generation
- `run_test_suite` / `record_test_result` — testing
- `deliver_release` / `stage_delivery` — release management

### 3. Node Continuous (`guide_node_lifecycle`)

**Scope:** Per node. **Duration:** Indefinite, always-on.

The node lifecycle has no phases and no sub-processes. A node registers, operates forever, and may unpair/re-pair. All desks are independent operations on a living entity:

- `register_identity` — join the mesh
- `configure_node` — set preferences
- `serve_llm` — provide LLM capabilities
- `manage_capabilities` — announce what this node can do
- ...

There is no lifecycle protocol — the node is simply alive and responding to commands.

---

## Lifecycle Protocol

Long-lived processes (domain discovery + planning + crafting) implement:

```
Command:  initiate_{process}_v1   →  Event: {process}_initiated_v1
Command:  open_{process}_v1       →  Event: {process}_opened_v1
Command:  shelve_{process}_v1     →  Event: {process}_shelved_v1
Command:  resume_{process}_v1     →  Event: {process}_resumed_v1
Command:  conclude_{process}_v1   →  Event: {process}_concluded_v1
Command:  archive_{process}_v1    →  Event: {process}_archived_v1
```

**Status bit flags** (powers of 2):

```
INITIATED = 1
ARCHIVED  = 2
OPEN      = 4
SHELVED   = 8
CONCLUDED = 16
```

**State transitions:**

```
             initiate
  (none) ──────────► initiated
                        │
                   open │
                        ▼
                      open
                        │ ▲
                 shelve │ │ resume
                        ▼ │
                      shelved
                        │
               conclude │
                        ▼
                    concluded

  (any state) ──archive──► archived
```

**Rules:**
- Domain-specific commands only work when state is `open`
- `shelve` records a reason (blocked, waiting, break)
- `resume` clears the shelve
- `conclude` is the hand-off — triggers the next process via PM

---

## Process Manager Chain

```
guide_venture_lifecycle              guide_division_planning           guide_division_crafting
        │                                      │                                │
  division_identified_v1  ──PM──►  initiate_planning_v1                         │
                                               │                                │
                                   planning_concluded_v1  ──PM──►  initiate_crafting_v1
```

Each PM is a gen_server that subscribes to the source event and dispatches the target command:

| PM | Subscribes To | Dispatches |
|----|--------------|------------|
| `on_division_identified_initiate_planning` | `division_identified_v1` | `initiate_planning_v1` |
| `on_planning_concluded_initiate_crafting` | `planning_concluded_v1` | `initiate_crafting_v1` |

---

## Fact Flow Diagram

```
                        guide_venture_lifecycle
                        (orchestrator)
                             │
          ┌──────────────────┼────────────────┐
          │                  │                │
   setup_venture    discover_divisions        │
          │                  │                │
          └────fact──────────┤                │
                             │                │
                    ┌────────▼─────────┐      │
                    │ planning         │──────┤
                    │ (design + plan)  │      │
                    └────────┬─────────┘      │
                             │ PM             │
                    ┌────────▼─────────┐      │
                    │ crafting         │──────┘
                    │ (build + deliver)│
                    └──────────────────┘
```

---

## The Guided Conversation Method

### The Protocol

1. **Frame a decision** — Ask a clear, bounded question with no ambiguity
2. **Present options** — Show tradeoffs as a table, not opinions. Include pros AND cons.
3. **User decides** — They own the choice. Never decide for them.
4. **Record the decision** — It becomes a constraint on all future decisions.
5. **Build forward** — Each decision narrows the next decision's option space.
6. **Produce an artifact** — The conversation output is structured data, not prose.

### Phase-Specific Conversations

| Phase | Guided Conversation Produces |
|-------|------------------------------|
| `setup_venture` | Domain name + brief |
| `discover_divisions` | Division list with names, descriptions, boundary rationale |
| `planning` | Aggregates, events, desk inventory, dependencies, sprint sequence |
| `crafting` | Module generation, test strategy, release manifest |

### The Decision Cascade

Decisions made in earlier phases constrain later phases:

```
setup: name = "my-saas-app"
  └─ constrains discovery scope

discover: divisions = [auth, billing, notify]
  └─ constrains planning: 3 divisions to plan

planning(auth): aggregates = [user, session, credential]
                desks = [register_user, authenticate_user, ...]
  └─ constrains crafting: exactly these desks to implement

crafting(auth): modules generated, tests passed, release delivered
```

Each phase's output is the next phase's input. The conversation at each phase only needs to cover that phase's decisions — everything else is already settled.

---

## What This Replaces

| Old Concept | Replaced By | Why |
|------------|-------------|-----|
| `manage_torches` | `setup_venture` + `guide_venture` | Process, not data management |
| `guide_division_alc` (8 processes) | `guide_division_planning` + `guide_division_crafting` | 2 focused dossiers |
| `query_division_alc` (monolithic) | `project_division_plannings/craftings` + `query_division_plannings/craftings` | Proper PRJ/QRY split per dossier |
| Parent-child aggregates | Orchestrator + PM chain | No hierarchy, just coordination |
| `torch_aggregate` | `setup` aggregate | Scoped to inception only |
| `cartwheel_aggregate` | `division_planning_aggregate` + `division_crafting_aggregate` | Each process owns its state |
| 8-process ALC | 2-process ALC (planning + crafting) | Simpler, focused dossiers |
| 4 frontend phases (DnA/AnP/TnI/DnO) | 2 frontend phases (Planning/Crafting) | Matches backend structure |
| `dna_status/anp_status/tni_status/dno_status` | `planning_status/crafting_status` | One status per dossier |
| start/pause/resume/complete | initiate/open/shelve/resume/conclude/archive | Full lifecycle protocol |
| spoke | desk | Work lands on a desk |

---

## Fact Transport

Inter-process communication uses **facts on the mesh** (external) or **pg groups** (internal to same BEAM VM). Processes never call each other directly.

---

*Process-centric architecture. Each phase is a first-class citizen.*
