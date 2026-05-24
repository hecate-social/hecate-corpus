# CODEGEN: Erlang Pipelines

*Templates and checklists for command pipelines.*

See also: [philosophy/COMMAND_PIPELINES.md](../../../philosophy/COMMAND_PIPELINES.md) for the conceptual model and invariant.

---

## Where Pipeline Code Lives

Pipelines and steps live alongside the desk or PM slice they serve.

### CMD desk with HOPE entry

```
apps/{cmd_app}/src/
└── {command}/                            # the desk
    ├── {command}_desk_sup.erl
    ├── {command}_v1.erl                  # Command record
    ├── {event}_v1.erl                    # Event record
    ├── maybe_{command}.erl               # Handler (consumes ENRICHED command)
    ├── {command}_responder_v1.erl        # mesh inbound → command
    ├── {command}_api.erl                 # HTTP inbound → command (optional)
    ├── {command}_pipeline.erl            # ← PIPELINE DECLARATION
    └── steps/                            # ← STEP MODULES
        ├── authenticate_{command}.erl
        ├── enforce_{command}_idempotency.erl
        ├── enrich_with_{thing}.erl
        └── validate_{rule}.erl
```

### PM sibling slice

PMs are sibling slices in the target CMD app (see [PROCESS_MANAGERS.md](../../../philosophy/PROCESS_MANAGERS.md)). They use pipelines too:

```
apps/{cmd_app}/src/
└── on_{src_event}_{action}_{target}/
    ├── on_{src_event}_{action}_{target}_sup.erl
    ├── on_{src_event}_{action}_{target}.erl           # gen_server: pg + dispatch
    ├── on_{src_event}_{action}_{target}_pipeline.erl  # ← PIPELINE
    └── steps/
        └── ...
```

### Cross-cutting steps (reusable across slices)

```
apps/{shared_lib}/src/
└── pipeline_steps/
    ├── authenticate_via_realm_cert.erl     # used by every CMD slice
    ├── enforce_command_id_idempotency.erl
    └── attach_actor_to_metadata.erl
```

A step belongs in a shared lib once at least two slices use it without modification.

---

## Step Module Template

```erlang
%%% @doc Pipeline step: enrich command with active permit status.
%%%
%%% Reads from local pricing projection (mesh-fact-fed) — never makes
%%% a cross-cluster call. Snapshots the permit at read time, attaches
%%% provenance to ctx metadata.
%%% @end
-module(enrich_with_permit_status).
-behaviour(evoq_pipeline_step).

-export([name/0, apply/2, failure_mode/0]).

-include_lib("kernel/include/logger.hrl").

name() -> enrich_with_permit_status.

failure_mode() -> fail_fast.   %% Without permit data, can't authorize egress.

apply(Cmd, Ctx) ->
    VehiclePlate = maps:get(vehicle_plate, Cmd),
    case parksim_pricing_local:active_permit_for_vehicle(VehiclePlate) of
        {ok, Permit} ->
            Cmd1 = Cmd#{
                permit_id          => maps:get(permit_id, Permit),
                permit_holder_id   => maps:get(holder_id, Permit),
                permit_valid_until => maps:get(valid_until, Permit)
            },
            Ctx1 = put_meta(Ctx, permit_snapshot_at, erlang:system_time(millisecond)),
            Ctx2 = put_meta(Ctx1, permit_source, <<"local_projection">>),
            {ok, Cmd1, Ctx2};
        {error, not_found} ->
            ?LOG_INFO("[~s] no active permit for ~s", [?MODULE, VehiclePlate]),
            {error, no_active_permit}
    end.

%% @private
put_meta(Ctx, Key, Value) ->
    Meta = maps:get('__meta', Ctx, #{}),
    maps:put('__meta', Meta#{Key => Value}, Ctx).
```

### Step naming conventions

| Step purpose | Naming | Example |
|--------------|--------|---------|
| Authentication | `authenticate[_via_{mechanism}]` | `authenticate_via_realm_cert` |
| Idempotency | `enforce_{cmd}_idempotency` | `enforce_record_exit_idempotency` |
| Enrichment | `enrich_with_{thing}` | `enrich_with_permit_status` |
| Validation needing external state | `validate_{rule}` | `validate_egress_permitted` |
| Pure input shape validation | usually in cmd's `from_map/1`, NOT a pipeline step | |
| Instrumentation | `record_{thing}` or `audit_{thing}` | `audit_command_received` |

---

## Pipeline Declaration Template

```erlang
%%% @doc Pipeline for the record_exit command.
%%%
%%% Six steps, ordered. Each step has a single responsibility.
%%% Enrichment runs before validation that depends on it.
%%% @end
-module(record_exit_pipeline).
-behaviour(evoq_pipeline).

-export([steps/0]).

steps() ->
    [authenticate_via_realm_cert,
     enforce_record_exit_idempotency,
     enrich_with_session,
     enrich_with_permit_status,
     enrich_with_outstanding_fee,
     validate_egress_permitted].
```

That's the whole module. The integration contract lives in this one file, in evaluation order. Reviewers can see at a glance what external data the command depends on.

---

## Wiring: Responder → Pipeline → Handler

```erlang
%%% record_exit_responder_v1.erl

handle_call({record_exit, Payload}, _From, State) ->
    case build_command(Payload) of
        {ok, Cmd} ->
            InitCtx = #{
                actor       => maps:get(actor, Payload, undefined),
                source      => mesh,
                command_id  => maps:get(command_id, Payload),
                received_at => erlang:system_time(millisecond)
            },
            case evoq_pipeline:run(record_exit_pipeline, Cmd, InitCtx) of
                {ok, EnrichedCmd, FinalCtx} ->
                    Reply = maybe_record_exit:dispatch(EnrichedCmd, FinalCtx),
                    {reply, Reply, State};
                {halt, CachedResult} ->
                    %% Idempotency hit — return cached response without dispatching
                    {reply, CachedResult, State};
                {error, Reason, FailedStep} ->
                    ?LOG_INFO("[~s] pipeline failed at ~s: ~p",
                              [?MODULE, FailedStep, Reason]),
                    {reply, {error, Reason}, State}
            end;
        {error, _} = Err ->
            {reply, Err, State}
    end.
```

---

## Wiring: PM → Pipeline → Dispatch

```erlang
%%% on_parking_card_read_authorize_egress.erl

handle_info({evoq_event, #{event_data := EventData, event_id := EventId}}, State) ->
    spawn(fun() -> run_pipeline_and_dispatch(EventData, EventId) end),
    {noreply, State}.

run_pipeline_and_dispatch(EventData, TriggerEventId) ->
    Cmd = derive_command_from_event(EventData),
    InitCtx = #{
        triggered_by => TriggerEventId,
        source       => pg,
        received_at  => erlang:system_time(millisecond)
    },
    case evoq_pipeline:run(authorize_egress_pipeline, Cmd, InitCtx) of
        {ok, EnrichedCmd, _Ctx} ->
            maybe_authorize_egress:dispatch(EnrichedCmd);
        {halt, _} ->
            ok;   %% already processed, drop
        {error, Reason, FailedStep} ->
            ?LOG_INFO("[on_card_read] pipeline failed at ~s: ~p",
                      [FailedStep, Reason])
    end.
```

The PM gen_server stays small — it `pg:join`s in `init/1` and spawns a worker per event. The worker runs the pipeline so the gen_server mailbox keeps draining under load.

---

## Bootstrap Helpers

Until `evoq_pipeline:run/3` ships, applications can hand-roll a minimal runner:

```erlang
%%% pipeline_runner.erl (~30 LOC, lives in a shared lib until evoq supports it)

-module(pipeline_runner).
-export([run/3]).

run(PipelineMod, Cmd, Ctx) ->
    Steps = PipelineMod:steps(),
    run_steps(Steps, Cmd, Ctx).

run_steps([], Cmd, Ctx) ->
    {ok, Cmd, Ctx};
run_steps([Step | Rest], Cmd, Ctx) ->
    StartUs = erlang:monotonic_time(microsecond),
    Result = safe_apply(Step, Cmd, Ctx),
    EndUs = erlang:monotonic_time(microsecond),
    telemetry:execute(
        [evoq, pipeline, step, stop],
        #{duration_us => EndUs - StartUs},
        #{step => Step, result => result_tag(Result)}
    ),
    case Result of
        {ok, Cmd1, Ctx1}    -> run_steps(Rest, Cmd1, Ctx1);
        {skip, Ctx1}        -> run_steps(Rest, Cmd, Ctx1);
        {halt, Result1}     -> {halt, Result1};
        {error, Reason}     -> {error, Reason, Step}
    end.

safe_apply(Step, Cmd, Ctx) ->
    try Step:apply(Cmd, Ctx)
    catch Class:Reason:Stack ->
        {error, {step_crashed, Class, Reason, Stack}}
    end.

result_tag({ok, _, _})   -> ok;
result_tag({skip, _})    -> skip;
result_tag({halt, _})    -> halt;
result_tag({error, _})   -> error.
```

Migrate to `evoq_pipeline:run/3` when it lands by changing only the call site.

---

## Testing

Each step is testable in isolation:

```erlang
enrich_with_permit_status_test() ->
    meck:new(parksim_pricing_local, [non_strict]),
    meck:expect(parksim_pricing_local, active_permit_for_vehicle,
                fun(<<"1-ABC-123">>) ->
                    {ok, #{permit_id => <<"p1">>,
                           valid_until => 999,
                           holder_id => <<"h1">>}}
                end),
    Cmd = #{vehicle_plate => <<"1-ABC-123">>},
    {ok, Cmd1, Ctx1} = enrich_with_permit_status:apply(Cmd, #{}),
    ?assertEqual(<<"p1">>, maps:get(permit_id, Cmd1)),
    ?assert(maps:is_key(permit_snapshot_at, maps:get('__meta', Ctx1))),
    meck:unload(parksim_pricing_local).
```

The pipeline is testable end-to-end with stub steps:

```erlang
record_exit_pipeline_happy_path_test() ->
    Cmd = #{vehicle_plate => <<"1-ABC-123">>, card_id => <<"c1">>},
    Ctx = #{actor => <<"test">>, source => test, command_id => <<"cmd-1">>},
    {ok, EnrichedCmd, _} = pipeline_runner:run(record_exit_pipeline, Cmd, Ctx),
    ?assertMatch(#{permit_id := _, session_id := _, fee_outstanding := _},
                 EnrichedCmd).
```

---

## Checklist: New Pipeline

Given a new command that needs external state for validation:

- [ ] Identify every cross-domain read the command needs (write them down).
- [ ] Decide failure mode per read: `fail_fast` (default), `fail_soft`, or `{default, V}`.
- [ ] Create one step module per read in `{command}/steps/` (or in shared lib if reusable across ≥2 slices).
- [ ] Create `{command}_pipeline.erl` declaring the step order.
- [ ] Wire the responder (and/or PM) to call `pipeline_runner:run/3` (or `evoq_pipeline:run/3`).
- [ ] Confirm the aggregate's `execute/2` reads NOTHING outside `(State, EnrichedCommand)`.
- [ ] Add unit tests per step (mock the external dependency).
- [ ] Add a pipeline happy-path test.
- [ ] Confirm enrichment metadata propagates to the emitted event (test against event envelope shape).
- [ ] Confirm idempotency step keys off HOPE identity, not enriched payload.

---

## Anti-Patterns

| Anti-pattern | Why wrong | Fix |
|--------------|-----------|-----|
| Reading external state inside aggregate's `execute/2` or `apply/2` | Cardinal sin (Demon 41) — breaks replay, testability, determinism | Move the read into a pipeline step |
| Reading external state inside `maybe_{command}` handler | Same — handler should be deterministic on enriched input | Pipeline step |
| Reading external state inside PM's `handle_info` event flow | Same — PM should dispatch, pipeline enriches | Move to pipeline step in PM's slice |
| Idempotency keyed off post-enrichment payload | Retries with different snapshots dedup wrongly, produce duplicate events | Key off HOPE identity (`command_id`) |
| Merging metadata into event payload | Demon 37 — silently overwrites business fields | Keep structurally separate (`'__meta'` ctx convention) |
| Branching inside the aggregate to RETRY enrichment | Read happens at wrong layer | Pipeline retries (failure_mode), or step fails fast |
| Single mega-step that does everything | Untestable, unobservable, can't reason about failures | Decompose into focused single-responsibility steps |
| Dynamic step lists (`steps/1` branching on command) | Hard to introspect, audit, document, telemetry-tag | Two pipelines for two command variants |
| Cross-cluster RPC inside a step (against another facility) | Couples to remote availability; defeats facility autonomy | Read from local denormalized projection of mesh-published facts |
| Logging at DEBUG when a step fails | Silent failures kill systems | Log at INFO/WARNING; failed-step name in telemetry |

---

*Date: 2026-05-25. Templates introduced alongside [philosophy/COMMAND_PIPELINES.md](../../../philosophy/COMMAND_PIPELINES.md).*
