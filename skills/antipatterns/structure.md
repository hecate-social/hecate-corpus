---
title: "ANTIPATTERNS: Structure"
layer: skill
audience: [agent, human]
stage: stable
---

# ANTIPATTERNS: Structure — Code Organization Violations

*Demons about code structure. Vertical slicing, not horizontal layers.*

[Back to Index](INDEX.md)

---

## 🔥 Incomplete Desks / Flat Workers

**Date:** 2026-02-04
**Origin:** hecate-daemon architecture review

### The Antipattern

CMD slices that are missing components or have workers directly supervised by the domain supervisor.

**Symptoms:**
```erlang
%% BAD: Domain sup directly supervises workers
manage_capabilities_sup
├── capability_announced_v1_to_mesh   % Worker — WRONG LEVEL
├── remote_capabilities_listener      % Worker — WRONG LEVEL
└── ...
```

**Missing pieces:**
- No desk supervisor (`*_desk_sup.erl`)
- No responder (`*_responder_v1.erl`) — can't receive HOPEs from mesh
- Emitters exist but float orphaned at domain level

### The Rule

> **Domain supervisors ONLY start desk supervisors + shared infra.**
> **Desk supervisors start all workers for that desk.**

```erlang
%% GOOD: Domain sup → Desk sups → Workers
manage_capabilities_sup
├── manage_capabilities_store         % Shared infra (OK at domain level)
├── announce_capability_desk_sup      % Supervisor
│   ├── announce_capability_responder_v1    % Worker
│   └── capability_announced_v1_to_mesh     % Worker
├── update_capability_desk_sup        % Supervisor
│   └── ...
└── retract_capability_desk_sup       % Supervisor
    └── ...
```

### Complete Desk Checklist

Every CMD desk MUST have:
- [ ] `*_desk_sup.erl` — Desk supervisor
- [ ] `*_v1.erl` — Command record
- [ ] `*_v1.erl` — Event record
- [ ] `maybe_*.erl` — Handler
- [ ] `*_responder_v1.erl` — HOPE → Command (mesh inbound)
- [ ] `*_to_mesh.erl` — Event → FACT emitter (mesh outbound)

Optional:
- [ ] `on_{event}_maybe_*.erl` — Policy/PM for cross-domain

### Why It Matters

Without responders, your domain can emit but not receive. You have a mouth but no ears.

Without desk supervisors, your supervision tree is flat and you lose fault isolation per feature.

See [CARTWHEEL.md](../../philosophy/CARTWHEEL.md) for the complete canonical structure.

---

## ⚠️ Listeners as Separate Desks (REVERSED 2026-05-24)

**Date:** 2026-02-08
**Reversed:** 2026-05-24
**Origin:** Division listener architecture discussion

### Status: REVERSED

This demon was reversed. The current rule (see [Demon 18 below](#-demon-18-process-managers-inside-desks)) is the opposite: **PMs and cross-domain listeners ARE their own slices**, sibling to desks, named `on_{source_event}_{action}_{target}/`. The discoverability of `on_*` directories at the filesystem level was judged more valuable than co-locating the trigger with the desk it serves.

The 2026-02-08 framing — "listener belongs IN desk X" — is no longer canonical. It applied a vertical-slice heuristic to the wrong axis: PMs cut *across* slices, so they get their own slice rather than being nested in one. A listener that does nothing but `pg:join` and forward to a single desk is still legitimate — it just lives in its own `on_*/` slice.

The single remaining caution: don't create an `on_*` slice that wraps a trivial in-process callback the desk could just do itself. Cross-domain integration via pg / mesh = sibling slice. Internal handler chaining within the same domain = stays in the desk.

---

## 🔥 Centralized Listener Supervisors

**Date:** 2026-02-08
**Origin:** hecate-daemon architecture refinement

### The Antipattern

Creating a central supervisor for all listeners across domains.

**Example (WRONG):**
```
apps/hecate_listeners/src/
├── hecate_listeners_sup.erl          # Central supervisor
├── venture_initiated_listener.erl
├── division_discovered_listener.erl
└── capability_announced_listener.erl
```

Or within a domain:
```
apps/design_division/src/
├── design_division_listeners_sup.erl     # Still wrong!
├── listeners/                             # Horizontal directory
│   ├── division_discovered_listener.erl
│   └── ...
```

### The Rule

> **Each PM / cross-domain listener owns its own slice supervisor.** No central listener supervisor, no `listeners/` directory.

### The Correct Structure

PMs are sibling slices to desks under the domain supervisor. Each PM slice owns a single-worker supervisor.

```
apps/design_division/src/
├── initiate_division/                                          # desk
│   ├── initiate_division_desk_sup.erl
│   └── ...
│
├── on_division_discovered_initiate_division/                   # PM sibling slice
│   ├── on_division_discovered_initiate_division_sup.erl
│   └── on_division_discovered_initiate_division.erl
│
└── on_all_desks_implemented_complete_division/                 # another PM sibling slice
    ├── on_all_desks_implemented_complete_division_sup.erl
    └── on_all_desks_implemented_complete_division.erl
```

The domain supervisor starts each PM slice's supervisor alongside the desk supervisors.

### Why It Matters

- **Fault isolation** — PM crash only affects its own slice.
- **Discoverability** — `on_*` directories scream which external events the domain reacts to.
- **No orphans** — Every PM has a clear owner (its slice sup).
- **Vertical slicing** — No horizontal grouping by technical concern.

See [PROCESS_MANAGERS.md](../../philosophy/PROCESS_MANAGERS.md) and [INTEGRATION_TRANSPORTS.md](../../philosophy/INTEGRATION_TRANSPORTS.md) for slice structures.

---

## Demon 14: God Module API Handlers

**Date exorcised:** 2026-02-10
**Where it appeared:** `apps/hecate_api/src/hecate_api_*.erl`
**Cost:** 137-file refactoring to fix

### The Demon

Putting all API endpoints for a domain in a single file with multiple `init/2` clauses:

```erlang
❌ WRONG: God module with 16 init/2 clauses
-module(hecate_api_mentors).
-export([init/2]).

init(Req0, [submit]) -> handle_submit(Req0);
init(Req0, [list_learnings]) -> handle_list_learnings(Req0);
init(Req0, [get_learning]) -> handle_get_learning(Req0);
init(Req0, [validate]) -> handle_validate(Req0);
init(Req0, [reject]) -> handle_reject(Req0);
init(Req0, [endorse]) -> handle_endorse(Req0);
%% ... 10 more clauses, 289 lines total
```

### Why It's Wrong

- **Horizontal grouping** — groups by "all mentors HTTP stuff" instead of by business operation
- **Violates vertical slicing** — the API handler is separated from the command/event/handler it serves
- **Growing forever** — every new endpoint adds to the same file
- **Hard to find** — `handle_validate` could be anything; you must read the whole file
- **Duplicated helpers** — each god module reinvents `dispatch_result/3`, `error_response/3`

### The Correct Pattern

Each desk owns its API handler:

```erlang
✅ CORRECT: Handler lives in its desk
apps/mentor_agents/src/validate_learning/
├── validate_learning_v1.erl
├── learning_validated_v1.erl
├── maybe_validate_learning.erl
└── validate_learning_api.erl    # ~30 lines, single-purpose
```

### The Lesson

> **API handlers are part of the desk, not part of the API app.**
> One endpoint = one `*_api.erl` file in the desk directory.
> Each handler exports `routes/0` — the central aggregator discovers them automatically.
> See Demon #25 for why centralized route files are wrong.

### How This Was Fixed

Replaced 11 god modules (1,700+ lines) with 50 desk-based handlers (~30-50 lines each).
All handlers use `hecate_api_utils` from the `shared` app for response helpers.
Routes standardized under `/api/` prefix.

Reference: `../skills/codegen/erlang/CODEGEN_ERLANG_TEMPLATES.md` → API Handler Templates

---

## 🔥 Demon 18: Process Managers Inside Desks

**Date exorcised:** 2026-02-12 (original direction)
**Reversed:** 2026-03-12 (rationale recorded); reinforced 2026-05-24
**Where it appeared:** Various scaffolds that nested `on_{event}_{action}_{target}.erl` inside the target desk directory.
**Cost:** Cross-domain integration points became invisible at the filesystem level — you couldn't tell which external events a domain reacted to without grepping inside every desk.

### The Lie

"A PM is a policy of the desk it triggers. It belongs inside that desk's directory."

This was the original (2026-02-12) framing. **It was wrong.** Production experience showed PMs are not subordinate to a single desk — they are cross-cutting integration points that belong at the same architectural level as desks themselves.

### Why It's Actually Wrong

1. **PMs are cross-slice.** A PM is an integration point between two domains, not a sub-feature of a single desk. Nesting it hides the cross-cutting nature.
2. **Business process flow becomes invisible.** When you `ls src/`, you cannot tell which external events the domain reacts to. You must grep inside every desk directory to find `on_*` files.
3. **One PM can dispatch to multiple desks.** A cancellation cascade, an evacuation force-settle, a fan-out reprice — these don't map 1:1 to any single desk. Nesting in one desk arbitrarily privileges that desk.
4. **The desk-completeness argument cuts both ways.** Yes, hiding the PM inside the desk makes "the desk owns everything." But that's exactly the cost: you lose visibility into integration points.

### The Rule (Current)

> **A PM is a first-class sibling slice in the target CMD app.** Own directory. Own supervisor. Own gen_server. Named `on_{source_event}_{action}_{target}/`.

```
apps/manage_capabilities/src/
├── announce_capability/                                # desk
│   ├── announce_capability_v1.erl
│   ├── capability_announced_v1.erl
│   ├── maybe_announce_capability.erl
│   └── announce_capability_api.erl
│
└── on_llm_detected_announce_capability/                # PM sibling slice
    ├── on_llm_detected_announce_capability_sup.erl
    └── on_llm_detected_announce_capability.erl
```

### Why Sibling, Not Nested

| Concern | Nested-in-desk | Sibling slice |
|---------|----------------|---------------|
| "What external events does this domain react to?" | Hidden — grep needed | Visible — `ls src/` |
| One PM → multiple desks | Awkward (which desk owns it?) | Natural |
| Supervision granularity | PM crash takes down desk infra | PM crash isolated |
| Cross-cutting nature | Implied subordination | Explicit peer |

### The Test

> "When I `ls src/` in a CMD app, can I immediately see every cross-domain integration point?"
>
> If no — PMs are nested inside desks — pull them out into sibling slices.

### The Lesson

> **PMs are cross-slice. They get their own slice. The `on_*` prefix and the slice directory together are the discoverability anchor for business process flow.**

See [PROCESS_MANAGERS.md](../../philosophy/PROCESS_MANAGERS.md) for the canonical pattern and code template.

---

---

## 🔥 Demon 25: Centralized Route Registration Files

**Date exorcised:** 2026-02-16
**Where it appeared:** 15 `*_routes.erl` files across all hecate apps
**Cost:** 15 centralized files deleted, ~102 handlers updated

### The Demon

One file per OTP app that lists all routes for that app's handlers:

```erlang
❌ WRONG: Centralized route file knows about all handlers
-module(breed_snake_gladiators_routes).
-export([routes/0]).

routes() ->
    [{"/api/arcade/gladiators/stables", initiate_stable_api, []},
     {"/api/arcade/gladiators/stables/:stable_id", get_stable_api, []},
     {"/api/arcade/gladiators/stables/:stable_id/champion/duel", start_champion_duel_api, []},
     {"/api/arcade/gladiators/heroes", heroes_api, []},
     {"/api/arcade/gladiators/heroes/:hero_id", get_hero_api, []},
     {"/api/arcade/gladiators/heroes/:hero_id/duel", start_hero_duel_api, []}].
```

### Why It's Wrong

- **Horizontal grouping** — routes are grouped by app, not by the handler that serves them
- **Every new handler requires editing TWO files** — the handler AND the route file
- **Route file knows too much** — it must import or reference every handler module
- **Merge conflicts** — multiple feature branches touching the same routes file
- **Violates screaming architecture** — a handler's URL path is part of its identity, not a separate concern
- **Same demon as #14** (God Module API Handlers) applied to routing

### The Correct Pattern

Each Cowboy handler exports `routes/0` declaring its own routes:

```erlang
✅ CORRECT: Handler owns its route
-module(initiate_stable_api).
-export([init/2, routes/0]).

routes() ->
    [{"/api/arcade/gladiators/stables", ?MODULE, []}].

init(Req0, State) ->
    %% ...
```

A single central aggregator discovers all handlers via OTP module introspection:

```erlang
✅ CORRECT: Auto-discovery aggregator (the ONLY central file)
-module(hecate_api_routes).
-export([compile/0]).

-define(HECATE_APPS, [hecate_api, guide_venture_lifecycle, ...]).

compile() ->
    cowboy_router:compile([{'_', discover_routes()}]).

discover_routes() ->
    lists:flatmap(fun collect_app_routes/1, ?HECATE_APPS).

collect_app_routes(App) ->
    Mods = app_modules(App),
    Handlers = [M || M <- Mods, M =/= ?MODULE, exports_routes(M)],
    lists:flatmap(fun(M) -> M:routes() end, Handlers).

app_modules(App) ->
    case application:get_key(App, modules) of
        {ok, Mods} -> Mods;
        _ -> []
    end.

exports_routes(Mod) ->
    code:ensure_loaded(Mod),
    erlang:function_exported(Mod, routes, 0).
```

### The Key Insight

Adding a new API endpoint requires touching exactly ONE file — the handler itself. The aggregator discovers it automatically because it exports `routes/0`.

### The Test

> "Can I add a new API endpoint by creating a single file?"
>
> **Yes** → routes/0 auto-discovery is working.
> **No, I also need to edit a routes file** → you have a centralized route file demon.

### The Lesson

> **Route ownership follows handler ownership.**
> The handler IS the route. The handler declares its path.
> The aggregator discovers — it never enumerates.

*Add more demons as we exorcise them.* 🔥🗝️🔥
