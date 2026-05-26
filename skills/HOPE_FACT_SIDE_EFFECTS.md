---
title: "Side Effects Must Follow Facts, Not Hopes"
layer: skill
audience: [agent, human]
stage: stable
---

# Side Effects Must Follow Facts, Not Hopes

*An architectural pattern for client-daemon communication.*

---

## The Principle

> **Any client (hecate-web, plugin frontend, CLI) is an external system. Side effects (UI updates, state transitions, user feedback) must be driven by received FACTS (events materialized into read models), never by command acknowledgments.**

A command sent to the daemon is a **HOPE** -- "I hope you'll do this." The HTTP response means "I received your hope and it looks valid." It does NOT mean the intent succeeded. Only the resulting **event** (fact), materialized into a read model by a projection, confirms that.

This principle applies universally:
- hecate-web (Tauri desktop app) sending commands via hecate:// protocol
- Plugin frontends (`*w` apps) sending commands via hecate:// protocol
- `hecate` CLI sending commands via Unix socket
- Any future client

---

## The Anti-Pattern

```
Client                           Daemon
──────                           ──────
POST /vision/refine  ──────→    receive hope
                                process command
                                store event (maybe)
← 200 OK  ←───────────────     "received"
update UI  ← WRONG!            side effect based on hope
```

What can go wrong between "received" and "event stored":
- Aggregate rejects the command (business rule violation)
- Event store write fails (disk full, replication error)
- Process crashes between accept and store
- Command is queued for async processing

The client updated its UI based on a **hope acknowledgment**, not a **fact**.

---

## The Correct Pattern: Read Model as Source of Truth

The daemon owns the truth. Events flow through projections into SQLite read models. Clients read from those read models to determine current state.

### hecate-web and Plugin Frontends

```
Client (hecate-web / plugin)     Daemon
────────────────────────         ──────
POST /vision/refine  ──────→    receive hope
← 202 Accepted  ←─────────     "hope received"
                                process command
                                vision_refined_v1 stored
                                projection updates SQLite

GET /query/visions/{id}  ──→    read from SQLite
← 200 OK + updated state ←     FACT confirmed
update UI                       side effect based on FACT
```

The client:
1. Sends command via hecate:// protocol (HOPE)
2. Receives 202 Accepted -- meaning "received, processing"
3. Polls the read model for updated state (or subscribes via WebSocket in the future)
4. The projection has materialized the event into SQLite
5. Client fetches the updated read model and updates UI based on FACT

### CLI

```
CLI                              Daemon
───                              ──────
command via Unix socket  ──→    receive hope
                                process command
                                event stored
                                projection updates SQLite
                                query read model
← result  ←───────────────     FACT confirmed
display output                  side effect based on FACT
```

The CLI can afford to wait synchronously. The daemon processes the command, waits for the projection to complete, queries the read model, and returns the result. The CLI never assumes success before the fact.

---

## Listener Architecture

### Daemon Side: Events Flow Through Established Channels

Events produced by command processing flow through the existing emitter infrastructure:

```
{event}_to_pg.erl       -- internal (same BEAM VM, intra-daemon)
{event}_to_mesh.erl     -- external (WAN, cross-daemon)
```

Projections subscribe to events via evoq and update SQLite read models. These read models are the single source of truth for all client queries.

```
Command processed
    |
    v
Event stored in ReckonDB
    |
    +---> {event}_to_pg.erl       (broadcast to internal listeners)
    +---> {event}_to_mesh.erl     (publish to mesh for other daemons)
    +---> Projection              (update SQLite read model)
```

### Client Side: Read Models Are the Interface

Clients do not subscribe to raw events. Clients read from projections (SQLite read models) via query endpoints.

| Client | Access Method | Pattern |
|--------|--------------|---------|
| hecate-web | hecate:// protocol (HTTP over Unix socket) | Poll query endpoints |
| Plugin frontends | hecate:// protocol (HTTP over Unix socket) | Poll query endpoints |
| CLI | Direct Unix socket | Synchronous request/response |
| Future | WebSocket | Push-based subscription |

The query endpoints are served by QRY department apps (e.g., `query_visions`, `query_ventures`). These read directly from the SQLite read models that projections maintain.

---

## Response Codes

Commands return **202 Accepted**, not 200 OK:

| Code | Meaning | Use |
|------|---------|-----|
| **200 OK** | "Here is your data" | Queries (GET) |
| **202 Accepted** | "Hope received, processing" | Commands (POST) |
| **400 Bad Request** | "Hope malformed" | Validation errors |
| **409 Conflict** | "Hope contradicts current state" | Business rule violations |

The distinction matters: 200 implies completion, 202 implies the work is still happening.

---

## Transport

There are three integration boundaries. Clients live outside all of them.

| Boundary | Transport | Scope | Purpose |
|----------|-----------|-------|---------|
| **Internal** | `pg` (OTP process groups) | Same BEAM VM | Intra-daemon event broadcast |
| **External** | `mesh` (Macula QUIC) | WAN | Cross-daemon fact publication |
| **Client** | HTTP over Unix socket | Same machine, cross-process | hecate:// protocol |

Clients access the daemon exclusively through HTTP over Unix socket. The hecate:// protocol proxies these requests. Clients never participate in pg groups or mesh topics -- they read the results of those integrations via query endpoints backed by SQLite read models.

---

## Implementation Order

1. **Daemon:** Return 202 for all command endpoints
2. **Daemon:** Ensure projections update SQLite read models from events
3. **Daemon:** Ensure QRY apps expose query endpoints for read models
4. **Client:** Send commands, receive 202, then poll query endpoints for updated state
5. **Future:** Add WebSocket support for push-based read model updates

---

## Relationship to Other Patterns

- **HOPE/FACT vocabulary** -- from the Macula mesh protocol, applied universally to all client-daemon communication
- **Projections** -- client reads ARE reads from projections; SQLite read models are the interface between daemon truth and client display
- **Event sourcing** -- commands produce events, events update read models via projections, clients read from read models
- **CQRS** -- commands and queries are strictly separated; 202 for writes, 200 for reads
- **Vertical slicing** -- each domain owns its projections and query endpoints; there is no central "client bridge" or "notification service"

---

*Recorded 2026-02-09. Origin: TUI vision command architecture review.*
*Updated 2026-02-09: Refined from generic event stream to per-fact listener pattern.*
*Rewritten 2026-02-18: Generalized from TUI-specific to universal client-daemon pattern. TUI (Go + Bubble Tea) is dead; principle now covers hecate-web, plugin frontends, CLI, and any future client.*
