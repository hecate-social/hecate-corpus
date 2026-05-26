---
title: "Example: Mesh Integration Patterns"
layer: example
audience: [agent, human]
stage: superseded
---

# Example: Mesh Integration Patterns

*Canonical example: FACTS vs EVENTS, Emitters and Listeners*

> **Layout note (2026-05-24):** This example places the listener INSIDE the desk it triggers (`initiate_division/subscribe_to_division_discovered.erl`). That layout is **superseded**. Current canon: listeners / PMs are **sibling slices** at the top of the target CMD app's `src/`, named `on_{src_event}_{action}_{target}/`, each with its own `_sup.erl` + gen_server. See [PROCESS_MANAGERS.md Location Rule](../philosophy/PROCESS_MANAGERS.md#location-rule) and [antipatterns/structure.md Demon 18](../skills/antipatterns/structure.md#-demon-18-process-managers-inside-desks). The mesh subscribe / emit code itself is still correct — only the directory placement and supervision wiring change.

---

## The Pattern

The mesh is a distributed communication layer between agents. It is **NOT** an event bus.

| Term | What It Is | Where It Lives |
|------|-----------|----------------|
| **FACT** | External truth published to mesh | Between agents |
| **EVENT** | Internal domain event (what happened) | Within an agent's event store |
| **COMMAND** | Intention (what should happen) | Within agent |

**Critical distinction:**

- **Events** are internal. They belong to an aggregate. They are stored in your event store.
- **Facts** are external. They are published to topics. Other agents receive them.

Treating mesh facts as events (or vice versa) breaks your architecture.

---

## Wrong Way: Bypassing the Command Layer

```
Mesh FACT → Subscriber → Projection (WRONG!)
```

This antipattern:
- Treats external facts as internal events
- Bypasses aggregate validation
- No command audit trail
- Projection state can diverge from event store
- Loses single source of truth

```erlang
%% ❌ WRONG: Direct projection from mesh message
handle_info({mesh_fact, Topic, FactData}, State) ->
    %% NO! Don't project directly from mesh facts!
    capability_projection:project(FactData),
    {noreply, State}.
```

---

## Correct Way: Full LISTENER → COMMAND → AGGREGATE Flow

### Receiving Facts (LISTENER)

```
Mesh FACT → LISTENER → converts to → COMMAND → AGGREGATE → DOMAIN EVENT → stored → projected
```

The listener's ONLY job is to convert external facts into internal commands.

### Publishing Facts (EMITTER)

```
DOMAIN EVENT → EMITTER → converts to → FACT → Mesh
```

The emitter's ONLY job is to convert internal events into external facts.

---

## Emitter Code Example

From `apps/discover_divisions/src/discover_division/division_discovered_v1_to_mesh.erl`:

```erlang
%%% @doc Emitter: Publish division_discovered_v1 events to mesh
%%%
%%% When a division is discovered within a domain, this emitter publishes
%%% the fact to mesh topic `hecate.domain.division_discovered`.
%%%
%%% The design_division service subscribes to this topic and initiates
%%% the division's lifecycle.
%%%
%%% Flow: division_discovered_v1 (event) -> emitter -> mesh fact
%%% @end
-module(division_discovered_v1_to_mesh).
-behaviour(gen_server).

-export([start_link/0, emit/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2, terminate/2]).

-define(TOPIC, <<"hecate.domain.division_discovered">>).

-record(state, {}).

%%====================================================================
%% API
%%====================================================================

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

%% @doc Emit a division_discovered event to mesh
-spec emit(map() | division_discovered_v1:division_discovered_v1()) -> ok.
emit(Event) when is_map(Event) ->
    gen_server:cast(?MODULE, {emit, Event});
emit(Event) ->
    gen_server:cast(?MODULE, {emit, division_discovered_v1:to_map(Event)}).

%%====================================================================
%% gen_server callbacks
%%====================================================================

init([]) ->
    logger:info("[division_discovered_v1_to_mesh] Starting emitter for topic ~s", [?TOPIC]),
    {ok, #state{}}.

handle_cast({emit, EventData}, State) ->
    do_emit(EventData),
    {noreply, State}.

%%====================================================================
%% Internal functions
%%====================================================================

do_emit(EventData) ->
    VentureId = maps:get(<<"venture_id">>, EventData, maps:get(venture_id, EventData, undefined)),
    DivisionId = maps:get(<<"division_id">>, EventData, maps:get(division_id, EventData, undefined)),

    logger:debug("[emitter] Publishing division ~s for domain ~s", [DivisionId, VentureId]),

    %% Publish FACT to mesh
    case hecate_mesh_client:publish(?TOPIC, EventData) of
        ok ->
            logger:info("[emitter] Published to ~s: division=~s", [?TOPIC, DivisionId]);
        {error, not_connected} ->
            logger:warning("[emitter] Mesh not connected, fact not published");
        {error, Reason} ->
            logger:error("[emitter] Failed to publish: ~p", [Reason])
    end.
```

**Key points:**
- Emitter converts domain events to mesh facts
- Uses topic naming: `hecate.{domain}.{event_name}`
- Handles connection failures gracefully
- Does NOT modify the event data (just publishes)

---

## Listener Code Example

From `apps/design_division/src/initiate_division/subscribe_to_division_discovered.erl`:

```erlang
%%% @doc Listener: Subscribe to division_discovered facts from mesh
%%%
%%% Subscribes to mesh topic `hecate.domain.division_discovered`.
%%% When a domain discovers a division, this listener receives the fact
%%% and forwards it to the policy for processing.
%%%
%%% This listener lives in the initiate_division desk because its
%%% sole purpose is to trigger division initiation.
%%%
%%% Flow: Mesh FACT -> Listener -> Policy -> Command -> Aggregate
%%% @end
-module(subscribe_to_division_discovered).
-behaviour(gen_server).

-export([start_link/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2, terminate/2]).

-define(TOPIC, <<"hecate.domain.division_discovered">>).

-record(state, {
    subscription :: reference() | undefined
}).

%%====================================================================
%% API
%%====================================================================

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

%%====================================================================
%% gen_server callbacks
%%====================================================================

init([]) ->
    self() ! subscribe,
    logger:info("[listener] Starting for topic ~s", [?TOPIC]),
    {ok, #state{}}.

handle_info(subscribe, State) ->
    SubRef = subscribe_to_topic(?TOPIC),
    {noreply, State#state{subscription = SubRef}};

handle_info({mesh_fact, ?TOPIC, FactData}, State) ->
    %% Forward to POLICY for processing
    %% Policy will create COMMAND and dispatch to AGGREGATE
    on_division_discovered_maybe_initiate_division:handle(FactData),
    {noreply, State};

handle_info(_Info, State) ->
    {noreply, State}.

terminate(_Reason, #state{subscription = SubRef}) ->
    unsubscribe(SubRef),
    ok.

%%====================================================================
%% Internal functions
%%====================================================================

subscribe_to_topic(Topic) ->
    case hecate_mesh_client:subscribe(Topic, self()) of
        {ok, SubRef} ->
            logger:info("[listener] Subscribed to ~s", [Topic]),
            SubRef;
        {error, not_connected} ->
            %% Retry subscription in 5 seconds
            logger:warning("[listener] Mesh not connected, retrying in 5s"),
            erlang:send_after(5000, self(), subscribe),
            undefined;
        {error, Reason} ->
            logger:error("[listener] Failed to subscribe: ~p", [Reason]),
            undefined
    end.

unsubscribe(undefined) -> ok;
unsubscribe(SubRef) -> hecate_mesh_client:unsubscribe(SubRef).
```

**Key points:**
- Listener subscribes to mesh topics
- Forwards facts to POLICY (not projection!)
- Lives in the desk it triggers (vertical slicing)
- Handles reconnection gracefully

---

## Policy Code Example

The policy decides what to do with the fact:

```erlang
%%% @doc Policy: React to division_discovered facts
%%%
%%% Naming convention: on_{source_event}_{action}_{target}
%%% - Source: division_discovered (from discover_divisions)
%%% - Action: maybe_initiate (policy decision)
%%% - Target: division (in design_division)
%%% @end
-module(on_division_discovered_maybe_initiate_division).

-export([handle/1]).

%% @doc Handle a division_discovered fact and potentially initiate the division
-spec handle(map()) -> ok | {error, term()}.
handle(FactData) ->
    DivisionId = get_field(division_id, FactData),
    VentureId = get_field(venture_id, FactData),
    ContextName = get_field(context_name, FactData),
    Description = get_field(description, FactData),

    logger:debug("[policy] Processing division ~s (~s) from domain ~s",
                [DivisionId, ContextName, VentureId]),

    %% Policy: Always initiate (future: conditional logic here)
    do_initiate(#{
        division_id => DivisionId,
        venture_id => VentureId,
        context_name => ContextName,
        description => Description
    }).

%% @doc Create and dispatch the initiate_division command
do_initiate(Params) ->
    case initiate_division_v1:new(Params) of
        {ok, Cmd} ->
            %% Dispatch to handler (which dispatches to aggregate)
            case maybe_initiate_division:dispatch(Cmd) of
                {ok, _Version, _Events} ->
                    logger:info("[policy] Division ~s initiated",
                               [maps:get(division_id, Params)]),
                    ok;
                {error, Reason} = Error ->
                    logger:error("[policy] Failed to initiate: ~p", [Reason]),
                    Error
            end;
        {error, Reason} = Error ->
            logger:error("[policy] Invalid command: ~p", [Reason]),
            Error
    end.
```

---

## Complete Flow Diagram

```
Agent A (discover_divisions)                 Agent B (design_division)
────────────────────────                     ──────────────────────────

1. User: POST /api/domains/:id/divisions/discover
   ↓
2. discover_division_v1 (COMMAND)
   ↓
3. maybe_discover_division:handle/1
   ↓
4. division_discovered_v1 (DOMAIN EVENT)
   ↓
5. Stored in Domain's event stream
   ↓
6. division_discovered_v1_to_mesh:emit/1 (EMITTER)
   ↓
═══════════════════════════════════════════════════════════════
                      MESH (topic: hecate.domain.division_discovered)
═══════════════════════════════════════════════════════════════
                                                     ↓
7. subscribe_to_division_discovered (LISTENER)
   ↓
8. on_division_discovered_maybe_initiate_division:handle/1 (POLICY)
   ↓
9. initiate_division_v1 (COMMAND)
   ↓
10. maybe_initiate_division:dispatch/1
    ↓
11. division_initiated_v1 (DOMAIN EVENT)
    ↓
12. Stored in Division's event stream
    ↓
13. Projected to read model
```

---

## Desk Structure

Both emitter and listener live in their respective desks:

```
discover_divisions/src/
└── discover_division/                       # Desk owns emitter
    ├── discover_division_v1.erl             # Command
    ├── division_discovered_v1.erl           # Event
    ├── maybe_discover_division.erl          # Handler
    └── division_discovered_v1_to_mesh.erl   # EMITTER

design_division/src/
└── initiate_division/                       # Desk owns listener
    ├── initiate_division_v1.erl             # Command
    ├── division_initiated_v1.erl            # Event
    ├── maybe_initiate_division.erl          # Handler
    ├── initiate_division_desk_sup.erl       # Supervisor
    ├── subscribe_to_division_discovered.erl                   # LISTENER
    └── on_division_discovered_maybe_initiate_division.erl     # POLICY
```

---

## Naming Conventions

| Component | Pattern | Example |
|-----------|---------|---------|
| **Topic** | `{namespace}.{domain}.{fact_name}` | `hecate.domain.division_discovered` |
| **Emitter** | `{event}_to_mesh` | `division_discovered_v1_to_mesh` |
| **Listener** | `subscribe_to_{fact}` | `subscribe_to_division_discovered` |
| **Policy** | `on_{fact}_{action}_{target}` | `on_division_discovered_maybe_initiate_division` |

---

## What NOT To Do

| Antipattern | Why It's Wrong | Correct Approach |
|-------------|----------------|------------------|
| Mesh fact → Projection | Bypasses aggregate | Fact → Command → Event → Projection |
| Domain event → Mesh directly | No transform layer | Event → Emitter → Fact → Mesh |
| Listener in `listeners/` folder | Horizontal thinking | Listener in desk it triggers |
| Central mesh subscriber | God module | Each domain owns its listeners |
| Treating facts as events | Different concepts | Facts external, events internal |

---

## Key Takeaways

1. **FACTS are NOT EVENTS** - External communication uses facts, internal storage uses events
2. **Emitters convert events to facts** - Controlled publication to mesh
3. **Listeners convert facts to commands** - Entry point to command layer
4. **Policy contains the "maybe"** - Business logic for conditional actions
5. **Single source of truth** - Projections read from event store, not mesh
6. **Vertical slicing** - Emitters and listeners live in their desks
7. **Graceful degradation** - Handle mesh disconnection without crashing

---

## Training Note

This example teaches:
- The fundamental distinction between FACTS and EVENTS
- How EMITTERS publish internal events as external facts
- How LISTENERS receive external facts and dispatch internal commands
- Why bypassing the command layer is an antipattern
- Correct placement of mesh integration components (vertical slicing)
- Policy/process manager patterns for cross-domain integration

*Date: 2026-02-08*
*Origin: Hecate daemon mesh integration architecture*
