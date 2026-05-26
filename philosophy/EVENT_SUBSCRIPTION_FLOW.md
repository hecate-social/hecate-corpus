---
title: Event Subscription Flow
layer: philosophy
audience: [agent, human]
stage: stable
---

# Event Subscription Flow

How events flow from command dispatch to projections, emitters, and external consumers.

---

## The Canonical Flow

```
HOPE (HTTP request)
    |
    v
API Handler (Cowboy)
    |  translates to domain command
    v
maybe_* Handler
    |  validates, dispatches via evoq
    v
evoq_dispatcher
    |  aggregate validates command
    |  aggregate produces events
    v
ReckonDB (Khepri/Ra)
    |  events stored, Khepri triggers fire
    v
evoq subscription (by event_type)
    |  emitters + projections receive {events, [Event]}
    |
    +------+------+------+
    |      |      |      |
    v      v      v      v
  to_pg  to_mesh  projection
  (pg)   (QUIC)   (SQLite)
```

**Key principle: The API handler's job ends after dispatch. It does NOT call emitters.**

---

## What Is an Emitter?

An emitter is a **projection** — it subscribes to events from the event store and produces a side effect. The side effect happens to be broadcasting to pg or publishing to mesh, rather than writing to SQLite.

| Component | Subscribes To | Produces |
|-----------|--------------|----------|
| `*_to_pg.erl` | ReckonDB via evoq | pg broadcast (internal) |
| `*_to_mesh.erl` | ReckonDB via evoq | Mesh fact (external) |
| `*_to_sqlite_*.erl` | ReckonDB via evoq | SQLite write (read model) |

**They are all projections.** The only difference is the output target.

---

## Subscription Mechanism

### How It Works (evoq -> reckon_evoq_adapter -> reckon_db)

1. Emitter/projection calls `reckon_evoq_adapter:subscribe/5`
2. Adapter calls `esdb_gater_api:save_subscription/6`
3. ReckonDB creates a **Khepri trigger** filtered by event type
4. When an event is appended, the trigger fires
5. `reckon_db_emitter_group:broadcast/3` delivers to subscriber
6. Subscriber receives `{events, [Event]}` message

```erlang
%% Emitter subscribes at startup (in init/1)
reckon_evoq_adapter:subscribe(
    dev_studio_store,           %% Store ID
    event_type,                 %% Subscription type
    <<"event_sticky_posted_v1">>,  %% Event type selector
    <<"sticky_posted_to_pg">>,  %% Subscription name (unique)
    #{subscriber_pid => self()} %% Options — deliver to this process
).
```

### Subscription Types

| Type | Selector | Use Case |
|------|----------|----------|
| `event_type` | `<<"event_sticky_posted_v1">>` | Emitters, projections — react to specific event types |
| `stream` | `<<"venture_aggregate-ven123">>` | Rare — react to all events on one aggregate |
| `event_pattern` | Pattern map | Advanced — wildcard matching |
| `event_payload` | Payload filter | Advanced — content-based filtering |

**`event_type` is the default for emitters and projections.**

---

## Three Event Delivery Paths

All paths originate from ReckonDB via evoq subscription.

### Path 1: Internal (pg)

For intra-BEAM, intra-daemon, same-division or inter-division communication.

```
ReckonDB -> evoq subscription -> emitter (*_to_pg.erl) -> pg broadcast
                                                              |
                            +----------+---------+------------+
                            |          |         |
                            v          v         v
                      QRY projection  CMD listener  other division listener
                      (same division) (same div)   (inter-division)
```

### Path 2: External (mesh)

For cross-node, WAN, agent-to-agent communication.

```
ReckonDB -> evoq subscription -> emitter (*_to_mesh.erl) -> Macula mesh (QUIC)
                                                                  |
                                                     other nodes/agents
```

---

## Query-Side Projection Sources

Query-side projections (QRY+PRJ apps) have three options for receiving events:

### Option A: Direct evoq Subscription (Same Division)

The projection subscribes directly to ReckonDB via evoq. Best for projections within the same division that own the events.

```erlang
%% In projection init/1
reckon_evoq_adapter:subscribe(
    dev_studio_store,
    event_type,
    <<"event_sticky_posted_v1">>,
    <<"sticky_posted_to_event_stickies">>,
    #{subscriber_pid => self()}
).
```

### Option B: pg Listener (Inter-Division)

The projection has a listener that joins a pg group. Best for cross-division integration within the same BEAM VM.

```erlang
%% In listener init/1
ok = pg:join(pg, event_sticky_posted_v1, self()).

%% Receives
handle_info({event_sticky_posted_v1, Event}, State) ->
    %% project to SQLite
```

### Option C: Mesh Listener (Cross-Node)

The projection has a listener that subscribes to mesh topics. Best for receiving events from other nodes/agents.

```erlang
%% In listener init/1
macula:subscribe(MeshPid, <<"storm.sticky.posted">>, self()).

%% Receives mesh fact, translates to command or direct projection
```

### When to Use Which

| Source | When | Example |
|--------|------|---------|
| **evoq subscription** | Same division owns the event | `query_venture_lifecycle` projecting `event_sticky_posted_v1` |
| **pg listener** | Different division, same BEAM | `query_capabilities` reacting to `llm_model_detected_v1` from `serve_llm` |
| **mesh listener** | Different node, WAN | Remote agent publishing capability facts |

---

## Emitter Implementation Pattern

### pg Emitter (subscribes via evoq, broadcasts to pg)

```erlang
-module(event_sticky_posted_v1_to_pg).
-behaviour(gen_server).
-export([start_link/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2, terminate/2]).

-define(EVENT_TYPE, <<"event_sticky_posted_v1">>).
-define(PG_GROUP, event_sticky_posted_v1).
-define(SUB_NAME, <<"event_sticky_posted_v1_to_pg">>).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

init([]) ->
    %% Subscribe to event store via evoq
    {ok, _SubId} = reckon_evoq_adapter:subscribe(
        dev_studio_store,
        event_type,
        ?EVENT_TYPE,
        ?SUB_NAME,
        #{subscriber_pid => self()}
    ),
    {ok, #{}}.

handle_info({events, Events}, State) ->
    lists:foreach(fun(Event) ->
        pg:send(pg, ?PG_GROUP, {?PG_GROUP, Event})
    end, Events),
    {noreply, State};
handle_info(_Info, State) ->
    {noreply, State}.

handle_call(_Req, _From, State) -> {reply, ok, State}.
handle_cast(_Msg, State) -> {noreply, State}.
terminate(_Reason, _State) -> ok.
```

### mesh Emitter (subscribes via evoq, publishes to mesh)

```erlang
-module(event_sticky_posted_v1_to_mesh).
-behaviour(gen_server).

%% Same pattern as pg emitter, but handle_info publishes to mesh:
handle_info({events, Events}, State) ->
    lists:foreach(fun(Event) ->
        Fact = translate_to_fact(Event),
        macula:publish(mesh_pid(), <<"storm.sticky.posted">>, Fact)
    end, Events),
    {noreply, State}.
```

---

## Projection Implementation Pattern (Direct evoq Subscription)

```erlang
-module(on_event_sticky_posted_v1_project_to_sqlite_event_stickies).
-behaviour(gen_server).
-export([start_link/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2, terminate/2]).

-define(EVENT_TYPE, <<"event_sticky_posted_v1">>).
-define(SUB_NAME, <<"event_sticky_posted_v1_to_sqlite_event_stickies">>).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

init([]) ->
    {ok, _SubId} = reckon_evoq_adapter:subscribe(
        dev_studio_store,
        event_type,
        ?EVENT_TYPE,
        ?SUB_NAME,
        #{subscriber_pid => self()}
    ),
    {ok, #{}}.

handle_info({events, Events}, State) ->
    lists:foreach(fun(Event) ->
        event_sticky_posted_v1_to_sqlite_event_stickies:project(Event)
    end, Events),
    {noreply, State};
handle_info(_Info, State) ->
    {noreply, State}.

handle_call(_Req, _From, State) -> {reply, ok, State}.
handle_cast(_Msg, State) -> {noreply, State}.
terminate(_Reason, _State) -> ok.
```

---

## What the API Handler Does NOT Do

```erlang
%% post_event_sticky_api.erl
handle_post(Req0, _State) ->
    %% 1. Parse request
    %% 2. Build command
    %% 3. Dispatch
    case maybe_post_event_sticky:dispatch(Cmd) of
        {ok, Version, Events} ->
            %% 4. Return response. DONE.
            hecate_api_utils:json_reply(201, Response, Req1);
        {error, Reason} ->
            hecate_api_utils:json_error(422, Reason, Req1)
    end.
    %% NO emit calls. NO pg broadcasts. NO mesh publishes.
    %% Emitters handle that autonomously via evoq subscriptions.
```

---

## Supervision

Emitters are started by the **desk supervisor** (CMD side), not the API handler.

```
guide_venture_lifecycle_sup (domain supervisor)
|-- post_event_sticky_sup (desk supervisor)
|   |-- event_sticky_posted_v1_to_pg (worker — subscribes via evoq)
|   +-- event_sticky_posted_v1_to_mesh (worker — subscribes via evoq)
```

Query-side projections that subscribe directly via evoq are started by the **query domain supervisor**.

```
query_venture_lifecycle_sup (domain supervisor)
|-- query_venture_lifecycle_store (SQLite connection)
|-- on_event_sticky_posted_v1_project_to_sqlite_event_stickies (worker — subscribes via evoq)
|-- on_event_sticky_pulled_v1_project_to_sqlite_event_stickies (worker — subscribes via evoq)
```

---

## Decision Record

| Date | Decision |
|------|----------|
| 2026-02-13 | Emitters subscribe to ReckonDB via evoq, not called manually from API handlers |
| 2026-02-13 | API handlers only dispatch commands and return responses |
| 2026-02-13 | Emitters are projections — same subscription mechanism, different output target |
| 2026-02-13 | Query projections can subscribe directly via evoq OR use pg/mesh listeners |
| 2026-02-13 | pg listeners for inter-division (same BEAM), mesh listeners for cross-node (WAN) |
| 2026-02-13 | Store ID is `dev_studio_store` (shared across all domains) |

---

*Events flow downhill. Subscriptions pull them where they need to go.*
