---
title: Testing Erlang Templates (TnI Guidelines)
layer: skill
audience: [agent, human]
stage: stable
---

# Testing Erlang Templates (TnI Guidelines)

Parameterized test templates for Hecate domain lifecycle apps.
Proven through `setup_venture` + `query_ventures` pilot (100 tests, 4 layers).

---

## 4 Testing Layers

| Layer | What | Scope | Framework | I/O |
|-------|------|-------|-----------|-----|
| **L1 Dossier** | Aggregate state reconstruction | `apply_event/2` chains | EUnit | None |
| **L2 Domain** | Commands, handlers, execute/2 state machine | Business rules | EUnit | None |
| **L3 Integration** | CMD to PRJ flow | Projection + Query | EUnit + SQLite | SQLite (temp) |
| **L4a pg Emission** | pg broadcast delivery | Observable effects | EUnit + pg | pg scope |
| **L4b Projection Rows** | Projection row verification | Column values, labels | EUnit + SQLite | SQLite (temp) |
| **L4c Dispatch** | ReckonDB event persistence | `dispatch/1` to store | EUnit + ReckonDB | ReckonDB (temp store) |

---

## Layer 1: Dossier Tests

**File:** `apps/{CmdApp}/test/{Aggregate}_tests.erl`

**What:** Pure functions only. No processes. Test that `apply_event/2` chains reconstruct state correctly.

### Template Variables

| Variable | Description |
|----------|-------------|
| `{Aggregate}` | Aggregate module (e.g., `setup_aggregate`) |
| `{StatusHrl}` | Status header include (e.g., `setup_venture/include/venture_status.hrl`) |
| `{StateRecord}` | Record name (e.g., `setup_state`) |
| `{Events}` | List of `{event_type_binary, flag_bit}` pairs |
| `{FlagMap}` | Macro for the flag map (e.g., `?VENTURE_FLAG_MAP`) |

### Template

```erlang
-module({Aggregate}_tests).
-include_lib("eunit/include/eunit.hrl").
-include_lib("{StatusHrl}").

%% Record field positions (record tag is element 1)
-define(F_STATUS, {StatusFieldPosition}).  %% Adjust per record

%% --- Fixtures ---
%% One fixture function per event type returning a map with binary keys

%% --- Tests ---

initial_state_test() ->
    S = {Aggregate}:initial_state(),
    ?assertEqual(0, element(?F_STATUS, S)).

init_callback_test() ->
    {ok, S} = {Aggregate}:init(<<"any-id">>),
    ?assertEqual(0, element(?F_STATUS, S)).

%% One test per event: apply event, verify status bit + fields
apply_{event_name}_test() ->
    S0 = {Aggregate}:initial_state(),
    S1 = {Aggregate}:apply_event({event_fixture}(), S0),
    ?assertNotEqual(0, element(?F_STATUS, S1) band ?{FLAG_BIT}).

%% Full lifecycle: chain all events, verify final status = sum of all bits
full_lifecycle_state_test() ->
    S0 = {Aggregate}:initial_state(),
    %% Chain: S0 -> S1 -> S2 -> ... -> SN
    ?assertEqual({ExpectedFinalStatus}, element(?F_STATUS, SN)).

%% Unknown event leaves state unchanged
apply_unknown_event_test() ->
    S0 = {Aggregate}:initial_state(),
    S1 = {Aggregate}:apply_event({first_event}(), S0),
    S2 = {Aggregate}:apply_event(#{<<"event_type">> => <<"unknown">>}, S1),
    ?assertEqual(S1, S2).

%% Atom keys work identically to binary keys
apply_with_atom_keys_test() ->
    %% Use atom key variant of first event, verify same result

%% flag_map/0 returns expected labels
flag_map_test() ->
    Map = {Aggregate}:flag_map(),
    ?assertEqual(<<"New">>, maps:get(0, Map)).

%% apply/2 callback order: (State, Event) - state first
apply_callback_order_test() ->
    S0 = {Aggregate}:initial_state(),
    S1 = {Aggregate}:apply(S0, {first_event}()),
    ?assertNotEqual(0, element(?F_STATUS, S1) band ?{FIRST_FLAG}).
```

### Key Patterns

- **Record field access:** Define `?F_*` macros for element positions. Count from 2 (element 1 is record tag).
- **State verification via execute/2:** If record is opaque, verify state through `execute/2` behavior instead of direct field access.
- **Partial event application:** Test events with optional fields omitted to verify defaults.

---

## Layer 2: Domain Internal Tests

### 2a. Command Tests

**File:** `apps/{CmdApp}/test/{CmdApp}_command_tests.erl`

```erlang
%% Per command module: {CmdModule}

{cmd_name}_new_valid_test() ->
    {ok, Cmd} = {CmdModule}:new(#{required_field => <<"value">>}),
    ?assertEqual(<<"value">>, {CmdModule}:get_required_field(Cmd)).

{cmd_name}_new_missing_fields_test() ->
    ?assertEqual({error, missing_required_fields}, {CmdModule}:new(#{})).

{cmd_name}_from_map_binary_keys_test() ->
    {ok, Cmd} = {CmdModule}:from_map(#{<<"required">> => <<"val">>}),
    ?assertEqual(<<"val">>, {CmdModule}:get_required(Cmd)).

{cmd_name}_to_map_round_trip_test() ->
    {ok, Cmd} = {CmdModule}:new(#{...}),
    Map = {CmdModule}:to_map(Cmd),
    ?assertEqual(<<"{cmd_type}">>, maps:get(<<"command_type">>, Map)),
    {ok, Cmd2} = {CmdModule}:from_map(Map),
    ?assertEqual({CmdModule}:get_id(Cmd), {CmdModule}:get_id(Cmd2)).
```

### 2b. Handler Tests

**File:** Same file or `apps/{CmdApp}/test/{CmdApp}_handler_tests.erl`

```erlang
%% Per handler: {HandlerModule} handling {CmdModule} -> {EventModule}

handle_valid_{operation}_test() ->
    {ok, Cmd} = {CmdModule}:new(#{...}),
    {ok, [Event]} = {HandlerModule}:handle(Cmd),
    Map = {EventModule}:to_map(Event),
    ?assertEqual(<<"{event_type}">>, maps:get(<<"event_type">>, Map)).
```

### 2c. Aggregate execute/2 Tests

**File:** Same file as handlers.

Template per command type — test success case + all error paths:

```erlang
%% Helpers: build state from event chains
fresh_state() -> {Aggregate}:initial_state().
{state_name}() -> {Aggregate}:apply_event({event}(), fresh_state()).

%% Success case
execute_{cmd}_on_{valid_state}_test() ->
    Cmd = #{<<"command_type">> => <<"{cmd_type}">>, ...},
    {ok, [EventMap]} = {Aggregate}:execute({valid_state}(), Cmd),
    ?assertEqual(<<"{event_type}">>, maps:get(<<"event_type">>, EventMap)).

%% Error cases (one per guard clause in the aggregate)
execute_{cmd}_{error_condition}_test() ->
    Cmd = #{<<"command_type">> => <<"{cmd_type}">>, ...},
    ?assertEqual({error, {error_atom}}, {Aggregate}:execute({invalid_state}(), Cmd)).

%% Unknown command
execute_unknown_command_test() ->
    ?assertEqual({error, unknown_command},
                 {Aggregate}:execute(fresh_state(), #{<<"command_type">> => <<"unknown">>})).

%% Callback order verification
execute_evoq_callback_order_test() ->
    {ok, _} = {Aggregate}:execute(fresh_state(), {valid_cmd_map}).
```

### 2d. Event Serialization Tests

**File:** `apps/{CmdApp}/test/{event_name}_tests.erl`

```erlang
{event}_new_test() ->
    E = {EventModule}:new(#{...}),
    ?assert(is_integer({EventModule}:get_timestamp(E))).

{event}_to_map_includes_event_type_test() ->
    E = {EventModule}:new(#{...}),
    Map = {EventModule}:to_map(E),
    ?assertEqual(<<"{event_type}">>, maps:get(<<"event_type">>, Map)).

{event}_round_trip_test() ->
    E = {EventModule}:new(#{...}),
    Map = {EventModule}:to_map(E),
    {ok, E2} = {EventModule}:from_map(Map),
    ?assertEqual({EventModule}:get_id(E), {EventModule}:get_id(E2)).

{event}_from_map_invalid_test() ->
    ?assertEqual({error, invalid_event}, {EventModule}:from_map(#{})).
```

---

## Layer 3: Integration Tests (CMD to PRJ)

**File:** `apps/{QryApp}/test/{qry_name}_cqrs_integration_tests.erl`

**Requires:** `test_store_proxy.erl` in the same test directory.

### test_store_proxy Pattern

A gen_server that:
1. Opens a temp SQLite file in `/tmp/`
2. Registers as `{store_module}` (e.g., `query_ventures_store`)
3. Implements the same `execute/1,2` and `query/1,2` API
4. Cleans up the temp file on terminate

```erlang
%% Setup/teardown pattern
cqrs_integration_test_() ->
    {foreach,
     fun setup/0,
     fun teardown/1,
     [fun test_case_1/1, fun test_case_2/1, ...]}.

setup() ->
    application:ensure_all_started(esqlite),
    DbPath = "/tmp/test_{table}_" ++
             integer_to_list(erlang:unique_integer([positive])) ++ ".db",
    {ok, _} = test_store_proxy:start(DbPath),
    #{db_path => DbPath}.

teardown(#{}) ->
    test_store_proxy:stop(),
    ok.
```

### Standard Test Cases

| Test | What It Verifies |
|------|-----------------|
| `setup_projects_to_sqlite` | Projection inserts a row |
| `query_by_id_found` | Query returns projected data |
| `query_by_id_not_found` | Query returns `{error, not_found}` |
| `archive_updates_status` | Archive projection sets bit flag |
| `page_returns_list` | Page query returns multiple results |
| `page_excludes_archived` | Default filter hides archived |
| `page_includes_archived` | Filter override shows archived |
| `page_limit_offset` | Pagination works correctly |
| `projection_idempotency` | Same event projected twice = no duplicate |
| `full_cqrs_flow` | Command -> handler -> event -> projection -> query |

---

## Layer 4: Side Effect Tests

### 4a. pg Emission Tests

**File:** `apps/{CmdApp}/test/{CmdApp}_side_effects_tests.erl`

```erlang
pg_emission_test_() ->
    {setup,
     fun start_pg/0,
     fun stop_pg/1,
     [fun emit_{event}_reaches_members/0, ...]}.

start_pg() ->
    case pg:start(pg) of
        {ok, Pid} -> {started, Pid};
        {error, {already_started, Pid}} -> {existing, Pid}
    end.

stop_pg({started, Pid}) -> gen_server:stop(Pid);
stop_pg({existing, _}) -> ok.

emit_{event}_reaches_members() ->
    Self = self(),
    Pid = spawn_link(fun() ->
        pg:join(pg, {group_atom}, self()),
        receive Msg -> Self ! {got, Msg} end
    end),
    timer:sleep(10),
    Event = #{<<"event_type">> => <<"{event_type}">>, ...},
    ok = {EmitterModule}:emit(Event),
    receive
        {got, {{group_atom}, E}} -> ?assertEqual(Event, E)
    after 1000 -> ?assert(false)
    end.

emit_with_no_members() ->
    ?assertEqual(ok, {EmitterModule}:emit(#{...})).

emit_reaches_multiple_members() ->
    %% Spawn N processes, join group, emit, verify all received
```

### 4b. Projection Row Verification Tests

**File:** `apps/{QryApp}/test/{table}_projection_tests.erl`

```erlang
%% Uses test_store_proxy, same setup/teardown as Layer 3

projection_creates_correct_row(#{}) ->
    fun() ->
        %% Project event, then SELECT raw row, verify columns
        {ok, [Row]} = {Store}:query("SELECT ... FROM {table} WHERE ...", [...]),
        Values = to_list(Row),  %% Handle tuple or list format
        %% Assert each column value
    end.

projection_status_label_correct(#{}) ->
    fun() ->
        %% Verify status_label matches evoq_bit_flags:to_string output
        ExpectedLabel = evoq_bit_flags:to_string(Status, ?FLAG_MAP),
        ?assertEqual(ExpectedLabel, ActualLabel)
    end.

projection_json_fields_encode(#{}) ->
    fun() ->
        %% Verify JSON fields decode correctly
        ?assertEqual([<<"r1">>], json:decode(ReposJson))
    end.

%% Helper for esqlite3 format flexibility
to_list(T) when is_tuple(T) -> tuple_to_list(T);
to_list(L) when is_list(L) -> L.
```

### 4c. ReckonDB Dispatch Tests

**File:** `apps/{CmdApp}/test/{CmdApp}_dispatch_tests.erl`

**What:** Verifies that `dispatch/1` persists events to ReckonDB and that they can be read back with correct data. Requires ReckonDB + evoq infrastructure (temp store created/destroyed per suite).

**Two variants exist:**

1. **setup_venture** (unique): `venture_id` auto-generated, second command is `refine_vision_v1`
2. **Lifecycle apps** (8 apps): Uses `unique_id/1` helper, second command is `archive_{X}_v1`

### Template: setup_venture variant

```erlang
%%% @doc Layer 4c: ReckonDB Dispatch Tests — setup_venture.
%%% Verifies that dispatch/1 persists events to ReckonDB and that
%%% they can be read back with correct data.
-module(setup_venture_dispatch_tests).

-include_lib("eunit/include/eunit.hrl").
-include_lib("reckon_db/include/reckon_db.hrl").

dispatch_test_() ->
    {setup, fun start_infra/0, fun stop_infra/1, [
        fun dispatch_persists_event/0,
        fun dispatch_returns_version_and_events/0,
        fun dispatch_second_command_appends_to_stream/0,
        fun read_back_event_data_matches/0
    ]}.

start_infra() ->
    application:ensure_all_started(reckon_db),
    application:ensure_all_started(evoq),
    application:set_env(evoq, event_store_adapter, reckon_evoq_adapter),
    TmpDir = "/tmp/test_dispatch_setup_" ++
        integer_to_list(erlang:unique_integer([positive])),
    StoreConfig = #store_config{
        store_id = setup_venture_store,
        data_dir = TmpDir,
        mode = single,
        options = #{}
    },
    {ok, _} = reckon_db_sup:start_store(StoreConfig),
    #{tmp_dir => TmpDir}.

stop_infra(#{tmp_dir := TmpDir}) ->
    reckon_db_sup:stop_store(setup_venture_store),
    os:cmd("rm -rf " ++ TmpDir),
    ok.

%% Dispatch persists exactly one event to ReckonDB
dispatch_persists_event() ->
    {ok, Cmd} = setup_venture_v1:new(#{name => <<"Dispatch Persist Test">>}),
    VentureId = setup_venture_v1:get_venture_id(Cmd),
    {ok, _Version, [_Event]} = maybe_setup_venture:dispatch(Cmd),
    {ok, Events} = esdb_gater_api:get_events(
        setup_venture_store, VentureId, 0, 100, forward),
    ?assertEqual(1, length(Events)).

%% Dispatch returns {ok, Version, [EventMaps]}
dispatch_returns_version_and_events() ->
    {ok, Cmd} = setup_venture_v1:new(#{name => <<"Return Type Test">>}),
    {ok, Version, EventMaps} = maybe_setup_venture:dispatch(Cmd),
    ?assert(is_integer(Version)),
    ?assert(is_list(EventMaps)),
    ?assertEqual(1, length(EventMaps)),
    [E] = EventMaps,
    ?assert(is_map(E)).

%% Second dispatch to same aggregate produces two events in store
dispatch_second_command_appends_to_stream() ->
    VentureId = <<"vent-incr-",
        (integer_to_binary(erlang:unique_integer([positive])))/binary>>,
    {ok, SetupCmd} = setup_venture_v1:new(#{
        venture_id => VentureId,
        name => <<"Version Incr Test">>
    }),
    {ok, _, _} = maybe_setup_venture:dispatch(SetupCmd),
    {ok, RefineCmd} = refine_vision_v1:new(#{
        venture_id => VentureId,
        brief => <<"Updated brief">>
    }),
    {ok, _, _} = maybe_refine_vision:dispatch(RefineCmd),
    {ok, Events} = esdb_gater_api:get_events(
        setup_venture_store, VentureId, 0, 100, forward),
    ?assertEqual(2, length(Events)).

%% Read-back event data matches the dispatched command
read_back_event_data_matches() ->
    {ok, Cmd} = setup_venture_v1:new(#{
        name => <<"ReadBack Test">>,
        brief => <<"Verify data">>,
        initiated_by => <<"test@host">>
    }),
    VentureId = setup_venture_v1:get_venture_id(Cmd),
    {ok, _, _} = maybe_setup_venture:dispatch(Cmd),
    {ok, [Event | _]} = esdb_gater_api:get_events(
        setup_venture_store, VentureId, 0, 100, forward),
    Data = extract_data(Event),
    Name = maps:get(<<"name">>, Data,
        maps:get(name, Data, undefined)),
    ?assertEqual(<<"ReadBack Test">>, Name).

extract_data(Event) when is_record(Event, event) ->
    Event#event.data;
extract_data(#{data := D}) when is_map(D) ->
    D;
extract_data(#{<<"data">> := D}) when is_map(D) ->
    D;
extract_data(Event) when is_map(Event) ->
    Event.
```

### Template: Lifecycle app variant (parameterized)

Replace `{CmdApp}`, `{short_name}`, `{StoreId}`, `{InitCmdModule}`, `{InitHandler}`, `{ArchiveCmdModule}`, `{ArchiveHandler}`, `{IdField}`.

```erlang
%%% @doc Layer 4c: ReckonDB Dispatch Tests — {CmdApp}.
%%% Verifies that dispatch/1 persists events to ReckonDB and that
%%% they can be read back with correct data.
-module({CmdApp}_dispatch_tests).

-include_lib("eunit/include/eunit.hrl").
-include_lib("reckon_db/include/reckon_db.hrl").

dispatch_test_() ->
    {setup, fun start_infra/0, fun stop_infra/1, [
        fun dispatch_persists_event/0,
        fun dispatch_returns_version_and_events/0,
        fun dispatch_second_command_appends_to_stream/0,
        fun read_back_event_data_matches/0
    ]}.

start_infra() ->
    application:ensure_all_started(reckon_db),
    application:ensure_all_started(evoq),
    application:set_env(evoq, event_store_adapter, reckon_evoq_adapter),
    TmpDir = "/tmp/test_dispatch_{short_name}_" ++
        integer_to_list(erlang:unique_integer([positive])),
    StoreConfig = #store_config{
        store_id = {StoreId},
        data_dir = TmpDir,
        mode = single,
        options = #{}
    },
    {ok, _} = reckon_db_sup:start_store(StoreConfig),
    #{tmp_dir => TmpDir}.

stop_infra(#{tmp_dir := TmpDir}) ->
    reckon_db_sup:stop_store({StoreId}),
    os:cmd("rm -rf " ++ TmpDir),
    ok.

dispatch_persists_event() ->
    DivisionId = unique_id(<<"{prefix}-persist">>),
    {ok, Cmd} = {InitCmdModule}:new(#{division_id => DivisionId}),
    {ok, _, [_]} = {InitHandler}:dispatch(Cmd),
    {ok, Events} = esdb_gater_api:get_events(
        {StoreId}, DivisionId, 0, 100, forward),
    ?assertEqual(1, length(Events)).

dispatch_returns_version_and_events() ->
    DivisionId = unique_id(<<"{prefix}-ret">>),
    {ok, Cmd} = {InitCmdModule}:new(#{division_id => DivisionId}),
    {ok, Version, EventMaps} = {InitHandler}:dispatch(Cmd),
    ?assert(is_integer(Version)),
    ?assert(is_list(EventMaps)),
    ?assertEqual(1, length(EventMaps)),
    [E] = EventMaps,
    ?assert(is_map(E)).

dispatch_second_command_appends_to_stream() ->
    DivisionId = unique_id(<<"{prefix}-stream">>),
    {ok, StartCmd} = {InitCmdModule}:new(#{division_id => DivisionId}),
    {ok, _, _} = {InitHandler}:dispatch(StartCmd),
    {ok, ArchiveCmd} = {ArchiveCmdModule}:new(#{division_id => DivisionId}),
    {ok, _, _} = {ArchiveHandler}:dispatch(ArchiveCmd),
    {ok, Events} = esdb_gater_api:get_events(
        {StoreId}, DivisionId, 0, 100, forward),
    ?assertEqual(2, length(Events)).

read_back_event_data_matches() ->
    DivisionId = unique_id(<<"{prefix}-read">>),
    {ok, Cmd} = {InitCmdModule}:new(#{
        division_id => DivisionId,
        started_by => <<"test@host">>
    }),
    {ok, _, _} = {InitHandler}:dispatch(Cmd),
    {ok, [Event | _]} = esdb_gater_api:get_events(
        {StoreId}, DivisionId, 0, 100, forward),
    Data = extract_data(Event),
    Id = maps:get(<<"division_id">>, Data,
        maps:get(division_id, Data, undefined)),
    ?assertEqual(DivisionId, Id).

unique_id(Prefix) ->
    <<Prefix/binary, "-",
      (integer_to_binary(erlang:unique_integer([positive])))/binary>>.

extract_data(Event) when is_record(Event, event) ->
    Event#event.data;
extract_data(#{data := D}) when is_map(D) ->
    D;
extract_data(#{<<"data">> := D}) when is_map(D) ->
    D;
extract_data(Event) when is_map(Event) ->
    Event.
```

### Special Case: rescue_division

`rescue_division` requires `incident_id` in the init command:

```erlang
dispatch_persists_event() ->
    DivisionId = unique_id(<<"res-persist">>),
    {ok, Cmd} = start_rescue_v1:new(#{
        division_id => DivisionId,
        incident_id => <<"inc-1">>
    }),
    {ok, _, [_]} = maybe_start_rescue:dispatch(Cmd),
    ...
```

### Key Patterns

- **`extract_data/1`**: Handles three ReckonDB return formats: `#event{}` record (atom-key `data` field), atom-key map (`#{data := D}`), binary-key map (`#{<<"data">> := D}`), or flat map fallback.
- **`unique_id/1`**: Used by lifecycle apps to prevent stream collisions between test runs. `setup_venture` doesn't need it because `venture_id` is auto-generated by `setup_venture_v1:new/1`.
- **Temp store lifecycle**: Each test suite creates a fresh ReckonDB store in `/tmp/`, destroyed in `stop_infra/1`.
- **`esdb_gater_api:get_events/5`**: `get_events(StoreId, StreamId, From, Count, Direction)` — reads back from the event store.

### Dispatch Template Variable Reference (9 CMD apps)

| App | Store ID | Init Cmd | Init Handler | Second Cmd | Second Handler | ID Field | Prefix | Extra Fields |
|-----|----------|----------|--------------|------------|----------------|----------|--------|--------------|
| `setup_venture` | `setup_venture_store` | `setup_venture_v1` | `maybe_setup_venture` | `refine_vision_v1` | `maybe_refine_vision` | `venture_id` (auto) | `vent` | `name` (required) |
| `discover_divisions` | `discover_divisions_store` | `start_discovery_v1` | `maybe_start_discovery` | `archive_discovery_v1` | `maybe_archive_discovery` | `division_id` | `dis` | — |
| `design_division` | `design_division_store` | `start_design_v1` | `maybe_start_design` | `archive_design_v1` | `maybe_archive_design` | `division_id` | `des` | — |
| `plan_division` | `plan_division_store` | `start_plan_v1` | `maybe_start_plan` | `archive_plan_v1` | `maybe_archive_plan` | `division_id` | `pln` | — |
| `generate_division` | `generate_division_store` | `start_generation_v1` | `maybe_start_generation` | `archive_generation_v1` | `maybe_archive_generation` | `division_id` | `gen` | — |
| `test_division` | `test_division_store` | `start_testing_v1` | `maybe_start_testing` | `archive_testing_v1` | `maybe_archive_testing` | `division_id` | `tst` | — |
| `deploy_division` | `deploy_division_store` | `start_deployment_v1` | `maybe_start_deployment` | `archive_deployment_v1` | `maybe_archive_deployment` | `division_id` | `dep` | — |
| `monitor_division` | `monitor_division_store` | `start_monitoring_v1` | `maybe_start_monitoring` | `archive_monitoring_v1` | `maybe_archive_monitoring` | `division_id` | `mon` | — |
| `rescue_division` | `rescue_division_store` | `start_rescue_v1` | `maybe_start_rescue` | `archive_rescue_v1` | `maybe_archive_rescue` | `division_id` | `res` | `incident_id` (required) |

`guide_venture` is excluded — passive orchestrator with no dispatch.

---

## Running Tests

```bash
# Layer 1+2: CMD app tests (auto-discovered via {module}_tests naming)
rebar3 eunit --app=setup_venture
rebar3 eunit --app=setup_venture,design_division,plan_division  # multiple

# Layer 3+4: QRY app tests (MUST use --module=, NOT --app=)
rebar3 eunit --module=monitoring_cqrs_integration_tests,monitoring_projection_tests

# All CMD apps at once (137 tests)
rebar3 eunit --app=setup_venture,discover_divisions,design_division,plan_division,generate_division,test_division,deploy_division,monitor_division,rescue_division

# All QRY apps at once (128 tests)
rebar3 eunit --module=venture_cqrs_integration_tests,venture_projection_tests,design_cqrs_integration_tests,design_projection_tests,discovery_cqrs_integration_tests,discovery_projection_tests,plan_cqrs_integration_tests,plan_projection_tests,generation_cqrs_integration_tests,generation_projection_tests,testing_cqrs_integration_tests,testing_projection_tests,deployment_cqrs_integration_tests,deployment_projection_tests,monitoring_cqrs_integration_tests,monitoring_projection_tests,rescue_cqrs_integration_tests,rescue_projection_tests
```

### Why `--app=` vs `--module=`

- **`--app=X`** runs EUnit on the application, which only discovers test modules matching `{app_module}_tests`. CMD app tests work because `setup_aggregate_tests` matches `setup_aggregate`.
- **`--module=X`** runs EUnit on explicitly named test modules. QRY tests REQUIRE this because `monitoring_cqrs_integration_tests` doesn't match any app module name.
- **`--dir=`** does NOT work reliably in rebar3 umbrella projects.

---

## Template Variable Reference (all 10 CMD apps)

| App | Aggregate | Status HRL | Init Event | Archive Event | Flags |
|-----|-----------|-----------|------------|---------------|-------|
| `setup_venture` | `setup_aggregate` | `venture_status.hrl` | `venture_setup_v1` | `venture_archived_v1` | SETUP=1, REFINED=2, SUBMITTED=4, ARCHIVED=8 |
| `discover_divisions` | `discovery_aggregate` | `discovery_status.hrl` | `discovery_started_v1` | `discovery_archived_v1` | INITIATED=1, ACTIVE=2, PAUSED=4, COMPLETED=8, ARCHIVED=16 |
| `design_division` | `design_aggregate` | `design_status.hrl` | `design_started_v1` | `design_archived_v1` | same lifecycle |
| `plan_division` | `plan_aggregate` | `plan_status.hrl` | `plan_started_v1` | `plan_archived_v1` | same lifecycle |
| `generate_division` | `generation_aggregate` | `generation_status.hrl` | `generation_started_v1` | `generation_archived_v1` | same lifecycle |
| `test_division` | `testing_aggregate` | `testing_status.hrl` | `testing_started_v1` | `testing_archived_v1` | same lifecycle |
| `deploy_division` | `deployment_aggregate` | `deployment_status.hrl` | `deployment_started_v1` | `deployment_archived_v1` | same lifecycle |
| `monitor_division` | `monitoring_aggregate` | `monitoring_status.hrl` | `monitoring_started_v1` | `monitoring_archived_v1` | same lifecycle |
| `rescue_division` | `rescue_aggregate` | `rescue_status.hrl` | `rescue_started_v1` | `rescue_archived_v1` | same lifecycle |
| `guide_venture` | `guide_aggregate` | `guide_status.hrl` | `guide_started_v1` | `guide_archived_v1` | passive orchestrator |

## Template Variable Reference (all 9 QRY apps)

| App | Store | Main Table | Proxy Module | Query API | Child Tables |
|-----|-------|-----------|--------------|-----------|-------------|
| `query_ventures` | `query_ventures_store` | `domains` | `test_store_proxy` | `get/1` | — |
| `query_discoveries` | `query_discoveries_store` | `discoveries` | `discoveries_test_store_proxy` | `execute/1` | `discovered_divisions` |
| `query_designs` | `query_designs_store` | `designs` | `designs_test_store_proxy` | `get/1` | `designed_aggregates`, `designed_events` |
| `query_plans` | `query_plans_store` | `plans` | `plans_test_store_proxy` | `get/1` | `planned_desks`, `planned_dependencies` |
| `query_generations` | `query_generations_store` | `generations` | `generations_test_store_proxy` | `get/1` | `generated_modules`, `generated_tests` |
| `query_tests` | `query_tests_store` | `testings` | `testings_test_store_proxy` | `get/1` | `test_suites`, `test_results` |
| `query_deployments` | `query_deployments_store` | `deployments` | `deployments_test_store_proxy` | `get/1` | `releases`, `rollout_stages` |
| `query_monitoring` | `query_monitoring_store` | `monitorings` | `monitoring_test_store_proxy` | `get/1` | `health_checks`, `incidents` |
| `query_rescues` | `query_rescues_store` | `rescues` | `rescues_test_store_proxy` | `get/1` | `diagnoses`, `fixes` |

**Note:** `query_discoveries` uses `execute/1` while all other QRY apps use `get/1`.

---

## Naming Conventions for Test Files

### CMD App Tests (L1+L2)

| Layer | Pattern | Example |
|-------|---------|---------|
| L1 Dossier | `{aggregate}_tests.erl` | `design_aggregate_tests.erl` |
| L2 Handlers | `{cmd_app}_handler_tests.erl` | `design_division_handler_tests.erl` |
| L2 Events | `{subject}_event_tests.erl` | `design_event_tests.erl` |
| L2 Side Effects | `{cmd_app}_side_effects_tests.erl` | `design_division_side_effects_tests.erl` |
| L4c Dispatch | `{cmd_app}_dispatch_tests.erl` | `design_division_dispatch_tests.erl` |

### QRY App Tests (L3+L4)

| Layer | Pattern | Example |
|-------|---------|---------|
| L3 Integration | `{subject}_cqrs_integration_tests.erl` | `design_cqrs_integration_tests.erl` |
| L4 Projections | `{subject}_projection_tests.erl` | `design_projection_tests.erl` |
| Helper | `{table}_test_store_proxy.erl` | `designs_test_store_proxy.erl` |

### CRITICAL: Unique Proxy Module Names

In rebar3 umbrella projects, **ALL test modules compile into the same namespace**. Each QRY app's test store proxy MUST have a unique module name:

```
WRONG:  test_store_proxy.erl         (collision between query_designs and query_plans!)
RIGHT:  designs_test_store_proxy.erl  (unique per app)
RIGHT:  plans_test_store_proxy.erl    (unique per app)
```

The proxy module registers as the actual store name (e.g., `query_designs_store`) so projections and queries work unchanged.

---

## Bugs & Lessons from Full Rollout (265 tests)

### 1. esqlite3 list/tuple mismatch

`esqlite3:fetchall` can return rows as lists `[[val]]` or tuples `[{val}]`. Lifecycle projections pattern-match on `{ok, [{CurrentStatus}]}` and silently fail when they get lists.

**Fix in test proxy:** Add `ensure_tuples/1` to normalize all fetchall results:

```erlang
ensure_tuples(Rows) -> [ensure_tuple(R) || R <- Rows].
ensure_tuple(T) when is_tuple(T) -> T;
ensure_tuple(L) when is_list(L) -> list_to_tuple(L).
```

### 2. esqlite3 API argument order in tests

Production stores use `esqlite3:q(Sql, Params, Db)` (Sql first). But in the test proxy, `esqlite3:exec(Sql, Db)` doesn't work — you must use `esqlite3:exec(Db, Sql)` (Db first). The test proxy uses `prepare/bind/fetchall` for parameterized queries instead of `q/3`.

### 3. Module name collision in umbrella test directories

Rebar3 compiles ALL test modules together in an umbrella. If two apps both have `test/test_store_proxy.erl`, only one survives. **Always prefix**: `designs_test_store_proxy`, `plans_test_store_proxy`, etc.

### 4. `warnings_as_errors` blocks all test compilation

A single unused function warning in ANY test file blocks ALL test compilation across the umbrella. Always clean up unused event fixture functions.

### 5. EUnit app discovery vs module discovery

`rebar3 eunit --app=query_designs` discovers ZERO tests because EUnit's `{application, X}` mode only matches `{app_module}_tests` modules. QRY test modules like `design_cqrs_integration_tests` don't correspond to any app module. **Must use `--module=`**.

### Proven Test Store Proxy Pattern

This is the canonical proxy pattern used across all 8 QRY app test suites:

```erlang
-module({table}_test_store_proxy).
-behaviour(gen_server).
-export([start/1, stop/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2, terminate/2]).
-record(state, {db :: reference(), path :: string()}).

start(DbPath) -> gen_server:start({local, {store_module}}, ?MODULE, DbPath, []).
stop() -> gen_server:stop({store_module}).

init(DbPath) ->
    {ok, Db} = esqlite3:open(DbPath),
    ok = esqlite3:exec(Db, "PRAGMA journal_mode=WAL;"),
    create_tables(Db),
    {ok, #state{db = Db, path = DbPath}}.

handle_call({execute, Sql, []}, _From, #state{db = Db} = S) ->
    {reply, esqlite3:exec(Db, Sql), S};
handle_call({execute, Sql, Params}, _From, #state{db = Db} = S) ->
    case esqlite3:prepare(Db, Sql) of
        {ok, St} -> ok = esqlite3:bind(St, Params), step(St), {reply, ok, S};
        E -> {reply, E, S}
    end;
handle_call({query, Sql, []}, _From, #state{db = Db} = S) ->
    case esqlite3:prepare(Db, Sql) of
        {ok, St} -> {reply, {ok, ensure_tuples(esqlite3:fetchall(St))}, S};
        E -> {reply, E, S}
    end;
handle_call({query, Sql, Params}, _From, #state{db = Db} = S) ->
    case esqlite3:prepare(Db, Sql) of
        {ok, St} -> ok = esqlite3:bind(St, Params),
                     {reply, {ok, ensure_tuples(esqlite3:fetchall(St))}, S};
        E -> {reply, E, S}
    end;
%% ...

terminate(_, #state{db = Db, path = P}) -> esqlite3:close(Db), file:delete(P), ok.
step(St) -> case esqlite3:step(St) of '$done' -> ok; {row,_} -> step(St); ok -> ok end.
ensure_tuples(Rows) -> [ensure_tuple(R) || R <- Rows].
ensure_tuple(T) when is_tuple(T) -> T;
ensure_tuple(L) when is_list(L) -> list_to_tuple(L).
```
