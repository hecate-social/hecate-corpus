---
title: "Example: Parent-Child Aggregate Pattern"
layer: philosophy
audience: [agent, human]
stage: stable
---

# Example: Parent-Child Aggregate Pattern

*Canonical example: Domain → Division relationship*

> **Note:** This document has been updated to use current terminology (2026-02-10).
> `torch` -> `domain`, `cartwheel` -> `division`, `spoke` -> `desk`.

---

## The Pattern

When a parent aggregate (Domain) needs to create child aggregates (Divisions):

1. **Parent DISCOVERS** children (declares "I need X")
2. **Child INITIATES** itself (starts its own lifecycle)

This separates:
- **What exists** (parent's decision)
- **How it works** (child's lifecycle)

---

## Domain Model

```
Domain (Business Endeavor)
├── Has 0..N Divisions (Bounded Contexts)
├── DISCOVERS what divisions it needs
└── Does NOT control division lifecycle

Division (Bounded Context)
├── Belongs to one Domain
├── INITIATES its own lifecycle
└── Manages its own phases (DnA, AnP, TnI, DnO)
```

---

## Correct Flow

```
User: "Create a domain with a division for user auth"

1. POST /api/domain/initiate
   → venture_initiated_v1 stored in Domain's stream
   → venture_initiated_v1 emitted to mesh (optional)

2. POST /api/domains/:id/divisions/discover
   → division_discovered_v1 stored in Domain's stream
   → division_discovered_v1 emitted to mesh

3. Division service listener receives fact
   → Policy decides to initiate
   → initiate_division_v1 command dispatched
   → division_initiated_v1 stored in Division's stream
```

---

## Wrong Flow (Antipattern)

```
❌ venture_initiated_v1 → automatically create division
```

**Why wrong:**
- Assumes 1:1 relationship
- No explicit decision about what divisions needed
- Child creation happens without parent consent
- Can't have domains with 0, 2, or 10 divisions

---

## Code Structure

### discover_divisions (Parent Domain)

```
apps/discover_divisions/src/
├── initiate_venture/
│   ├── initiate_venture_v1.erl            # Command
│   ├── venture_initiated_v1.erl           # Event
│   ├── maybe_initiate_venture.erl         # Handler
│   └── venture_initiated_v1_to_mesh.erl   # Emitter (optional)
│
└── discover_division/                      # Parent discovers children
    ├── discover_division_v1.erl           # Command
    ├── division_discovered_v1.erl         # Event (in Domain stream!)
    ├── maybe_discover_division.erl        # Handler
    └── division_discovered_v1_to_mesh.erl # Emitter → mesh
```

### design_division (Child Domain)

```
apps/design_division/src/
├── initiate_division/                                  # desk
│   ├── initiate_division_desk_sup.erl
│   ├── initiate_division_v1.erl                        # Command
│   ├── division_initiated_v1.erl                       # Event (in Division stream!)
│   ├── maybe_initiate_division.erl                     # Handler
│   └── initiate_division_api.erl
│
└── on_division_discovered_initiate_division/           # PM sibling slice
    ├── on_division_discovered_initiate_division_sup.erl
    └── on_division_discovered_initiate_division.erl   # pg:join + dispatch
```

---

## Event Stream Ownership

| Event | Stored In | Reason |
|-------|-----------|--------|
| `venture_initiated_v1` | Domain stream | Birth of domain |
| `division_discovered_v1` | Domain stream | Parent's decision |
| `division_initiated_v1` | Division stream | Birth of division |

**Key insight:** `division_discovered_v1` belongs to the domain because it's the parent's decision about what children exist.

---

## Semantic Distinction

| Action | Owner | Verb | Meaning |
|--------|-------|------|---------|
| **Discover** | Parent | "I need X" | Parent declares what children exist |
| **Initiate** | Child | "I exist" | Child starts its lifecycle |

---

## API Endpoints

```http
# Domain endpoints
POST /api/domain/initiate           # Create a domain
GET  /api/domains                   # List all domains
GET  /api/domains/:venture_id       # Get specific domain

# Division discovery (on Domain!)
POST /api/domains/:venture_id/divisions/discover

# Division lifecycle (separate domain)
GET  /api/divisions                  # List all divisions
GET  /api/divisions/:id              # Get specific division
```

---

## Listener / PM Placement Rule

The PM (`on_division_discovered_initiate_division`) lives as a **sibling slice** of `initiate_division/` in the target CMD app, with its own directory, supervisor, and gen_server.

**Why?** PMs are cross-slice integration points, not sub-features of any one desk. Putting them at the top level of `src/` with the `on_*` naming convention means the cross-domain integration points are visible from `ls`.

```
❌ WRONG (nested inside the desk it triggers):
design_division/src/
└── initiate_division/
    ├── ...command, event, handler...
    └── on_division_discovered_initiate_division.erl   # buried

✅ CORRECT (sibling slice):
design_division/src/
├── initiate_division/                                  # desk
│   ├── initiate_division_v1.erl
│   ├── division_initiated_v1.erl
│   ├── maybe_initiate_division.erl
│   └── initiate_division_api.erl
│
└── on_division_discovered_initiate_division/           # PM sibling slice
    ├── on_division_discovered_initiate_division_sup.erl
    └── on_division_discovered_initiate_division.erl   # gen_server: pg:join + dispatch
```

> Earlier guidance (2026-02-08) placed the listener INSIDE the desk. That was reversed 2026-03-12, reinforced 2026-05-24. See [antipatterns/structure.md Demon 18](../skills/antipatterns/structure.md#-demon-18-process-managers-inside-desks) and [PROCESS_MANAGERS.md Location Rule](PROCESS_MANAGERS.md#location-rule).

---

## Process Manager Naming

```
on_{source_event}_{action}_{target}
```

Example: `on_division_discovered_maybe_initiate_division`

- **Source:** `division_discovered` (from discover_divisions)
- **Action:** `maybe_initiate` (policy decision)
- **Target:** `division` (in design_division)

---

## Complete Erlang Example

### discover_division_v1.erl (Command)

```erlang
-module(discover_division_v1).
-export([new/1, validate/1, to_map/1, from_map/1]).

-record(discover_division_v1, {
    venture_id     :: binary(),
    context_name   :: binary(),
    description    :: binary() | undefined,
    discovered_by  :: binary() | undefined
}).

new(#{venture_id := VentureId, context_name := ContextName} = Params) ->
    {ok, #discover_division_v1{
        venture_id = VentureId,
        context_name = ContextName,
        description = maps:get(description, Params, undefined),
        discovered_by = maps:get(discovered_by, Params, undefined)
    }};
new(_) ->
    {error, missing_required_fields}.
```

### division_discovered_v1.erl (Event)

```erlang
-module(division_discovered_v1).
-export([new/1, to_map/1, from_map/1]).

-record(division_discovered_v1, {
    venture_id     :: binary(),
    division_id    :: binary(),
    context_name   :: binary(),
    description    :: binary() | undefined,
    discovered_by  :: binary() | undefined,
    discovered_at  :: non_neg_integer()
}).

new(#{venture_id := VentureId, context_name := ContextName} = Params) ->
    DivisionId = maps:get(division_id, Params, generate_id()),
    {ok, #division_discovered_v1{
        venture_id = VentureId,
        division_id = DivisionId,
        context_name = ContextName,
        description = maps:get(description, Params, undefined),
        discovered_by = maps:get(discovered_by, Params, undefined),
        discovered_at = erlang:system_time(millisecond)
    }}.
```

### Policy (Clean Code)

```erlang
-module(on_division_discovered_maybe_initiate_division).
-export([handle/1]).

handle(FactData) ->
    Params = extract_params(FactData),
    Result = do_initiate(Params),
    log_result(maps:get(division_id, Params), Result),
    Result.

extract_params(FactData) ->
    #{
        division_id => get_field(division_id, FactData),
        venture_id => get_field(venture_id, FactData),
        context_name => get_field(context_name, FactData),
        description => get_field(description, FactData)
    }.

do_initiate(Params) ->
    with_command(initiate_division_v1:new(Params)).

with_command({ok, Cmd}) -> dispatch(maybe_initiate_division:dispatch(Cmd));
with_command({error, _} = E) -> E.

dispatch({ok, _, _}) -> ok;
dispatch({error, _} = E) -> E.
```

---

## Key Takeaways

1. **Parent discovers, child initiates** - Clear separation of concerns
2. **Events belong to aggregate that makes decision** - `division_discovered` in domain stream
3. **Listeners live in desk they trigger** - Vertical slicing
4. **No auto-creation** - Explicit decisions about what exists
5. **Policy contains "maybe"** - Conditional logic is explicit

---

## Training Note

This example can be used for fine-tuning to teach:
- Parent-child aggregate relationships
- Event stream ownership
- Vertical slice organization
- Listener placement rules
- Process manager naming conventions

*Date: 2026-02-08*
*Origin: Hecate Walking Skeleton implementation*
