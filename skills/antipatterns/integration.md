---
title: "ANTIPATTERNS: Integration"
layer: skill
audience: [agent, human]
stage: stable
---

# ANTIPATTERNS: Integration — Subscriptions, Messaging, Process Managers

*Demons about integration channels, subscription pipelines, and cross-domain communication.*

[Back to Index](INDEX.md)

---

## 🔥 Using Mesh for Internal Integration

**Date:** 2026-02-08
**Origin:** hecate-daemon walking skeleton debugging

### The Antipattern

Using Macula mesh (`*_to_mesh.erl` emitters) for communication between umbrella apps within the same BEAM VM.

**Example (WRONG):**
```
setup_venture (CMD app)
    → venture_initiated_v1_to_mesh.erl
    → Macula mesh (QUIC, DHT, NAT traversal)
    → on_venture_initiated_from_mesh.erl
    → query_ventures (PRJ+QRY app)
```

This uses WAN-grade infrastructure for intra-process communication.

### Why It's Wrong

1. **Massive overhead** — QUIC, DHT discovery, NAT traversal for processes in the same VM
2. **QUIC needs addressable URIs** — Doesn't work inside containers (no stable public address)
3. **Adds latency** — Network round-trip for what should be direct message passing

### The Rule

> **Use `pg` (OTP process groups) for internal integration.**
> **Mesh (Macula/QUIC) is ONLY for:**
> - **NAT traversal** — when peers are behind different NATs
> - **Direct Internet** — agent-to-agent over the public Internet
> - **LAN ↔ LAN** — communication between separate physical networks
>
> QUIC requires addressable URIs. Containers don't have them. Mesh does NOT work in K8s.

### The Correct Pattern

```
setup_venture (CMD app)
    → venture_initiated_v1_to_pg.erl    # Internal via pg
    → Direct Erlang message passing
    → on_venture_initiated_v1_from_pg_project_to_sqlite_ventures.erl
    → query_ventures (PRJ+QRY app)
```

### Two Integration Layers

| Layer | Transport | Scope |
|-------|-----------|-------|
| **Internal** | `pg` | Same BEAM VM, intra-daemon, intra-LAN (Erlang VM cluster) |
| **External** | `mesh` | NAT traversal, direct Internet, LAN ↔ LAN (QUIC, addressable URIs required) |

### Naming Convention

| Transport | Emitter | Listener |
|-----------|---------|----------|
| pg | `{event}_to_pg.erl` | `on_{event}_from_pg_*.erl` |
| mesh | `{event}_to_mesh.erl` | `on_{event}_from_mesh_*.erl` |

See [INTEGRATION_TRANSPORTS.md](../../philosophy/INTEGRATION_TRANSPORTS.md) for full details.

---

## 🔥 Side Effects Based on Hope Acknowledgment

**Date:** 2026-02-09
**Origin:** Vision command architecture review

### The Antipattern

Performing side effects (file I/O, state changes) in a client based on a command's HTTP response rather than on a received event (fact).

**Example (WRONG):**
```javascript
// Client sends command to daemon
const res = await fetch('hecate://localhost/api/domains/refine-vision', {
    method: 'POST', body: JSON.stringify(params)
});
if (res.ok) {
    // WRONG: treating 200 OK as a fact
    updateVisionDisplay(params);
}
```

The 200 OK means "I received your hope." Not "the vision was refined." Between acknowledgment and event storage, anything can fail.

### The Rule

> **Side effects in external systems (frontends, other agents) must be triggered by received FACTS (events), not by command acknowledgments (hope receipts).**

### The Correct Pattern

```javascript
// Client subscribes to event stream
const events = new EventSource('hecate://localhost/api/domains/events');
events.addEventListener('vision_refined_v1', (e) => {
    updateVisionDisplay(JSON.parse(e.data));  // NOW it's safe
});

// Client sends hope (fire and forget the response)
fetch('hecate://localhost/api/domains/refine-vision', {
    method: 'POST', body: JSON.stringify(params)
});  // 202 Accepted — don't act on the response
```

### Why It Matters

- Commands can be rejected by aggregate business rules AFTER acknowledgment
- Event store writes can fail
- Async processing means acknowledgment ≠ completion
- The frontend is an **external system** — it must treat the daemon as eventually consistent

### Reference

See [HOPE_FACT_SIDE_EFFECTS.md](../HOPE_FACT_SIDE_EFFECTS.md) for the full architectural pattern.

---

## Demon 15: Consumer-Generated Command IDs for Framework Idempotency

**Date exorcised:** 2026-02-11
**Where it appeared:** All 76 dispatch modules across hecate-daemon CMD apps
**Cost:** 9/9 L4b dispatch tests returning cached first-command results (silent data loss)

### The Lie

"Each dispatch module should generate its own `command_id` for idempotency."

### What Happened

All 76 dispatch modules had identical `generate_command_id(Id, Timestamp)` functions using `hash(AggregateId + Timestamp_ms)`. Two different commands to the same aggregate within 1ms produced identical command IDs, causing the idempotency cache to silently return the first command's cached result — a production-grade silent data loss bug. The codegen template propagated the bug to every module.

### Why It's Wrong

- **76 identical functions = wrong responsibility placement.** If every consumer must implement the same logic, it belongs in the framework.
- **Conflates two concepts.** The design conflates command identification (tracing, unique per invocation) with command deduplication (idempotency, deterministic per intent).
- **`hash(Id + Timestamp)` fails at BOTH:** Not unique within 1ms (commands collide), not deterministic across retries (timestamps differ).

### The Truth

- The **FRAMEWORK** (evoq) should auto-generate `command_id` if not provided (unique per invocation, for tracing)
- True idempotency requires a separate `idempotency_key` field — caller-provided, deterministic, based on business intent
- Dispatch modules should NOT contain `generate_command_id` at all
- These are two separate fields on `#evoq_command{}`: `command_id` (framework-owned) and `idempotency_key` (caller-optional)

### The Fix

evoq v1.3.0+ auto-generates `command_id`. Dispatch modules drop `generate_command_id` entirely.

### The Lesson

> **If every consumer implements the same function, it belongs in the framework.**
> **If one field serves two purposes (identification and deduplication), split it into two fields.**

---

## 🔥 Demon 24: Silent Subscription Pipeline Failures

**Date exorcised:** 2026-02-13
**Where it appeared:** reckon-db subscription delivery pipeline (three bugs: v1.2.4, v1.2.5, v1.2.6)
**Cost:** 7 days of debugging. POST /api/domains/initiate returns 201, GET /api/domains returns []. No errors, no crashes, no warnings.

### The Lie

"If commands succeed and events are stored, projections will populate."

### What Happened

Three separate bugs in reckon-db's subscription delivery pipeline each caused **complete silence** — events stored correctly but never delivered to subscribers. Each bug was in a different layer:

| Bug | Layer | What Went Wrong | Symptom |
|-----|-------|-----------------|---------|
| Filter path mismatch | Khepri trigger | `by_stream` stripped category prefix from stream ID → filter path `[streams, <<"delivery-001">>]` never matched storage path `[streams, <<"test$delivery-001">>]` | Trigger never fires |
| Record vs map matching | Khepri trigger | `by_event_type` used map pattern `#{event_type => Type}` but stored data is `#event{}` record (tuple) — maps can't match tuples | Trigger never fires |
| Subscription id undefined | pg group routing | `subscribe/5` never set `#subscription.id` → emitter pool joined pg group `{Store, undefined, emitters}` → trigger broadcast to `{Store, CorrectKey, emitters}` | Broadcast finds no emitters |

**Result:** Commands succeed (201). Events stored in ReckonDB. Triggers registered. But the chain breaks silently at different points — subscribers never receive `{events, [Event]}` messages. SQLite read models stay empty. GET returns `[]`.

### Why This Is the Deadliest Bug Class

1. **No errors.** No crashes, no warnings, no log messages. The system appears healthy.
2. **Partial success misleads.** POST returns the event data. The developer thinks "it worked."
3. **Blame diffusion.** Is it the projection? The emitter? The subscription? The trigger? The filter? The pg group? Each component appears correct in isolation.
4. **Unit tests pass.** Each layer's unit tests passed. No integration test exercised the full pipeline.
5. **The gap is invisible.** The distance between "event stored" and "event delivered to subscriber" is a black box.

### The Pipeline

```
append(Event)
  → Khepri put (stored)              ← Events live here
  → Khepri trigger fires             ← Bug 1, 2: filter doesn't match
  → ProcFun executes
  → reckon_db_emitter_group:broadcast(StoreId, SubKey, Event)
  → pg:get_members(scope, {StoreId, SubKey, emitters})  ← Bug 3: wrong key
  → emitter receives
  → subscriber receives {events, [Event]}  ← Never reached
```

### Three Rules That Would Have Prevented This

**Rule 1: Integration-test the full delivery pipeline.**

Before trusting subscriptions, write a CT test that:
1. Creates a store
2. Subscribes a process
3. Starts an emitter for that subscription
4. Appends an event
5. Asserts the subscriber receives it within N seconds

```erlang
%% This test caught ALL THREE bugs
subscribe_then_append_delivers_event(Config) ->
    StoreId = proplists:get_value(store_id, Config),
    {ok, SubKey} = reckon_db_subscriptions:subscribe(
        StoreId, stream, <<"test$stream-1">>, <<"test_sub">>,
        #{subscriber => self()}),
    EmitterName = reckon_db_emitter_group:emitter_name(StoreId, SubKey),
    {ok, _} = reckon_db_emitter:start_link(StoreId, SubKey, self(), EmitterName),
    {ok, _} = reckon_db_streams:append(StoreId, <<"test$stream-1">>, -2,
        [#{event_type => <<"thing_happened_v1">>, data => #{}}]),
    receive
        {events, [E]} -> ?assertEqual(<<"thing_happened_v1">>, E#event.event_type)
    after 5000 -> ct:fail("Event not delivered")
    end.
```

**This test did not exist until after 7 days of debugging.**

**Rule 2: Computed identifiers must travel with their records.**

When you compute a key/id from a record's fields, set it ON the record immediately:

```erlang
%% WRONG — id computed but not set on record
Key = subscriptions_store:key(Subscription),
notify_created(StoreId, subscriptions, Subscription),  %% id = undefined!

%% CORRECT — id travels with the record
Key = subscriptions_store:key(Subscription),
SubWithId = Subscription#subscription{id = Key},
notify_created(StoreId, subscriptions, SubWithId),  %% id = Key ✓
```

If downstream code needs `Record.id`, the code that computes the id must set it. Don't hope someone else will.

**Rule 3: When storage paths and filter paths must match, test that they match.**

Khepri triggers use path-based event filters. If the filter path doesn't **exactly** match the storage path, the trigger silently never fires. There's no "filter didn't match" warning.

```
Storage path:  [streams, <<"test$delivery-001">>, <<"000000000">>]
Filter path:   [streams, <<"delivery-001">>, #if_path_matches{...}]
                         ^^^^^^^^^^^^^^^^^
                         WRONG — stripped the category prefix
```

This is a general rule for any trigger/filter/subscription system: **the selector must mirror the storage path exactly.** Write a test that verifies this by storing data and checking the trigger fires.

### The Meta-Lesson

> **Event sourcing's biggest vulnerability is the invisible gap between "stored" and "delivered."**
> **If you can't prove delivery with an integration test, you can't trust it.**
> **Silent failure + no integration tests = days of debugging that one CT test would have prevented.**

### For Hecate Agents Specifically

When generating code that uses ReckonDB subscriptions:

1. **Always generate a delivery smoke test** alongside subscription code
2. **Never assume subscriptions "just work"** — verify with `receive ... after` in a test
3. **If GET returns empty but POST succeeds** — the subscription pipeline is broken, not the projection logic
4. **Check these layers in order:** filter match → trigger firing → pg group membership → emitter running → subscriber PID alive

### See Also

- [reckon-db CHANGELOG](https://github.com/reckon-db-org/reckon-db/blob/main/CHANGELOG.md) — bugs documented in v1.2.4, v1.2.5, v1.2.6
- Demon #22 (Manual Event Emission) — the application-level version of this infrastructure-level problem
- Demon #23 (Raw Records in Projections) — another "silent failure" in the event delivery chain
- [SESSION_LEVEL_CONSISTENCY.md](../../philosophy/SESSION_LEVEL_CONSISTENCY.md) — the mitigation pattern: return aggregate state from commands so callers don't depend on projection timing

---

## 🔥 Demon 26: Calling PG Emitters "Dead Code" Without Subscribers

**Date exorcised:** 2026-02-23
**Where it appeared:** hecate-app-appstored audit — 6 `*_to_pg.erl` emitters
**Cost:** Nearly deleted working infrastructure that enables inter-domain integration

### The Lie

"These `*_to_pg.erl` emitters have no subscribers — they're dead code and should be removed."

### What Happened

During an audit of `hecate-app-appstored`, all 6 PG emitters were flagged as "dead code" because no process currently calls `pg:join/3` on their topics. The reasoning was:

1. No module subscribes to these pg groups → nobody receives the broadcasts → the emitters do nothing
2. Therefore they are dead code and should be removed to reduce complexity

This reasoning is **fundamentally wrong** and misunderstands two things:

### Two Misconceptions Corrected

**Misconception 1: "No current subscribers = dead code"**

PG emitters are **pub/sub publishers**. In pub/sub, the publisher does NOT need to know about consumers. The publisher's job is to PUBLISH. Consumers arrive when they need the data — possibly in a different app, possibly months later, possibly in a process manager that doesn't exist yet.

Calling a publisher "dead" because it has no current subscribers is like calling a radio tower "dead" because nobody is tuned in right now. The tower's job is to broadcast. Listeners come and go.

**Misconception 2: "ReckonDB is just an Event Store"**

ReckonDB (like any Event Store) serves **two roles**:

1. **Event Store** — durable, ordered storage of domain events
2. **Event Bus** — processes subscribe via `reckon_evoq_adapter:subscribe/5` and receive events as they're appended

The `*_to_pg.erl` emitters bridge from the Event Bus to OTP `pg` groups. This serves a different audience than direct evoq subscriptions:

| Channel | Audience | Scope |
|---------|----------|-------|
| Direct evoq subscription | Projections within the same division | Intra-domain |
| PG emitter → pg group | Process managers, listeners in OTHER divisions | Inter-domain |

Both channels are valid. They serve different integration needs.

### The Architecture

```
ReckonDB (Store + Bus)
  ↓ reckon_evoq_adapter:subscribe/5
  ├── Projections → SQLite (read models, same domain)
  └── PG Emitters → pg groups (inter-domain integration)
                      └── Future consumers join when needed
```

### Why It's Dangerous to Remove

1. **Silent breakage** — A future process manager that `pg:join`s an emitter's topic will receive nothing if the emitter was deleted
2. **No error signal** — pg groups with no publishers don't error — they just never deliver messages
3. **Architectural intent lost** — The emitter documents that "this event is important enough to broadcast inter-domain"
4. **Rebuilding is expensive** — Re-creating the emitter, its supervisor child spec, and its evoq subscription is significant work

### The Rule

> **Never remove a pub/sub publisher because it has no current subscribers.**
> **PG emitters are inter-domain integration infrastructure — they exist to PUBLISH, not to serve known consumers.**
> **An Event Store is also an Event Bus. Emitters bridge from store-bus to pg-bus for a different audience.**

### Prevention

Before calling any emitter "dead code," ask:

1. Is it a pub/sub publisher? → Publishers don't need subscribers to be valid
2. Does it bridge between integration layers? → It's infrastructure
3. Was it intentionally created as part of a vertical slice? → It documents intent
4. Could a future consumer need this topic? → Leave it alone

### The Lesson

> **Pub/sub publishers are infrastructure. Infrastructure exists before its consumers.**
> **ReckonDB = Event Store + Event Bus. PG emitters bridge the bus to pg groups for inter-domain use.**
> **"No subscribers" ≠ "dead code." It means "no consumers yet."**

---

## Demon 39: Bypassing Evoq Behaviours with Raw gen_servers

**Date exorcised:** 2026-03-07
**Where it appeared:** ALL 63 event-handling modules across hecate-daemon (PMs, projections, emitters)
**Cost:** Auto-start bug — containers started on EVERY event, not just `plugin_execution_started_v1`. Unfiltered event delivery caused invisible production-grade bugs.

### The Lie

"I'll just use `evoq_subscriptions:subscribe/5` in a raw `gen_server` — it's simpler than implementing `evoq_event_handler` or `evoq_process_manager`."

### What Happened

Every single event-handling module in hecate-daemon was implemented as a raw `gen_server` that called `evoq_subscriptions:subscribe/5` directly in `init/1`:

```erlang
%% WRONG — Raw gen_server, no evoq behaviour
-module(on_plugin_execution_started_start_container).
-behaviour(gen_server).

init([]) ->
    {ok, _} = evoq_subscriptions:subscribe(
        plugins_store, event_type, <<"plugin_execution_started_v1">>,
        <<"on_plugin_execution_started_start_container">>,
        #{subscriber_pid => self()}),
    {ok, #{}}.

handle_info({events, Events}, State) ->
    lists:foreach(fun handle_event/1, Events),
    {noreply, State}.

handle_event(#evoq_event{data = Data}) ->
    %% BUG: No event_type check! Receives ALL events from plugins_store.
    PluginId = get_value(plugin_id, Data),
    start_container(PluginId).
```

The `event_type` parameter in `evoq_subscriptions:subscribe/5` is passed to reckon-db as a filter hint, but reckon-db's subscription delivery does NOT guarantee filtering. **All events from the store are delivered regardless.** The raw gen_server had no event_type guard, so it started containers on `container_pull_completed_v1`, `plugin_installed_v1`, and every other event in `plugins_store`.

### Why It's Wrong

1. **No event filtering.** Evoq behaviours register with `evoq_event_type_registry` via `interested_in/0`. The `evoq_event_router` only delivers matching events. Raw gen_servers receive EVERYTHING.
2. **No telemetry.** Evoq behaviours emit `telemetry:execute` events for every event processed — duration, success, failure. Raw gen_servers are invisible.
3. **No error handling.** Evoq behaviours support `on_error/4` with retry, skip, stop, dead-letter strategies. Raw gen_servers crash or silently swallow errors.
4. **No checkpointing.** `evoq_projection` tracks checkpoint positions and skips already-processed events (idempotency). Raw gen_servers reprocess everything on restart.
5. **No rebuild.** `evoq_projection` supports `rebuild/1` to replay all events and reconstruct read models. Raw gen_servers have no replay mechanism.
6. **Invisible bugs.** When a PM receives the wrong event type and acts on it, the bug is silent — it looks like correct behavior until someone notices the side effect.

### The Three Evoq Behaviours

| Behaviour | Purpose | Key Callbacks |
|-----------|---------|---------------|
| `evoq_event_handler` | Side effects (pg broadcast, mesh publish, external calls) | `interested_in/0`, `init/1`, `handle_event/4` |
| `evoq_projection` | Read model updates (SQLite, ETS) | `interested_in/0`, `init/1`, `project/4` |
| `evoq_process_manager` | Cross-aggregate coordination (saga, policy) | `interested_in/0`, `correlate/2`, `handle/3`, `apply/2` |

### Correct Patterns

**Event Handler (emitters, side-effect workers):**

```erlang
-module(plugin_installed_v1_to_pg).
-behaviour(evoq_event_handler).

interested_in() -> [<<"plugin_installed_v1">>].

init(_Config) -> {ok, #{}}.

handle_event(<<"plugin_installed_v1">>, Event, _Metadata, State) ->
    pg:send(plugins, {plugin_installed_v1, Event}),
    {ok, State}.
```

**Projection (read model writers):**

```erlang
-module(plugin_installed_v1_to_sqlite_plugins).
-behaviour(evoq_projection).

interested_in() -> [<<"plugin_installed_v1">>].

init(_Config) ->
    {ok, RM} = evoq_read_model:new(evoq_read_model_ets, #{}),
    {ok, #{}, RM}.

project(#{event_type := <<"plugin_installed_v1">>} = Event, _Meta, State, RM) ->
    ok = do_sqlite_insert(Event),
    {ok, State, RM};
project(_Event, _Meta, State, RM) ->
    {skip, State, RM}.
```

**Process Manager (policies, sagas):**

```erlang
-module(on_plugin_installed_register_entry).
-behaviour(evoq_process_manager).

interested_in() -> [<<"plugin_installed_v1">>].

correlate(#{data := #{plugin_id := PluginId}}, _Meta) ->
    {start, PluginId}.

handle(_State, #{event_type := <<"plugin_installed_v1">>} = Event, _Meta) ->
    Cmd = build_register_entry_command(Event),
    {ok, #{}, [Cmd]};
handle(State, _Event, _Meta) ->
    {ok, State}.

apply(State, _Event) -> State.
```

### The 63 Violations (Audit)

Every module below was a raw gen_server using `evoq_subscriptions:subscribe/5`:

**CMD apps (guide_plugin_lifecycle, guide_launcher_lifecycle, guide_license_lifecycle, mentor_llms):**
- All `*_to_pg.erl` emitters (should be `evoq_event_handler`)
- All `*_to_mesh.erl` emitters (should be `evoq_event_handler`)
- All `on_*` process managers (should be `evoq_process_manager`)

**PRJ apps (project_plugins, project_licenses, project_launcher, project_settings, project_realm_memberships, project_llm_mentorships):**
- All `on_*_to_sqlite_*.erl` projections (should be `evoq_projection`)
- All `*_to_settings.erl`, `*_to_memberships.erl` projections

### Prevention

Before writing ANY module that handles domain events:

1. **Ask: "What is this module's job?"**
   - Side effect (broadcast, call external system) -> `evoq_event_handler`
   - Update read model (SQLite, ETS) -> `evoq_projection`
   - Dispatch commands to other aggregates -> `evoq_process_manager`

2. **Never use `evoq_subscriptions:subscribe/5` directly in application code.**
   It is infrastructure for evoq internals. Application modules implement behaviours.

3. **If the module has `handle_info({events, Events}, State)` -> it is WRONG.**
   Evoq behaviours receive events through `handle_event/4`, `project/4`, or `handle/3`.

### The Meta-Lesson

> **When you build on a framework, USE the framework.**
> **Bypassing evoq's behaviours to use raw gen_servers is like bypassing OTP to use raw processes.**
> **The framework exists to enforce contracts (interested_in), provide guarantees (filtering, checkpointing, telemetry), and prevent bugs (event-type mismatches).**
> **63 modules. Zero behaviours. One auto-start bug that took days to find. Use. The. Framework.**

---

*We burned these demons so you don't have to. Keep the fire going.*
