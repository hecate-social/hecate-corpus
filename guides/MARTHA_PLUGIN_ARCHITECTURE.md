---
title: Martha Plugin Architecture
layer: guide
audience: [agent, human]
stage: stable
---

# Martha Plugin Architecture

_How Martha implements the ALC as a hecate-web plugin._

**Date:** 2026-02-19
**Status:** Active

---

## What Is Martha?

Martha is the Hecate plugin that implements the Application Lifecycle (ALC). She guides users through domain setup, division discovery, and the eight-process development cycle -- from design through rescue.

Martha follows the standard Hecate plugin model: a daemon (`hecate-marthad`) communicating via Unix socket, a frontend (`hecate-marthaw`) compiled to an ES module, and CLI subcommands.

**Martha is to the ALC what QuickBooks is to accounting** -- the tool that implements the discipline.

---

## Repository Structure

```
hecate-martha/
  Dockerfile                         # Unified build: frontend + daemon

  hecate-marthad/                    # Erlang/OTP daemon
    rebar.config
    config/
      sys.config
      vm.args
    apps/
      guide_venture_lifecycle/       # CMD: domain + discovery lifecycle
      guide_division_alc/            # CMD: division 8-process lifecycle
      query_venture_lifecycle/       # QRY+PRJ: domain read models
      query_division_alc/            # QRY+PRJ: division read models
    src/                             # App shell: Cowboy, socket, health, manifest
    priv/static/                     # Frontend bundle (component.js)

  hecate-marthaw/                    # SvelteKit frontend
    vite.config.lib.ts               # Library build -> dist/component.js
    src/lib/
      MarthaStudio.svelte            # Root component (receives {api} prop)
      ... (vertical slices, see below)
```

---

## Runtime

```
~/.hecate/hecate-marthad/
  sqlite/            # Division/domain read models
  reckon-db/         # Event store data (martha_store)
  sockets/
    api.sock         # Discovered by hecate-web plugin watcher
  run/
  connectors/
  hecate-corpus/     # AI knowledge base (cloned at install)
```

Socket path: `~/.hecate/hecate-marthad/sockets/api.sock`
Manifest: `GET /manifest` returns `{name: "martha", icon: ..., version: ..., description: ...}`

---

## Event Store

Martha owns a single shared event store: **`martha_store`**.

Both CMD apps (`guide_venture_lifecycle`, `guide_division_alc`) write to this store using separate event streams:

| Stream Pattern | Owner | Example |
|---------------|-------|---------|
| `domain-{id}` | `guide_venture_lifecycle` | `domain-abc123` |
| `division-{id}` | `guide_division_alc` | `division-def456` |

The store is started by `hecate_marthad_app` (the app shell), not by domain apps. Domain supervisors only manage their own emitters, listeners, and process managers.

**Migration note:** In hecate-daemon this store is currently named `dev_studio_store`. The rename to `martha_store` happens during extraction.

---

## Frontend Vertical Slices

The frontend is organized by **ALC task**, not by technical concern. Each vertical slice contains its own component, store, and types. No `components/`, `stores/`, `utils/` directories.

### Domain-Level Slices

| Slice | ALC Task | What It Does |
|-------|----------|--------------|
| `compose_vision/` | Setup Domain | Initiate domain, AI-guided vision creation (Oracle conversation + live preview), submit vision |
| `brainstorm_venture_events/` | Discovery (chaotic) | Timeboxed high-octane event brainstorming -- free-form domain event stickies on a timeline |
| `storm_venture_big_picture/` | Discovery (structured) | Big Picture Event Storming organization: stack, groom, cluster, name, map, promote to divisions |

**On the brainstorm/Big Picture split:** Event Storming's Big Picture technique starts with a chaotic brainstorming phase (posting stickies) and then moves through structured organization (stacking duplicates, grooming, clustering into bounded contexts, naming, context mapping, and finally promoting clusters to divisions). The brainstorm is the high-energy creative burst; the Big Picture phases are the methodical distillation.

Both slices share the same underlying data model (stickies, stacks, clusters, fact arrows) via the domain aggregate. The `brainstorm_venture_events` slice manages the `storm` phase; `storm_venture_big_picture` manages `stack` through `promoted`.

### Division ALC Slices (8 processes)

| Slice | ALC Process | What It Does |
|-------|-------------|--------------|
| `design_division/` | Design | Design-level event storming, desk cards, aggregate grouping |
| `plan_division/` | Planning | Desk inventory, dependency mapping, sprint sequencing |
| `craft_division/` | Crafting | Code generation, module scaffolding |
| `refactor_division/` | Refactoring | Structural improvement analysis and execution |
| `debug_division/` | Debugging | Test suites, acceptance criteria, defect diagnosis |
| `deploy_division/` | Deployment | Release management, staged rollout |
| `monitor_division/` | Monitoring | Health checks, SLA tracking, anomaly detection |
| `rescue_division/` | Rescue | Incident response, diagnosis, escalation |

### Orchestration + Shared

| Slice | Purpose |
|-------|---------|
| `guide_venture/` | Domain header, division list, manual division identification, phase progress, lifecycle controls |
| `shared/` | TaskCard, AIAssistPanel, EventStreamViewer, ModelSelector |

### Slice Anatomy

Every slice follows the same pattern:

```
{slice_name}/
  {SliceName}.svelte     # Main view component
  {slice_name}.ts        # Stores + API calls (reactive state)
  types.ts               # Types scoped to this slice
```

Examples:

```
brainstorm_venture_events/
  BrainstormVentureEvents.svelte   # Chaotic sticky board UI
  brainstorm_venture_events.ts     # Sticky post/pull, timer, state
  types.ts                         # StickyNote, BrainstormSession

storm_venture_big_picture/
  StormVentureBigPicture.svelte    # Multi-phase board UI (stack/groom/cluster/name/map)
  storm_venture_big_picture.ts     # Phase transitions, cluster ops, fact arrows
  types.ts                         # EventCluster, FactArrow, StormPhase
```

### Why Vertical Slicing?

The current hecate-web DevOps studio uses horizontal layers:

```
components/devops/     # 13 components grouped by tech
stores/devops.ts       # 1242-line god-store for ALL state
types.ts               # All types in one file
```

This means understanding "how does event storming work?" requires reading 3+ files scattered across the tree. With vertical slicing:

- **To understand brainstorming** -- read `brainstorm_venture_events/` (one directory)
- **To understand Big Picture organization** -- read `storm_venture_big_picture/` (one directory)
- **To understand deployment** -- read `deploy_division/` (one directory)
- **To add monitoring features** -- edit `monitor_division/` (one directory)

The directory name screams what it does. All related code lives together.

---

## Daemon Architecture

The daemon is an Erlang/OTP umbrella with apps mirroring the ALC:

### CMD Apps (Process-Centric)

| App | Manages |
|-----|---------|
| `guide_venture_lifecycle` | Domain initiation, vision, Big Picture storm, division discovery |
| `guide_division_alc` | All 8 division processes: design through rescue |

### QRY+PRJ Apps (Data-Centric)

| App | Read Models |
|-----|-------------|
| `query_venture_lifecycle` | Domains, storm sessions, stickies, clusters, discovered divisions |
| `query_division_alc` | Divisions, aggregates, events, desks, deps, modules, tests, releases, incidents |

### API Shell

| Module | Purpose |
|--------|---------|
| `hecate_marthad_app` | Starts Cowboy on Unix socket, starts `martha_store`, ensures directory layout |
| `hecate_marthad_sup` | Top-level supervisor |
| `marthad_paths` | Path helpers for `~/.hecate/hecate-marthad/` |
| `marthad_health_api` | `GET /health` |
| `marthad_manifest_api` | `GET /manifest` |
| `marthad_api_routes` | Auto-discovery route handler for `/api/[...]` |
| `marthad_api_utils` | JSON response helpers, body parsing, field extraction |
| `marthad_projection_event` | Flattens `#event{}` records to maps for projections |
| `marthad_mesh_proxy` | Routes mesh publishes to hecate-daemon via pg groups |
| `marthad_plugin_registrar` | Registers Martha with hecate-daemon on startup |

---

## Domain-to-Division Bridge

The two CMD apps (`guide_venture_lifecycle` and `guide_division_alc`) are **completely decoupled**. They share `martha_store` but use separate event streams and have zero direct coupling.

### How Divisions Are Discovered

Two paths produce `division_identified_v1` in the domain aggregate:

| Path | Command | Context |
|------|---------|---------|
| Manual | `identify_division_v1` | User manually names a division during discovery |
| Storm promotion | `promote_event_cluster_v1` | Promotes a named cluster from Big Picture (emits TWO events atomically: `event_cluster_promoted_v1` + `division_identified_v1`) |

Both paths add to the domain's `discovered_divisions` map (`context_name => division_id`).

### Missing Process Manager

Currently, **no process manager bridges** the domain lifecycle to division ALC. When `division_identified_v1` fires, nothing automatically creates the division aggregate in `guide_division_alc`.

This bridge is needed:

```
guide_venture_lifecycle                          guide_division_alc
─────────────────────                            ─────────────────────
division_identified_v1  ────  PM  ────────►  initiate_division_v1
                               │
              on_division_identified_initiate_division
```

The process manager would:
1. Subscribe to `martha_store` for `division_identified_v1` events
2. Extract `division_id`, `context_name`, `description` from the event
3. Dispatch `initiate_division_v1` to `guide_division_alc`

Per convention, this PM lives in the **target domain** (`guide_division_alc`):

```
apps/guide_division_alc/src/
  on_division_identified_initiate_division/
    on_division_identified_initiate_division.erl
    on_division_identified_initiate_division_sup.erl
```

### Big Picture Storm Phases (Domain Aggregate)

The storm runs as a linear state machine within the domain aggregate:

```
storm ──► stack ──► groom ──► cluster ──► name ──► map ──► promoted
```

| Phase | Activity | Commands |
|-------|----------|----------|
| `storm` | Chaotic brainstorming -- post domain event stickies | `post_event_sticky_v1`, `pull_event_sticky_v1` |
| `stack` | Group duplicate/related stickies | `stack_event_sticky_v1`, `unstack_event_sticky_v1` |
| `groom` | Pick canonical sticky per stack, absorb duplicates | `groom_event_stack_v1` |
| `cluster` | Group stickies into bounded context clusters | `cluster_event_sticky_v1`, `uncluster_event_sticky_v1` |
| `name` | Name each cluster (the bounded context name) | `name_event_cluster_v1` |
| `map` | Draw fact arrows between clusters (context mapping) | `draw_fact_arrow_v1`, `erase_fact_arrow_v1` |
| `promoted` | Clusters promoted to divisions | `promote_event_cluster_v1` |

Phase transitions are strictly linear via `advance_storm_phase_v1`. Lifecycle controls: `shelve_big_picture_storm_v1`, `resume_big_picture_storm_v1`, `archive_big_picture_storm_v1`.

---

## Cross-Daemon Communication

Martha's frontend component receives an `api` prop from hecate-web that routes to the Martha daemon socket. But Martha also needs capabilities from the main hecate-daemon:

- **LLM streaming** -- AI-assisted vision, event storming, domain expertise
- **Agent prompts** -- Loaded from hecate-corpus knowledge base

This flows through BEAM clustering (same cookie, same network):

```
MarthaStudio.svelte
  |  api.post('/chat', {model, messages})
  v
hecate-marthad (Cowboy)
  |  erlang:send / pg
  v
hecate-daemon (serve_llm)
  |  LLM provider API
  v
Response streams back through the chain
```

Martha daemon proxies LLM requests to hecate-daemon. The frontend never talks to hecate-daemon directly -- all requests go through the Martha socket.

---

## Migration from hecate-web

Martha is being extracted from hecate-web's built-in DevOps studio. The extraction involves:

1. **Frontend**: Move 13 components + 1 store + types from `hecate-web/src/lib/` to `hecate-marthaw/src/lib/`, restructured as vertical slices
2. **Backend**: Move 4 Erlang apps from `hecate-daemon/apps/` to `hecate-marthad/apps/`
3. **Store rename**: `dev_studio_store` becomes `martha_store`
4. **API routes**: Extract domain/division routes from `hecate_api_routes.erl` to Martha's own API handler
5. **Decouple**: Remove Martha-specific error codes from shared `api.ts`, Martha-specific types from shared `types.ts`, Martha-specific StatusBar logic
6. **Add bridge PM**: Implement `on_division_identified_initiate_division` process manager

After extraction, Martha registers as `{id: "martha", path: "/martha"}` in the plugin system, discovered automatically via the socket convention.

---

## Reference Implementation

The Trader plugin (`hecate-social/hecate-trader/`) is the reference for:
- Daemon structure (Cowboy on Unix socket, health/manifest endpoints)
- Frontend build (Vite library mode, externalized Svelte runtime)
- Plugin discovery (directory naming, socket watching)
- Component loading (ES module via blob URL, `api` prop injection)

Martha follows the same pattern but is significantly more complex (event sourcing, AI integration, 122 Erlang source files vs Trader's handful).
