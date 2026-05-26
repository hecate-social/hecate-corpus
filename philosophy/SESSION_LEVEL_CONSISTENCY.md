---
title: SESSION-LEVEL CONSISTENCY
layer: philosophy
audience: [agent, human]
stage: stable
---

# SESSION-LEVEL CONSISTENCY — Mitigating Projection Race Conditions

*When the user initiates a command, the response should be immediately usable without waiting for projections.*

---

## The Problem

In event-sourced CQRS systems, there is a gap between **command acceptance** and **read model update**:

```
POST /api/domains/initiate
  -> Command dispatched
  -> Event stored in ReckonDB          (milliseconds)
  -> Projection receives event         (eventual — 10-500ms)
  -> SQLite read model updated         (eventual — depends on load)

GET /api/domains/:id                  (immediately after POST)
  -> Queries SQLite read model
  -> 404 Not Found                     (projection hasn't caught up!)
```

The user sees: "I just created something, but it doesn't exist."

This is not a bug. This is **eventual consistency working as designed**. But for the user's session, it's unacceptable.

---

## The Pattern

> **Initiate commands return FEEDBACK containing the resulting aggregate state.**
>
> The caller uses this FEEDBACK directly, without querying the read model.

### The Rules

1. **Every `initiate_*` command** carries a payload that represents an empty or near-empty default read model shape
2. **The command handler** returns a FEEDBACK containing either an Error or the resulting aggregate state
3. **The caller** (API handler, UI, agent) uses the FEEDBACK directly to update its local state
4. **Background refresh** of the read model happens asynchronously — the projection will catch up

### Why "Session-Level"?

The consistency guarantee is scoped to the **caller's session**:

- The user who initiated the command sees the result immediately
- Other users/sessions see it after projection (eventually consistent)
- This is the correct trade-off for interactive systems

---

## Correct Flow

```
POST /api/domains/initiate
  -> Command dispatched
  -> Event stored in ReckonDB
  -> Handler returns FEEDBACK with aggregate state
  -> API returns aggregate state to caller         <-- IMMEDIATE

Caller:
  -> Sets local state from POST response           <-- NO second query needed
  -> Background refresh of read model (optional)

Meanwhile (async):
  -> Projection processes event
  -> SQLite read model updated
  -> Other sessions now see the domain
```

---

## Wrong Flow (Antipattern)

```
POST /api/domains/initiate
  -> Command dispatched
  -> Returns only {ok, venture_id}

Caller:
  -> Immediately calls GET /api/domains/:venture_id    <-- RACE CONDITION
  -> 404 (projection hasn't caught up)
  -> Sets state to null
  -> User sees empty form / "not found"
```

**Why wrong:**
- Relies on projection completing before the next HTTP request
- Under load, projections may lag significantly
- Creates flickering UIs (form -> empty -> populated)
- Forces polling or retry logic as a workaround

---

## Implementation Examples

### Erlang: API Handler

```erlang
%% initiate_venture_api.erl
dispatch_event(Params, Req) ->
    case maybe_initiate_venture:dispatch(Cmd) of
        {ok, Version, Events} ->
            %% Return aggregate state in response — not just "ok"
            hecate_api_utils:json_ok(200, #{
                <<"ok">> => true,
                <<"venture_id">> => VentureId,
                <<"name">> => Name,
                <<"brief">> => Brief,
                <<"status">> => 1,
                <<"status_label">> => <<"Initiated">>,
                <<"initiated_at">> => erlang:system_time(millisecond),
                <<"initiated_by">> => InitiatedBy,
                <<"version">> => Version
            }, Req);
        {error, Reason} ->
            hecate_api_utils:bad_request(Reason, Req)
    end.
```

### TypeScript: Caller (Svelte Store)

```typescript
// Set active domain DIRECTLY from POST response
const resp = await api.post('/api/domains/initiate', { name, brief });

const domain: Domain = {
    venture_id: resp.venture_id as string,
    name: resp.name as string,
    brief: (resp.brief as string) ?? null,
    status: resp.status as number,
    status_label: (resp.status_label as string) ?? 'Initiated',
    // ... all fields from response
};

activeVenture.set(domain);   // Immediate — no second query
fetchVentures();              // Background refresh — projection catches up
```

---

## When to Apply

| Scenario | Apply Session-Level Consistency? |
|----------|--------------------------------|
| `initiate_*` commands (aggregate birth) | **Always** |
| `archive_*` commands (aggregate removal) | **Always** — caller should remove from local state |
| Status-changing commands (start, pause, complete) | **Recommended** — return updated status |
| Internal process manager commands | **No** — no user session to satisfy |
| Mesh-triggered commands | **No** — eventual consistency is fine |

---

## Relationship to CQRS

This pattern does **NOT** violate CQRS:

- The **write side** still stores events in ReckonDB
- The **read side** still projects events to SQLite
- The **separation** is maintained

What changes is: **the command response carries enough state for the caller's immediate needs**. This is standard practice in systems like:

- Commanded (Elixir) — aggregates return state after execution
- Axon Framework (Java) — command gateway returns result
- EventStoreDB — append returns write position

The projection remains the **source of truth for queries**. Session-level consistency is a **performance optimization for the initiating session only**.

---

## Key Takeaways

1. **Commands return state, not just acknowledgment** — especially `initiate_*` commands
2. **Callers use command response directly** — no immediate follow-up query
3. **Background refresh** handles eventual consistency for the read model
4. **Other sessions** are eventually consistent (this is fine)
5. **Never rely on immediate GET after POST** in event-sourced systems

---

## See Also

- [HECATE_WALKING_SKELETON.md](HECATE_WALKING_SKELETON.md) — The skeleton's `initiate_*` desk should follow this pattern from day 1
- [CARTWHEEL.md](CARTWHEEL.md) — CMD department architecture
- [antipatterns/INDEX.md](../skills/antipatterns/INDEX.md) — Demon #24 (Silent Subscription Pipeline Failures) is a consequence of ignoring this pattern

---

*The user who took the action should never have to wait for the system to agree it happened.*
