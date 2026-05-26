---
title: Evoq Behaviours
layer: codegen
audience: [codegen]
stage: stable
---

# Evoq Behaviours — Complete Reference

**Package:** `evoq` (hex.pm, ~> 1.4)
**Source:** `reckon-db-org/evoq/`
**Last verified:** 2026-03-03

Evoq provides **14 behaviours** in three categories: Core Domain (what you implement for business logic), Infrastructure/Adapter (pluggable backends), and Lifecycle/Extension (customization hooks).

---

## Core Domain Behaviours

### 1. `evoq_aggregate` — Consistency Boundary

The aggregate is the consistency boundary in event sourcing. It processes commands and emits events.

**Required callbacks:**

| Callback | Spec | Description |
|----------|------|-------------|
| `init/1` | `init(AggregateId :: binary()) -> {ok, State}` | Initialize aggregate state |
| `execute/2` | `execute(State, Command :: map()) -> {ok, [Event]} \| {error, Reason}` | Execute command, return events |
| `apply/2` | `apply(State, Event :: map()) -> NewState` | Apply event to update state |

**CRITICAL:** `execute(State, Command)` and `apply(State, Event)` — **State comes FIRST!**

**Optional callbacks:**

| Callback | Description |
|----------|-------------|
| `snapshot/1` | Serialize state for snapshot storage |
| `from_snapshot/1` | Restore state from snapshot |

**Started via:** `evoq_aggregate:start_link/2,3`. Managed by `evoq_aggregate_registry` (on-demand via `evoq_aggregate_partition_sup`).

**Example:**

```erlang
-module(territory_auction_aggregate).
-behaviour(evoq_aggregate).
-export([init/1, execute/2, apply/2]).

init(_AggregateId) ->
    {ok, #territory_auction{}}.

execute(State, #{command_type := open_territory_auction_v1} = Cmd) ->
    maybe_open_territory_auction:execute(State, Cmd);
execute(State, #{command_type := close_bidding_v1} = Cmd) ->
    maybe_close_bidding:execute(State, Cmd).

apply(State, #{event_type := <<"territory_auction_opened_v1">>} = Event) ->
    %% Update state from event data
    Data = maps:get(data, Event),
    State#territory_auction{status = ?OPENED, ...}.
```

---

### 2. `evoq_command` — Command Validation

Commands represent intentions to change state. Validation is optional but recommended.

**Required callbacks:** None.

**Optional callbacks:**

| Callback | Spec | Description |
|----------|------|-------------|
| `validate/1` | `validate(Command :: map()) -> ok \| {error, Reason}` | Custom validation |

**Usage:**

```erlang
%% Create a command
Cmd = evoq_command:new(
    open_territory_auction_v1,    %% command_type
    territory_auction_aggregate,  %% aggregate_type
    AuctionId,                    %% aggregate_id
    #{territory => Territory, ...}  %% payload
).

%% Dispatch (validate + execute)
evoq_router:dispatch(Cmd).
```

---

### 3. `evoq_event_handler` — Event Subscription (Side Effects)

Event handlers subscribe to event types (not streams) and process events as they are published. Use for side effects like publishing to pg groups, sending emails, etc.

**Required callbacks:**

| Callback | Spec | Description |
|----------|------|-------------|
| `interested_in/0` | `-> [EventType :: binary()]` | Event types to subscribe to |
| `init/1` | `init(Config :: map()) -> {ok, State}` | Initialize handler state |
| `handle_event/4` | `handle_event(EventType, Event, Metadata, State) -> {ok, NewState}` | Process event |

**Optional callbacks:**

| Callback | Description |
|----------|-------------|
| `on_error/4` | Custom error handling strategy (see `evoq_error_handler`) |

**Started via:** `evoq_event_handler:start_link/2,3`. Registers with `evoq_event_type_registry` by event type.

**Example (pg emitter):**

```erlang
-module(franchise_territory_awarded_v1_to_pg).
-behaviour(evoq_event_handler).
-export([interested_in/0, init/1, handle_event/4]).

interested_in() -> [<<"franchise_territory_awarded_v1">>].

init(_Config) ->
    ensure_pg_scope(),
    {ok, #{}}.

handle_event(<<"franchise_territory_awarded_v1">>, Event, Metadata, State) ->
    Data = maps:get(data, Event, Event),
    Payload = #{auction_id => maps:get(aggregate_id, Metadata)},
    Members = pg:get_members(macula_integration, territory_awarded),
    [Pid ! {territory_awarded, Payload} || Pid <- Members],
    {ok, State}.
```

---

### 4. `evoq_projection` — Read Model Builder

Projections transform events into read model updates. They are idempotent (can be replayed safely) and track progress via checkpoints.

**Required callbacks:**

| Callback | Spec | Description |
|----------|------|-------------|
| `interested_in/0` | `-> [EventType :: binary()]` | Event types to project |
| `init/1` | `init(Config) -> {ok, State, ReadModel}` | Initialize with read model |
| `project/4` | `project(Event, Metadata, State, ReadModel) -> {ok, NewState, NewReadModel} \| {skip, State, ReadModel}` | Transform event |

**Optional callbacks:**

| Callback | Description |
|----------|-------------|
| `on_error/4` | Custom error handling |

**Started via:** `evoq_projection:start_link/2,3` (opts: `checkpoint_store`, `start_from`). Supports `rebuild/1` to clear and replay.

**Difference from `evoq_event_handler`:** Projections manage a read model, have checkpoint support, and can be rebuilt. Event handlers are for fire-and-forget side effects.

---

### 5. `evoq_process_manager` — Cross-Aggregate Saga

Coordinates long-running business processes spanning multiple aggregates. Correlates events to process instances and dispatches commands.

**Required callbacks:**

| Callback | Spec | Description |
|----------|------|-------------|
| `interested_in/0` | `-> [EventType :: binary()]` | Event types to handle |
| `correlate/2` | `correlate(Event, Metadata) -> {start, ProcId} \| {continue, ProcId} \| {stop, ProcId} \| false` | Route event to process instance |
| `handle/3` | `handle(State, Event, Metadata) -> {ok, NewState} \| {ok, NewState, [Command]} \| {error, Reason}` | Process event, optionally dispatch commands |
| `apply/2` | `apply(State, Event) -> NewState` | Apply event to PM state |

**Optional callbacks:**

| Callback | Description |
|----------|-------------|
| `init/1` | Initialize PM state for new process instance |
| `compensate/2` | Generate compensating commands for saga rollback |

**Started via:** `evoq_process_manager:start/2,3`. Routes via `evoq_pm_router` -> `evoq_pm_instance`.

---

## Infrastructure/Adapter Behaviours

### 6. `evoq_adapter` — Event Store Backend

Pluggable event store interface. Implementation: `reckon_evoq` (connects to ReckonDB via reckon_gater).

| Callback | Description |
|----------|-------------|
| `append/4` | Append events with version check (-1=NO_STREAM, -2=ANY, N>=0=exact) |
| `read/5` | Read events from stream (start, count, direction) |
| `read_all/3` | Read all events from stream |
| `read_by_event_types/3` | Read events by type across all streams |
| `version/2` | Get current stream version |
| `exists/2` | Check if stream exists |
| `list_streams/1` | List all streams |
| `delete_stream/2` | Delete stream and events |

Config: `{evoq, [{event_store_adapter, reckon_evoq}]}`

### 7. `evoq_subscription_adapter` — Subscription Backend

| Callback | Description |
|----------|-------------|
| `subscribe/5` | Create subscription (types: stream, event_type, event_pattern, event_payload, tags) |
| `unsubscribe/2` | Remove subscription |
| `ack/4` | Acknowledge event processed |
| `get_checkpoint/2` | Get current checkpoint position |
| `list/1` | List all subscriptions |
| `get_by_name/2` | Get subscription by name |

### 8. `evoq_snapshot_adapter` — Snapshot Backend

| Callback | Description |
|----------|-------------|
| `save/5` | Save snapshot at version |
| `read/2` | Read latest snapshot |
| `read_at_version/3` | Read snapshot at exact version |
| `delete/2` | Delete all snapshots for stream |
| `delete_at_version/3` | Delete specific snapshot version |
| `list_versions/2` | List all snapshot versions |

### 9. `evoq_read_model` — Read Model Store

**Required:** `init/1`, `get/2`, `put/3`, `delete/2`
**Optional:** `list/2`, `clear/1`
**Reference impl:** `evoq_read_model_ets` (ETS-backed, in-memory)

### 10. `evoq_checkpoint_store` — Projection Checkpoint Persistence

**Required:** `load/1`, `save/2`
**Optional:** `delete/1`
**Reference impl:** `evoq_checkpoint_store_ets` (ETS-backed, lost on restart)

---

## Lifecycle/Extension Behaviours

### 11. `evoq_aggregate_lifespan` — Aggregate Lifecycle Control

Controls idle timeouts, hibernation, and passivation (snapshot + stop).

**Required:**

| Callback | Description |
|----------|-------------|
| `after_event/1` | Return action after processing event |
| `after_command/1` | Return action after processing command |
| `after_error/1` | Return action after error |

**action() type:** `timeout() | infinity | hibernate | stop | passivate`

**Optional:** `on_timeout/1`, `on_passivate/1`, `on_activate/2`
**Default impl:** `evoq_aggregate_lifespan_default` (30min idle, 5min error, auto-snapshot on passivate)

### 12. `evoq_middleware` — Command Pipeline Interceptor

All optional: `before_dispatch/1`, `after_dispatch/1`, `after_failure/1`
Config: `{evoq, [{middleware, [my_middleware]}]}` or per-dispatch opts.

Pipeline record: `#evoq_pipeline{command, context, assigns, halted, response}`

### 13. `evoq_event_upcaster` — Event Schema Evolution

**Required:** `upcast(Event, Metadata) -> {ok, Transformed} | {ok, Transformed, NewEventType} | skip`
**Optional:** `version() -> pos_integer()`
Registered with `evoq_type_provider`. Chainable via `chain_upcasters/3`.

### 14. `evoq_error_handler` — Error Handling Strategy

All optional: `on_error/4`, `max_retries/0`, `backoff_ms/1`
**error_action():** `retry | {retry, DelayMs} | skip | stop | {dead_letter, Reason}`
Default: exponential backoff (100ms base, 30s max), 5 retries, then dead letter.

---

## Quick Decision Guide

| Need | Behaviour |
|------|-----------|
| Enforce business rules, emit events | `evoq_aggregate` |
| Validate command before dispatch | `evoq_command` |
| React to events (side effects, pg/mesh) | `evoq_event_handler` |
| Build read model from events | `evoq_projection` |
| Coordinate across aggregates (saga) | `evoq_process_manager` |
| Transform old event versions | `evoq_event_upcaster` |
| Add logging/auth/metrics to dispatch | `evoq_middleware` |
| Control aggregate memory lifecycle | `evoq_aggregate_lifespan` |
| Plug in different event store | `evoq_adapter` |
| Plug in subscription mechanism | `evoq_subscription_adapter` |
| Plug in snapshot storage | `evoq_snapshot_adapter` |
| Plug in read model storage | `evoq_read_model` |
| Plug in checkpoint persistence | `evoq_checkpoint_store` |
| Custom error recovery strategy | `evoq_error_handler` |

## Common Patterns in Our Codebase

**Emitter (event handler → pg):** `{event}_to_pg.erl` — subscribes to event, publishes to pg group
**Emitter (event handler → mesh):** `{event}_to_mesh.erl` — subscribes to event, publishes to mesh
**Projection (event → SQLite):** `{event}_to_{table}.erl` — projects event to SQLite read model table
**Process Manager:** `on_{event}_{verb}_{subject}.erl` — reacts to event, dispatches command to another aggregate
