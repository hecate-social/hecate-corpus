---
title: The Canonical Architecture
layer: philosophy
audience: [agent, human]
stage: stable
---

# CARTWHEEL.md — The Canonical Architecture

*The Division Architecture in practice. This is the blueprint.*

> **Note:** "Cartwheel" is the historical name for what is now called "Division Architecture".
> `cartwheel` -> `division`, `spoke` -> `desk`.

> **Runtime equivalence:** A Division maps 1-to-1 to a Hecate App (plugin).
> The daemon IS the Division's runtime; its Erlang umbrella apps ARE the
> CMD/PRJ/QRY departments. See [Hecate Plugin Directory Convention](../guides/HECATE_PLUGIN_DIRECTORY_CONVENTION.md#app--division-the-fundamental-equivalence).

---

## Overview

**Division Architecture** has three sequences and four mesh components:

| Sequence | Purpose |
|----------|---------|
| **CMD** | Command/Write side — receives intents, produces events |
| **PRJ** | Projection — subscribes to events, updates read models |
| **QRY** | Query/Read side — serves queries from read models |

| Mesh Component | Direction | Purpose |
|----------------|-----------|---------|
| **RESPONDER** | Mesh → Domain | Receives HOPEs, translates to Commands |
| **EMITTER** | Domain → Mesh | Subscribes to Events, publishes FACTs |
| **REQUESTER** | Domain → Mesh | Sends HOPEs, awaits responses |
| **LISTENER** | Mesh → Domain | Receives FACTs, triggers local processing |

---

## CMD Domain Structure

A CMD domain app is composed of **desks** (vertical slices) + optional **shared infrastructure**.

```
apps/manage_capabilities/
├── src/
│   ├── manage_capabilities_app.erl
│   ├── manage_capabilities_sup.erl       # Starts desks + shared infra ONLY
│   │
│   ├── (shared infrastructure - optional)
│   │   ├── manage_capabilities_store.erl # ReckonDB instance
│   │   └── manage_capabilities_exchange.erl # Internal pub/sub
│   │
│   ├── announce_capability/              # DESK (vertical slice)
│   │   ├── announce_capability_desk_sup.erl
│   │   ├── announce_capability_v1.erl
│   │   ├── capability_announced_v1.erl
│   │   ├── maybe_announce_capability.erl
│   │   ├── announce_capability_responder_v1.erl
│   │   └── capability_announced_v1_to_mesh.erl
│   │
│   ├── update_capability/                # DESK
│   │   └── ...
│   │
│   └── retract_capability/               # DESK
│       └── ...
│
└── rebar.config                          # Include src_dirs for spokes
```

---

## CMD Desk Contents

Every CMD desk contains:

| File | Type | Purpose |
|------|------|---------|
| `*_desk_sup.erl` | Supervisor | Supervises all workers in this desk |
| `*_v1.erl` | Record | Command struct (`new/N`, `to_map/1`, `from_map/1`) |
| `*_v1.erl` | Record | Event struct (what happened) |
| `maybe_*.erl` | Handler | Validates command, dispatches via evoq |
| `*_responder_v1.erl` | gen_server | HOPE → Command translator (mesh inbound) |
| `*_to_pg.erl` | gen_server | Subscribes via evoq, broadcasts to pg (internal) |
| `*_to_mesh.erl` | gen_server | Subscribes via evoq, publishes to mesh (external) |

**Optional:**
| File | Type | Purpose |
|------|------|---------|
| `on_{event}_maybe_*.erl` | Policy | Reacts to internal domain event, dispatches command |
| `on_{fact}_maybe_*.erl` | Listener | Reacts to external fact (pg/mesh), dispatches command |

---

## CMD Desk Flow

### Inbound (Mesh → Domain)

```
HOPE (announce_capability_hope_v1)
    ↓
Responder (announce_capability_responder_v1)
    ↓ translates
Command (announce_capability_v1)
    ↓
Handler (maybe_announce_capability)
    ↓ validates, dispatches
Event (capability_announced_v1)
    ↓
Store (ReckonDB)
```

### Outbound (Domain → Mesh / pg)

```
ReckonDB (event stored)
    ↓ evoq subscription (by event_type)
Emitter (capability_announced_v1_to_mesh)
    ↓ receives {events, [Event]}, transforms
FACT (published to mesh topic / pg group)
```

**Emitters subscribe to the event store via evoq at startup.** They are NOT called manually by API handlers. See [EVENT_SUBSCRIPTION_FLOW.md](EVENT_SUBSCRIPTION_FLOW.md).

### Internal Reaction (via Policy)

```
Event from own domain (license_revoked_v1)
    ↓ evoq subscription (by event_type)
Policy (on_license_revoked_v1_maybe_remove_plugin)
    ↓ dispatches command to own aggregate
Command (remove_plugin_v1)
    ↓
Normal CMD flow
```

### Cross-Domain Reaction (via Listener)

```
FACT from another domain (app_available)
    ↓ arrives via pg group or mesh
Listener (on_app_available_maybe_install_plugin)
    ↓ dispatches command to local aggregate
Command (install_plugin_v1)
    ↓
Normal CMD flow
```

---

## Domain Supervisor Pattern

**Domain supervisor ONLY starts:**
1. Shared infrastructure (store, exchange)
2. Desk supervisors

**Domain supervisor NEVER directly supervises workers.**

```erlang
%% manage_capabilities_sup.erl
init([]) ->
    Children = [
        %% Shared infra
        #{id => manage_capabilities_store,
          start => {manage_capabilities_store, start_link, []},
          type => worker},
        
        %% Desks (supervisors, not workers!)
        #{id => announce_capability_desk_sup,
          start => {announce_capability_desk_sup, start_link, []},
          type => supervisor},
        #{id => update_capability_desk_sup,
          start => {update_capability_desk_sup, start_link, []},
          type => supervisor},
        #{id => retract_capability_desk_sup,
          start => {retract_capability_desk_sup, start_link, []},
          type => supervisor}
    ],
    {ok, {#{strategy => one_for_one, intensity => 5, period => 10}, Children}}.
```

---

## Desk Supervisor Pattern

**Desk supervisor starts all workers for that desk. Emitters start first — they subscribe via evoq and react autonomously.**

```erlang
%% announce_capability_desk_sup.erl
init([]) ->
    Children = [
        %% Emitters first — they subscribe to ReckonDB via evoq
        #{id => capability_announced_v1_to_pg,
          start => {capability_announced_v1_to_pg, start_link, []},
          type => worker},
        #{id => capability_announced_v1_to_mesh,
          start => {capability_announced_v1_to_mesh, start_link, []},
          type => worker},
        %% Responder — receives HOPEs from mesh
        #{id => announce_capability_responder_v1,
          start => {announce_capability_responder_v1, start_link, []},
          type => worker}
    ],
    {ok, {#{strategy => one_for_one, intensity => 5, period => 10}, Children}}.
```

---

## Naming Conventions

| Component | Pattern | Example |
|-----------|---------|---------|
| Desk directory | `{verb}_{noun}/` | `announce_capability/` |
| Desk supervisor | `{verb}_{noun}_desk_sup.erl` | `announce_capability_desk_sup.erl` |
| Command | `{verb}_{noun}_v1.erl` | `announce_capability_v1.erl` |
| Event | `{noun}_{past_verb}_v1.erl` | `capability_announced_v1.erl` |
| Handler | `maybe_{verb}_{noun}.erl` | `maybe_announce_capability.erl` |
| Responder | `{verb}_{noun}_responder_v1.erl` | `announce_capability_responder_v1.erl` |
| Emitter (pg) | `{event}_to_pg.erl` | `capability_announced_v1_to_pg.erl` |
| Emitter (mesh) | `{event}_to_mesh.erl` | `capability_announced_v1_to_mesh.erl` |
| Policy | `on_{event}_maybe_{verb}_{noun}.erl` | `on_license_revoked_v1_maybe_remove_plugin.erl` |
| Listener | `on_{fact}_maybe_{verb}_{noun}.erl` | `on_app_available_maybe_install_plugin.erl` |
| HOPE struct | `{verb}_{noun}_hope_v1.erl` | `announce_capability_hope_v1.erl` |

---

## PRJ Domain Structure

**PRJ is a SEPARATE app from QRY.** Projections are writers (react to events, update read models). Queries are readers (serve HTTP, read from read models). Different responsibilities, different apps.

PRJ app naming: `project_{read_model_plural}` (e.g., `project_capabilities`).

```
apps/project_capabilities/
├── src/
│   ├── project_capabilities.app.src
│   ├── project_capabilities_app.erl
│   ├── project_capabilities_sup.erl
│   ├── project_capabilities_store.erl      # SQLite read model (owned by PRJ)
│   │
│   ├── capability_announced_v1_to_capabilities/  # PRJ desk
│   │   ├── capability_announced_v1_to_capabilities_sup.erl
│   │   └── capability_announced_v1_to_capabilities.erl
│   │
│   └── capability_retracted_v1_to_capabilities/  # PRJ desk
│       └── ...
```

PRJ desks subscribe to ReckonDB via `reckon_evoq_adapter:subscribe/5` and write to SQLite.

---

## QRY Domain Structure

**QRY is a SEPARATE app from PRJ.** Queries read from the read models that PRJ populates. Simple functions, no events, no subscriptions.

QRY app naming: `query_{read_model_plural}` (e.g., `query_capabilities`).

```
apps/query_capabilities/
├── src/
│   ├── query_capabilities.app.src
│   ├── query_capabilities_app.erl
│   ├── query_capabilities_sup.erl
│   │
│   ├── get_capability_by_id/
│   │   └── get_capability_by_id_api.erl    # GET /api/.../capabilities/:id
│   │
│   └── get_capabilities_page/
│       └── get_capabilities_page_api.erl   # GET /api/.../capabilities
```

QRY depends on PRJ (for the SQLite store module). QRY has NO subscriptions, NO event handling — it only reads.

---

## Three-App Division Structure

Every division with event sourcing has three apps:

| App | Department | Responsibility | Naming |
|-----|-----------|---------------|--------|
| CMD | Command | Receives intents, produces events | `{process_verb}_{subject}` |
| PRJ | Projection | Subscribes to events, writes read models | `project_{read_model_plural}` |
| QRY | Query | Serves HTTP, reads from read models | `query_{read_model_plural}` |

```
apps/
├── manage_capabilities/        # CMD — event sourcing, aggregates, commands
├── project_capabilities/       # PRJ — projections, SQLite store, event subscriptions
└── query_capabilities/         # QRY — API handlers, reads from SQLite
```

---

## Code Generation

This architecture is **strict enough for deterministic code generation**.

Given:
- Domain name: `manage_capabilities`
- Desk name: `announce_capability`
- Event name: `capability_announced`

A generator can produce:
- [ ] `announce_capability_desk_sup.erl`
- [ ] `announce_capability_v1.erl`
- [ ] `capability_announced_v1.erl`
- [ ] `maybe_announce_capability.erl`
- [ ] `announce_capability_responder_v1.erl`
- [ ] `capability_announced_v1_to_mesh.erl`
- [ ] Update `manage_capabilities_sup.erl` to include desk
- [ ] Update `rebar.config` src_dirs

**No AI needed.** Templates + naming conventions = complete desk.

---

*The division stands. The desks deliver.* 🔥🗝️🔥
