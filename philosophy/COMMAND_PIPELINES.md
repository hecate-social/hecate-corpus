# Command Pipelines

*The chain-of-responsibility that prepares commands so aggregates remain pure functions of state + payload.*

---

## The Invariant

> **An aggregate's `execute(State, Payload)` is a pure function of two inputs.**

Any external information the aggregate needs MUST be in the payload by the time the aggregate sees it. If it isn't, the pipeline didn't do its job.

This invariant has consequences:

- Aggregates are deterministically replayable from event history alone.
- Aggregates are trivially testable (no mocking, no I/O, no time travel).
- A replay 12 months later does not need the same external services still to be up.
- Event flow never blocks on cross-domain reads.
- Cross-domain reads are explicit, named, traceable steps — not inline calls in handler code.

This is the structural cure for [Demon 41 (THE Cardinal Sin)](../skills/ANTIPATTERNS_EVENT_SOURCING.md#-demon-41-reading-from-read-models-during-event-flow--the-cardinal-sin).

---

## The Pattern

Every request that mutates state — whether a HOPE from outside or a domain-internal command from a PM — flows through a pipeline of steps before reaching the aggregate.

```
HOPE  /  Trigger Event
   ↓
Responder (HOPE → Command)  /  PM (Event → Command)
   ↓
[Pipeline of Steps]
   ├── authenticate
   ├── enforce_idempotency
   ├── enrich_with_{external_state}
   ├── enrich_with_{...}
   └── validate_business_rules
   ↓
Enriched Command
   ↓
maybe_{command} (handler)
   ↓
Aggregate.execute(State, EnrichedCommand)
   ↓
Event (carrying enrichment provenance in metadata)
```

The pipeline owns every external read. The aggregate owns every business decision. They never overlap.

---

## The Step Contract

A pipeline step has a fixed shape:

```erlang
-callback name() -> atom().
-callback apply(Cmd, Ctx) ->
      {ok,   Cmd1, Ctx1}     %% proceed with possibly modified cmd/ctx
    | {skip, Ctx1}            %% proceed unchanged; useful for instrumentation
    | {halt, Result}          %% pipeline succeeds, skip remaining steps
    | {error, Reason}.        %% halt with error
```

**Four return shapes, each meaningful:**

- **`{ok, Cmd, Ctx}`** — the usual path. Step transformed the command and/or enriched the context.
- **`{skip, Ctx}`** — step didn't touch the command. Useful for logging, audit, telemetry steps that only observe.
- **`{halt, Result}`** — successful early termination. The canonical case is idempotency: "this command was already processed; here's the cached response, don't dispatch." Structurally different from error.
- **`{error, Reason}`** — pipeline halts; caller translates to error response.

The `halt` return is load-bearing. Without it, idempotency steps have to fake success by skipping aggregate dispatch in a follow-up check, leaking pipeline knowledge into callers.

---

## The Pipeline Declaration

A pipeline is a module that declares its steps in order:

```erlang
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

The order matters. Enrichment steps run before validation steps that depend on them. The pipeline declaration IS the integration contract — every cross-domain dependency for this command is visible here, in a single file, in evaluation order.

If a command genuinely needs two different chains, that's usually a sign it should be two different commands (e.g., `record_card_exit_v1` vs `record_permit_exit_v1`), each with its own pipeline. Dynamic step lists hide what static ones reveal.

---

## Failure Semantics

Each step declares its failure mode:

| Mode | Behaviour |
|------|-----------|
| `fail_fast` (default) | Step returns `{error, Reason}` → pipeline halts |
| `fail_soft` | Step returns `{error, Reason}` → pipeline continues; error recorded in ctx metadata |
| `{default, Value}` | Step returns `{error, Reason}` → continues with `Value` used in place of enrichment |

`fail_soft` is for enrichment that's nice-to-have (cosmetic field, optional flag, advisory enrichment). `{default, V}` is for enrichment that has a known safe fallback. `fail_fast` is the default — most enrichments are load-bearing.

**Be conservative.** Prefer `fail_fast` unless you have thought hard about what "continuing without this data" actually means for the aggregate's decision.

---

## Enrichment Discipline

### Timestamp every snapshot

Enrichment that reads external state captures the read-time, attaches it to context, and the runner propagates it to event metadata:

```erlang
apply(Cmd, Ctx) ->
    case lookup_permit(Cmd) of
        {ok, Permit} ->
            Cmd1 = Cmd#{permit_id => maps:get(permit_id, Permit),
                        permit_valid_until => maps:get(valid_until, Permit)},
            Ctx1 = put_meta(Ctx, permit_snapshot_at, erlang:system_time(millisecond)),
            Ctx2 = put_meta(Ctx1, permit_source, <<"local_pricing_projection">>),
            {ok, Cmd1, Ctx2};
        {error, not_found} ->
            {error, no_active_permit}
    end.
```

The aggregate branches on `permit_id` and `permit_valid_until` (payload). The event metadata carries `permit_snapshot_at` and `permit_source` (provenance). Downstream consumers reason about freshness without re-reading.

### Idempotency keys live on the HOPE, not the enriched command

Two retries of the same HOPE should dedup even if enrichment produced slightly different snapshots between them. The dedup key is on the inbound request's stable identity (`command_id` assigned by the responder, or the HOPE's `request_id`), not on the post-enrichment payload. Otherwise retries get treated as new commands and you write duplicate events.

### Payload vs metadata: structural separation

Enrichment splits into two:

- **Payload** — data the aggregate decides on (`permit_id`, `fee_amount`, `session_status`). Part of the command. Becomes part of the event.
- **Metadata** — provenance (snapshot timestamps, source projection, query duration, filter chain ID). Lives in a separate metadata block.

Never `maps:merge` them. [Demon 37 (Flattening Event Envelopes into Business Data)](../skills/ANTIPATTERNS_EVENT_SOURCING.md) is real and silent.

Convention: ctx keys under a reserved `'__meta'` map get serialized into the dispatched command's `metadata` field by the runner just before handing off to the aggregate.

---

## PMs Are Pipelines Too

A process manager that reacts to a cross-domain event is structurally the same as a responder reacting to a HOPE: it receives a trigger, runs an enrichment pipeline, dispatches an enriched command.

```
Trigger Event (from pg subscription)
   ↓
PM gen_server (in its sibling slice — see PROCESS_MANAGERS.md)
   ↓
[Pipeline of Steps]
   ↓
Dispatch to target aggregate
```

The PM module stays thin — it `pg:join`s in `init/1` and runs `evoq_pipeline:run(PipelineMod, Cmd, Ctx)` on each event. All the cross-domain work lives in the pipeline. See [PROCESS_MANAGERS.md](PROCESS_MANAGERS.md) for the PM slice structure.

This collapses two patterns into one. There is no separate "PM with enrichment logic" pattern — there is just "pipeline, with HOPE-driven or event-driven entry."

---

## What Goes In The Pipeline, What Doesn't

| Belongs in pipeline | Belongs elsewhere |
|---------------------|-------------------|
| Authentication | Transport-level concerns (TLS, mesh handshake) |
| Authorization (capability check) | Routing (which aggregate stream id?) |
| Idempotency check | Storage (event store append) |
| External-state enrichment | Event folding (aggregate's `apply/2`) |
| Business validation that needs external data | Event emission (aggregate's `execute/2`) |
| Snapshot-timestamping | Projection writes |
| Pure input shape validation | Anything that runs AFTER the aggregate |
| Audit / telemetry of command receipt | Audit of event emission (in projection or emitter) |

The pipeline runs BEFORE the aggregate. Once the aggregate fires, the event flows to projections and PMs — that's a different concern.

---

## The HOPE Identity Rule

> **The HOPE identity defines idempotency. The enriched command defines what the aggregate sees. These are not the same thing.**

If your idempotency step uses the post-enrichment command as its key, retries with the same HOPE but slightly different enrichment snapshots will produce duplicate aggregate writes. Always key idempotency off the inbound request's stable identity (`command_id` assigned by the responder, or HOPE's `request_id`).

---

## Connection to Other Concepts

- **Cardinal sin cure.** This pattern is the structural cure for [Demon 41](../skills/ANTIPATTERNS_EVENT_SOURCING.md#-demon-41-reading-from-read-models-during-event-flow--the-cardinal-sin). Aggregates that need external data get it via pipeline, never by reading during event flow.
- **PM placement.** PMs live as sibling slices in the target CMD app (see [PROCESS_MANAGERS.md](PROCESS_MANAGERS.md)). The pipeline lives in those slices alongside the gen_server and supervisor.
- **Session consistency.** Enrichment timestamps in metadata enable downstream consumers to reason about freshness, which underpins [Session-Level Consistency](SESSION_LEVEL_CONSISTENCY.md).
- **Mesh boundary.** Across facilities, enrichment reads from local denormalized projections of mesh-published facts — not from cross-WAN RPCs. The local projection IS the enrichment cache.

---

## Future: evoq Behaviour Support

This pattern is a candidate for framework-level support in `evoq`:

- `evoq_pipeline_step` — the step contract above
- `evoq_pipeline` — the pipeline declaration contract above
- `evoq_pipeline:run/3` and `evoq_pipeline:run_async/3` — runners
- Telemetry spans: `[evoq, pipeline, run, start|stop]`, `[evoq, pipeline, step, start|stop|exception]`

Applications that adopt the pattern today by hand-rolling the runner can migrate to `evoq_pipeline:run/3` when it ships without code changes — only the call site changes.

See `reckon-db-org/evoq/proposals/PROPOSAL_EVOQ_PIPELINE.md` for the framework-level proposal.

---

## Key Takeaways

1. **Aggregates are pure functions of state + payload.** This is the invariant the pipeline protects.
2. **All external reads happen in pipeline steps** — never in aggregates, handlers, or PMs' event-flow code.
3. **Steps return one of four shapes**: `ok` / `skip` / `halt` / `error`. `halt` is for successful early termination (idempotency).
4. **Pipelines are declarative module callbacks** — `steps/0` returns the chain.
5. **Enrichment is timestamped** and propagated to event metadata as provenance.
6. **Payload and metadata stay structurally separate** (`'__meta'` ctx convention).
7. **Idempotency keys live on the HOPE**, not on the enriched command.
8. **PMs use pipelines too** — same shape, different trigger.
9. **Cross-cluster reads happen at pipeline time, against local projections of mesh-published facts** — never via runtime cross-cluster RPC.

---

*Date: 2026-05-25. Pattern introduced to resolve [Demon 41](../skills/ANTIPATTERNS_EVENT_SOURCING.md#-demon-41-reading-from-read-models-during-event-flow--the-cardinal-sin) for the hecate-parksim family and generalized as a framework-level pattern.*
