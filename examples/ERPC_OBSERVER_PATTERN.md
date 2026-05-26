---
title: erpc Observer Pattern
layer: example
audience: [agent, human]
stage: stable
---

# erpc Observer Pattern

_A gen_server that polls another BEAM node for state via erpc._

**Date:** 2026-03-01
**Used in:** `hecate-apps/hecate-app-meshview/hecate-app-meshviewd/src/mesh_observer.erl`

---

## The Problem

You need to read state from another BEAM node (e.g. hecate-daemon)
without that node knowing about you. One-way dependency.

## The Solution

A gen_server that:
1. Discovers the target node via `net_adm:ping/1`
2. Polls on a timer via `erpc:call/4`
3. Caches results in state
4. Exposes cached state through a public API

## Template

```erlang
-module(my_observer).
-behaviour(gen_server).

-export([start_link/0, get_snapshot/0]).
-export([init/1, handle_info/2, handle_call/3, handle_cast/2, terminate/2]).

-define(POLL_MS, 10000).

-record(state, {
    target_node :: atom() | undefined,
    connected :: boolean(),
    snapshot :: map(),
    poll_ref :: reference() | undefined
}).

%%% Public API

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

-spec get_snapshot() -> map().
get_snapshot() ->
    gen_server:call(?MODULE, get_snapshot).

%%% Callbacks

init([]) ->
    Ref = erlang:send_after(0, self(), poll),
    {ok, #state{
        target_node = undefined,
        connected = false,
        snapshot = #{},
        poll_ref = Ref
    }}.

handle_call(get_snapshot, _From, State) ->
    {reply, State#state.snapshot, State};
handle_call(_Req, _From, State) ->
    {reply, {error, unknown}, State}.

handle_cast(_Msg, State) ->
    {noreply, State}.

handle_info(poll, State) ->
    NewState = do_poll(State),
    Ref = erlang:send_after(?POLL_MS, self(), poll),
    {noreply, NewState#state{poll_ref = Ref}};
handle_info(_Msg, State) ->
    {noreply, State}.

terminate(_Reason, _State) ->
    ok.

%%% Internal

do_poll(State) ->
    case ensure_target(State) of
        {ok, Node, State1} ->
            Snapshot = collect_data(Node),
            State1#state{connected = true, snapshot = Snapshot};
        {error, _Reason, State1} ->
            State1#state{connected = false, snapshot = #{}}
    end.

ensure_target(#state{target_node = undefined} = State) ->
    discover_target(State);
ensure_target(#state{target_node = Node} = State) ->
    case net_adm:ping(Node) of
        pong -> {ok, Node, State};
        pang -> discover_target(State#state{target_node = undefined})
    end.

discover_target(State) ->
    %% Resolve target node from own hostname
    Hostname = hostname(),
    TargetNode = list_to_atom("hecate@" ++ Hostname),
    case net_adm:ping(TargetNode) of
        pong ->
            logger:info("[my_observer] Connected to ~p", [TargetNode]),
            {ok, TargetNode, State#state{target_node = TargetNode}};
        pang ->
            {error, unreachable, State}
    end.

hostname() ->
    case string:split(atom_to_list(node()), "@") of
        [_Name, Host] -> Host;
        _ -> net_adm:localhost()
    end.

collect_data(Node) ->
    %% Call whatever functions you need on the target node
    Value = safe_erpc(Node, some_module, some_function, []),
    #{value => Value, polled_at => erlang:system_time(millisecond)}.

safe_erpc(Node, Mod, Fun, Args) ->
    try
        erpc:call(Node, Mod, Fun, Args, 5000)
    catch
        error:{erpc, Reason} ->
            logger:debug("[my_observer] erpc ~p:~p/~p failed: ~p",
                        [Mod, Fun, length(Args), Reason]),
            {error, Reason};
        Class:Reason ->
            logger:debug("[my_observer] erpc ~p:~p/~p crashed: ~p:~p",
                        [Mod, Fun, length(Args), Class, Reason]),
            {error, {Class, Reason}}
    end.
```

## Key Design Decisions

### Why erpc, not rpc

| `rpc:call/4` | `erpc:call/4` |
|--------------|---------------|
| Returns `{badrpc, timeout}` | Raises `error:{erpc, timeout}` |
| Silent failures if not matched | Explicit catch required |
| Legacy API | Modern replacement (OTP 23+) |

### Why poll, not subscribe

| Subscribe (pg/pubsub) | Poll (timer) |
|------------------------|--------------|
| Source must publish | Source needs no changes |
| Real-time updates | Configurable interval |
| Coupling in source | Zero coupling |
| Complex error handling | Simple retry on next tick |

Polling is the correct choice when:
- The source should not know about observers
- Latency of seconds is acceptable
- You want zero-coupling between nodes

### Why gen_server state, not ETS

| ETS | gen_server state |
|-----|------------------|
| Concurrent reads | Serialized reads |
| Survives process restart | Lost on restart |
| More complex | Simpler |

For a single observer with infrequent reads (frontend polls every 5s),
gen_server state is simpler and sufficient. Graduate to ETS if profiling
shows the gen_server mailbox becoming a bottleneck.

## Graceful Degradation Rules

1. **Never crash** when target is unreachable -- return empty/stale state
2. **Retry discovery** on next poll tick, not immediately
3. **Keep last known state** when a single poll fails
4. **Frontend shows connection status** -- no error dialogs for transient disconnects
5. **Observer can start before target** -- it will discover when target appears

## Trust Level Note

The erpc observer pattern requires BEAM clustering (shared cookie,
`net_adm:ping/1`). This places the plugin in the **trusted** tier.

When the session-scoped cookie system is implemented, untrusted plugins
will poll via HTTP instead of erpc. The observer gen_server pattern still
applies -- only the transport changes from `erpc:call/4` to an HTTP GET.

See [PLUGIN_SECURITY_MODEL.md](../philosophy/PLUGIN_SECURITY_MODEL.md).

---

## Pairing with a Cowboy Endpoint

```erlang
-module(my_status_api).
-export([init/2]).

init(Req0, _State) ->
    case cowboy_req:method(Req0) of
        <<"GET">> ->
            Snapshot = my_observer:get_snapshot(),
            Body = json:encode(maps:merge(#{ok => true}, Snapshot)),
            Req = cowboy_req:reply(200,
                #{<<"content-type">> => <<"application/json">>},
                Body, Req0),
            {ok, Req, []};
        _ ->
            Req = cowboy_req:reply(405,
                #{<<"content-type">> => <<"application/json">>},
                <<"{\"ok\":false,\"error\":\"method_not_allowed\"}">>, Req0),
            {ok, Req, []}
    end.
```
