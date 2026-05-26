# ANTIPATTERNS: Event Sourcing — Aggregates, Events, Envelopes

*Demons about event sourcing mechanics. Aggregates, event records, envelopes, and stores.*

[Back to Index](ANTIPATTERNS.md)

---

## 🔥 Wrong Aggregate Callback Argument Order

**Date:** 2026-02-09
**Origin:** hecate-daemon division auto-initiation bug

### The Antipattern

Writing aggregate `execute/2` and `apply/2` callbacks with wrong argument order.

**Example (WRONG):**
```erlang
%% WRONG - Payload first, State second
execute(#{command_type := <<"my_command">>} = Payload, State) ->
    do_something(Payload, State).

apply_event(#{event_type := <<"my_event_v1">>} = Event, State) ->
    update_state(Event, State).
```

### The Rule

> **evoq behaviour callbacks expect: State first, then Payload/Event.**

evoq_aggregate.erl calls:
```erlang
Module:execute(AggState, Command#evoq_command.payload)
Module:apply(AccState, Event)
```

### The Correct Implementation

```erlang
-module(my_aggregate).
-behaviour(evoq_aggregate).

%% Behaviour callbacks
-export([init/1, execute/2, apply/2]).

%% init/1 returns {ok, State}
init(_AggregateId) ->
    {ok, initial_state()}.

%% execute/2: State first, Payload second
execute(State, #{command_type := <<"my_command">>} = Payload) ->
    do_something(State, Payload).

%% apply/2: State first, Event second
apply(State, #{event_type := <<"my_event_v1">>} = Event) ->
    update_state(State, Event).
```

### Why This Happens

1. It's natural to write `execute(CommandPayload, State)` — "execute this command on this state"
2. Many examples online show the wrong order
3. Without tests, the bug only appears at runtime

### The Symptom

All commands fail with `{error, unknown_command}` because:
- evoq passes `(State, Payload)`
- Aggregate receives State where it expects a map
- Pattern match `#{command_type := ...}` fails on a record
- Falls through to catch-all: `execute(_State, _Payload) -> {error, unknown_command}`

### Prevention: Always Test Aggregates

```erlang
execute_argument_order_test() ->
    State = my_aggregate:initial_state(),
    Payload = #{command_type => <<"my_command">>, id => <<"test">>},

    %% This test catches wrong argument order immediately
    Result = my_aggregate:execute(State, Payload),
    ?assertMatch({ok, [_]}, Result).
```

### The Lesson

> **Use `-behaviour(evoq_aggregate).`** — The compiler will check callbacks exist.
> **Write aggregate tests before push** — A 10-line test would have caught this bug.

---

## 🔥 Demon 23: Raw #event{} Records Passed to Projections

**Date exorcised:** 2026-02-13
**Where it appeared:** All 45 projection subscribers across `query_venture_lifecycle` and `query_division_alc`
**Cost:** Every projection crashed on boot — read models permanently empty

### The Lie

"The subscriber receives events and passes them to `project/1`. It just works."

### What Happened

ReckonDB emitters send `{events, [#event{}]}` messages where `#event{}` is a **record** (tuple) from `reckon_gater/include/esdb_gater_types.hrl`. The business payload lives inside `#event.data`.

All 45 projection subscribers passed the raw record to `project/1`:

```erlang
%% WRONG — E is an #event{} record (tuple), not a map
handle_info({events, Events}, State) ->
    lists:foreach(fun(E) ->
        case venture_initiated_v1_to_sqlite_ventures:project(E) of
            ok -> ok;
            {error, Reason} -> logger:warning("failed: ~p", [Reason])
        end
    end, Events),
    {noreply, State}.
```

But `project/1` calls `maps:find(Key, Event)` — which crashes with `{badmap, #event{...}}` because `#event{}` is a tuple, not a map.

The gen_server crashes, the supervisor restarts it (subscribing to ReckonDB again), it crashes again on the first event — infinite restart loop until the supervisor hits its max restart intensity and dies.

**Result:** Events stored correctly in ReckonDB, but SQLite read models permanently empty. POST succeeds but GET returns `[]`.

### Why It's Wrong

1. **Records are tuples.** Erlang records compile to tuples. `maps:find/2` on a tuple raises `{badmap, _}`.
2. **Silent failure.** The supervisor restart loop happens in the background — no visible error to the user, just empty query results.
3. **Template propagation.** The subscriber template had this bug, so every subscriber generated from it inherited it.
4. **No tests caught it.** All existing projection tests passed flat maps directly to `project/1`, never testing with actual `#event{}` records.

### The Truth

Projections need a **flat map** with both envelope fields (event_id, event_type, stream_id, version, timestamp) and business data fields merged at top level.

### The Fix

Created `projection_event:to_map/1` in the `shared` app:

```erlang
-module(projection_event).
-include_lib("reckon_gater/include/esdb_gater_types.hrl").
-export([to_map/1]).

to_map(#event{} = E) ->
    Envelope = #{
        event_id => E#event.event_id,
        event_type => E#event.event_type,
        stream_id => E#event.stream_id,
        version => E#event.version,
        metadata => E#event.metadata,
        timestamp => E#event.timestamp,
        epoch_us => E#event.epoch_us
    },
    case E#event.data of
        Data when is_map(Data) -> maps:merge(Data, Envelope);
        _ -> Envelope
    end;
to_map(Map) when is_map(Map) ->
    Map.  %% Already flat — passthrough for backward compat
```

All 45 subscribers updated:
```erlang
%% CORRECT — convert record to flat map first
case venture_initiated_v1_to_sqlite_ventures:project(projection_event:to_map(E)) of
```

### Prevention

**Always test with the actual input type.** Write at least one test per projection that constructs an `#event{}` record and passes it through the full pipeline:

```erlang
raw_record_crashes_project() ->
    EventRecord = #event{
        event_id = <<"evt-1">>, event_type = <<"venture_initiated_v1">>,
        stream_id = <<"venture_aggregate-v1">>, version = 0,
        data = #{venture_id => <<"v-1">>, name => <<"Test">>},
        metadata = #{}, timestamp = 1000, epoch_us = 1000000
    },
    ?assertError({badmap, _}, my_projection:project(EventRecord)).

record_via_to_map_projects() ->
    EventRecord = ...,  %% same as above
    FlatMap = projection_event:to_map(EventRecord),
    ok = my_projection:project(FlatMap).
```

### The Lesson

> **Records are tuples. Maps are maps. Never assume one works where the other is expected.**
> **If all your tests pass flat maps but production receives records, your tests are lying.**
> **Write a failing test FIRST — then fix the code.**

---

## 🔥🔥🔥 Evoq Without ReckonDB ("In-Memory" Event Sourcing)

**Date:** 2026-03-02
**Origin:** macula-realm initial plan said "In-Memory Evoq"

### The Antipattern

Using evoq CQRS framework without ReckonDB as the event store, claiming events can be stored "in memory."

### Why It's Wrong

1. **No event persistence** — Without an event store, events are lost on restart. There's nothing to replay.
2. **No projections** — Projections subscribe to the event store. No store = no subscriptions = no projections.
3. **No audit trail** — The entire point of event sourcing is an immutable log of facts. "In-memory" means no log.
4. **Defeats CQRS** — Without stored events flowing to projections, you're back to CRUD with extra steps.
5. **No snapshots** — Aggregate state reconstruction requires replaying events from the store.

### The Rule

> **Evoq REQUIRES ReckonDB. There is no "in-memory" mode for production.**
>
> The stack is: `reckon_db` (event store) + `reckon_evoq` (adapter) + `evoq` (framework).
> All three are required. Remove any one and the system collapses.

### Configuration (MANDATORY)

```erlang
%% sys.config / config.exs
{evoq, [
    {event_store_adapter, reckon_evoq_adapter},
    {subscription_adapter, reckon_evoq_adapter}
]}.
```

### Store Creation (MANDATORY at app startup)

```erlang
Config = #store_config{
    store_id = my_domain_store,
    data_dir = "/path/to/store",
    mode = single
},
{ok, _Pid} = reckon_db_sup:start_store(Config).
```

Without both of these, evoq will crash on first dispatch.

**Alternative (declarative) — list stores in the `reckon_db` app env; `reckon_db_sup` starts them at boot, no explicit `start_store/1` call:**

```erlang
{reckon_db, [
    {stores, [{my_domain_store, [{mode, single}, {data_dir, "/path/to/store"}]}]}
]}.
```

⚠️ The default `data_dir` is **`/var/lib/reckon_db/<store_id>`** — not writable in dev/test. Always set an explicit, writable `data_dir` (in prod, the persistent volume), or the store dies at boot with `{badmatch, {error, enoent}}` from `reckon_db_store:init/1`.

---

## 🔥 Binary Keys in Event `to_map/1` Functions

**Date:** 2026-03-05
**Origin:** "Publish from URL" never projects to SQLite — 11 event modules affected

### The Antipattern

Using binary keys in `to_map/1` return maps:

```erlang
%% WRONG — binary keys
to_map(#license_initiated_v1{} = E) ->
    #{
        <<"event_type">> => <<"license_initiated_v1">>,
        <<"license_id">> => E#license_initiated_v1.license_id
    }.
```

### Why It's Wrong

`evoq_aggregate:append_events` extracts the event type with an **atom** key lookup:

```erlang
EventType = maps:get(event_type, Event, undefined),
```

In Erlang, `event_type` (atom) and `<<"event_type">>` (binary) are **completely different map keys**. The atom lookup on a binary-keyed map returns `undefined`. The event gets stored in ReckonDB with `event_type = undefined`, so Khepri trigger filters (`#event{event_type = <<"license_initiated_v1">>}`) never match. Projections never receive events. Zero errors logged anywhere.

### The Correct Pattern

```erlang
%% CORRECT — atom keys
to_map(#license_initiated_v1{} = E) ->
    #{
        event_type => <<"license_initiated_v1">>,
        license_id => E#license_initiated_v1.license_id
    }.
```

**Rule:** `to_map/1` MUST use atom keys. The values remain binaries, but the keys must be atoms.

### How We Caught It

Settings events (atom keys) worked. License events (binary keys) didn't. Comparing `settings_initiated_v1.erl` to `license_initiated_v1.erl` revealed the difference. The bug was completely silent — dispatch returned `{ok, 2, Events}`, no errors, no warnings.

### Detection Checklist

If `POST` succeeds but `GET` returns empty:
1. Check `to_map/1` key types — atom vs binary
2. Check stored events in ReckonDB — is `event_type` populated or `undefined`?
3. Check Khepri trigger filter — does the pattern match the stored event?

---

## 🔥 Flattening Event Envelopes Into Business Data

**Date:** 2026-03-06
**Origin:** hecate-daemon — `projection_event:to_map/1` caused silent field collisions

### The Antipattern

Creating a "helper" that merges the event envelope (`#evoq_event{}` fields like `event_id`, `stream_id`, `version`, `timestamp`) with the business payload (`data` map) into one flat map.

**Example (WRONG):**
```erlang
%% projection_event.erl — THE BUG
to_map(#evoq_event{} = E) ->
    Envelope = #{event_id => ..., version => ..., timestamp => ...},
    case E#evoq_event.data of
        Data when is_map(Data) -> maps:merge(Data, Envelope);  %% COLLISION!
        _ -> Envelope
    end.

%% Every listener did this:
handle_info({events, Events}, State) ->
    lists:foreach(fun(E) ->
        Map = projection_event:to_map(E),  %% Flatten envelope + data
        my_projection:project(Map)
    end, Events).
```

### Why It's Dangerous

**Silent data corruption via field name collisions.** The envelope and business data share common field names:

| Envelope Field | Business Field | Result |
|---------------|---------------|--------|
| `version` (integer: stream position 0, 1, 2...) | `version` (string: `"0.1.0"`) | Catalog stores `0` instead of `"0.1.0"` |
| `timestamp` (integer: envelope creation time) | `timestamp` (any: business timestamp) | Wrong timestamp in read model |
| `metadata` (map: correlation/causation IDs) | `metadata` (map: business metadata) | Lost business metadata |

Because `maps:merge(Data, Envelope)` gives envelope fields priority, the business value is **silently replaced**. No error, no warning — just wrong data in the read model.

### The Real Bug We Found

`license_initiated_v1` events carry a business `version` field (e.g., `"0.1.0"`). After flattening, the projection read `version` and got the evoq stream version (integer `0`) instead. The plugin catalog stored stream positions as app versions.

### The Fix

**Never flatten.** Use evoq's documented pattern — pattern match `#evoq_event{data = Data}` and pass `Data` directly:

```erlang
-include_lib("evoq/include/evoq_types.hrl").

handle_info({events, Events}, State) ->
    lists:foreach(fun(#evoq_event{data = Data}) ->
        my_projection:project(Data)
    end, Events),
    {noreply, State}.
```

This is exactly what evoq's own documentation shows in `evoq_subscriptions.erl`:

```erlang
%% From evoq source (line 31):
%% handle_info({events, Events}, State) ->
%%     lists:foreach(fun(#evoq_event{data = Data}) ->
%%         project(Data)
%%     end, Events),
```

If a projection or PM needs envelope fields (rare), extract them explicitly:

```erlang
fun(#evoq_event{data = Data, stream_id = StreamId}) ->
    my_projection:project(Data, StreamId)
end
```

### Rules

1. **Business data lives in `#evoq_event.data`** — extract it, pass it, done
2. **Never merge envelope with data** — field collisions are silent and destructive
3. **No "convenience" flattening helpers** — they hide the two-level structure that exists for a reason
4. **Include `evoq_types.hrl`** to pattern match the record — don't treat events as opaque terms

---

---

## 🔥 Reading Business Fields Directly from Event Envelope

**Date:** 2026-03-08
**Origin:** hecate-daemon — all license domain PMs silently got `undefined` for every business field

### The Antipattern

In `evoq_event_handler` callbacks, reading business fields directly from the `Event` argument without extracting `data` first.

**Example (WRONG):**
```erlang
handle_event(_EventType, Event, _Metadata, State) ->
    LicenseId = hecate_api_utils:get_field(license_id, Event),  %% WRONG!
    FeeCents = hecate_api_utils:get_field(fee_cents, Event),    %% WRONG!
    ...
```

### Why It's Dangerous

The `Event` passed to `handle_event/4` is the **full evoq event envelope** — a map with `event_type`, `event_id`, `stream_id`, `version`, `data`, `metadata`, `tags`, `timestamp`, `epoch_us`. Business fields live inside the `data` sub-map.

`get_field(license_id, Event)` looks for `license_id` at the top level of the envelope. It's not there. It returns `undefined`. **No error, no warning.**

In our case:
- `license_id` = `undefined` → PM tried to grant a license with `aggregate_id = undefined` → dispatched to a fresh aggregate (status=0) → `{error, license_not_initiated}`
- `fee_cents` = `undefined` → `is_free(undefined) = true` → PM wrongly treated EVERY offering as free
- The error was logged but swallowed — the install flow silently stalled at "Downloading"

### The Fix

**The event IS the full envelope. Extract `data` when you need business fields:**

```erlang
handle_event(_EventType, Event, _Metadata, State) ->
    Data = maps:get(data, Event),
    LicenseId = hecate_api_utils:get_field(license_id, Data),
    FeeCents = hecate_api_utils:get_field(fee_cents, Data),
    ...
```

If you need envelope fields, use them directly from `Event`:
```erlang
handle_event(_EventType, Event, _Metadata, State) ->
    Data = maps:get(data, Event),
    StreamId = maps:get(stream_id, Event),
    Version = maps:get(version, Event),
    %% Business fields from Data, envelope fields from Event
    ...
```

### Correct Examples Already in Codebase

The plugin domain PMs were already correct:
```erlang
%% on_plugin_removed_deprovision_container.erl
handle_event(_EventType, Event, _Metadata, State) ->
    Data = maps:get(data, Event),
    PluginId = get_value(plugin_id, Data),
    ...
```

Projections (evoq_projection behaviour) receive events differently — `project(#{data := Data} = Event, ...)` — and pattern-match `data` in the function head.

### How We Caught It

Free license install flow: POST /api/appstore/install returned 201, but the plugin never installed. PM logs showed `[PM] Failed to grant license undefined: license_not_initiated`. The `undefined` was the giveaway — `license_id` wasn't being extracted.

### Rules

1. **`handle_event/4` receives the full envelope** — business data is in `maps:get(data, Event)`
2. **Never read business fields from the envelope directly** — they will always be `undefined`
3. **Envelope fields from `Event`, business fields from `Data`** — no flattening, no fallback hacks
4. **When writing a new PM, copy the pattern from an existing working PM** — don't reinvent

---

## 🔥🔥🔥 Demon 41: Reading from Read Models During Event Flow — THE Cardinal Sin

**Date:** 2026-03-08
**Origin:** hecate-daemon — `on_license_granted_install_plugin` PM read from `project_licenses_store` to build the `install_plugin_v1` command. Race condition caused "stuck at downloading" — projection hadn't caught up when PM fired.
**Severity:** THE most important antipattern in event sourcing. If you learn only one rule, learn this one.

### The Antipattern

A Process Manager, Event Handler, or Projection reading from a read model (ETS, SQLite, external API) during event processing to obtain data it needs for downstream actions.

**Example (WRONG):**
```erlang
%% PM receives license_granted_v1, needs plugin details for install command
handle_event(_EventType, Event, _Metadata, State) ->
    Data = maps:get(data, Event),
    LicenseId = hecate_api_utils:get_field(license_id, Data),
    %% WRONG! Reading from read model during event flow!
    case project_licenses_store:get_license(LicenseId) of
        {ok, License} ->
            OciImage = maps:get(oci_image, License),     %% from read model
            PluginId = maps:get(plugin_id, License),     %% from read model
            dispatch_install(PluginId, OciImage, ...);
        {error, not_found} ->
            %% "Projection may not have caught up yet"  ← THIS COMMENT IS THE SMELL
            logger:warning("not found, skipping")
    end,
    {ok, State}.
```

### Why It's THE Cardinal Sin

1. **Race condition by design.** The PM and the projection are separate processes subscribing to the same event store. There is NO ordering guarantee. When the PM fires on `license_granted_v1`, the projection may not have processed `license_initiated_v1` yet. The read model is empty. The PM silently skips. The plugin never installs.

2. **Tight coupling.** The PM now depends on a specific projection in a different department (PRJ). If the projection changes its schema, the PM breaks. Event flow components should depend ONLY on the event stream, never on read models.

3. **Silent failure.** The `{error, not_found}` branch logs a warning and moves on. No retry, no error, no crash. The user sees "Downloading..." forever. The system looks healthy but is silently broken.

4. **Architectural violation.** Read models exist to serve QUERIES (the Q in CQRS). They are optimized for reading, eventually consistent, and may be rebuilt at any time. Using them as a data source during the COMMAND flow (C in CQRS) breaks the fundamental separation.

### The Root Cause

The event payload was too poor. `license_granted_v1` only carried `license_id` and `grant_reason`. The PM needed `plugin_id`, `oci_image`, `plugin_name`, `version`, `icon`, `group_name` to build the install command. Because the event didn't carry this data, the PM reached for the read model.

### The Fix: Enrich the Event

**The handler has access to the aggregate state.** Echo the fields downstream consumers need into the event:

```erlang
%% maybe_grant_license.erl — receives State, echoes install fields
handle(Cmd, #license_state{} = State) ->
    Event = license_granted_v1:new(#{
        license_id   => grant_license_v1:get_license_id(Cmd),
        grant_reason => grant_license_v1:get_grant_reason(Cmd),
        %% Echoed from aggregate state — PM needs these
        plugin_id    => State#license_state.plugin_id,
        plugin_name  => State#license_state.plugin_name,
        oci_image    => State#license_state.oci_image,
        version      => State#license_state.version,
        icon         => State#license_state.icon,
        group_name   => State#license_state.group_name
    }),
    {ok, [Event]}.
```

**The PM reads ONLY from the event — zero external dependencies:**

```erlang
%% on_license_granted_install_plugin.erl — self-contained
handle_event(_EventType, Event, _Metadata, State) ->
    Data = maps:get(data, Event),
    PluginId = hecate_api_utils:get_field(plugin_id, Data),
    OciImage = hecate_api_utils:get_field(oci_image, Data),
    %% Everything comes from the event. No read model. No race condition.
    CmdParams = #{
        plugin_id => PluginId,
        oci_image => OciImage,
        ...
    },
    dispatch_install(CmdParams),
    {ok, State}.
```

### The Rule

> **During event flow, components MUST NEVER read from read models.**
>
> The event MUST carry a payload rich enough for ALL downstream processing.
>
> If the event is too poor, ENRICH IT. Fix the command/handler to echo aggregate state into the event.
>
> Read models may ONLY be consulted when constructing a Command payload — in API handlers, CLI handlers, or user-facing code. NEVER in event handlers, projections, or process managers.

### Where Read Model Access IS Allowed

| Component | Read Model Access | Why |
|-----------|-------------------|-----|
| **API handler** | YES | Constructing command payload from user request + read model |
| **CLI handler** | YES | Same — user-facing command construction |
| **Convenience API** (e.g., `install_offering_api`) | YES | Orchestrating multiple commands for UX |
| **Event handler / PM** | **NEVER** | Must use event data only |
| **Projection** | **NEVER** (reads own state only) | Writes to read model, never reads others |
| **Aggregate execute** | **NEVER** | Uses aggregate state only |

### The Smell

If you see ANY of these in an event handler or PM, it's a Demon #41 violation:

```erlang
project_licenses_store:get_license(...)     %% WRONG
project_plugins_store:get(...)              %% WRONG
evoq_read_model:get(...)                    %% WRONG (unless in a projection reading its OWN state)
esqlite3:q(...)                             %% WRONG
ets:lookup(some_other_table, ...)           %% WRONG
```

### The Comment That Reveals the Bug

If you ever write this comment:

```erlang
%% Projection may not have caught up yet
```

**STOP.** You are about to commit the cardinal sin. The projection WILL NOT have caught up — that's the nature of eventual consistency. The fix is not to "hope it catches up" or add retries. The fix is to make the event carry the data.

### The Lesson

> **Events are facts. Facts must be self-contained.**
> **A process manager that reads from a read model is a process manager that will fail.**
> **If your event is too poor, your event is wrong. Fix the event.**

### The Structural Cure: Command Pipelines

Enriching the source event is the right fix when the data lives in the source aggregate. But it doesn't address the case where the consuming domain needs data from a **third** domain entirely (e.g., the egress saga needs both session state from `entry2exit` and permit status from `pricing`). The source aggregate has neither.

The general structural cure is the **command pipeline**: all external reads happen in named pipeline steps BEFORE the aggregate sees the command. The aggregate stays a pure function of `(State, EnrichedCommand)`. Any cross-domain read — whether to enrich a HOPE-driven command or to enrich a PM-dispatched command — happens in the pipeline, at command-preparation time, never inside event handling.

See [philosophy/COMMAND_PIPELINES.md](../philosophy/COMMAND_PIPELINES.md) for the pattern and [skills/codegen/erlang/CODEGEN_ERLANG_PIPELINES.md](codegen/erlang/CODEGEN_ERLANG_PIPELINES.md) for templates.

The two cures stack: **enrich the event at the source** when data is local, **enrich the command via pipeline** when data is cross-domain. Together they keep aggregates pure and PMs free of read-model lookups.

---

## 🔥 Aggregate `apply/2` Sees Two Event Shapes (and `execute/2` Must Not Pre-Wrap `data`)

**Date:** 2026-05-21
**Origin:** reckon-portal blog Division (dogfooding ReckonDB). Verified against **evoq 1.15.0** — the `evoq_aggregate` reference in `codegen/erlang/EVOQ_BEHAVIOURS.md` is dated to ~1.4, and this corner behaves as below in 1.15.

### The Antipattern

Returning events from `execute/2` already wrapped in a `data` key, AND writing `apply/2` to read `maps:get(data, Event)` unconditionally.

```erlang
%% WRONG — execute pre-wraps under `data`
execute(_S, #{command_type := publish_post} = P) ->
    {ok, [#{event_type => <<"post_published_v1">>,
            data => #{title => maps:get(title, P)}}]}.   %% the trap

%% WRONG — apply assumes a `data` key is always present
apply(S, #{event_type := <<"post_published_v1">>, data := D}) -> ...
```

### Why It's Wrong

`evoq_aggregate:append_events/5` builds the stored envelope itself:

```erlang
Data = maps:without([event_type], Event),   %% everything-but-event_type IS the data
Wrapped = #{event_type => EventType, data => Data, metadata => BaseMeta}.
```

So if `execute/2` returns `#{event_type, data => #{...}}`, the stored data becomes `#{data => #{...}}` — **double-nested**. The projection reads `data.data.title` and finds nothing. Silent, like every envelope demon.

Worse, there are **two delivery shapes for the same `apply/2`**:

| When | What `apply/2` receives |
|------|--------------------------|
| In-memory, right after `execute/2` | the **raw** `execute` output — business fields **inline**, NO `data` key (evoq folds the unwrapped events into state *before* appending) |
| On reload / replay from the store | the **wrapped** envelope — business fields under `data` |

An `apply/2` that matches `data := D` works on reload but silently no-ops in-memory (falls through to the catch-all), so status flags never get set on the live aggregate — and the bug only shows on the **second** command, never the first.

### The Fix

1. **`execute/2` returns business fields INLINE** (atom keys — see "Binary Keys" above) alongside `event_type`; let evoq do the wrapping:

```erlang
execute(_S, #{command_type := publish_post} = P) ->
    {ok, [#{event_type => <<"post_published_v1">>,
            title => maps:get(title, P),
            body  => maps:get(body, P)}]}.
```

2. **`apply/2` normalises both shapes** (the same `maps:get(data, Event, Event)` idiom evoq's own handlers use):

```erlang
apply(State, #{event_type := Type} = Event) ->
    apply_event(Type, event_data(Event), State).

event_data(#{data := D}) when is_map(D) -> D;     %% reloaded envelope
event_data(Event) ->                              %% raw in-memory event
    maps:without([event_type, version, metadata, event_id,
                  stream_id, timestamp, epoch_us, tags], Event).
```

### The Rule

> **`execute/2` emits inline fields; evoq owns the `data` wrapper.**
> **`apply/2` reads fields tolerantly — inline in-memory, under `data` on reload.**
> **Test the SECOND command, not just the first** — the in-memory fold only bites on replay-of-self (publish → "already published").

---

## 🔥 Demon 49: Discarding `evoq_dispatcher:dispatch/2`'s Return Value

**Date exorcised:** 2026-05-26
**Where it appeared:** `hecate-services/hecate-parksim/apps/simulate_visit/src/simulate_visit.erl` — three call sites, `_ = maybe_X:dispatch(...)`
**Cost:** ~2 hours of "events vanish silently" debugging across multiple boot probes

### The Lie

"Dispatch is fire-and-forget. Errors will log themselves."

### What Happened

`evoq_dispatcher:dispatch/2` returns `{ok, Version, Events}` on success
and `{error, Reason}` on failure. The error channel is the ONLY channel
for validation rejections (stream-id format, payload shape) and
aggregate-side `{error, _}` returns from `execute/2`.

Discarding the result with `_ = dispatch(...)` throws away both
channels. Symptom: the caller appears to succeed (no crash, no log
line, no exception), but events never land in the store. Downstream
projections are silent because nothing was written to react to.

In the worked example, `simulate_visit:enter/4` minted UUID-v4 session
ids (`9D9D16B6-F56E-...`) that failed the reckon-db stream-id validator.
Each dispatch returned
`{error, {invalid_stream_id, malformed_user_id, ...}}` — and the
underscore discarded every one. A 12-second e2e probe produced zero
events; the actual cause was 100% caller-side error swallow.

```erlang
%% WRONG — the only error channel goes into the void
enter(SessionId, LotId, Plate, CardId) ->
    _ = maybe_initiate_parking_session:dispatch(#{
        session_id => SessionId,
        lot_id     => LotId,
        plate      => Plate,
        card_id    => CardId,
        entered_at => simulate_clock:now_iso8601()
    }),
    ok.
```

### The Fix

Pattern-match the dispatch result. The right choice depends on the
caller:

```erlang
%% RIGHT — caller decides what to do with errors
case maybe_initiate_X:dispatch(Cmd) of
    {ok, _Version, _Events} -> ok;
    {error, Reason}         -> logger:warning("dispatch failed: ~p", [Reason])
end.

%% ALSO RIGHT — let the process die on dispatch error
%% Default for short-lived spawned workers (per-visit, per-request).
%% proc_lib's crash handler logs to error_logger, making the failure
%% visible without extra plumbing.
{ok, _V, _Events} = maybe_initiate_X:dispatch(Cmd).
```

### The Rule

> **The dispatch result IS the only error channel. Never `_ = dispatch(...)`.**
> **Pattern-match. Log or crash on `{error, _}`.**

Silent dispatch failures are the worst kind: the caller thinks
everything is fine while the store stays empty. Eventual-consistency
flows that depend on the event then look like they have a "race
condition" or a "delivery bug" — but the bug is upstream in a
discarded return value.

This is the application-side mirror of the Feedback pattern (see
`guides/INTEGRATION_ACTORS.md` § Session-Level Consistency). The same
return shape that carries `{ok, Version, Events}` (or
`{ok, Version, Events, State}` from `dispatch_with_state/1`) is also
the error channel. Throwing it away means throwing away both the
canonical answer AND the canonical error report.

The same rule applies to integration-layer dispatches into evoq
adapters, to `reckon_evoq_adapter:append_events/4` returns, and to any
other "the result IS the answer" API across the family.

---

*We burned these demons so you don't have to. Keep the fire going.* 🔥🗝️🔥
