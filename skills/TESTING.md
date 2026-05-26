---
title: Testing Patterns
layer: skill
audience: [agent, human]
stage: stable
---

# Testing Patterns

Guidelines for testing hecate-daemon Erlang applications.

---

## Test Philosophy

> **Tests verify behavior, not implementation.**

Write tests that:
1. Verify the contract (inputs → outputs)
2. Test integration points (pg, SQLite, mesh)
3. Catch regressions before deployment
4. Document expected behavior

---

## Test Before Push

**Never push code without local tests passing.**

```bash
# Run unit tests for specific apps
rebar3 eunit --app=setup_venture,query_ventures

# Run specific test modules
rebar3 eunit --module=venture_initiated_v1_to_pg_tests

# Verbose output
rebar3 eunit --app=query_ventures -v
```

---

## Test File Location

Tests live alongside the code they test:

```
apps/setup_venture/
├── src/
│   └── initiate_venture/
│       └── venture_initiated_v1_to_pg.erl
└── test/
    └── venture_initiated_v1_to_pg_tests.erl   # {module}_tests.erl
```

**Naming:** `{module}_tests.erl` - EUnit auto-discovers tests for a module.

---

## pg Integration Tests

Testing process groups requires proper setup/teardown.

### Emitter Test Pattern

```erlang
-module(my_event_to_pg_tests).
-include_lib("eunit/include/eunit.hrl").

-define(GROUP, my_event).
-define(SCOPE, pg).

emit_test() ->
    ensure_pg(),
    ok = pg:join(?SCOPE, ?GROUP, self()),

    Event = #{id => <<"test">>},
    ok = my_event_to_pg:emit(Event),

    receive
        {my_event, ReceivedEvent} ->
            ?assertEqual(Event, ReceivedEvent)
    after 1000 ->
        ?assert(false)
    end,

    ok = pg:leave(?SCOPE, ?GROUP, self()).

ensure_pg() ->
    case pg:start(?SCOPE) of
        {ok, _} -> ok;
        {error, {already_started, _}} -> ok
    end.
```

### Listener Test Pattern (Fixtures)

```erlang
-module(my_projection_tests).
-include_lib("eunit/include/eunit.hrl").

-define(GROUP, my_event).
-define(SCOPE, pg).

%% Setup/teardown for tests needing full stack
setup() ->
    ensure_pg(),
    StorePid = start_or_get(my_store, start_link, []),
    SupPid = start_or_get(my_desk_sup, start_link, []),
    timer:sleep(100),  % Let listener join pg
    {StorePid, SupPid}.

cleanup({_StorePid, _SupPid}) ->
    ok.  % Don't kill registered processes

%% Fixture test
integration_test_() ->
    {setup,
     fun setup/0,
     fun cleanup/1,
     fun(_) ->
         Event = #{id => <<"test-", (integer_to_binary(erlang:system_time(microsecond)))/binary>>},

         %% Send via pg
         Message = {my_event, Event},
         lists:foreach(fun(Pid) -> Pid ! Message end, pg:get_members(?SCOPE, ?GROUP)),

         timer:sleep(200),  % Let projection run

         %% Verify in database
         {ok, Rows} = my_store:query("SELECT id FROM my_table WHERE id = ?1", [maps:get(id, Event)]),

         [?_assertEqual(1, length(Rows))]
     end}.

%% Helper
start_or_get(Module, Fun, Args) ->
    case apply(Module, Fun, Args) of
        {ok, Pid} -> Pid;
        {error, {already_started, Pid}} -> Pid
    end.
```

---

## SQLite Integration Tests

### Testing Projections

```erlang
projection_test() ->
    %% Use unique IDs to avoid test pollution
    Id = <<"test-", (integer_to_binary(erlang:system_time(microsecond)))/binary>>,

    Event = #{
        id => Id,
        name => <<"Test Item">>,
        created_at => erlang:system_time(millisecond)
    },

    %% Call projection directly
    ok = my_event_to_sqlite_items:project(Event),

    %% Verify
    {ok, Rows} = my_store:query("SELECT id, name FROM items WHERE id = ?1", [Id]),
    ?assertEqual(1, length(Rows)),
    ?assertMatch([[Id, <<"Test Item">>]], Rows).
```

### Note: esqlite3 Returns Lists

```erlang
%% esqlite3:fetchall returns rows as LISTS, not tuples
{ok, [[Id, Name, Status]]} = my_store:query("SELECT id, name, status FROM items WHERE id = ?1", [Id])

%% NOT tuples:
%% {ok, [{Id, Name, Status}]}  % WRONG assumption
```

---

## Test Patterns by Component

| Component | What to Test | How |
|-----------|-------------|-----|
| **Aggregate** | Execute returns events | Call execute(State, Payload), assert {ok, [EventMaps]} |
| **Emitter** | Subscribers receive messages | Join pg, emit, assert receive |
| **Listener** | Events trigger projections | Emit to pg, check database |
| **Projection** | Correct data written | Call directly, query database |
| **Query** | Correct data returned | Seed database, call query |
| **Handler** | Business logic | Call handle/1, assert events |

---

## Aggregate Tests (CRITICAL)

**Always test aggregates match evoq behaviour callback signatures!**

evoq calls:
- `Module:init(AggregateId)` → `{ok, State}`
- `Module:execute(State, Payload)` → `{ok, [EventMaps]}` or `{error, Reason}`
- `Module:apply(State, Event)` → NewState

### Aggregate Test Pattern

```erlang
-module(my_aggregate_tests).
-include_lib("eunit/include/eunit.hrl").

%% CRITICAL: Test argument order matches evoq expectations
execute_argument_order_test() ->
    %% Initial state
    State = my_aggregate:initial_state(),

    %% Command payload (from command:to_map/1)
    Payload = #{
        command_type => <<"my_command">>,
        id => <<"test-123">>,
        name => <<"Test">>
    },

    %% Execute with correct order: State, Payload
    Result = my_aggregate:execute(State, Payload),

    %% Should return {ok, [EventMap]}
    ?assertMatch({ok, [_]}, Result),

    {ok, [EventMap]} = Result,
    ?assertEqual(<<"my_event_v1">>, maps:get(event_type, EventMap)).

%% Test unknown command returns error
unknown_command_test() ->
    State = my_aggregate:initial_state(),
    Payload = #{command_type => <<"unknown">>},
    ?assertEqual({error, unknown_command}, my_aggregate:execute(State, Payload)).
```

### Why This Matters

The aggregate bug (2026-02-09) was caused by wrong argument order:
- **Wrong:** `execute(Payload, State)` - fails silently with "unknown_command"
- **Correct:** `execute(State, Payload)` - matches evoq behaviour

This test catches the bug immediately. Without it, the bug only appeared at runtime when evoq dispatched a command and the aggregate returned `{error, unknown_command}`.

---

## Common Test Fixtures

### Unique IDs to Avoid Pollution

```erlang
unique_id() ->
    <<"test-", (integer_to_binary(erlang:system_time(microsecond)))/binary>>.
```

### Waiting for Async Operations

```erlang
%% Give async processes time to work
timer:sleep(200),

%% Or poll with timeout
wait_for(Fun, Timeout) ->
    wait_for(Fun, Timeout, 50).

wait_for(Fun, Timeout, _Interval) when Timeout =< 0 ->
    Fun();  % Final attempt
wait_for(Fun, Timeout, Interval) ->
    case Fun() of
        {ok, _} = Result -> Result;
        _ ->
            timer:sleep(Interval),
            wait_for(Fun, Timeout - Interval, Interval)
    end.
```

---

## Anti-Patterns

| Anti-Pattern | Why It's Wrong | Correct Approach |
|--------------|----------------|------------------|
| Push without tests | Bugs reach prod | `rebar3 eunit` before push |
| Test implementation | Brittle tests | Test behavior/contracts |
| Shared test state | Flaky tests | Unique IDs per test |
| No async wait | Race conditions | `timer:sleep` or polling |
| Killing registered processes | Affects other tests | Don't cleanup registered |

---

## Running Tests in CI

The CI workflow runs:

```yaml
- name: Run tests
  run: rebar3 eunit
```

All tests must pass before the Docker image is built.

---

## Decision Record

| Date | Decision |
|------|----------|
| 2026-02-08 | Tests required before push |
| 2026-02-08 | Use `{module}_tests.erl` naming for auto-discovery |
| 2026-02-08 | pg tests use scope `pg` (OTP default) |
| 2026-02-08 | Use unique IDs to avoid test pollution |
