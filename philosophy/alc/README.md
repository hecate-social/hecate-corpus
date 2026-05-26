---
title: The Division Application Lifecycle
layer: philosophy
audience: [agent, human]
stage: stable
---

# HECATE ALC -- The Division Application Lifecycle

*Two processes. One chain. Planning feeds crafting.*

**Updated:** 2026-03-10 — Consolidated from 8 processes to 2 process-centric dossiers.

---

## Overview

The ALC governs how a **division** (bounded context, cohesive piece of software) evolves from design to delivery. It uses **two independent processes**, each with its own dossier (aggregate), its own event stream, and its own CMD/PRJ/QRY app trio.

The ALC applies to **divisions specifically**. Domains have their own lifecycle. Nodes run continuously. The division is where craft happens, and the ALC is the rhythm of that craft.

---

## The Two Processes

| # | Process | CMD App | Purpose |
|---|---------|---------|---------|
| 1 | **Planning** | `guide_division_planning` | Event storming, aggregate design, desk inventory, dependencies |
| 2 | **Crafting** | `guide_division_crafting` | Code generation, testing, release delivery |

Each process is a first-class citizen -- its own dossier, its own desks, its own read models.

### Apps

| Process | CMD | PRJ | QRY |
|---------|-----|-----|-----|
| Planning | `guide_division_planning` | `project_division_plannings` | `query_division_plannings` |
| Crafting | `guide_division_crafting` | `project_division_craftings` | `query_division_craftings` |

---

## Lifecycle Protocol

Every process is long-lived. Every process speaks with **screaming verbs** -- not generic `start_phase` / `pause_phase`, but verbs that name the process they govern.

```
initiate_{process}  --> {process}_initiated_v1   (birth, auto via PM)
open_{process}      --> {process}_opened_v1      (begin work)
shelve_{process}    --> {process}_shelved_v1      (blocked, set aside)
resume_{process}    --> {process}_resumed_v1      (pick back up)
conclude_{process}  --> {process}_concluded_v1    (hand off to next)
archive_{process}   --> {process}_archived_v1     (soft delete)
```

### Status Bit Flags

```
INITIATED = 1
ARCHIVED  = 2
OPEN      = 4
SHELVED   = 8
CONCLUDED = 16
```

### State Machine

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

A process that has never been initiated is not yet born. Initiating creates it (usually via process manager). Opening it makes it active. Shelving pauses it. Resuming reactivates it. Concluding completes it and signals readiness for the next process.

See **Demon #16** in [antipatterns/INDEX.md](../../skills/antipatterns/INDEX.md) for why generic lifecycle verbs are forbidden.

---

## Process Manager Chain

```
guide_venture_lifecycle              guide_division_planning           guide_division_crafting
        │                                      │                                │
  division_identified_v1  ──PM──►  initiate_planning_v1                         │
                                               │                                │
                                   planning_concluded_v1  ──PM──►  initiate_crafting_v1
```

| PM | Lives In | Subscribes To | Dispatches |
|----|----------|--------------|------------|
| `on_division_identified_initiate_planning` | `guide_division_planning` | `division_identified_v1` | `initiate_planning_v1` |
| `on_planning_concluded_initiate_crafting` | `guide_division_crafting` | `planning_concluded_v1` | `initiate_crafting_v1` |

---

## Planning Dossier

**Aggregate:** `division_planning_aggregate` (stream: `division-planning-{id}`)

**Desks (10):**

| Desk | Type | Purpose |
|------|------|---------|
| `initiate_planning` | lifecycle | Birth (auto via PM) |
| `archive_planning` | lifecycle | Soft delete |
| `open_planning` | lifecycle | Begin work |
| `shelve_planning` | lifecycle | Pause with reason |
| `resume_planning` | lifecycle | Resume from shelve |
| `conclude_planning` | lifecycle | Complete, trigger crafting PM |
| `design_aggregate` | domain | Define aggregate boundaries |
| `design_event` | domain | Define domain events |
| `plan_desk` | domain | Add desk to inventory |
| `plan_dependency` | domain | Map desk dependencies |

**Read model tables (PRJ):** `division_plannings`, `designed_aggregates`, `designed_events`, `planned_desks`, `planned_dependencies`

**Query endpoints (QRY):**
- `GET /api/plannings/:id`
- `GET /api/domains/:venture_id/plannings`
- `GET /api/plannings`

---

## Crafting Dossier

**Aggregate:** `division_crafting_aggregate` (stream: `division-crafting-{id}`)

**Desks (12):**

| Desk | Type | Purpose |
|------|------|---------|
| `initiate_crafting` | lifecycle | Birth (auto via PM) |
| `archive_crafting` | lifecycle | Soft delete |
| `open_crafting` | lifecycle | Begin work |
| `shelve_crafting` | lifecycle | Pause with reason |
| `resume_crafting` | lifecycle | Resume from shelve |
| `conclude_crafting` | lifecycle | Complete |
| `generate_module` | domain | Generate Erlang module from template |
| `generate_test` | domain | Generate test module |
| `run_test_suite` | domain | Execute test suite |
| `record_test_result` | domain | Record test outcome |
| `deliver_release` | domain | Deliver a versioned release |
| `stage_delivery` | domain | Stage release for rollout |

**Read model tables (PRJ):** `division_craftings`, `generated_modules`, `generated_tests`, `test_suites`, `test_results`, `releases`, `delivery_stages`

**Query endpoints (QRY):**
- `GET /api/craftings/:id`
- `GET /api/domains/:venture_id/craftings`
- `GET /api/craftings`

---

## What This Replaced

The previous 8-process ALC (design, planning, crafting, refactoring, debugging, deployment, monitoring, rescue) was consolidated into 2 focused processes. The rationale:

1. **Planning** absorbs design -- aggregate design and desk planning are part of the same dossier
2. **Crafting** absorbs implementation, testing, and delivery -- code generation through release is one continuous process
3. Monitoring, rescue, debugging, and refactoring are operational concerns, not lifecycle phases -- they happen continuously, not sequentially

### Superseded Files

The following per-process detail files describe the old 8-process model and are **superseded** by this document:

- `design.md` — absorbed into Planning
- `planning.md` — absorbed into Planning
- `crafting.md` — absorbed into Crafting
- `debugging.md` — absorbed into Crafting
- `deployment.md` — absorbed into Crafting
- `monitoring.md` — operational, not lifecycle
- `rescue.md` — operational, not lifecycle
- `refactoring.md` — operational, not lifecycle
- `../HECATE_DISCOVERY_N_ANALYSIS.md` — old 4-phase naming
- `../HECATE_ARCHITECTURE_N_PLANNING.md` — old 4-phase naming
- `../HECATE_TESTING_N_IMPLEMENTATION.md` — old 4-phase naming
- `../HECATE_DEPLOYMENT_N_OPERATIONS.md` — old 4-phase naming

---

## Three Lifecycles

The ALC is one of three lifecycle types in the Hecate ecosystem:

| Lifecycle | Scope | Nature |
|-----------|-------|--------|
| **Domain Lifecycle** | The overall business endeavor | Setup, discovery, orchestration |
| **Division ALC** | A single bounded context | The two-process chain described here |
| **Node Lifecycle** | Infrastructure | Continuous operation, no phases |

See [HECATE_DOMAIN_LIFECYCLE.md](../HECATE_DOMAIN_LIFECYCLE.md) for the domain-level view.

---

## Related Doctrines

| Doctrine | Relevance | Description |
|----------|-----------|-------------|
| [Walking Skeleton](../HECATE_WALKING_SKELETON.md) | Crafting | Fully operational system from day one |
| [Dossier Principle](../DDD.md) | Planning | Process-centric domain modeling |
| [Vertical Slicing](../VERTICAL_SLICING.md) | Planning, Crafting | Features live together, no horizontal layers |
| [Screaming Architecture](../SCREAMING_ARCHITECTURE.md) | Planning, Crafting | Names reveal intent |
| [Division Model](../../guides/CARTWHEEL_COMPANY_MODEL.md) | All | CMD/PRJ/QRY department structure |

---

## For Agents

When working on a division:

1. **Know which process is active.** Do not craft during planning. Each process has a purpose -- respect it.
2. **Use the lifecycle verbs.** `open_planning`, `conclude_crafting`, `shelve_planning` -- not `start`, `stop`, `pause`. The process screams its name.
3. **Follow the chain.** Plan before crafting. The order exists for a reason.
4. **Conclude fast.** Small iterations. Conclude planning early, start crafting. The goddess favors momentum over perfection.

---

## Terminology

| Term | Meaning | Old Term |
|------|---------|----------|
| **Domain** | The overall business endeavor | Torch |
| **Division** | A bounded context, cohesive software unit | Cartwheel / Company |
| **Department** | CMD, PRJ, or QRY within a division | Department |
| **Desk** | A single capability within a department | Spoke |
| **Dossier** | The aggregate -- folder of event slips | Dossier |

---

*Two processes. Planning feeds crafting. Each with its own dossier.*
