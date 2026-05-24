# CODEGEN_ERLANG_TEMPLATES.md — Erlang Code Templates

_Complete module templates for generating Division Architecture (Cartwheel) code. Fill variables, generate code._

**Target:** Erlang/OTP with `reckon_evoq`

**Related files:**
- [CODEGEN_ERLANG_CHECKLISTS.md](CODEGEN_ERLANG_CHECKLISTS.md) — Generation checklists
- [CODEGEN_ERLANG_NAMING.md](CODEGEN_ERLANG_NAMING.md) — Naming conventions
- [ANTIPATTERNS_EVENT_SOURCING.md](../../ANTIPATTERNS_EVENT_SOURCING.md) — canonical reckon-db + evoq wiring (MANDATORY for CMD/PRJ)

---

## Service Shell (umbrella root)

For a new `hecate-<svc>` daemon, **DON'T hand-write the shell**.
Use the canonical scaffolder in `hecate-om`:

```bash
cd ~/work/codeberg.org/hecate-services/hecate-om
./scripts/scaffold-service.sh \
    ~/work/codeberg.org/hecate-services/hecate-<svc> \
    hecate-<svc> \
    "One-line description of <svc>"
```

This renders, from `hecate-om/templates/`:

| File | Purpose |
|------|---------|
| `Containerfile` | Multi-stage OTP build image |
| `quadlet/hecate-<svc>.container` | Podman Quadlet unit |
| `manifest.json` | hecate-realm capability declaration |
| `.github/workflows/build-push.yml` | ghcr.io publish on push to main |
| `src/<app>_app.erl` | OTP entry — one line: `hecate_om:boot(<service_module>)` |
| `src/<app>_service.erl` | hecate_om_service callbacks (incl. optional `store_id/0` + `data_dir/0`) |
| `config/sys.config.src` | Canonical reckon_db + evoq + hecate_om blocks |

### What you still write by hand

- `rebar.config` (deps + relx releases — copy from a sibling service)
- `src/<app>.app.src`
- `src/<app>_sup.erl` (top-level sup; if the service has no
  domain-specific children, just leave the child list empty)
- The `apps/<domain>/` CMD/PRJ/QRY apps themselves (see the
  CMD/PRJ/QRY templates below)

### Why `_app.erl` is one line

`hecate_om:boot/1` does everything: persistent-term registration of
the service module, capability advertisement, /health wiring, and
(when the service exports `store_id/0` + `data_dir/0`)
`reckon_db_sup:start_store/1` + `evoq_store_subscription:start_link/1`.
The wiring is in `hecate_om_store`, shipped in hecate_om ≥ 0.3.0.

If the service is **producer-only** (no event store of its own —
e.g. a simulator, a metrics exporter), delete the two optional
callbacks from `_service.erl` AND the `{reckon_db, [...]}` /
`{evoq, [...]}` blocks from `sys.config.src`. The scaffolder
includes them by default since CMD/PRJ is the common case.

---

## CMD Templates

### {domain}\_app.erl

```erlang
-module({domain}_app).
-behaviour(application).

-export([start/2, stop/1]).

start(_StartType, _StartArgs) ->
    {domain}_sup:start_link().

stop(_State) ->
    ok.
```

### {domain}\_sup.erl

```erlang
-module({domain}_sup).
-behaviour(supervisor).

-export([start_link/0, init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

init([]) ->
    Children = [
        %% Shared infrastructure (optional)
        % #{id => {domain}_store,
        %   start => {{domain}_store, start_link, []},
        %   type => worker},

        %% Desks (ADD DESK SUPERVISORS HERE)
        #{id => {command}_desk_sup,
          start => {{command}_desk_sup, start_link, []},
          type => supervisor}
    ],
    {ok, {#{strategy => one_for_one, intensity => 5, period => 10}, Children}}.
```

### {command}\_desk_sup.erl

```erlang
-module({command}_desk_sup).
-behaviour(supervisor).

-export([start_link/0, init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

init([]) ->
    Children = [
        %% Frontdesk: HOPE responder
        #{id => {command}_responder_v1,
          start => {{command}_responder_v1, start_link, []},
          type => worker},

        %% Backoffice: Emitter (PRJ filer to mesh)
        #{id => {event}_to_mesh,
          start => {{event}_to_mesh, start_link, []},
          type => worker}

        %% NOTE: PMs are NOT children of desk sups.
        %% Each PM lives in its own sibling slice `on_{src_event}_{action}_{target}/`
        %% with its own supervisor, started by the domain sup. See Demon 18.
    ],
    {ok, {#{strategy => one_for_one, intensity => 5, period => 10}, Children}}.
```

### {command}\_v1.erl (Command Record)

```erlang
-module({command}_v1).

-export([new/1, new/2, to_map/1, from_map/1]).
-export([stream_id/1]).
%% Getters
-export([{field1}/1, {field2}/1]).

-record({command}_v1, {
    {field1} :: binary(),
    {field2} :: binary(),
    metadata = #{} :: map()
}).

-opaque t() :: #{{command}_v1{}}.
-export_type([t/0]).

%% Constructor
new(#{field1} := Field1, {field2} := Field2}) ->
    #{{command}_v1{
        {field1} = Field1,
        {field2} = Field2
    }}.

new(Field1, Field2) ->
    #{{command}_v1{
        {field1} = Field1,
        {field2} = Field2
    }}.

%% Stream ID for this dossier
stream_id(#{{command}_v1{{field1} = Field1}}) ->
    <<"{domain_noun}-", Field1/binary>>.

%% Serialization
to_map(#{{command}_v1{} = Cmd) ->
    #{
        {field1} => Cmd#{{command}_v1.{field1},
        {field2} => Cmd#{{command}_v1.{field2},
        metadata => Cmd#{{command}_v1.metadata
    }.

from_map(#{<<"{field1}">> := Field1, <<"{field2}">> := Field2} = Map) ->
    #{{command}_v1{
        {field1} = Field1,
        {field2} = Field2,
        metadata = maps:get(<<"metadata">>, Map, #{})
    }}.

%% Getters
{field1}(#{{command}_v1{{field1} = V}}) -> V.
{field2}(#{{command}_v1{{field2} = V}}) -> V.
```

### {event}\_v1.erl (Event Record)

```erlang
-module({event}_v1).

-export([new/1, from_command/1, to_map/1, from_map/1]).
-export([event_type/0]).
%% Getters
-export([{field1}/1, {field2}/1, timestamp/1]).

-record({event}_v1, {
    {field1} :: binary(),
    {field2} :: binary(),
    timestamp :: integer()
}).

-opaque t() :: #{{event}_v1{}}.
-export_type([t/0]).

%% Event type identifier
event_type() -> <<"{event}_v1">>.

%% Constructor from command
from_command(Cmd) ->
    #{{event}_v1{
        {field1} = {command}_v1:{field1}(Cmd),
        {field2} = {command}_v1:{field2}(Cmd),
        timestamp = erlang:system_time(millisecond)
    }}.

new(#{field1} := Field1, {field2} := Field2}) ->
    #{{event}_v1{
        {field1} = Field1,
        {field2} = Field2,
        timestamp = erlang:system_time(millisecond)
    }}.

%% Serialization
to_map(#{{event}_v1{} = E) ->
    #{
        {field1} => E#{{event}_v1.{field1},
        {field2} => E#{{event}_v1.{field2},
        timestamp => E#{{event}_v1.timestamp
    }.

from_map(#{<<"{field1}">> := Field1, <<"{field2}">> := Field2} = Map) ->
    #{{event}_v1{
        {field1} = Field1,
        {field2} = Field2,
        timestamp = maps:get(<<"timestamp">>, Map, 0)
    }}.

%% Getters
{field1}(#{{event}_v1{{field1} = V}}) -> V.
{field2}(#{{event}_v1{{field2} = V}}) -> V.
timestamp(#{{event}_v1{timestamp = V}}) -> V.
```

### maybe\_{command}.erl (Handler)

```erlang
-module(maybe_{command}).

-export([handle/1, handle/2, dispatch/1]).

-include_lib("kernel/include/logger.hrl").

%% Entry point: dispatch command through evoq
dispatch(Cmd) ->
    StreamId = {command}_v1:stream_id(Cmd),
    case handle(Cmd) of
        {ok, Event} ->
            %% Persist event to store
            EventMap = {event}_v1:to_map(Event),
            EventType = {event}_v1:event_type(),
            ok = reckon_evoq:append({domain}_store, StreamId, EventType, EventMap),
            {ok, Event};
        {error, _} = Error ->
            Error
    end.

%% Handle command (stateless validation)
handle(Cmd) ->
    case validate(Cmd) of
        ok ->
            Event = {event}_v1:from_command(Cmd),
            {ok, Event};
        {error, _} = Error ->
            Error
    end.

%% Handle with aggregate state (for invariant checks)
handle(Cmd, AggregateState) ->
    case can_execute(Cmd, AggregateState) of
        true ->
            handle(Cmd);
        {false, Reason} ->
            {error, Reason}
    end.

%% Validation
validate(Cmd) ->
    Field1 = {command}_v1:{field1}(Cmd),
    case byte_size(Field1) > 0 of
        true -> ok;
        false -> {error, {field1}_required}
    end.

%% Business rule check against current state
can_execute(_Cmd, _State) ->
    %% TODO: Add business rules
    true.
```

### {command}\_responder_v1.erl (Frontdesk - HOPE inbox)

```erlang
-module({command}_responder_v1).
-behaviour(gen_server).

-export([start_link/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2, terminate/2]).

-include_lib("kernel/include/logger.hrl").

-define(TOPIC, <<"hecate.{domain_noun}.{command}">>).

%%====================================================================
%% API
%%====================================================================

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

%%====================================================================
%% gen_server callbacks
%%====================================================================

init([]) ->
    %% Subscribe to HOPE topic on mesh
    ok = hecate_mesh:subscribe(?TOPIC),
    ?LOG_INFO("[~s] Responder started, subscribed to ~s", [?MODULE, ?TOPIC]),
    {ok, #{}}.

handle_call(_Request, _From, State) ->
    {reply, {error, not_implemented}, State}.

handle_cast(_Msg, State) ->
    {noreply, State}.

handle_info({mesh_hope, Topic, Hope, ReplyTo}, State) ->
    ?LOG_DEBUG("[~s] Received HOPE on ~s", [?MODULE, Topic]),
    Result = handle_hope(Hope),
    %% Send FEEDBACK
    Feedback = case Result of
        {ok, Event} ->
            #{ok => true, event => {event}_v1:to_map(Event)};
        {error, Reason} ->
            #{ok => false, error => Reason}
    end,
    ok = hecate_mesh:publish(ReplyTo, Feedback),
    {noreply, State};

handle_info(_Info, State) ->
    {noreply, State}.

terminate(_Reason, _State) ->
    ok.

%%====================================================================
%% Internal
%%====================================================================

handle_hope(Hope) ->
    %% Translate HOPE to Command
    Cmd = hope_to_command(Hope),
    %% Dispatch through normal command flow
    maybe_{command}:dispatch(Cmd).

hope_to_command(Hope) ->
    {command}_v1:from_map(Hope).
```

### {event}\_to_mesh.erl (Emitter - subscribes via evoq, publishes to mesh)

**Subscribes to ReckonDB via evoq. NOT called manually from API handlers.**
See [EVENT_SUBSCRIPTION_FLOW.md](../philosophy/EVENT_SUBSCRIPTION_FLOW.md).

```erlang
-module({event}_to_mesh).
-behaviour(gen_server).

-export([start_link/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2, terminate/2]).

-include_lib("kernel/include/logger.hrl").

-define(EVENT_TYPE, <<"{event}">>).
-define(SUB_NAME, <<"{event}_to_mesh">>).
-define(TOPIC, <<"hecate.{domain_noun}.{event}">>).

%%====================================================================
%% API
%%====================================================================

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

%%====================================================================
%% gen_server callbacks
%%====================================================================

init([]) ->
    %% Subscribe to event store via evoq
    {ok, _SubId} = reckon_evoq_adapter:subscribe(
        dev_studio_store,
        event_type,
        ?EVENT_TYPE,
        ?SUB_NAME,
        #{subscriber_pid => self()}
    ),
    ?LOG_INFO("[~s] Emitter started, publishing to ~s", [?MODULE, ?TOPIC]),
    {ok, #{}}.

handle_call(_Request, _From, State) ->
    {reply, {error, not_implemented}, State}.

handle_cast(_Msg, State) ->
    {noreply, State}.

handle_info({events, Events}, State) ->
    lists:foreach(fun(EventData) ->
        %% Transform EVENT to FACT (may have different structure)
        Fact = event_to_fact(EventData),
        %% Publish to mesh
        ok = hecate_mesh:publish(?TOPIC, Fact),
        ?LOG_DEBUG("[~s] Published FACT to ~s", [?MODULE, ?TOPIC])
    end, Events),
    {noreply, State};

handle_info(_Info, State) ->
    {noreply, State}.

terminate(_Reason, _State) ->
    ok.

%%====================================================================
%% Internal
%%====================================================================

%% Transform internal EVENT to external FACT
%% FACT is a PUBLIC CONTRACT - may differ from EVENT structure
event_to_fact(EventData) ->
    #{
        {field1} => maps:get(<<"{field1}">>, EventData),
        {field2} => maps:get(<<"{field2}">>, EventData),
        published_at => erlang:system_time(millisecond)
    }.
```

### Process Manager (Sibling Slice) — `on_{src_event}_{action}_{target}/`

A PM is a sibling slice of desks in the target CMD app. It has its own directory, supervisor, and gen_server. Generate **both files** below and start the supervisor from the **domain sup** (peer to desk sups), not from a desk sup.

#### `on_{src_event}_{action}_{target}_sup.erl`

```erlang
-module(on_{src_event}_{action}_{target}_sup).
-behaviour(supervisor).

-export([start_link/0, init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

init([]) ->
    Children = [
        #{id => on_{src_event}_{action}_{target},
          start => {on_{src_event}_{action}_{target}, start_link, []},
          restart => permanent,
          type => worker}
    ],
    {ok, {#{strategy => one_for_one, intensity => 10, period => 10}, Children}}.
```

#### `on_{src_event}_{action}_{target}.erl`

```erlang
-module(on_{src_event}_{action}_{target}).
-behaviour(gen_server).

-export([start_link/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2, terminate/2]).

-include_lib("kernel/include/logger.hrl").

%%====================================================================
%% API
%%====================================================================

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

%%====================================================================
%% gen_server callbacks
%%====================================================================

init([]) ->
    %% Subscribe to trigger events from another domain
    ok = reckon_evoq:subscribe({trigger_domain}_store, self(), #{
        event_types => [<<"{trigger_event}_v1">>]
    }),
    ?LOG_INFO("[~s] Policy started", [?MODULE]),
    {ok, #{}}.

handle_call(_Request, _From, State) ->
    {reply, {error, not_implemented}, State}.

handle_cast(_Msg, State) ->
    {noreply, State}.

handle_info({evoq_event, _StreamId, EventType, EventData, _Position}, State)
  when EventType =:= <<"{trigger_event}_v1">> ->
    %% Transform trigger event to command
    case should_trigger(EventData) of
        true ->
            Cmd = event_to_command(EventData),
            case maybe_{command}:dispatch(Cmd) of
                {ok, _Event} ->
                    ?LOG_DEBUG("[~s] Triggered {command} from {trigger_event}", [?MODULE]);
                {error, Reason} ->
                    ?LOG_WARNING("[~s] Failed to trigger: ~p", [?MODULE, Reason])
            end;
        false ->
            ok
    end,
    {noreply, State};

handle_info(_Info, State) ->
    {noreply, State}.

terminate(_Reason, _State) ->
    ok.

%%====================================================================
%% Internal
%%====================================================================

should_trigger(_EventData) ->
    %% TODO: Add policy logic
    true.

event_to_command(EventData) ->
    %% Transform trigger event data to command
    {command}_v1:new(#{
        {field1} => maps:get(<<"{trigger_field}">>, EventData),
        {field2} => maps:get(<<"{other_field}">>, EventData, <<>>)
    }).
```

---

## PRJ Templates

### query\_{domain_noun}\_sup.erl

```erlang
-module(query_{domain_noun}_sup).
-behaviour(supervisor).

-export([start_link/0, init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

init([]) ->
    Children = [
        %% Store (SQLite)
        #{id => query_{domain_noun}_store,
          start => {query_{domain_noun}_store, start_link, []},
          type => worker},

        %% PRJ Desks (ADD PROJECTION SUPERVISORS HERE)
        #{id => {event}_to_{read_store}_sup,
          start => {{event}_to_{read_store}_sup, start_link, []},
          type => supervisor}
    ],
    {ok, {#{strategy => one_for_one, intensity => 5, period => 10}, Children}}.
```

### {event}_to_{read_store}\_sup.erl

```erlang
-module({event}_to_{read_store}_sup).
-behaviour(supervisor).

-export([start_link/0, init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

init([]) ->
    Children = [
        #{id => {event}_to_{read_store},
          start => {{event}_to_{read_store}, start_link, []},
          type => worker}
    ],
    {ok, {#{strategy => one_for_one, intensity => 5, period => 10}, Children}}.
```

### {event}_to_{read_store}.erl (Projection)

```erlang
-module({event}_to_{read_store}).
-behaviour(gen_server).

-export([start_link/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2, terminate/2]).

-include_lib("kernel/include/logger.hrl").

%%====================================================================
%% API
%%====================================================================

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

%%====================================================================
%% gen_server callbacks
%%====================================================================

init([]) ->
    %% Subscribe to events from CMD domain store
    ok = reckon_evoq:subscribe({domain}_store, self(), #{
        event_types => [<<"{event}_v1">>]
    }),
    ?LOG_INFO("[~s] Projection started", [?MODULE]),
    {ok, #{}}.

handle_call(_Request, _From, State) ->
    {reply, {error, not_implemented}, State}.

handle_cast(_Msg, State) ->
    {noreply, State}.

handle_info({evoq_event, StreamId, EventType, EventData, Position}, State)
  when EventType =:= <<"{event}_v1">> ->
    ok = project(EventData),
    %% Checkpoint position for recovery
    ok = reckon_evoq:ack({domain}_store, Position),
    {noreply, State};

handle_info(_Info, State) ->
    {noreply, State}.

terminate(_Reason, _State) ->
    ok.

%%====================================================================
%% Internal
%%====================================================================

project(EventData) ->
    Row = #{
        {pk_field} => maps:get(<<"{pk_field}">>, EventData),
        {field1} => maps:get(<<"{field1}">>, EventData),
        {field2} => maps:get(<<"{field2}">>, EventData),
        updated_at => erlang:system_time(millisecond)
    },
    %% Upsert for idempotency
    query_{domain_noun}_store:upsert({read_store}, Row).
```

---

## QRY Templates

### get\_{aggregate}\_by\_id.erl (Single Lookup)

```erlang
-module(get_{aggregate}_by_id).

-export([execute/1]).

-spec execute(binary()) -> {ok, map()} | {error, not_found | term()}.
execute(Id) ->
    Sql = "SELECT {aggregate}_id, name, status, status_label "
          "FROM {aggregates} WHERE {aggregate}_id = ?1",
    case query_{domain_noun}_store:query(Sql, [Id]) of
        {ok, [Row]} ->
            {ok, row_to_map(Row)};
        {ok, []} ->
            {error, not_found};
        {error, Reason} ->
            {error, Reason}
    end.
```

### get\_{aggregates}\_page.erl (Paged List)

```erlang
-module(get_{aggregates}_page).

-include_lib("{cmd_domain}/include/{aggregate}_status.hrl").

-export([execute/1]).

-define(DEFAULT_PAGE_SIZE, 50).
-define(MAX_PAGE_SIZE, 200).

-spec execute(map()) -> {ok, map()} | {error, term()}.
execute(Opts) ->
    Page = maps:get(page, Opts, 1),
    PageSize = min(maps:get(page_size, Opts, ?DEFAULT_PAGE_SIZE), ?MAX_PAGE_SIZE),
    Offset = (Page - 1) * PageSize,

    {ok, [[Total]]} = query_{domain_noun}_store:query(
        "SELECT COUNT(*) FROM {aggregates} WHERE (status & ?1) = 0",
        [?{AGG}_ARCHIVED]),

    Sql = "SELECT {aggregate}_id, name, status, status_label, initiated_at "
          "FROM {aggregates} "
          "WHERE (status & ?1) = 0 "
          "ORDER BY initiated_at DESC "
          "LIMIT ?2 OFFSET ?3",
    case query_{domain_noun}_store:query(Sql, [?{AGG}_ARCHIVED, PageSize, Offset]) of
        {ok, Rows} ->
            {ok, #{
                items => [row_to_map(R) || R <- Rows],
                total => Total,
                page => Page,
                page_size => PageSize
            }};
        {error, Reason} ->
            {error, Reason}
    end.
```

---

## Walking Skeleton Templates

### archive\_{noun}\_v1.erl

```erlang
-module(archive_{noun}_v1).

-export([new/1, to_map/1, from_map/1, event_type/0]).
-export([{noun}_id/1, archived_at/1, archived_by/1, reason/1]).

-record(archive_{noun}_v1, {
    {noun}_id :: binary(),
    archived_at :: integer(),
    archived_by :: binary(),
    reason :: binary() | undefined
}).

event_type() -> <<"archive_{noun}_v1">>.

new(#{{noun}_id := Id, archived_by := By} = Params) ->
    {ok, #archive_{noun}_v1{
        {noun}_id = Id,
        archived_at = erlang:system_time(millisecond),
        archived_by = By,
        reason = maps:get(reason, Params, undefined)
    }}.

to_map(#archive_{noun}_v1{} = E) ->
    #{
        event_type => event_type(),
        {noun}_id => E#archive_{noun}_v1.{noun}_id,
        archived_at => E#archive_{noun}_v1.archived_at,
        archived_by => E#archive_{noun}_v1.archived_by,
        reason => E#archive_{noun}_v1.reason
    }.

from_map(#{<<"{noun}_id">> := Id} = Map) ->
    {ok, #archive_{noun}_v1{
        {noun}_id = Id,
        archived_at = maps:get(<<"archived_at">>, Map, 0),
        archived_by = maps:get(<<"archived_by">>, Map, <<"unknown">>),
        reason = maps:get(<<"reason">>, Map, undefined)
    }}.

{noun}_id(#archive_{noun}_v1{{noun}_id = V}) -> V.
archived_at(#archive_{noun}_v1{archived_at = V}) -> V.
archived_by(#archive_{noun}_v1{archived_by = V}) -> V.
reason(#archive_{noun}_v1{reason = V}) -> V.
```

### {noun}\_archived\_v1\_to\_{noun}s.erl (Archive Projection)

```erlang
-module({noun}_archived_v1_to_{noun}s).

-export([project/1]).

-define(ARCHIVED, 2).  %% Must match aggregate flag

project(EventData) ->
    Id = maps:get(<<"{noun}_id">>, EventData),
    ArchivedAt = maps:get(<<"archived_at">>, EventData),

    %% Update status to include ARCHIVED flag
    Sql = "UPDATE {noun}s SET status = status | ?1, archived_at = ?2 WHERE {noun}_id = ?3",
    query_{noun}s_store:execute(Sql, [?ARCHIVED, ArchivedAt, Id]).
```

---

## Aggregate Template

### {noun}\_aggregate.erl

**Every aggregate MUST declare the evoq behaviour.** The compiler checks callback signatures exist.

```erlang
-module({noun}_aggregate).
-behaviour(evoq_aggregate).

%% Behaviour callbacks
-export([init/1, execute/2, apply/2]).

%% Testing alias
-export([initial_state/0, apply_event/2]).

%% Bit flag display
-export([flag_map/0]).

%% Status bit flags (powers of 2)
-define(INITIATED,    1).   %% 2^0
-define(DNA_ACTIVE,   2).   %% 2^1
-define(DNA_COMPLETE, 4).   %% 2^2
-define(ARCHIVED,    32).   %% 2^5 (soft deleted)

-record({noun}_state, {
    {noun}_id :: binary() | undefined,
    name      :: binary() | undefined,
    status    :: non_neg_integer()
}).

-type state() :: #{noun}_state{}.
-export_type([state/0]).

flag_map() -> #{
    0           => <<"New">>,
    ?INITIATED  => <<"Initiated">>,
    ?ARCHIVED   => <<"Archived">>
}.

%% Behaviour callback: init/1
init(_AggregateId) ->
    {ok, initial_state()}.

initial_state() ->
    #{noun}_state{
        {noun}_id = undefined,
        name = undefined,
        status = 0
    }.

%% CRITICAL: execute(State, Payload) — State FIRST!
execute(State, #{command_type := <<"initiate_{noun}">>} = Payload) ->
    execute_initiate(Payload, State);
execute(State, #{command_type := <<"archive_{noun}">>} = Payload) ->
    execute_archive(Payload, State);
execute(_State, _Payload) ->
    {error, unknown_command}.

%% Guards pattern: check preconditions with bit flags
execute_initiate(_Payload, #{noun}_state{status = 0}) ->
    %% Only uninitiated aggregates can be initiated
    {ok, Cmd} = initiate_{noun}_v1:from_map(_Payload),
    convert_events(maybe_initiate_{noun}:handle(Cmd), fun {noun}_initiated_v1:to_map/1);
execute_initiate(_Payload, _State) ->
    {error, {noun}_already_initiated}.

execute_archive(_Payload, #{noun}_state{{noun}_id = undefined}) ->
    {error, {noun}_not_found};
execute_archive(_Payload, #{noun}_state{status = Status}) when Status band ?ARCHIVED =/= 0 ->
    {error, {noun}_already_archived};
execute_archive(Payload, _State) ->
    {ok, Cmd} = archive_{noun}_v1:from_map(Payload),
    convert_events(maybe_archive_{noun}:handle(Cmd), fun {noun}_archived_v1:to_map/1).

convert_events({ok, Events}, ToMapFun) ->
    {ok, [ToMapFun(E) || E <- Events]};
convert_events({error, Reason}, _ToMapFun) ->
    {error, Reason}.

%% Behaviour callback: apply(State, Event) — State FIRST!
apply(State, Event) ->
    apply_event(Event, State).

%% apply_event has reversed args for testing convenience
%% Handle both atom and binary keys (evoq may pass either)
apply_event(#{<<"event_type">> := <<"{noun}_initiated_v1">>} = E, State) ->
    apply_initiated(E, State);
apply_event(#{event_type := <<"{noun}_initiated_v1">>} = E, State) ->
    apply_initiated(E, State);
apply_event(#{<<"event_type">> := <<"{noun}_archived_v1">>} = _E, State) ->
    apply_archived(State);
apply_event(#{event_type := <<"{noun}_archived_v1">>} = _E, State) ->
    apply_archived(State);
apply_event(_E, State) ->
    State.

apply_initiated(E, State) ->
    State#{noun}_state{
        {noun}_id = get_value({noun}_id, E),
        name = get_value(name, E),
        status = ?INITIATED bor ?DNA_ACTIVE
    }.

apply_archived(#{noun}_state{status = Status} = State) ->
    State#{noun}_state{status = evoq_bit_flags:set(Status, ?ARCHIVED)}.

%% Helper: get value from map with atom or binary keys
get_value(Key, Map) -> get_value(Key, Map, undefined).
get_value(Key, Map, Default) when is_atom(Key) ->
    BinKey = atom_to_binary(Key, utf8),
    case maps:find(Key, Map) of
        {ok, V} -> V;
        error ->
            case maps:find(BinKey, Map) of
                {ok, V} -> V;
                error -> Default
            end
    end.
```

### Aggregate Guard Pattern (Mid-Lifecycle Desks)

Desks that operate between initiation and archive follow a guard chain pattern:

```erlang
%% Iterative update desk (repeatable, no status change)
execute(State, #{command_type := <<"refine_{noun}">>} = Payload) ->
    execute_refine(Payload, State);

execute_refine(_Payload, #{noun}_state{{noun}_id = undefined}) ->
    {error, {noun}_not_found};
execute_refine(_Payload, #{noun}_state{status = S}) when S band ?ARCHIVED =/= 0 ->
    {error, {noun}_archived};
execute_refine(_Payload, #{noun}_state{status = S}) when S band ?DNA_ACTIVE =:= 0 ->
    {error, dna_not_active};
execute_refine(Payload, _State) ->
    {ok, Cmd} = refine_{noun}_v1:from_map(Payload),
    convert_events(maybe_refine_{noun}:handle(Cmd), fun {noun}_refined_v1:to_map/1).

%% State transition desk (one-shot, changes bit flags)
execute(State, #{command_type := <<"submit_{noun}">>} = Payload) ->
    execute_submit(Payload, State);

execute_submit(_Payload, #{noun}_state{{noun}_id = undefined}) ->
    {error, {noun}_not_found};
execute_submit(_Payload, #{noun}_state{status = S}) when S band ?ARCHIVED =/= 0 ->
    {error, {noun}_archived};
execute_submit(_Payload, #{noun}_state{status = S}) when S band ?DNA_ACTIVE =:= 0 ->
    {error, dna_not_active};
execute_submit(_Payload, #{noun}_state{status = S}) when S band ?DNA_COMPLETE =/= 0 ->
    {error, dna_already_complete};
execute_submit(Payload, _State) ->
    {ok, Cmd} = submit_{noun}_v1:from_map(Payload),
    convert_events(maybe_submit_{noun}:handle(Cmd), fun {noun}_submitted_v1:to_map/1).
```

### Partial Update Apply Pattern

When an event carries optional fields, only update state for non-undefined values:

```erlang
apply_refined(E, State) ->
    S1 = case get_value(brief, E) of
        undefined -> State;
        Brief -> State#{noun}_state{brief = Brief}
    end,
    S2 = case get_value(repos, E) of
        undefined -> S1;
        Repos -> S1#{noun}_state{repos = Repos}
    end,
    %% ... chain for each optional field
    S2.
```

### Status Transition Apply Pattern

When an event transitions between phases, set new flag and unset old:

```erlang
apply_submitted(#{noun}_state{status = Status} = State) ->
    NewStatus = evoq_bit_flags:unset(
        evoq_bit_flags:set(Status, ?DNA_COMPLETE),
        ?DNA_ACTIVE
    ),
    State#{noun}_state{status = NewStatus}.
```

---

## Internal Emitter Template (pg)

### {event}\_to\_pg.erl

**For intra-daemon integration** (projections, process managers within the same BEAM VM).
pg emitters are **gen_servers** that subscribe to ReckonDB via evoq at startup.

**They are NOT called manually from API handlers.** See [EVENT_SUBSCRIPTION_FLOW.md](../philosophy/EVENT_SUBSCRIPTION_FLOW.md).

```erlang
-module({event}_to_pg).
-behaviour(gen_server).

-export([start_link/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2, terminate/2]).

-define(EVENT_TYPE, <<"{event}">>).
-define(PG_GROUP, {event}).
-define(SUB_NAME, <<"{event}_to_pg">>).

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
        pg:send(pg, ?PG_GROUP, {?PG_GROUP, Event})
    end, Events),
    {noreply, State};
handle_info(_Info, State) ->
    {noreply, State}.

handle_call(_Req, _From, State) -> {reply, ok, State}.
handle_cast(_Msg, State) -> {noreply, State}.
terminate(_Reason, _State) -> ok.
```

---

## rebar.config Template

```erlang
{erl_opts, [debug_info]}.

{deps, [
    {reckon_evoq, {git, "https://github.com/reckon-db-org/reckon_evoq.git", {branch, "main"}}}
]}.

%% Include desk directories
{src_dirs, [
    "src",
    "src/{command1}",
    "src/{command2}",
    "src/{event1}_to_{read_store1}",
    "src/get_{aggregate}_by_id",
    "src/get_{aggregates}_page"
]}.
```

---

## Test Templates

### CRITICAL: Every aggregate MUST have tests before push.

### {noun}\_aggregate\_tests.erl

```erlang
-module({noun}_aggregate_tests).
-include_lib("eunit/include/eunit.hrl").

%%====================================================================
%% State Helpers
%%====================================================================

%% Build an initiated aggregate state for testing mid-lifecycle spokes
initiated_state() ->
    Initial = {noun}_aggregate:initial_state(),
    {ok, [EventMap]} = {noun}_aggregate:execute(Initial, #{
        command_type => <<"initiate_{noun}">>,
        {noun}_id => <<"test-123">>,
        name => <<"Test">>
    }),
    {noun}_aggregate:apply_event(EventMap, Initial).

%%====================================================================
%% Argument Order Tests (CRITICAL — catches the #1 bug)
%%====================================================================

execute_argument_order_test() ->
    State = {noun}_aggregate:initial_state(),
    Payload = #{
        command_type => <<"initiate_{noun}">>,
        {noun}_id => <<"test-123">>,
        name => <<"Test">>
    },
    Result = {noun}_aggregate:execute(State, Payload),
    ?assertMatch({ok, [_]}, Result),
    {ok, [EventMap]} = Result,
    ?assertEqual(<<"{noun}_initiated_v1">>, maps:get(<<"event_type">>, EventMap)).

unknown_command_test() ->
    State = {noun}_aggregate:initial_state(),
    ?assertEqual({error, unknown_command},
        {noun}_aggregate:execute(State, #{command_type => <<"bogus">>})).

%%====================================================================
%% Guard Tests (verify business rules)
%%====================================================================

double_initiate_test() ->
    State = initiated_state(),
    Result = {noun}_aggregate:execute(State, #{
        command_type => <<"initiate_{noun}">>,
        {noun}_id => <<"test-456">>
    }),
    ?assertEqual({error, {noun}_already_initiated}, Result).

archive_uninitiated_test() ->
    State = {noun}_aggregate:initial_state(),
    Result = {noun}_aggregate:execute(State, #{
        command_type => <<"archive_{noun}">>,
        {noun}_id => <<"test-123">>
    }),
    ?assertEqual({error, {noun}_not_found}, Result).

%%====================================================================
%% Status Transition Tests (verify bit flags)
%%====================================================================

archive_sets_flag_test() ->
    State = initiated_state(),
    {ok, [EventMap]} = {noun}_aggregate:execute(State, #{
        command_type => <<"archive_{noun}">>,
        {noun}_id => <<"test-123">>
    }),
    NewState = {noun}_aggregate:apply_event(EventMap, State),
    %% ARCHIVED (32) must be set
    Status = element(4, NewState),  %% #state.status
    ?assertNotEqual(0, Status band 32).
```

### Emitter Tests

```erlang
-module({event}_to_pg_tests).
-include_lib("eunit/include/eunit.hrl").

-define(GROUP, {event}).
-define(SCOPE, pg).

emit_test() ->
    ensure_pg(),
    ok = pg:join(?SCOPE, ?GROUP, self()),
    Event = #{id => <<"test">>},
    ok = {event}_to_pg:emit(Event),
    receive
        {{event}, Received} -> ?assertEqual(Event, Received)
    after 1000 -> ?assert(false)
    end,
    ok = pg:leave(?SCOPE, ?GROUP, self()).

ensure_pg() ->
    case pg:start(?SCOPE) of
        {ok, _} -> ok;
        {error, {already_started, _}} -> ok
    end.
```

---

## API Handler Templates

API handlers are Cowboy `init/2` handlers that live **inside desk directories**, not in `hecate_api`.
All handlers use `hecate_api_utils` from the `shared` app.

### CMD API Handler Template — {command}\_api.erl

For command desks (POST endpoints that dispatch commands via `maybe_*`).
Each handler exports `routes/0` — the central aggregator discovers them automatically.

```erlang
-module({command}_api).
-export([init/2, routes/0]).

routes() ->
    [{"/api/{domain}/{action}", ?MODULE, []}].

init(Req0, State) ->
    case cowboy_req:method(Req0) of
        <<"POST">> -> handle_post(Req0, State);
        _ -> hecate_api_utils:method_not_allowed(Req0)
    end.

handle_post(Req0, _State) ->
    case hecate_api_utils:read_json_body(Req0) of
        {ok, Params, Req1} ->
            do_command(Params, Req1);
        {error, invalid_json, Req1} ->
            hecate_api_utils:bad_request(<<"Invalid JSON">>, Req1)
    end.

do_command(Params, Req) ->
    CmdParams = #{
        %% Extract fields from Params here
        field_a => maps:get(<<"field_a">>, Params, undefined),
        field_b => maps:get(<<"field_b">>, Params, undefined)
    },
    case {command}_v1:new(CmdParams) of
        {ok, Cmd} ->
            dispatch_result(maybe_{command}:dispatch(Cmd), Req);
        {error, Reason} ->
            hecate_api_utils:json_error(400, Reason, Req)
    end.

dispatch_result({ok, Version, Events}, Req) ->
    hecate_api_utils:json_ok(201, #{version => Version, events => Events}, Req);
dispatch_result({error, Reason}, Req) ->
    hecate_api_utils:json_error(400, Reason, Req).
```

**Variations:**

- **URL binding** (e.g., `:learning_id`): Extract with `cowboy_req:binding(learning_id, Req0)` before reading body
- **No body needed** (e.g., validate, endorse): Skip `read_json_body`, just extract binding and dispatch
- **Custom status code**: Use `json_ok(201, ...)` for creation, `json_ok(Result, Req)` for 200

### QRY Paged API Handler Template — get\_{nouns}\_page\_api.erl

For paged list queries (GET endpoints with optional filters).
Each handler exports `routes/0` — the central aggregator discovers them automatically.

```erlang
-module(get_{nouns}_page_api).
-export([init/2, routes/0]).

routes() ->
    [{"/api/{domain}", ?MODULE, []}].

init(Req0, State) ->
    case cowboy_req:method(Req0) of
        <<"GET">> -> handle_get(Req0, State);
        _ -> hecate_api_utils:method_not_allowed(Req0)
    end.

handle_get(Req0, _State) ->
    QS = cowboy_req:parse_qs(Req0),
    Filters = build_filters(QS),
    case get_{nouns}_page:execute(Filters) of
        {ok, Result} ->
            hecate_api_utils:json_ok(#{{nouns} => Result}, Req0);
        {error, Reason} ->
            hecate_api_utils:json_error(500, Reason, Req0)
    end.

build_filters(QS) ->
    lists:foldl(fun({K, V}, Acc) ->
        case K of
            <<"limit">> -> safe_int(V, limit, Acc);
            <<"offset">> -> safe_int(V, offset, Acc);
            %% Add domain-specific filters here
            _ -> Acc
        end
    end, #{}, QS).

safe_int(V, Key, Acc) ->
    case catch binary_to_integer(V) of
        I when is_integer(I) -> Acc#{Key => I};
        _ -> Acc
    end.
```

### QRY By-ID API Handler Template — get\_{noun}\_by\_id\_api.erl

For single-record lookups (GET endpoints with URL binding).
Each handler exports `routes/0` — the central aggregator discovers them automatically.

```erlang
-module(get_{noun}_by_id_api).
-export([init/2, routes/0]).

routes() ->
    [{"/api/{domain}/:{noun}_id", ?MODULE, []}].

init(Req0, State) ->
    case cowboy_req:method(Req0) of
        <<"GET">> -> handle_get(Req0, State);
        _ -> hecate_api_utils:method_not_allowed(Req0)
    end.

handle_get(Req0, _State) ->
    Id = cowboy_req:binding({noun}_id, Req0),
    case get_{noun}_by_id:execute(Id) of
        {ok, {Noun}} ->
            hecate_api_utils:json_ok(#{{noun} => {Noun}}, Req0);
        {error, not_found} ->
            hecate_api_utils:not_found(Req0);
        {error, Reason} ->
            hecate_api_utils:json_error(500, Reason, Req0)
    end.
```

### Route Ownership (Auto-Discovery)

**Each handler exports `routes/0`.** There are no centralized route files.
The aggregator (`hecate_api_routes.erl`) discovers handlers automatically via OTP module introspection.

**Adding a new endpoint requires touching exactly ONE file** — the handler itself.

All routes MUST use the `/api/` prefix.

**Route conventions:**
- `POST /api/{domain}/{verb}` — commands (e.g., `/api/social/follow`)
- `GET /api/{domain}` — paged list (e.g., `/api/capabilities`)
- `GET /api/{domain}/:id` — single lookup (e.g., `/api/capabilities/:mri`)
- `POST /api/{domain}/:id/{verb}` — commands on existing aggregates (e.g., `/api/divisions/:id/transition`)

See [ANTIPATTERNS_STRUCTURE.md](../../ANTIPATTERNS_STRUCTURE.md) Demon #25 for why centralized route files are wrong.

---

_Templates are deterministic. Fill in variables. Generate code. No creativity required._
