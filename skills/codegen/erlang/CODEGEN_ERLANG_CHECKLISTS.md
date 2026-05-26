---
title: Generation Checklists
layer: codegen
audience: [codegen]
stage: stable
---

# CODEGEN_ERLANG_CHECKLISTS.md — Generation Checklists

_Step-by-step checklists for generating Division Architecture (Cartwheel) code._

**Target:** Erlang/OTP with `reckon_evoq`

**Related files:**
- [CODEGEN_ERLANG_UMBRELLA.md](CODEGEN_ERLANG_UMBRELLA.md) — Umbrella project layout
- [CODEGEN_ERLANG_TEMPLATES.md](CODEGEN_ERLANG_TEMPLATES.md) — Code templates
- [CODEGEN_ERLANG_NAMING.md](CODEGEN_ERLANG_NAMING.md) — Naming conventions
- [../../NAMING_CONVENTIONS.md](../../NAMING_CONVENTIONS.md) — Consolidated quick-reference

---

## Variables

Templates use these placeholders:

| Variable          | Example                | Description                    |
| ----------------- | ---------------------- | ------------------------------ |
| `{domain}`        | `manage_capabilities`  | Domain app name                |
| `{command}`       | `announce_capability`  | Command/desk name (verb_noun)  |
| `{event}`         | `capability_announced` | Event name (noun_past_verb)    |
| `{read_store}`    | `capabilities`         | Read model read_store name     |
| `{query}`         | `find_capability`      | Query name                     |
| `{trigger_event}` | `llm_model_detected`   | Event that triggers a policy   |

---

## Directory Structures

### CMD Domain App (WRITE EVENTS)

```
apps/{domain}/
├── src/
│   ├── {domain}_app.erl
│   ├── {domain}_sup.erl
│   ├── {domain}_store.erl              # ReckonDB instance
│   │
│   ├── {command}/                      # DESK directory
│   │   ├── {command}_desk_sup.erl
│   │   ├── {command}_v1.erl
│   │   ├── {event}_v1.erl
│   │   ├── maybe_{command}.erl
│   │   ├── {command}_responder_v1.erl
│   │   ├── {event}_to_mesh.erl
│   │   └── {command}_api.erl
│   │
│   └── on_{src_event}_{action}_{target}/    # PM SIBLING SLICE (one per cross-domain integration)
│       ├── on_{src_event}_{action}_{target}_sup.erl
│       └── on_{src_event}_{action}_{target}.erl
│
└── rebar.config
```

PMs are siblings of desks, not nested. The `on_*/` directories at the top level of `src/` make every cross-domain integration point visible from `ls`. (See [antipatterns/structure.md Demon 18](../../antipatterns/structure.md#-demon-18-process-managers-inside-desks).)

### PRJ Domain App

```
apps/query_{domain_noun}/
├── src/
│   ├── query_{domain_noun}_app.erl
│   ├── query_{domain_noun}_sup.erl
│   ├── query_{domain_noun}_store.erl   # SQLite instance
│   │
│   └── {event}_to_{read_store}/             # PRJ DESK directory
│       ├── {event}_to_{read_store}_sup.erl
│       └── {event}_to_{read_store}.erl
│
└── rebar.config
```

### QRY (inside PRJ app)

**CRITICAL: No `list_*` or `get_all_*`. Use `get_{aggs}_page` for lists, `get_{agg}_by_id` for lookups.**

```
apps/query_{domain_noun}/
└── src/
    ├── get_{aggregate}_by_id/           # QRY DESK: single lookup
    │   ├── get_{aggregate}_by_id.erl
    │   └── get_{aggregate}_by_id_api.erl
    │
    └── get_{aggregates}_page/           # QRY DESK: paged list
        ├── get_{aggregates}_page.erl
        └── get_{aggregates}_page_api.erl
```

---

## Walking Skeleton: Mandatory Desks

**Every new aggregate MUST implement TWO desks from the start:**

1. `initiate_{aggregate}` — Birth event, starts the lifecycle
2. `archive_{aggregate}` — End event, marks as archived (not deleted!)

### Why Both Are Required

Event sourcing means **we never delete**. Without `archive_{aggregate}`:
- Test data pollutes production-like environments
- No way to "undo" accidental creations
- Lists grow unbounded with obsolete records

### Aggregate Status Bit Flags

Every aggregate MUST have a status integer field with AT LEAST these flags:

```erlang
%% Minimum required status flags
-define(STATUS_INITIATED, 1).   %% 2^0 - Created/born
-define(STATUS_ARCHIVED,  2).   %% 2^1 - Soft-deleted, hidden from queries
```

Additional flags are domain-specific:

```erlang
%% Example: Domain-specific flags
-define(STATUS_INITIATED, 1).
-define(STATUS_ACTIVE,    2).
-define(STATUS_PAUSED,    4).
-define(STATUS_COMPLETED, 8).
-define(STATUS_ARCHIVED, 16).
```

### Walking Skeleton Checklist

When creating a new aggregate `{noun}`:

**CMD Domain (`manage_{noun}s`):**
- [ ] `initiate_{noun}/` desk — emits `{noun}_initiated_v1`
- [ ] `archive_{noun}/` desk — emits `{noun}_archived_v1`
- [ ] `{noun}_aggregate.erl` — with `STATUS_INITIATED` and `STATUS_ARCHIVED` flags

**PRJ Domain (`query_{noun}s`):**
- [ ] `{noun}_initiated_v1_to_{noun}s/` desk — inserts row
- [ ] `{noun}_archived_v1_to_{noun}s/` desk — sets `status |= ARCHIVED`

**Query Behavior:**
- Default `list_all/0` MUST filter out archived records: `WHERE status & ?ARCHIVED = 0`
- Provide `list_all_including_archived/0` for admin/debug purposes

---

## Generation Checklists

### New Aggregate (Walking Skeleton)

Given: `noun=capability`, `domain=manage_capabilities`

Generate:

- [ ] `src/capability_aggregate.erl` — with `-behaviour(evoq_aggregate).`
- [ ] `test/capability_aggregate_tests.erl` — argument order + guard tests
- [ ] `initiate_capability/` desk (see New CMD Desk)
- [ ] `archive_capability/` desk (see New CMD Desk)

### New CMD Desk

Given: `domain=manage_capabilities`, `command=announce_capability`, `event=capability_announced`

Generate:

- [ ] `src/announce_capability/announce_capability_desk_sup.erl`
- [ ] `src/announce_capability/announce_capability_v1.erl`
- [ ] `src/announce_capability/capability_announced_v1.erl`
- [ ] `src/announce_capability/maybe_announce_capability.erl`
- [ ] `src/announce_capability/announce_capability_api.erl` — API handler (see CMD API Template)
- [ ] `src/announce_capability/announce_capability_responder_v1.erl`
- [ ] `src/announce_capability/capability_announced_to_mesh.erl`
- [ ] `src/announce_capability/capability_announced_to_pg.erl` — internal emitter
- [ ] `test/capability_announced_to_pg_tests.erl` — emitter test
- [ ] Update `manage_capabilities_sup.erl` to include desk supervisor
- [ ] Update `rebar.config` src_dirs
- [ ] Add route to `hecate_api_routes.erl`

### New QRY Desk

Given: `noun=capability`, `query_app=query_capabilities`

**For paged list query:**
- [ ] `src/get_capabilities_page/get_capabilities_page.erl` — query module
- [ ] `src/get_capabilities_page/get_capabilities_page_api.erl` — API handler (see QRY Paged API Template)
- [ ] Update `rebar.config` src_dirs
- [ ] Add route to `hecate_api_routes.erl`: `GET /api/capabilities`

**For single-by-id query:**
- [ ] `src/get_capability_by_mri/get_capability_by_mri.erl` — query module
- [ ] `src/get_capability_by_mri/get_capability_by_mri_api.erl` — API handler (see QRY By-ID API Template)
- [ ] Update `rebar.config` src_dirs
- [ ] Add route to `hecate_api_routes.erl`: `GET /api/capabilities/:mri`

### New PRJ Desk

Given: `event=capability_announced`, `read_store=capabilities`

Generate:

- [ ] `src/capability_announced_to_capabilities/capability_announced_to_capabilities_sup.erl`
- [ ] `src/capability_announced_to_capabilities/capability_announced_to_capabilities.erl`
- [ ] Update `query_capabilities_sup.erl` to include desk supervisor
- [ ] Update `rebar.config` src_dirs

**Subscription delivery smoke test** (See Demon #24):
- [ ] Write a CT test that subscribes, appends an event, and asserts the subscriber receives it
- [ ] Test with `#event{}` record input, not just flat maps (See Demon #23)

### New Policy/PM

Given: `src_event=llm_model_detected`, `action=announce`, `target=capability`

Generate a SIBLING SLICE in the target CMD app (not nested inside the desk):

- [ ] `src/on_llm_model_detected_announce_capability/on_llm_model_detected_announce_capability_sup.erl` (single-worker supervisor)
- [ ] `src/on_llm_model_detected_announce_capability/on_llm_model_detected_announce_capability.erl` (gen_server: `pg:join` in `init/1`, spawn worker for dispatch in `handle_info`)
- [ ] Add `on_llm_model_detected_announce_capability_sup` as a child of the **domain supervisor** (`{domain}_sup.erl`), peer to desk sups — NOT a child of `announce_capability_desk_sup.erl`

---

_Checklists are deterministic. Check every box. Skip nothing._
