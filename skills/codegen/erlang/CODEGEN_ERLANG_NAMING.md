---
title: Erlang Code Naming Conventions
layer: codegen
audience: [codegen]
stage: stable
---

# CODEGEN_ERLANG_NAMING.md — Erlang Code Naming Conventions

_Strict naming rules for all components in the Domain/Division/Desk architecture._

**Target:** Erlang/OTP with `reckon_evoq`

**Related files:**
- [CODEGEN_ERLANG_TEMPLATES.md](CODEGEN_ERLANG_TEMPLATES.md) — Code templates
- [CODEGEN_ERLANG_CHECKLISTS.md](CODEGEN_ERLANG_CHECKLISTS.md) — Generation checklists
- [../../NAMING_CONVENTIONS.md](../../NAMING_CONVENTIONS.md) — Consolidated quick-reference

---

## Master Naming Table

| Component      | Pattern                      | Example                                     |
| -------------- | ---------------------------- | ------------------------------------------- |
| Domain app     | `{verb}_{noun}`              | `manage_capabilities`                       |
| Query app      | `query_{noun}`               | `query_capabilities`                        |
| CMD Desk dir   | `{command}/`                 | `announce_capability/`                      |
| PRJ Desk dir   | `{event}/` (version-agnostic) | `capability_announced/`                   |
| CMD Desk sup   | `{command}_desk_sup`         | `announce_capability_desk_sup`              |
| PRJ Desk sup   | `{event}_sup`                | `capability_announced_sup`                  |
| Command        | `{command}_v1`               | `announce_capability_v1`                    |
| Event          | `{noun}_{past_verb}_v1`      | `capability_announced_v1`                   |
| Handler        | `maybe_{command}`            | `maybe_announce_capability`                 |
| CMD API        | `{command}_api`              | `announce_capability_api`                   |
| Responder      | `{command}_responder_v1`     | `announce_capability_responder_v1`          |
| Emitter (mesh) | `{event}_to_mesh`            | `capability_announced_to_mesh`              |
| Emitter (pg)   | `{event}_to_pg`              | `capability_announced_to_pg`                |
| Aggregate      | `{noun}_aggregate`           | `capability_aggregate`                      |
| Projection     | `{event}_to_{read_store}`    | `capability_announced_to_capabilities`      |
| Policy/PM      | `on_{event}_maybe_{command}` | `on_llm_detected_maybe_announce_capability` |
| Query (by PK)  | `get_{noun}_by_id`           | `get_capability_by_id`                      |
| QRY by-ID API  | `get_{noun}_by_id_api`       | `get_capability_by_id_api`                  |
| Query (paged)  | `get_{nouns}_page`           | `get_capabilities_page`                     |
| QRY paged API  | `get_{nouns}_page_api`       | `get_capabilities_page_api`                 |
| Tests          | `{module}_tests`             | `capability_aggregate_tests`                |

---

## Query Module Naming (Critical)

### Rule 1: No Raw `get_{aggregate}` — Use `get_{aggregate}_by_id`

A module named `get_venture` is ambiguous. Get it how? By name? By status?

**The name MUST specify the lookup strategy.**

| Module Name | What It Screams |
|-------------|----------------|
| `get_venture_by_id` | "I fetch one domain by its primary key" |
| `get_venture_by_name` | "I fetch one domain by its unique name" |
| `get_active_venture` | "I fetch the currently active domain" |

### Rule 2: No Raw `list_{aggregates}` — Use `get_{aggregates}_page`

A module named `list_ventures` sounds like it returns ALL domains. That's a scaling time bomb.

**Every list query MUST be paged by default.**

| Name | Screams | Danger |
|------|---------|--------|
| `list_ventures` | "Give me everything" | Unbounded. Will crash at scale. |
| `get_all_ventures` | "Give me literally all of them" | Even worse. Explicit unbounded intent. |
| `get_ventures_page` | "Give me one page" | Bounded. Safe. Correct. |

### Rule 3: Filtered Queries Include the Filter in the Name

| Module Name | What It Screams |
|-------------|----------------|
| `get_divisions_by_venture` | "I return divisions belonging to a specific domain" |
| `get_findings_by_division` | "I return findings for a specific division" |
| `get_ventures_by_status` | "I return domains matching a status filter" |

### Complete QRY Naming Convention

| Pattern | Module Name | Intent |
|---------|-------------|--------|
| Single by PK | `get_{aggregate}_by_id` | One record by primary key |
| Single by unique field | `get_{aggregate}_by_{field}` | One record by unique constraint |
| Single active/current | `get_active_{aggregate}` | The one currently active |
| Paged list | `get_{aggregates}_page` | Bounded page of records |
| Filtered paged list | `get_{aggregates}_by_{field}` | Filtered + paged |
| Search | `search_{aggregates}` | Full-text search (paged) |
| Count/stats | `count_{aggregates}` | Aggregate statistics |

### Route Alignment

| Module | Route | HTTP |
|--------|-------|------|
| `get_venture_by_id_api` | `/api/domains/:venture_id` | GET |
| `get_active_venture_api` | `/api/domain` | GET |
| `get_ventures_page_api` | `/api/domains` | GET |
| `get_division_by_id_api` | `/api/divisions/:division_id` | GET |
| `get_divisions_page_api` | `/api/divisions` | GET |

---

## Anti-Patterns

| Anti-Pattern | Correct | Why |
|--------------|---------|-----|
| `get_venture` | `get_venture_by_id` | "get how?" is ambiguous |
| `list_ventures` | `get_ventures_page` | Unbounded result set |
| `get_all_ventures` | `get_ventures_page` | "all" is a scaling time bomb |
| `find_ventures` | `get_ventures_page` or `search_ventures` | "find" is vague |
| `query_ventures` | Specific module per query | Provider module, not a desk |

---

## Migration: Renaming Existing Modules

When renaming a query module (e.g., `get_venture` -> `get_venture_by_id`):

### Checklist

1. **Rename directory**: `src/get_venture/` -> `src/get_venture_by_id/`
2. **Create new module files** with updated `-module()` declarations
3. **Delete old files** from the renamed directory
4. **Update `rebar.config`**: `src_dirs` entry
5. **Update callers**:
   - Route in `hecate_api_routes.erl`
   - Any API handler calling `old_module:execute/1`
   - Any internal caller (e.g., `get_active_*_api.erl`)
6. **Delete stale beam files**: `_build/default/lib/*/ebin/old_module.beam`
7. **Compile + test**: `rebar3 compile && rebar3 eunit`

---

## Banned Suffixes (Technical Noise)

These suffixes reveal implementation, not intent:

- `*_handler`, `*_manager`, `*_processor`, `*_worker`, `*_service`, `*_helper`, `*_util`, `*_impl`

## Allowed Suffixes (With Meaning)

- `*_v1` (Version)
- `*_desk_sup` (CMD desk supervisor)
- `*_sup` (PRJ desk supervisor, e.g., `capability_announced_sup`)
- `*_responder_v1` (HOPE receiver)
- `*_to_mesh` (Emitter to mesh)
- `*_to_pg` (Emitter to pg)
- `*_to_{table}` (Projection)
- `*_store` (Storage accessor)
- `*_api` (API handler)

---

_Names scream intent. Pages prevent disasters. Be explicit or be sorry._
