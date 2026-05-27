---
title: "Example: DCB Counter with Maximum"
layer: example
audience: [agent, human]
stage: stable
---

# Example: DCB Counter with Maximum

*Canonical example: a counter that respects a maximum value, using `evoq_decision` instead of `evoq_aggregate`.*

---

## The Use Case

"Allow up to N events tagged X. After N, refuse."

Classic shapes:
- License seat allocation (max 10 seats; reject 11th claim)
- Idempotent registration (max 1 event per email tag)
- Rate limiting per tenant (max K events per window)
- Reservation systems (max 30 students per course)

In every case the consistency boundary is *cross-cutting* — it doesn't belong to any single entity. There's no natural Dossier to hang it on. Inventing a "counter aggregate" works but feels like ceremony, and forces every operation through a single stream's Ra consensus.

`evoq_decision` (DCB) handles this directly. The "consistency unit" is a *query*, not a stream.

---

## When to Use Decision Instead of Aggregate

| Question | Aggregate | Decision |
|----------|-----------|----------|
| Does the work have a natural identity-carrier (a "thing")? | Yes | No |
| Does the work have a lifecycle (open → conclude)? | Yes | No |
| Is the consistency rule per-thing? | Yes | No |
| Is the consistency rule cross-cutting (over many things)? | No | Yes |
| Is uniqueness/allocation/rate-limiting the core check? | No | Yes |

See `philosophy/CONSISTENCY_BOUNDARIES.md` for the full discrimination rule.

---

## The Pattern

Three pieces:

1. **Commands** — what the caller asks
2. **Events** — what happened (immutable facts)
3. **Decision module** — implements the `evoq_decision` behaviour

The runtime (`evoq_decision_runtime`) wires everything else: reads the context, calls the user's decide function, conditionally appends, retries on conflict.

---

## Code

### Commands and events

```erlang
%% commands.erl
-record(increment_counter, {
    counter_id :: binary(),  %% e.g., <<"license-seats:tenant-42">>
    max        :: pos_integer()
}).

%% events.erl
-record(counter_incremented_v1, {
    counter_id :: binary(),
    new_value  :: pos_integer()  %% 1..max
}).
```

Events carry no tags in the record itself; the tags travel in the
event map passed to the runtime (see below).

### Decision module

```erlang
-module(dcb_counter).
-behaviour(evoq_decision).

-export([context/1, decide/2]).

-include("commands.hrl").
-include("events.hrl").

%% The consistency boundary: all events tagged with this counter's id.
%% That's what we need to count to know whether we've hit the max.
context(#increment_counter{counter_id = CounterId}) ->
    {any_of, [counter_tag(CounterId)]}.

%% Pure decision: given the events tagged this counter, decide whether
%% appending one more would exceed max. If not, return the event to
%% append. If yes, return an error.
decide(ContextEvents, #increment_counter{counter_id = CounterId, max = Max}) ->
    CurrentCount = length(ContextEvents),
    case CurrentCount < Max of
        true ->
            NewValue = CurrentCount + 1,
            Event = #{
                event_type => <<"counter_incremented_v1">>,
                data => #{
                    counter_id => CounterId,
                    new_value  => NewValue
                },
                tags => [counter_tag(CounterId)]
            },
            {ok, [Event]};
        false ->
            {error, counter_full}
    end.

%% Stable tag scheme — pick one and keep it.
counter_tag(CounterId) ->
    <<"counter:", CounterId/binary>>.
```

That's the whole Decision. No supervisor, no `init/1`, no state to
manage. The runtime owns the lifecycle.

### Dispatch

```erlang
%% Anywhere user code wants to increment:
case evoq_decision_runtime:dispatch(
       dcb_counter,
       my_store,
       #increment_counter{counter_id = <<"license-seats:tenant-42">>, max = 10}) of
    {ok, [_Event]} ->
        %% Counter incremented. Caller can read the projection now.
        ok;
    {error, counter_full} ->
        %% Reject the request — limit reached.
        {error, no_seats_available};
    {error, retry_budget_exhausted} ->
        %% Three sequential context_changed conflicts. Either real
        %% contention or a runaway loop; surface to the caller.
        {error, try_again};
    {error, Other} ->
        %% Backend error — propagate.
        {error, Other}
end.
```

---

## The Concurrent Test

The headline correctness property: under heavy contention, the counter
never exceeds `max`. 100 processes simultaneously call dispatch with
the same counter_id and `max = 10`. Exactly 10 succeed; the rest get
`{error, counter_full}` or `{error, retry_budget_exhausted}`.

```erlang
concurrent_increment_respects_max_test(_Config) ->
    StoreId = my_store,
    CounterId = <<"license-seats:test">>,
    Max = 10,
    NumWorkers = 100,
    Self = self(),

    Workers =
        [spawn_link(fun() ->
             receive go -> ok end,
             Result = evoq_decision_runtime:dispatch(
                 dcb_counter, StoreId,
                 #increment_counter{counter_id = CounterId, max = Max}),
             Self ! {result, N, Result}
         end) || N <- lists:seq(1, NumWorkers)],

    [W ! go || W <- Workers],

    Results = [receive {result, _N, R} -> R end || _ <- Workers],
    Successes = [R || {ok, _} = R <- Results],
    Errors    = Results -- Successes,

    ?assertEqual(Max, length(Successes)),
    ?assertEqual(NumWorkers - Max, length(Errors)),

    %% And the store holds exactly Max events under the counter tag.
    {ok, Events} = evoq_event_store:read_by_tags(
        StoreId, [<<"counter:license-seats:test">>], any, 1000),
    DcbEvents = [E || #{stream_id := <<"_dcb">>} = E <- Events],
    ?assertEqual(Max, length(DcbEvents)).
```

This works because Ra serializes the `khepri:transaction/2` bodies that
implement `append_if_no_tag_matches`. Concurrent contenders all read the
same context, all see the same cutoff, all call `decide`, but the
*conditional append* fails for all but the winner of each batch — the
others retry with a fresh read of the context, observe the new event,
and either succeed (if there's still room) or correctly fail with
`counter_full`.

---

## Why This Pattern Is Awkward With Aggregates

Two ways to implement the counter with `evoq_aggregate`:

**Option A: a "counter aggregate" per counter_id.**

```
Stream: counter-license-seats-tenant-42
  - counter_incremented_v1 (version 0, value=1)
  - counter_incremented_v1 (version 1, value=2)
  - ...
```

Works, but invents an aggregate where there isn't naturally one. Every
licence-seat tenant has its own one-stream-deep "counter object" that
exists solely to enforce a numerical limit. That's structurally
suspicious — there's no `initiate_counter` / `conclude_counter`
lifecycle, no business state beyond "how many." The aggregate is
ceremony.

**Option B: bolt it onto an existing aggregate (e.g., Tenant).**

```
Stream: tenant-42
  - tenant_registered_v1
  - tenant_licence_seat_claimed_v1
  - ...
```

Also works, but now every operation on the tenant (registration,
suspension, anything) serializes through the same Ra consensus group as
licence-seat claims. Per-tenant write throughput cap. The "counter"
domain bleeds into the "tenant" domain.

**Option C: `evoq_decision`.**

No fake aggregate. No bleed across domains. Each call hits its own
transaction; Ra serializes only the contended ones. The events
themselves still flow through the same store and the same projections —
you read the counter's value with the same `read_by_tags` you'd use for
analytics.

---

## v1 Limitations Worth Knowing

- **Compound filters shipped 2026-05-27 (evoq 1.19.0).** `context/1`
  can return `{any_of, [Tag]}`, `{all_of, [Tag]}`, or any recursive
  nesting via `{and_, [Filter]}` / `{or_, [Filter]}`. The runtime
  plumbs compound filters through the read path: walks the tree,
  reads tag-union once, refines client-side. Empty `{and_, []}` /
  `{or_, []}` short-circuits to an empty context, no read.
- **DCB-stream only.** The runtime considers events written via
  `evoq_decision` (under the `<<"_dcb">>` pseudo-stream) when computing
  the cutoff. Aggregate-stream events tagged the same way are visible
  to read APIs but invisible to the consistency check. For
  pure-DCB use cases this is correct.
- **Default retry budget = 3.** Override via `retry_budget/0` callback.
  Heavy contention should escalate to the caller rather than spin
  indefinitely.
- **HMAC chain shipped 2026-05-27 (reckon-db 3.1.0).** DCB events on
  integrity-enabled stores carry `prev_event_hash` + `mac` linked
  from genesis. Implementation pre-computes MAC chains outside the
  transaction (Horus extractor rejects `crypto:*` in transaction
  bodies); the transaction verifies the chain-tip + counter still
  match. Concurrent-writer contention on the chain tip retries up
  to `?INTEGRITY_RETRY_BUDGET = 5` times; exhaustion surfaces as
  `{error, dcb_concurrent_writer_exhausted}` and the caller should
  back off + retry the whole dispatch.

---

## Related

- [`philosophy/CONSISTENCY_BOUNDARIES.md`](../philosophy/CONSISTENCY_BOUNDARIES.md) — the discrimination rule (Dossier vs Decision) in full
- [`GLOSSARY.md`](../GLOSSARY.md) — definitions for Decision, Dossier, tag_filter
- [`philosophy/DDD.md`](../philosophy/DDD.md) — why we think in processes (Dossiers) by default
- External: `reckon-db-org/reckon-db/plans/PLAN_DCB_IMPLEMENTATION.md` — the implementation record across the four-repo stack

---

*Counter respects max. No aggregate ceremony. Ra serializes only the contended writes.*
