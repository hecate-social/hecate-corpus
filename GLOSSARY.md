---
title: Glossary
layer: index
audience: [agent, human]
stage: stable
---

# Glossary

*Canonical vocabulary for Hecate. One term per concept. Old names noted once, never repeated inline.*

---

## The 5D Hierarchy

The structural backbone of Hecate. Four nested containers plus one flowing artifact. Each begins with `D`.

```
Domain (1)
  └── Division (N)        — bounded context
       └── Department (3)  — CMD, PRJ, QRY
            └── Desk (N)   — single capability

       Dossier — the artifact that flows through desks
```

### Domain

The overall business endeavor. The whole problem space. One per domain.

A Domain is the conglomerate that contains all Divisions belonging to a single business intent. It is the unit of governance, branding, and strategic alignment.

| Aspect | Detail |
|--------|--------|
| Cardinality | 1 per endeavor |
| Lifecycle | Setup → Discovery → ongoing |
| Owns | Brand, Divisions list, strategic intent |
| Old terms | Venture, Torch |

> **⚠ Override notice — read this carefully.**
>
> "Domain" is a heavily-overloaded word in Domain-Driven Design literature. Eric Evans uses it at two different scales: (1) *the entire problem space the business operates in* (the senior usage), and (2) loosely for *a sub-area / bounded context* (the junior usage, scattered through chapters about domain objects, domain services, domain models).
>
> **In Hecate, "Domain" means only the senior usage.** It is the whole problem space — the domain as a coherent endeavor. The bounded-context concept that DDD readers reflexively call "domain" is called **Division** in Hecate, never Domain.
>
> Concretely:
>
> | DDD term | Hecate term | Notes |
> |----------|-------------|-------|
> | Domain (senior, whole problem space) | **Domain** | Same word, same meaning |
> | Subdomain (Core / Supporting / Generic) | **Division** | Each Division IS a (sub)domain in DDD terms |
> | Bounded Context | **Division** | One Division = one bounded context |
> | Ubiquitous Language | (per Division) | Each Division has its own |
> | Domain Model | **Dossier** + Slips for that Division | The Aggregate IS the dossier |
> | Domain Event | **Slip** (internal) or **FACT** (cross-context) | Hecate distinguishes by transport |
> | Domain Service | (banned as horizontal) | Each Desk owns its logic |
>
> When you read DDD literature and see "domain" — pause and check the scale. If it's the whole-business scale, it maps to Hecate **Domain**. If it's the bounded-context scale, it maps to Hecate **Division**. Never let the word slip between scales mid-sentence.
>
> In code: use `domain_*` for new top-level processes; existing `*_venture` apps remain valid. Use `division_*` everywhere bounded contexts are involved.

### Division

A cohesive piece of software. One bounded context. The unit of repo or umbrella app.

A Division is a specialist firm within the Domain, responsible for exactly one bounded context. It produces three apps (CMD/PRJ/QRY) and may live in its own repo or under a shared umbrella.

| Aspect | Detail |
|--------|--------|
| Cardinality | N per Domain |
| Lifecycle | Discovery → Planning → Crafting → Operations |
| Owns | One bounded context, three Departments |
| Old terms | Cartwheel, Company |

### Department

A CQRS role inside a Division. Exactly three Departments per Division.

| Department | Role | App naming |
|------------|------|-----------|
| **CMD** | Receives intents, produces events | `{process_verb}_{subject}` (e.g., `manage_capabilities`, `design_division`) |
| **PRJ** | Subscribes to events, writes read models | `project_{read_model_plural}` (e.g., `project_capabilities`) |
| **QRY** | Reads from read models, serves queries | `query_{read_model_plural}` (e.g., `query_capabilities`) |

CMD is process-centric (verb-named). PRJ and QRY are data-centric (noun-named).

### Desk

A single capability inside a Department. The atomic unit of work.

A Desk is one folder, one supervisor, one vertical slice. It contains a command, an event, a handler, optionally a responder/emitter/listener. Everything related to that capability lives in that folder.

| Aspect | Detail |
|--------|--------|
| Cardinality | N per Department |
| Naming | `{verb}_{noun}/` for CMD, `get_{...}/` for QRY, `{event}_to_{table}/` for PRJ |
| Old terms | Spoke |

### Dossier

The aggregate. A folder of event slips that accumulates as the dossier passes through Desks.

The Dossier is not a data structure; it is its history. The stream of slips IS the dossier. When you "rebuild state," you read the dossier front to back. When validating a command, you ask "can this slip be added to this dossier?"

| Aspect | Detail |
|--------|--------|
| Cardinality | N per Division (one per process instance) |
| Identity | Stream ID = Dossier ID |
| Contents | Ordered Slips (events) |
| Synonym | Aggregate |

---

## Adjacent Terms

### Slip

A single event inside a Dossier. Immutable, append-only, ordered.

A Slip is what a Desk adds to a Dossier. It carries a subset of the Dossier's state plus any new data the Desk introduced. Naming: `{subject}_{verb_past}_v{N}` (e.g., `capability_announced_v1`).

| Synonym | Event |

### Process

A long-lived Dossier with lifecycle protocol (`initiate / open / shelve / resume / conclude / archive`). The Dossier of a Process is its full event stream.

| Examples | `setup_venture`, `discover_divisions`, `design_division`, `plan_division`, `generate_division`, `test_division`, `deploy_division` |

### Process Manager (PM)

A sibling slice inside a Department whose job is to react to a source event (often from another Domain or Division) and dispatch a target command. Provides cross-Division integration with discoverability at the filesystem level.

| Naming | `on_{source_event}_{verb}_{target}/` |
| Lives in | Target Department (the one issuing the command) |
| Examples | `on_division_identified_initiate_planning`, `on_planning_concluded_initiate_crafting` |

### Handler

The Desk function that validates a command and decides whether to dispatch an event.

| Naming | `maybe_{verb}_{noun}.erl` (e.g., `maybe_announce_capability`) |
| Returns | `{ok, Event} | {error, Reason}` |

### Projection

A PRJ Desk that subscribes to a slip type and writes a row to a read model table.

| Naming | `{event}_to_{table}` (e.g., `capability_announced_v1_to_capabilities`) |

### Filing Cabinet

Metaphor for the read-model store. Dossiers live in the Event Store; the Filing Cabinet holds the index cards (read models) that PRJ populates so QRY can find Dossiers without opening every one.

---

## Integration Vocabulary

### HOPE

A request sent from one agent to another over the mesh. Optimistic. Maps to a Command when a Responder receives it.

| Direction | Mesh → Domain (inbound) or Domain → Mesh (outbound) |
| Carries | Intent + parameters |
| Naming | `{verb}_{noun}_hope_v1.erl` |

### FACT

An integration event published from one bounded context to others over the mesh. Stable public schema. Distinct from internal Slips.

| Direction | Domain → Mesh |
| Carries | Public-contract subset of an internal Slip, possibly transformed |
| Naming | `{subject}.{verb_past}` topic |

### FEEDBACK

The response to a HOPE.

### Responder, Emitter, Listener, Requester

| Role | Direction | Purpose |
|------|-----------|---------|
| **Responder** | Mesh → Domain | Translates HOPE into Command |
| **Emitter** | Domain → Mesh | Publishes FACTs derived from internal Slips |
| **Requester** | Domain → Mesh | Sends HOPEs to other domains |
| **Listener** | Mesh → Domain | Receives FACTs, dispatches local Commands |

---

## Architecture Layers

### Tier Model

Hecate's runtime is organized in tiers. (See `philosophy/HECATE_TIER_MODEL.md`.)

| Tier | Name | Purpose |
|------|------|---------|
| L0 | Kernel | OS substrate (Linux today, Macula-Kernel future) |
| L1 | Identity | Macula Realm membership |
| L2 | Services | Stations, Daemons, Gateways |
| L3 | Session | Per-user agent runtime |
| L4 | Apps | Plugins, Studios, Booklets |

### ALC (Application Lifecycle)

The 2-process Division lifecycle: **Planning** + **Crafting**. Replaces the older 8-process model.

| Phase | Dossier | Apps |
|-------|---------|------|
| Planning | `division_planning` | `guide_division_planning`, `project_division_plannings`, `query_division_plannings` |
| Crafting | `division_crafting` | `guide_division_crafting`, `project_division_craftings`, `query_division_craftings` |

PMs chain them: `planning_concluded_v1 → initiate_crafting_v1`.

### Walking Skeleton

The minimum vertically-sliced implementation that proves the architecture. Built during Crafting. One Desk per Department, end-to-end, before any breadth.

---

## Status & State

### Bit Flags

All Dossier status fields are integers treated as bit flags, manipulated with `evoq_bit_flags`. Flags are powers of 2.

```
INITIATED = 1
ARCHIVED  = 2
OPEN      = 4
SHELVED   = 8
CONCLUDED = 16
```

### Lifecycle Protocol

Long-lived Processes share this protocol:

```
initiate → open ⇄ shelve / resume → conclude → archive
```

Domain-specific commands work only when state is `open`.

---

## Anti-Concepts (Banned Terminology)

| Banned | Why | Use Instead |
|--------|-----|-------------|
| `services/`, `utils/`, `helpers/` | Horizontal layers, graves where understanding dies | Vertical Desks |
| `user_created`, `order_updated` | CRUD events leak implementation; they don't describe business | `user_registered`, `order_confirmed` |
| `did:macula:...` legacy prefixes | Pre-MRI compatibility shim | `mri:...` |
| `manage_X` for Hecate Processes | Hecate Processes are verb-centric | `{process_verb}_{subject}` (e.g., `design_division`) |
| `torch`, `cartwheel`, `spoke` | Pre-2026-02 vocabulary | `domain`, `division`, `desk` |
| `created`, `updated`, `deleted` events | CRUD; no business meaning | Business verbs in past tense |

---

## Versioning

Slips and Commands carry `_v{N}` suffix. Versions are forever; we add new versions rather than mutate old ones.

```
capability_announced_v1.erl   ← original
capability_announced_v2.erl   ← added with new field
```

Both coexist. PRJ handles both. Old Slips in the store never change.

---

## Quick Decision Tree

When you don't know what to call something:

1. Is it a process step? → It's a **Desk**.
2. Is it something that happened? → It's a **Slip** (event).
3. Is it something that should happen? → It's a **Command**.
4. Is it the whole history of a thing? → It's a **Dossier** (aggregate).
5. Is it a folder of related Desks? → It's a **Department**.
6. Is it three Departments under one bounded context? → It's a **Division**.
7. Is it the whole problem space? → It's a **Domain**.
8. Is it a thing that flows in or out across the mesh? → It's a **HOPE** (request) or **FACT** (event).
9. Is it the reaction to a cross-Division event? → It's a **Process Manager**.

If you can't fit it into this list, you're probably reaching for a horizontal layer. Stop.

---

*Five Ds. Process over data. Names that scream.*
