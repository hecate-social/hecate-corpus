# Example: Process Manager Patterns

*Canonical example: Cross-domain coordination without tight coupling*

> **Note:** This document has been updated to use current terminology (2026-02-10).
> `torch` -> `venture`, `cartwheel` -> `division`, `spoke` -> `desk`.

---

## The Pattern

A **Process Manager** (also called Policy or Saga) coordinates actions across domain boundaries:

1. **Subscribes** to events from a source domain
2. **Makes decisions** about what should happen next
3. **Dispatches commands** to a target domain

This enables:
- **Loose coupling** between domains
- **Explicit integration points** (easy to find and reason about)
- **Testable domains** (each domain can be tested in isolation)

---

## The Problem

When Domain A needs to trigger actions in Domain B, the naive approach is:

```
Domain A event → Domain A handler → DIRECT CALL to Domain B
```

This creates **tight coupling**. Domain A must know:
- Domain B's command structure
- Domain B's aggregate interface
- How to construct valid commands for Domain B

---

## Wrong Way (Direct Cross-Domain Calls)

```erlang
%% In discover_divisions/src/discover_division/maybe_discover_division.erl
%% ❌ WRONG: Handler directly calls another domain

handle(Cmd) ->
    %% Handle our domain event
    Event = division_discovered_v1:new(Cmd),
    ok = store_event(Event),

    %% ❌ WRONG: Direct call to another domain!
    DivisionCmd = initiate_division_v1:new(#{
        division_id => Event#division_discovered_v1.division_id,
        venture_id => Event#division_discovered_v1.venture_id
    }),
    maybe_initiate_division:handle(DivisionCmd),  %% ❌ TIGHT COUPLING

    {ok, [Event]}.
```

**Problems:**

| Issue | Impact |
|-------|--------|
| Domain A knows Domain B's API | Changes to B require changes to A |
| Can't test A without B | Slower, more complex tests |
| Circular dependencies possible | Compilation errors, confusion |
| Hidden integration points | Hard to understand data flow |
| No policy decisions | Always creates division, no conditions |

---

## Correct Way (Process Manager)

```
Domain A (discover_divisions)
    ↓ stores division_discovered_v1 event
    ↓ emitter publishes fact to mesh (optional)

Process Manager (on_division_discovered_maybe_initiate_division)
    ↓ subscribes to division_discovered facts
    ↓ policy: decides IF and HOW to initiate
    ↓ creates initiate_division_v1 command
    ↓ dispatches to target aggregate

Domain B (design_division)
    ↓ receives command
    ↓ stores division_initiated_v1 event
```

**Benefits:**

| Benefit | Why |
|---------|-----|
| Loose coupling | Domains don't know about each other |
| Clear integration points | PM is the only connection |
| Testable in isolation | Mock the PM's dependencies |
| Policy decisions explicit | "maybe" in the name = conditional logic |
| Easy to find | Naming convention reveals purpose |

---

## Naming Convention

```
on_{source_event}_{action}_{target}
```

| Component | Meaning | Example |
|-----------|---------|---------|
| `on_` | Triggered by | Prefix |
| `{source_event}` | What happened | `division_discovered` |
| `{action}` | What we do | `maybe_initiate` |
| `{target}` | What we affect | `division` |

**Examples:**

| Name | Source | Action | Target |
|------|--------|--------|--------|
| `on_division_discovered_maybe_initiate_division` | division_discovered | maybe_initiate | division |
| `on_user_registered_send_welcome_email` | user_registered | send | welcome_email |
| `on_order_placed_reserve_inventory` | order_placed | reserve | inventory |
| `on_payment_received_fulfill_order` | payment_received | fulfill | order |
| `on_capability_announced_update_registry` | capability_announced | update | registry |

**"Maybe" indicates policy decision:** The process manager may choose NOT to act based on conditions.

---

## Location Rule

**Process managers live in the TARGET domain as SIBLING SLICES**, never nested inside the target desk's directory.

```
design_division/src/                                    # TARGET domain
├── initiate_division/                                  # The desk that handles the command
│   ├── initiate_division_v1.erl
│   ├── division_initiated_v1.erl
│   ├── maybe_initiate_division.erl
│   └── initiate_division_api.erl
│
└── on_division_discovered_initiate_division/           # The PM — SIBLING slice
    ├── on_division_discovered_initiate_division_sup.erl
    └── on_division_discovered_initiate_division.erl   # gen_server: pg:join + dispatch
```

**Why sibling, not nested?**

1. **PMs are cross-slice.** A PM is an integration point between two domains, not a sub-feature of a single desk. Nesting it inside a desk hides the cross-cutting nature.
2. **Business process flow is discoverable at the filesystem level.** When you `ls src/`, the `on_*` directories scream which external events this domain reacts to. Embedding them inside desks buries that information.
3. **One PM may dispatch to multiple desks.** A cancellation cascade, a fan-out reprice, an evacuation force-settle — these don't map 1:1 to a single desk. Embedding in one desk would arbitrarily privilege that desk.
4. **Same justification as `_api` handlers being co-located with desks (see Demon 14):** the structure should reflect what the code is for, not what it touches.

**Why target domain (not source)?**
- PM needs to know how to construct target commands.
- PM is a consumer of source events, producer of target commands.
- Source domain shouldn't know who reacts to its events.

> **Decision recorded 2026-03-12, reinforced 2026-05-24.** Earlier versions of this doc placed the PM inside the target desk. That guidance was reversed. The sibling-slice pattern is canonical. See [ANTIPATTERNS_STRUCTURE.md Demon 18](../skills/ANTIPATTERNS_STRUCTURE.md#-demon-18-process-managers-inside-desks).

---

## Complete Code Example

The PM is a single gen_server that joins the source domain's pg scope in `init/1` and dispatches the target command on each event. No separate "listener" module is needed.

### 1. The Process Manager (Single Module)

```erlang
%%% @doc Process Manager: React to division_discovered events and initiate divisions.
%%%
%%% Subscribes to the source domain's pg scope (`discover_divisions`)
%%% and dispatches `initiate_division_v1` for each discovered division.
%%%
%%% Naming convention: on_{source_event}_{action}_{target}
%%% - Source: division_discovered (from discover_divisions)
%%% - Action: initiate (unconditional; would be `maybe_initiate` if conditional)
%%% - Target: division (in design_division)
%%% @end
-module(on_division_discovered_initiate_division).
-behaviour(gen_server).

-export([start_link/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2, terminate/2]).

-include_lib("kernel/include/logger.hrl").

-define(SCOPE, discover_divisions).
-define(TOPIC, <<"division_discovered">>).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

init([]) ->
    ok = pg:join(?SCOPE, ?TOPIC, self()),
    ?LOG_INFO("[~s] joined scope=~s topic=~s", [?MODULE, ?SCOPE, ?TOPIC]),
    {ok, #{}}.

handle_call(_Request, _From, State) ->
    {reply, {error, unknown_call}, State}.

handle_cast(_Msg, State) ->
    {noreply, State}.

handle_info({evoq_event, #{event_data := EventData}}, State) ->
    spawn(fun() -> dispatch_initiate(EventData) end),
    {noreply, State};
handle_info(Other, State) ->
    ?LOG_WARNING("[~s] unexpected message: ~p", [?MODULE, Other]),
    {noreply, State}.

terminate(_Reason, _State) ->
    ok.

%%====================================================================
%% Internal
%%====================================================================

dispatch_initiate(EventData) ->
    Params = #{
        division_id => get_field(division_id, EventData),
        venture_id => get_field(venture_id, EventData),
        context_name => get_field(context_name, EventData),
        description => get_field(description, EventData)
    },
    case initiate_division_v1:new(Params) of
        {ok, Cmd} ->
            log_result(Params, maybe_initiate_division:dispatch(Cmd));
        {error, _} = Err ->
            ?LOG_ERROR("[~s] bad params: ~p", [?MODULE, Err])
    end.

log_result(#{division_id := Id}, {ok, _Version, _Events}) ->
    ?LOG_INFO("[~s] initiated division ~s", [?MODULE, Id]);
log_result(#{division_id := Id}, {error, Reason}) ->
    ?LOG_ERROR("[~s] failed to initiate division ~s: ~p", [?MODULE, Id, Reason]).

get_field(Key, Map) when is_atom(Key) ->
    BinKey = atom_to_binary(Key, utf8),
    maps:get(Key, Map, maps:get(BinKey, Map, undefined)).
```

The PM **spawns a worker** for the actual dispatch so the gen_server mailbox keeps draining. This matters when the dispatch path makes any blocking call (read model lookup, mesh RPC).

### 2. The PM's Own Supervisor (the Slice's Sup)

```erlang
%%% @doc Supervisor for the on_division_discovered_initiate_division slice.
%%% Owns the PM gen_server. One-for-one, permanent restart.
%%% @end
-module(on_division_discovered_initiate_division_sup).
-behaviour(supervisor).

-export([start_link/0, init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

init([]) ->
    Children = [
        #{id => on_division_discovered_initiate_division,
          start => {on_division_discovered_initiate_division, start_link, []},
          restart => permanent,
          type => worker}
    ],
    {ok, {{one_for_one, 10, 10}, Children}}.
```

### 3. Wiring the Slice into the Domain Sup

The target domain's CMD app supervisor starts each PM slice's supervisor alongside the desk supervisors. PMs are first-class slices, not children of desks.

```erlang
%%% design_division CMD app: domain supervisor
init([]) ->
    Children = [
        %% Desks
        #{id => initiate_division_desk_sup,
          start => {initiate_division_desk_sup, start_link, []},
          type => supervisor},

        %% PM slices (cross-domain integration points)
        #{id => on_division_discovered_initiate_division_sup,
          start => {on_division_discovered_initiate_division_sup, start_link, []},
          type => supervisor}
    ],
    {ok, {#{strategy => one_for_one, intensity => 10, period => 10}, Children}}.
```

---

## Process Manager with Conditional Logic

```erlang
%% Example: Only initiate for certain division types

handle(FactData) ->
    ContextName = get_field(context_name, FactData),
    case should_auto_initiate(ContextName) of
        true ->
            do_initiate(extract_params(FactData));
        false ->
            logger:info("[policy] Skipping auto-initiation for ~s", [ContextName]),
            ok
    end.

should_auto_initiate(<<"authentication", _/binary>>) -> true;
should_auto_initiate(<<"core_", _/binary>>) -> true;
should_auto_initiate(_) -> false.  %% Manual initiation required
```

---

## Anti-Patterns to Avoid

| Anti-Pattern | Why It's Wrong | Correct Approach |
|--------------|----------------|------------------|
| Handler calls other domain | Tight coupling | Use Process Manager |
| PM in source domain | Source knows too much about target | PM in target domain |
| PM nested inside target desk | Hides the cross-slice integration point | PM as sibling slice with own `_sup` |
| PM without "maybe" in name when conditional | Hides conditional nature | Include "maybe" if conditional; omit if always-act |
| Direct event passing | Bypasses command validation | Create proper command |
| Global event bus subscription | Hidden dependencies | Explicit pg join in PM's `init/1` |
| Inline dispatch in PM's `handle_info` | Blocks gen_server mailbox | Spawn worker for dispatch |

---

## Supervision Hierarchy

```
design_division_sup (domain supervisor)
├── initiate_division_desk_sup                          (desk supervisor)
│   └── initiate_division workers
├── on_division_discovered_initiate_division_sup        (PM slice supervisor)
│   └── on_division_discovered_initiate_division        (gen_server: pg + dispatch)
├── on_lot_evacuated_force_settle_division_sup          (another PM slice)
│   └── on_lot_evacuated_force_settle_division
└── ...
```

PMs are first-class siblings of desks under the domain supervisor. Each PM slice owns its own supervisor and gen_server.

---

## Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                   DOMAIN A (discover_divisions)                  │
│                                                                 │
│  Command: discover_division_v1                                  │
│      ↓                                                          │
│  Handler: maybe_discover_division                               │
│      ↓                                                          │
│  Event: division_discovered_v1 → stored in Venture stream       │
│      ↓                                                          │
│  Emitter: division_discovered_v1_to_mesh → publishes FACT       │
└─────────────────────────────────────────────────────────────────┘
                                ↓
                    ════════════════════════════
                         MESH (loose coupling)
                         Topic: hecate.venture.division_discovered
                    ════════════════════════════
                                ↓
┌─────────────────────────────────────────────────────────────────┐
│                   DOMAIN B (design_division)                    │
│                                                                 │
│  PM slice: on_division_discovered_initiate_division             │
│      ↓ gen_server pg:joins source scope                         │
│      ↓ receives FACT, spawns worker                             │
│      ↓ worker decides + creates command                         │
│  Command: initiate_division_v1                                  │
│      ↓                                                          │
│  Handler: maybe_initiate_division                               │
│      ↓                                                          │
│  Event: division_initiated_v1 → stored in Division stream       │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

1. **Domains don't call each other directly** - Process managers bridge the gap
2. **PM lives in TARGET domain as a SIBLING slice** - own directory, own `_sup`, own gen_server
3. **`on_*` directories scream business process flow at the filesystem level** - this is the primary justification for the sibling-slice rule
4. **Naming convention reveals purpose** - `on_{event}_{action}_{target}`
5. **"Maybe" indicates conditional policy** - omit when the PM always acts
6. **PM is a gen_server that joins pg in `init/1` and dispatches** - no separate listener module
7. **Dispatch runs in a spawned worker** - keeps the gen_server mailbox draining under load
8. **Loose coupling enables testing** - each domain testable in isolation

---

## When to Use Process Managers

| Scenario | Use PM? | Reason |
|----------|---------|--------|
| Domain A event triggers Domain B | Yes | Cross-domain coordination |
| Same-domain event triggers action | No | Handler can call directly |
| Complex multi-step workflow | Yes | Saga pattern |
| Conditional cross-domain action | Yes | Policy decisions |
| Simple 1:1 mapping | Yes | Still provides loose coupling |

---

## Training Note

This example teaches:
- Process Manager / Policy / Saga pattern
- Cross-domain coordination without coupling
- Naming conventions for process managers
- Location rule (PM in target domain)
- Vertical slicing of PM + Listener
- Flow from source event to target command

*Date: 2026-02-08*
*Origin: Hecate Venture → Division integration*
