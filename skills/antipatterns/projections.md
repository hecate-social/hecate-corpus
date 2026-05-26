---
title: "ANTIPATTERNS: Projections & Read Models"
layer: skill
audience: [agent, human]
stage: stable
---

# ANTIPATTERNS: Projections & Read Models

*Demons about projections, read model management, and PRJ/QRY separation.*

[Back to Index](INDEX.md)

---

## 🔥 Read-Time Status Enrichment

**Date:** 2026-02-10
**Origin:** hecate-daemon domain/division status handling

### The Antipattern

Computing `status_label` at query time instead of storing it in the read model at projection write time.

**Symptoms:**
```erlang
%% BAD: Query module enriches at read time
list(Opts) ->
    {ok, Rows} = store:query("SELECT * FROM domains"),
    [enrich_status(row_to_map(R)) || R <- Rows].

enrich_status(#{status := Status} = Row) ->
    Label = evoq_bit_flags:to_string(Status, venture_aggregate:flag_map()),
    Row#{status_label => Label}.
```

**Related violations:**
- Magic numbers: `Status = 3` instead of `evoq_bit_flags:set_all(0, [?VENTURE_INITIATED, ?VENTURE_DNA_ACTIVE])`
- Duplicated flags: `-define(ARCHIVED, 32)` redefined in projections instead of using shared `.hrl`
- Query modules importing aggregate internals (`venture_aggregate:flag_map()`)
- Binary key mismatch: Projections match `#{venture_id := ...}` but events arrive with `<<"venture_id">>` keys

### The Rule

> **Read models store DENORMALIZED data. Compute everything at write time (projection), never at read time (query).**

### The Correct Pattern

**1. Extract flag macros to `.hrl` header in CMD app:**
```erlang
%% apps/setup_venture/include/venture_status.hrl
-define(VENTURE_INITIATED,   1).
-define(VENTURE_DNA_ACTIVE,  2).
-define(VENTURE_ARCHIVED,   32).

-define(VENTURE_FLAG_MAP, #{
    0                    => <<"New">>,
    ?VENTURE_INITIATED   => <<"Initiated">>,
    ?VENTURE_DNA_ACTIVE  => <<"Discovering">>,
    ?VENTURE_ARCHIVED    => <<"Archived">>
}).
```

**2. Projection computes and stores `status_label` at write time:**
```erlang
-include_lib("setup_venture/include/venture_status.hrl").

project(Event) ->
    VentureId = get(venture_id, Event),
    Status = evoq_bit_flags:set_all(0, [?VENTURE_INITIATED, ?VENTURE_DNA_ACTIVE]),
    Label = evoq_bit_flags:to_string(Status, ?VENTURE_FLAG_MAP),
    store:execute(
        "INSERT INTO domains (venture_id, status, status_label) VALUES (?1, ?2, ?3)",
        [VentureId, Status, Label]).
```

**3. Query module reads `status_label` directly — no enrichment:**
```erlang
list(Opts) ->
    {ok, Rows} = store:query("SELECT venture_id, status, status_label FROM domains"),
    [row_to_map(R) || R <- Rows].
%% NO enrich_status function at all
```

**4. Handle binary keys from events (events arrive with binary keys from evoq/ReckonDB):**
```erlang
get(Key, Map) when is_atom(Key) ->
    case maps:find(Key, Map) of
        {ok, V} -> V;
        error -> maps:get(atom_to_binary(Key, utf8), Map, undefined)
    end.
```

### Why It Matters

- **CPU waste**: `enrich_status` runs `evoq_bit_flags:to_string/2` on every row, every query, every request
- **Coupling**: Query modules (PRJ app) depend on aggregate internals (CMD app's `flag_map()`)
- **Inconsistency**: API handlers compute their own labels with hardcoded magic numbers
- **Fragility**: If `flag_map()` changes, all cached/stored data still shows old labels until re-queried
- **CQRS violation**: Read models should be pre-computed and ready to serve — no computation at query time

### The Lesson

> **Projections exist to do the heavy lifting. Queries exist to be dumb and fast.**

---

## 🔥 Demon 22: Manual Event Emission from API Handlers

**Date exorcised:** 2026-02-13
**Where it appeared:** All 17 storm desk API handlers in `guide_venture_lifecycle`
**Cost:** Projections never received events — read models stayed empty

### The Lie

"After dispatch, the API handler should call emitters to broadcast events."

### What Happened

API handlers manually called pg and mesh emitters after dispatching:

```erlang
%% post_event_sticky_api.erl — WRONG
case maybe_post_event_sticky:dispatch(Cmd) of
    {ok, _Version, Events} ->
        lists:foreach(fun(E) ->
            event_sticky_posted_v1_to_pg:emit(E),    %% Manual call
            event_sticky_posted_v1_to_mesh:emit(E)    %% Manual call
        end, Events),
        hecate_api_utils:json_reply(201, Body, Req);
```

The pg emitter had a simple `emit/1` function that broadcast to a pg group:

```erlang
%% event_sticky_posted_v1_to_pg.erl — WRONG
emit(Event) ->
    Members = pg:get_members(pg, event_sticky_posted_v1),
    lists:foreach(fun(Pid) -> Pid ! {event_sticky_posted_v1, Event} end, Members),
    ok.
```

Projections joined pg groups and waited for messages:

```erlang
%% on_event_sticky_posted_v1_from_pg_project_to_sqlite_event_stickies.erl — WRONG
init([]) ->
    ok = pg:join(pg, event_sticky_posted_v1, self()),
    {ok, #{}}.

handle_info({event_sticky_posted_v1, Event}, State) ->
    event_sticky_posted_v1_to_sqlite_event_stickies:project(Event),
    {noreply, State}.
```

### Why It's Wrong

1. **Fragile coupling** — Every API handler must know which emitters to call for which events. Forget one? Silent data loss.
2. **Wrong responsibility** — The API handler's job is request/response. Event delivery is infrastructure.
3. **No replay** — If an emitter wasn't running when the event was stored, the event is lost forever. No catch-up.
4. **Duplicates on retry** — If the HTTP request is retried, the event is emitted twice (even though evoq deduplicates the command).
5. **Testing burden** — Every API handler test must verify emit calls in addition to dispatch.

### The Truth

**Emitters are projections.** They subscribe to the event store via evoq and react autonomously. The API handler dispatches the command and returns. Period.

ReckonDB has a built-in subscription mechanism:
1. Subscriber calls `reckon_evoq_adapter:subscribe(StoreId, event_type, EventType, Name, #{subscriber_pid => self()})`
2. ReckonDB registers a **Khepri trigger** filtered by event type
3. When events are appended, the trigger fires
4. Subscriber receives `{events, [Event]}` messages automatically

### The Correct Pattern

**API handler — dispatch and return:**
```erlang
%% post_event_sticky_api.erl — CORRECT
case maybe_post_event_sticky:dispatch(Cmd) of
    {ok, _Version, Events} ->
        %% Return response. DONE. No emit calls.
        hecate_api_utils:json_reply(201, Body, Req);
    {error, Reason} ->
        hecate_api_utils:json_error(422, Reason, Req)
end.
```

**Emitter — subscribes via evoq at startup:**
```erlang
%% event_sticky_posted_v1_to_pg.erl — CORRECT
-behaviour(gen_server).

init([]) ->
    {ok, _} = reckon_evoq_adapter:subscribe(
        dev_studio_store,
        event_type,
        <<"event_sticky_posted_v1">>,
        <<"event_sticky_posted_v1_to_pg">>,
        #{subscriber_pid => self()}
    ),
    {ok, #{}}.

handle_info({events, Events}, State) ->
    lists:foreach(fun(Event) ->
        pg:send(pg, event_sticky_posted_v1, {event_sticky_posted_v1, Event})
    end, Events),
    {noreply, State}.
```

**Projection — subscribes via evoq OR listens on pg:**
```erlang
%% Direct evoq subscription (same division):
init([]) ->
    {ok, _} = reckon_evoq_adapter:subscribe(
        dev_studio_store, event_type,
        <<"event_sticky_posted_v1">>,
        <<"sticky_posted_to_event_stickies">>,
        #{subscriber_pid => self()}
    ),
    {ok, #{}}.

%% OR pg listener (inter-division):
init([]) ->
    ok = pg:join(pg, event_sticky_posted_v1, self()),
    {ok, #{}}.
```

### The Flow

```
WRONG:  API handler -> manual emit -> pg -> projection
RIGHT:  ReckonDB -> evoq subscription -> emitter -> pg -> listener
        ReckonDB -> evoq subscription -> projection (direct)
```

### Prevention

- API handlers must NEVER import or call `*_to_pg` or `*_to_mesh` modules
- Emitters must be `gen_server`s that subscribe in `init/1`
- If an emitter has an `emit/1` export (not `start_link/0`), it's wrong

### The Lesson

> **The API handler dispatches commands. The event store delivers events. These are separate concerns.**
> **Emitters subscribe to the store — they are NOT called by application code.**
> **See [EVENT_SUBSCRIPTION_FLOW.md](../../philosophy/EVENT_SUBSCRIPTION_FLOW.md) for the canonical pattern.**

---

## 🔥🔥🔥 Inline Projections After Command Dispatch

**Date:** 2026-03-02
**Origin:** macula-realm franchise storefront LiveViews

### The Antipattern

Writing to the read model (Repo.insert, Repo.update_all) immediately after dispatching an evoq command inside a LiveView or handler.

**Example (WRONG):**
```elixir
case :evoq_router.dispatch(cmd) do
  {:ok, _version, _events} ->
    # BAD: LiveView does the projection's job
    Repo.insert(%RealmLicense{license_id: id, status: 1, sold_at: now})
    {:noreply, put_flash(socket, :info, "License sold.")}
end
```

### Why It's Wrong

1. **Duplicates projection logic** — Same write exists in the projection AND the LiveView
2. **Bypasses the event stream** — If you replay events, the LiveView writes are lost
3. **Couples UI to persistence** — LiveViews should only dispatch commands and query read models
4. **Breaks CQRS** — The whole point is separating writes (commands → events) from reads (projections → read models)
5. **Untestable** — You can't test the projection independently if the LiveView does the write

### The Rule

> **LiveViews dispatch commands. Projections update read models. NEVER both.**
>
> After dispatch succeeds, allow the projection to process (brief sleep if needed), then reload from the read model.

### The Correct Pattern

```elixir
case :evoq_router.dispatch(cmd) do
  {:ok, _version, _events} ->
    # Projection gen_server handles the read model update
    Process.sleep(50)
    {:noreply, socket |> put_flash(:info, "License sold.") |> reload_license()}
end
```

The projection gen_server subscribes to events via `evoq_subscriptions:subscribe/5` and writes to the read model autonomously.

---

## 🔥🔥🔥 Consolidating PRJ and QRY Into One Department

**Date:** 2026-03-02
**Origin:** macula-realm franchise storefront architecture

### The Antipattern

Treating PRJ (projections) and QRY (queries) as one combined "QRY+PRJ" department.

### Why It's Wrong

1. **Different responsibilities** — PRJ subscribes to events and WRITES to read models. QRY READS from read models.
2. **Different lifecycles** — PRJ is event-driven (reactive). QRY is request-driven (on demand).
3. **Different scaling concerns** — PRJ throughput is event volume. QRY throughput is query volume.
4. **Violates separation of concerns** — Write path and read path must be independent.

### The Rule

> **PRJ and QRY are ALWAYS separate departments.**
>
> | Department | Nature | Direction |
> |-----------|--------|-----------|
> | **PRJ** | Event-driven | Events → Read Model (WRITE) |
> | **QRY** | Request-driven | Read Model → Response (READ) |
>
> Never combine them. Even if they share the same database tables.

### In Practice

```
Division: procure_realm_license
├── CMD: procure_realm_license (commands, aggregates, handlers)
├── PRJ: project_realm_licenses (event → PostgreSQL projections)
└── QRY: query_realm_licenses (Ecto queries on read model)
```

---

## 🔥🔥🔥 Separate Projections Racing on Shared ETS Table

**Date:** 2026-03-07
**Origin:** hecate-daemon project_licenses, project_plugins, project_launcher

### The Antipattern

Multiple separate `evoq_projection` modules each subscribing to different event types but writing to the **same ETS table**. Each projection is a separate gen_server with its own subscription — there is NO ordering guarantee between them.

**Example (WRONG):**
```
project_licenses_sup starts:
  license_initiated_v1_to_catalog   (projection gen_server A)
  license_announced_v1_to_catalog   (projection gen_server B)
  license_published_v1_to_catalog   (projection gen_server C)
```

Each subscribes independently via `evoq_projection`. When events arrive:
1. `license_initiated_v1` — projection A creates ETS entry (INSERT)
2. `license_announced_v1` — projection B looks up entry by license_id (READ-MODIFY-WRITE)
3. `license_published_v1` — projection C looks up entry by license_id (READ-MODIFY-WRITE)

**The race:** Projections B and C can process BEFORE projection A has created the entry. `find_by_license_id/1` returns `not_found`, and the announced/published events are silently dropped. The catalog card never appears.

### Why It Happens

Each `evoq_projection` is a separate gen_server. Even though ReckonDB delivers events in order to each subscriber, **different subscribers are different processes**. The BEAM guarantees message ordering from process A to process B, but NOT ordering across multiple independent receivers.

```
ReckonDB store
  |
  ├─ delivers initiated_v1 to Projection A (gen_server pid 1)
  ├─ delivers announced_v1 to Projection B (gen_server pid 2)
  └─ delivers published_v1 to Projection C (gen_server pid 3)

Pid 2 may process its event BEFORE pid 1 processes its event.
```

### The Rule

> **One ETS table = one projection module.**
>
> If multiple event types write to the same ETS table, they MUST be handled by a SINGLE `evoq_projection` module. The single gen_server's mailbox guarantees sequential processing.

### The Correct Pattern: Merged Projection

```erlang
-module(license_lifecycle_to_catalog).
-behaviour(evoq_projection).
-export([interested_in/0, init/1, project/4]).

interested_in() ->
    [<<"license_initiated_v1">>,
     <<"license_announced_v1">>,
     <<"license_published_v1">>,
     <<"license_amended_v1">>,
     <<"license_retracted_v1">>].

init(_Config) ->
    {ok, RM} = evoq_read_model:new(evoq_read_model_ets, #{name => catalog}),
    {ok, #{}, RM}.

project(#{data := Data} = Event, _Metadata, State, RM) ->
    case get_event_type(Event) of
        <<"license_initiated_v1">>  -> project_initiated(Data, State, RM);
        <<"license_announced_v1">>  -> project_announced(Data, State, RM);
        <<"license_published_v1">>  -> project_published(Data, State, RM);
        <<"license_amended_v1">>    -> project_amended(Data, State, RM);
        <<"license_retracted_v1">>  -> project_retracted(Data, State, RM);
        _                           -> {ok, State, RM}
    end.
```

**One gen_server** processes all 5 event types sequentially. `initiated` is always processed before `announced` because the gen_server's mailbox preserves order.

### Naming Convention

Merged projections that handle multiple events for the same table are named:

```
{domain}_lifecycle_to_{table}.erl
```

Examples:
- `license_lifecycle_to_catalog.erl` — 5 license lifecycle events -> catalog ETS
- `plugin_lifecycle_to_plugins.erl` — 10 plugin lifecycle events -> plugins ETS
- `launcher_lifecycle_to_launcher.erl` — 4 launcher events -> launcher_entries + launcher_groups ETS

### When This Does NOT Apply

- **SQLite projections**: Writes go through a gen_server call to the store. The store serializes writes. Still, if projection B reads from SQLite and the row from projection A hasn't been written yet, the same race exists. Prefer merged projections for SQLite too when events are dependent.
- **Independent tables**: If each projection writes to a DIFFERENT ETS table, separate projections are fine. No shared state = no race.

### Prevention Checklist

Before adding a new projection, verify:
- [ ] Does this write to a table that OTHER projections also write to?
- [ ] Does this READ from a table that ANOTHER projection INSERTS into?
- [ ] If yes to either: MERGE into the existing lifecycle projection for that table.

### The Lesson

> **Separate gen_servers = separate timelines. If your projections share state, they must share a process.**

---

*We burned these demons so you don't have to. Keep the fire going.* 🔥🗝️🔥
