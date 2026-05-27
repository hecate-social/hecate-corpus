---
title: Consistency Boundaries
layer: philosophy
audience: [agent, human]
stage: stable
---

# Consistency Boundaries

*Where Hecate stands in the aggregate vs aggregateless debate, and why.*

**Date:** 2026-05-26
**Status:** Active doctrine
**Origin:** Aggregateless ES (Rico Fritzsche / FACTSTR) and DCB (Sara Pellegrini) entered mainstream discourse; the corpus needed an explicit position.

---

## The two schools

Event sourcing has split into two camps over the last several years. The split is not about events themselves; both camps agree on the immutability of events, replay-as-truth, and projections for read models. The split is about **where the consistency boundary lives**.

### School A — stream-per-aggregate (classical DDD + ES)

- Each "thing with identity" gets its own stream
- The stream's version is the consistency unit
- An append checks `expected_version == current_version` atomically
- The aggregate is the unit of identity, lifetime, and concurrency, all at once
- Implementations: EventStoreDB, Marten, Axon, **ReckonDB**

### School B — aggregateless / DCB / facts-first

- Events are pure facts in a flat log
- The log itself is the storage primitive; "streams" are filter views
- An append checks `no events matching this tag-filter have appeared since sequence N`
- The consistency unit is a *query*, defined per-decision at write time
- Implementations: FACTSTR (Rust), [dcb.events](https://dcb.events/) reference implementations, parts of NServiceBus

The two schools agree on what an event is. They disagree on what an aggregate is and on what concurrency check protects a write.

---

## Hecate's position: stream-per-Dossier, process-centric framing

Hecate stays firmly in School A, with a process-centric twist that goes beyond classical DDD.

The Dossier is not just a write-side concurrency unit. It is:

- An **identity** unit (the stream ID names this thing)
- A **lifetime** unit (initiate → open → shelve → resume → conclude → archive)
- A **structural** unit (one folder per Dossier in the codebase, screaming what it does)
- A **process** unit (the Dossier carries facts as it passes through Desks)
- A **concurrency** unit (the stream-version check protects the write)

The Dossier earns its keep on the *modelling* side. The concurrency primitive is the smallest of its jobs. If we adopted aggregateless writes, we would still want the Dossier for the other four reasons.

---

## The aggregateless critique, addressed

The critique deserves a serious answer because parts of it are correct.

### "Aggregates pre-decide the consistency boundary, and you keep getting it wrong."

True in classical DDD where aggregates are designed around *data shapes* (customer, order, account). Boundaries shift as the business shifts; stream migrations cost.

Less true in Hecate. Dossiers are designed around *processes*, not data. A process has a clear scope and a clear lifetime by construction. Refactoring a Dossier boundary is rarer than refactoring a data-shape aggregate, because processes are stabler than data shapes.

### "Cross-entity invariants force you into sagas or process-manager dances."

True. The cure in Hecate is **process guards using read models inside command pipelines** (see `philosophy/COMMAND_PIPELINES.md`). The pipeline pre-loads cross-entity state, enriches the command, and hands the aggregate a self-contained decision input. The Dossier records WHAT WE KNEW at decision time, including the cross-entity facts.

This costs:
- One extra step in the pipeline (a named, traceable read)
- Slightly weaker consistency than a real distributed transaction (the read can race; idempotency handles the rare conflict)

It pays:
- The cross-entity decision is auditable in the event stream (the enriched command's metadata records the source of the cross-entity reads)
- No saga complexity
- No tag index maintenance in the consensus layer
- No double-query cost at append time

### "Reading from read models inside the aggregate is forbidden (Demon 41) — that's a tell."

True, and the workaround (pipelines) is real machinery. Aggregateless ES doesn't have this constraint because the decision function is allowed to query whatever it needs.

Hecate's answer: the constraint is the point. The structural separation between **decision data (loaded at the boundary)** and **commit data (the aggregate's own slips)** preserves replay determinism and makes audit straightforward. Aggregateless ES trades replay determinism for write-time flexibility. We chose the opposite trade deliberately.

---

## The discrimination rule

When you're about to model a new process or capability, you choose between two write-time shapes. The discrimination rule is straightforward.

**Use a Dossier when:**
- The work has a natural identity-carrier (a thing, an instance, a workflow)
- The work has a lifetime (something starts, something ends)
- You want one folder in the codebase that screams what this is
- Multiple Desks contribute to the same instance over time
- You want a stream you can replay to audit the whole instance

This covers ~90% of business work. Customers, orders, devices, capabilities, plugins, sessions, divisions, plannings, craftings, all Dossiers.

**The remaining 10% — cross-cutting decisions without an identity-carrier:**
- Uniqueness checks (email, username, MRI)
- Idempotency / replay protection on cross-cutting commands
- Allocation against shared resources (license seats, slots, quotas)
- Rate limits across a tenant
- Anti-fraud / anomaly decisions over recent activity

These do NOT fit the Dossier shape cleanly. Three options for handling them in Hecate today:

1. **Invent a registry Dossier.** Make `email_uniqueness` or `license_seats` a Dossier in its own right. The "thing" becomes the registry, not the individual record. Works but feels forced.
2. **Process guard via command pipeline.** Pre-load the relevant read-model slice at the boundary. Append into a fresh Dossier (e.g., the per-user signup Dossier) with the uniqueness fact embedded in metadata. The risk is a race-condition window between read and append; mitigation is idempotency keys + reconciliation.
3. **Process Manager dance.** Issue a request, wait for a fact, condition the local command on the fact. Heaviest but most explicit.

Aggregateless ES handles these natively with one query + one conditional append. Hecate handles them with one of the three options above. This is a real cost. We pay it.

---

## What would change the position?

The current doctrine is not unconditional. It would be revisited if:

1. **A real Hecate process surfaces where all three workarounds above feel structurally wrong.** Not just inconvenient. Structurally wrong, repeatedly, across several Divisions.
2. **The 10% case grows to 30%+ in some Domain.** If a Domain's work is dominated by cross-cutting decisions without identity-carriers, the Dossier-first defaults are wrong for that Domain.
3. **Khepri or Ra gain native tag-filter conditional append.** Today the cost of adding it to ReckonDB is non-trivial (one new behaviour callback, one new Khepri custom command, scan performance work). Upstream support would lower that cost dramatically.

Until one of these is true, the position holds. ReckonDB ships tags and tag-filter reads + subscriptions (Phase 1 + 2 of the existing research). It does NOT ship query-based concurrency (Phase 3, deferred). See `reckon-db-org/reckon-db/plans/PLAN_FUTURE_RESEARCH.md` for the technical detail.

---

## Decision — active capability (shipped 2026-05-27)

**Updated 2026-05-27.** This section previously described Decision as a reserved-but-unimplemented term. That position changed: between P3.0 and P3.9 (one focused build sprint, see `reckon-db-org/reckon-db/plans/PLAN_DCB_IMPLEMENTATION.md`), the full four-repo stack landed and Decision became a first-class user-facing construct alongside Dossier.

### What sits alongside Dossier now

| | **Dossier** | **Decision** |
|---|-------------|--------------|
| Identity | Yes — stream id | No |
| Lifetime | Yes (initiate → conclude → archive) | No |
| Folder home | Yes — one folder per Dossier | No — stateless behaviour module |
| Concurrency primitive | Stream-version optimistic concurrency | Tag-filter context-query optimistic concurrency |
| Erlang behaviour | `evoq_aggregate` | `evoq_decision` |
| Storage append | `append_events/3,4` | `append_if_no_tag_matches/4` |
| When to reach for it | The work has a natural "thing" with state and history | The work is cross-cutting (uniqueness, allocation, idempotency, rate limits) |
| Discrimination rule | Default for ~90% of business work | Escape hatch for the ~10% genuinely cross-cutting case |

Both produce events. Both feed projections. Both flow through the same store. The difference is whether identity-and-lifetime are first-class for the consistency boundary.

### Discrimination rule: when to reach for which

**Use a Dossier when:**

- The work has a natural identity-carrier (a customer, an order, a division)
- There's a lifecycle (something starts, something ends)
- Multiple Desks contribute to the same instance over time
- You want one folder in the codebase that screams what it is

**Use a Decision when:**

- The consistency check is uniqueness across many things (email, MRI, idempotency key)
- The consistency check is allocation against a shared resource (seats, slots, quotas)
- The consistency check is a count/window over recent activity (rate limits, anti-fraud)
- There's no natural "thing" to hang the work on, and inventing one would be ceremony

If you're tempted to invent a "registry aggregate" (e.g., `email_uniqueness_aggregate` that owns all emails) to satisfy an aggregate-first architecture, that's the tell — use a Decision instead.

### Reference example

See [`../examples/DCB_COUNTER.md`](../examples/DCB_COUNTER.md) for the canonical walkthrough: a counter that respects a maximum, implemented as an `evoq_decision`, with the concurrent-contention test showing 100 procs racing on `max=10` and exactly 10 succeeding.

### Stack reference

| Repo | Version | What it ships |
|------|---------|---------------|
| reckon-gater | 2.3.0 | `tag_filter()` + `seq_cutoff()` types, `append_if_no_tag_matches/4` wire verb |
| reckon-db | 3.1.0 | Khepri-transaction primitive + `/by_tag/{tag}/{seq}` tag index + gateway dispatch |
| reckon-evoq | 2.2.0 | Adapter passthrough |
| evoq | 1.18.0 | `evoq_decision` behaviour + `evoq_decision_runtime:dispatch/3` |

### Known v1 limitations

- **Compound filters at the evoq layer shipped 2026-05-27 (evoq 1.19.0).** `context/1` can return `{any_of, [Tag]}`, `{all_of, [Tag]}`, or any recursive nesting via `{and_, [Filter]}` / `{or_, [Filter]}` matching the backend's `tag_filter()` exactly. The runtime collects the union of referenced tags, reads broadly via `read_by_tags`, and refines client-side using per-event semantics.
- **DCB-stream only.** The runtime considers events under the `<<"_dcb">>` pseudo-stream when computing the cutoff. Mixed-mode use cases (aggregate streams + DCB sharing tags) are unsupported — use `evoq_aggregate` for per-aggregate, `evoq_decision` for pure cross-cutting.
- **HMAC chain shipped 2026-05-27.** DCB events on integrity-enabled stores carry `prev_event_hash` + `mac` linked from genesis. Implementation uses outside-the-transaction MAC pre-computation with inside-the-transaction chain-tip + counter verification (Horus rejects `crypto:*` inside transaction bodies). Concurrent-writer contention on the chain tip is bounded by `?INTEGRITY_RETRY_BUDGET = 5`; exhaustion surfaces as `{error, dcb_concurrent_writer_exhausted}`. Tampering detection works via the existing `reckon_gater_integrity:verify_event/3`.
- **No options API.** Retry budget set via `retry_budget/0` callback only; no per-call options map yet.

None of these block normal Decision use — they're flagged so callers know the edges.

---

## What this doc is NOT

- It is not a critique of FACTSTR or Rico Fritzsche's article. The aggregateless case is principled and well-argued.
- It is not a dismissal of DCB. Sara Pellegrini's framework is the cleanest articulation of the alternative; her enrollment example is the canonical case to study.
- It is not unconditional. It is a stance based on Hecate's current process-centric defaults, with clear conditions under which it would be revisited.

---

## Related

- [DDD.md](DDD.md) — the Dossier Principle, the foundation of the process-centric stance.
- [CARTWHEEL.md](CARTWHEEL.md) — Division Architecture, where Dossiers live.
- [HECATE_DOMAIN_LIFECYCLE.md](HECATE_DOMAIN_LIFECYCLE.md) — the venture-level process model.
- [COMMAND_PIPELINES.md](COMMAND_PIPELINES.md) — the structural cure for Demon 41; the mechanism that handles cross-entity decisions without leaving the Dossier model.
- [../skills/antipatterns/event_sourcing.md](../skills/antipatterns/event_sourcing.md) — Demon 41 (reading from read models during event flow).
- External: [PLAN_FUTURE_RESEARCH.md in reckon-db](https://codeberg.org/reckon-db-org/reckon-db/src/branch/main/plans/PLAN_FUTURE_RESEARCH.md) — the technical research that grounds this doctrine.
- External: Rico Fritzsche, [Aggregateless Event Sourcing](https://ricofritzsche.me/aggregateless-event-sourcing/) and [FACTSTR](https://factstr.com/).
- External: Sara Pellegrini, [dcb.events](https://dcb.events/), [event-thinking.io](https://sara.event-thinking.io/).

---

*The Dossier is the cornerstone. The Decision is the escape hatch.*
