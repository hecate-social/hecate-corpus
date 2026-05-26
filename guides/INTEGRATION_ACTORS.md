---
title: Integration Actors and Artifacts
layer: guide
audience: [agent, human]
stage: stable
---

# Integration Actors and Artifacts

How domain events cross boundaries. This guide covers the 4 artifact types, 4 actor types, feedback, and the conventions that bind them.

---

## Overview

Integration in evoq has two dimensions:

- **Artifacts** -- the data shapes that cross boundaries
- **Actors** -- the processes that move artifacts across boundaries

```
DOMAIN (bounded context)                    OUTSIDE
-----------------------------------------  ----------------------------------------
Command  ->  Aggregate  ->  Event
                               |
                               +-> Emitter  -----> Fact  -----> pg / mesh
                               |
                               +-> Emitter  -----> Fact  -----> pg / mesh
                                                                    |
                                                          Listener <+
                                                              |
                                                          Command (in target domain)

Requester  -----> Hope  -----> mesh  -----> Responder
                                               |
                                           Command (dispatched)
                                               |
                                           Feedback <-- aggregate state
                                               |
Requester <---- Feedback <---- mesh <----------+
```

---

## The 4 Artifact Types

Artifacts are the DATA that crosses boundaries. They are defined as evoq behaviours.

| Artifact | Level | Key Type | Serialization | Direction | Behaviour |
|----------|-------|----------|---------------|-----------|-----------|
| **Command** | Domain | atom | Erlang terms | Inbound | `evoq_command` |
| **Event** | Domain | atom | Erlang terms | Internal | `evoq_event` |
| **Fact** | Integration | binary | JSON | Outbound (fire-and-forget) | `evoq_fact` |
| **Hope** | Integration | binary | JSON | Outbound RPC (expects feedback) | `evoq_hope` |

### Domain Artifacts: Command and Event

Domain artifacts stay inside the bounded context. They use atom keys because they never leave the BEAM VM.

- **Command** (`evoq_command`) -- user intent. Validated before execution. `new/1` returns `{ok, Cmd}`.
- **Event** (`evoq_event`) -- immutable fact. `new/1` returns `Event` directly (no wrapper -- event construction from a validated handler should not fail).

### Integration Artifacts: Fact and Hope

Integration artifacts cross boundaries. They use binary keys because they must be JSON-serializable.

- **Fact** (`evoq_fact`) -- fire-and-forget. A domain event happens, a fact is published. Nobody waits for a response.
- **Hope** (`evoq_hope`) -- request. An agent sends a hope and hopes for a response. Think RPC, but honest about reliability.

### Feedback

**Feedback** is the response to a Hope. It carries the result of command execution at the responder side:

- `{ok, AggregateState}` -- command succeeded, here is the post-event state
- `{error, Reason}` -- command failed, here is why

Feedback enables **session-level consistency**: the requester gets immediate truth about what happened, without waiting for projections to catch up at the remote end.

Feedback is serialized as a map with binary keys:
```erlang
%% Success
#{<<"status">> => <<"ok">>,
  <<"state">> => #{<<"venture_id">> => ..., <<"name">> => ..., ...}}

%% Failure
#{<<"status">> => <<"error">>,
  <<"reason">> => <<"venture_already_exists">>}
```

Feedback is defined as `evoq_feedback` behaviour for typed serialization/deserialization.

---

## The 4 Actor Types

Actors are the PROCESSES that move artifacts across boundaries. Each is an evoq behaviour and a gen_server.

| Actor | What It Does | Input | Output | Transport | Behaviour |
|-------|-------------|-------|--------|-----------|-----------|
| **Emitter** | Publishes facts from domain events | Domain event | Fact | pg / mesh | `evoq_emitter` |
| **Listener** | Receives facts, dispatches commands | Fact | Command | pg / mesh | `evoq_listener` |
| **Requester** | Sends hopes, receives feedback | Hope | Feedback | mesh | `evoq_requester` |
| **Responder** | Receives hopes, dispatches commands, returns feedback | Hope | Feedback | mesh / hecate:// | `evoq_responder` |

### Emitter

Subscribes to domain events via evoq. When an event matches, translates it to a fact (using the `evoq_fact` module) and publishes to the target transport.

```erlang
-behaviour(evoq_emitter).

%% Which event triggers this emitter
source_event() -> venture_initiated_v1.

%% Which fact module translates the event
fact_module() -> venture_initiated_fact_v1.

%% Where to publish
transport() -> pg.  %% or mesh
```

**Placement:** Emitters live in the **source desk** -- the same desk that processes the command producing the event. They are supervised by the desk supervisor.

**Naming:** `emit_{event}_to_{transport}`

```
apps/guide_venture_lifecycle/src/initiate_venture/
    initiate_venture_v1.erl             # command
    venture_initiated_v1.erl            # event
    venture_initiated_fact_v1.erl       # fact (translates event for integration)
    emit_venture_initiated_v1_to_pg.erl   # emitter (pg)
    emit_venture_initiated_v1_to_mesh.erl # emitter (mesh)
    maybe_initiate_venture.erl          # handler
    initiate_venture_desk_sup.erl       # supervisor (starts emitters)
```

### Listener

Receives facts from pg groups or mesh topics. Translates the fact into a command and dispatches it to the local aggregate.

```erlang
-behaviour(evoq_listener).

%% Which fact type to listen for
source_fact() -> <<"hecate.domain.initiated">>.

%% Where to listen
transport() -> pg.  %% or mesh

%% Handle received fact
handle_fact(FactType, Payload, Metadata) ->
    case initiate_division_v1:from_fact(Payload) of
        {ok, Cmd} -> maybe_initiate_division:dispatch(Cmd);
        {error, _} -> skip
    end.
```

**Placement:** Listeners get their **own slice directory** in the target CMD app -- like process managers. This gives filesystem-level discoverability of external integration points.

**Naming:** `on_{fact}_from_{transport}_{command}`

```
apps/design_division/src/
    on_venture_initiated_from_pg_initiate_division/
        on_venture_initiated_from_pg_initiate_division.erl      # listener
        on_venture_initiated_from_pg_initiate_division_sup.erl  # supervisor
```

**Why own slice?** When you browse `src/`, the `on_*` directories scream which external facts this domain reacts to. Without dedicated directories, integration points are buried inside handlers -- invisible when browsing the codebase. This is the same reasoning as process managers.

### Requester

Sends a hope over mesh and waits for feedback. Used for cross-daemon RPC.

```erlang
-behaviour(evoq_requester).

%% Which hope type
hope_module() -> discover_venture_hope_v1.

%% Send and wait
request(Hope, Opts) ->
    Payload = discover_venture_hope_v1:to_payload(Hope),
    case hecate_mesh:call(<<"hecate.domain.discover">>, Payload, Opts) of
        {ok, FeedbackPayload} ->
            evoq_feedback:from_payload(FeedbackPayload);
        {error, Reason} ->
            {error, Reason}
    end.
```

**Placement:** Requesters live in the desk that initiates the RPC call.

**Naming:** `request_{hope_type}`

### Responder

Receives hopes (via mesh or hecate:// API), dispatches commands to the local aggregate, and returns feedback containing the post-event aggregate state.

```erlang
-behaviour(evoq_responder).

%% Which hope type this responder handles
hope_type() -> <<"hecate.domain.discover">>.

%% Handle incoming hope
handle_hope(HopeType, Payload, Metadata) ->
    case discover_venture_hope_v1:from_payload(Payload) of
        {ok, Hope} ->
            Cmd = build_command(Hope),
            case evoq_dispatcher:dispatch_with_state(Cmd) of
                {ok, _Version, _Events, State} ->
                    {ok, State};
                {error, Reason} ->
                    {error, Reason}
            end;
        {error, Reason} ->
            {error, Reason}
    end.
```

**Placement:** Responders live in the target desk (like API handlers).

**Naming:** `{command}_responder_v1`

---

## Session-Level Consistency

The feedback pattern enables session-level consistency for RPC:

1. Requester sends Hope over mesh
2. Responder dispatches command to aggregate
3. Aggregate executes command, produces events, applies them to state
4. Responder returns the **post-event aggregate state** as feedback
5. Requester receives immediate truth -- no need to query remote read models

This is critical for interactive systems where the caller needs to know the result NOW, not after projections catch up at the remote end.

```
Console A (requester)                Console B (responder)
─────────────────────                ────────────────────
Hope: "discover domain X"  ──mesh──>  receive hope
                                       dispatch command
                                       event stored
                                       state updated
                              <──mesh──  Feedback: {ok, State}
Use State directly
(no remote query needed)
```

### When to Use

| Scenario | Session-Level? | Why |
|----------|---------------|-----|
| `initiate_*` commands | Always | Caller needs the born aggregate state |
| `archive_*` commands | Always | Caller needs confirmation to remove from UI |
| Status-changing commands | Recommended | Caller needs updated status |
| Internal process manager commands | No | No user session to satisfy |
| Fire-and-forget mesh facts | No | No response expected |

### Implementation in evoq

Session-level consistency requires `evoq_dispatcher:dispatch_with_state/1` -- an additive API that returns `{ok, Version, Events, State}` instead of just `{ok, Version, Events}`. The aggregate state after applying all new events is included in the response.

---

## Naming Conventions Summary

### Artifacts (Data)

| Artifact | Module Pattern | Example |
|----------|---------------|---------|
| Command | `{verb}_{subject}_v1` | `initiate_venture_v1` |
| Event | `{subject}_{verb_past}_v1` | `venture_initiated_v1` |
| Fact | `{event}_fact_v1` | `venture_initiated_fact_v1` |
| Hope | `{verb}_{subject}_hope_v1` | `discover_venture_hope_v1` |
| Feedback | `{hope}_feedback_v1` | `discover_venture_feedback_v1` |

### Actors (Processes)

| Actor | Module Pattern | Example |
|-------|---------------|---------|
| Emitter (pg) | `emit_{event}_to_pg` | `emit_venture_initiated_v1_to_pg` |
| Emitter (mesh) | `emit_{event}_to_mesh` | `emit_venture_initiated_v1_to_mesh` |
| Listener (pg) | `on_{fact}_from_pg_{command}` | `on_venture_initiated_from_pg_initiate_division` |
| Listener (mesh) | `on_{fact}_from_mesh_{command}` | `on_app_available_from_mesh_install_plugin` |
| Requester | `request_{hope_type}` | `request_discover_venture` |
| Responder | `{command}_responder_v1` | `initiate_venture_responder_v1` |

### Placement Rules

| Actor | Lives In | Supervised By | Gets Own Slice? |
|-------|----------|--------------|-----------------|
| Emitter | Source desk (where event is produced) | Desk supervisor | No (part of desk) |
| Listener | Target CMD app | Own supervisor | **Yes** (own directory) |
| Requester | Desk that initiates RPC | Desk supervisor | No (part of desk) |
| Responder | Target desk (like API handler) | Desk supervisor | No (part of desk) |

---

## Complete Desk Example

### Source Side: `initiate_venture` desk

```
apps/guide_venture_lifecycle/src/initiate_venture/
    initiate_venture_v1.erl                 # Command (evoq_command)
    venture_initiated_v1.erl                # Event (evoq_event)
    venture_initiated_fact_v1.erl           # Fact (evoq_fact)
    emit_venture_initiated_v1_to_pg.erl     # Emitter (evoq_emitter)
    emit_venture_initiated_v1_to_mesh.erl   # Emitter (evoq_emitter)
    maybe_initiate_venture.erl              # Handler
    initiate_venture_api.erl                # API entry point
    initiate_venture_responder_v1.erl       # Responder (evoq_responder)
    initiate_venture_desk_sup.erl           # Supervisor
```

### Target Side: `design_division` CMD app reacting to domain facts

```
apps/design_division/src/
    on_venture_initiated_from_pg_initiate_division/
        on_venture_initiated_from_pg_initiate_division.erl      # Listener
        on_venture_initiated_from_pg_initiate_division_sup.erl  # Supervisor
    initiate_division/
        initiate_division_v1.erl
        division_initiated_v1.erl
        emit_division_initiated_v1_to_pg.erl
        maybe_initiate_division.erl
        initiate_division_desk_sup.erl
```

### RPC Side: Agent A requesting Agent B

```
Agent A (requester side):
apps/discover_ventures/src/discover_venture/
    discover_venture_hope_v1.erl            # Hope (evoq_hope)
    request_discover_venture.erl            # Requester (evoq_requester)

Agent B (responder side):
apps/guide_venture_lifecycle/src/initiate_venture/
    initiate_venture_responder_v1.erl       # Responder (evoq_responder)
    discover_venture_feedback_v1.erl        # Feedback (evoq_feedback)
```

---

## Relationship to Existing Patterns

### Emitter vs evoq_event_handler

`evoq_event_handler` is a general-purpose event subscriber for side effects. Emitters are a **specialized subset**: their only job is translating domain events to integration facts and publishing them. The `evoq_emitter` behaviour makes this intent explicit and enforces the fact-translation contract.

### Listener vs evoq_process_manager

Listeners and process managers share the "own slice" placement pattern for the same reason: filesystem-level discoverability. The difference:

- **Process manager**: reacts to internal domain events, orchestrates multi-step workflows
- **Listener**: reacts to external facts (from another domain via pg/mesh), dispatches a local command

### Policy vs Listener

Both dispatch commands. The difference is the source:

| Actor | Source | Naming |
|-------|--------|--------|
| Policy | Internal domain event (own event store) | `on_{event}_maybe_{command}` |
| Listener | External fact (pg/mesh from another domain) | `on_{fact}_from_{transport}_{command}` |

---

## Migration Guide

### Renaming existing emitters

Old: `{event}_to_{transport}.erl`
New: `emit_{event}_to_{transport}.erl`

```bash
# Example migration
git mv venture_initiated_v1_to_pg.erl emit_venture_initiated_v1_to_pg.erl
# Update -module() declaration inside the file
# Update references in desk supervisor
```

### Adding evoq behaviours to existing emitters/listeners

Existing emitters and listeners implemented as plain gen_servers continue to work. Adding `-behaviour(evoq_emitter)` or `-behaviour(evoq_listener)` gives compile-time enforcement of the required callbacks. The behaviours are opt-in.

---

## Decision Record

| Date | Decision |
|------|----------|
| 2026-03-14 | Emitters renamed from `{event}_to_{transport}` to `emit_{event}_to_{transport}` |
| 2026-03-14 | Listeners get own slice directory (like process managers) for screaming architecture |
| 2026-03-14 | 4 actor behaviours: `evoq_emitter`, `evoq_listener`, `evoq_requester`, `evoq_responder` |
| 2026-03-14 | `evoq_feedback` behaviour for typed feedback serialization |
| 2026-03-14 | Session-level consistency via `dispatch_with_state/1` in evoq_dispatcher |
| 2026-03-14 | Fact modules live in the same desk as the event they translate (vertical slicing) |

---

*Every integration point screams at the filesystem level. If you can't see it in `ls src/`, it doesn't exist.*
