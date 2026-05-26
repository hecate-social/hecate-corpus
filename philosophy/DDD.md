---
title: The Dossier Principle
layer: philosophy
audience: [agent, human]
stage: stable
---

# DDD.md — The Dossier Principle

*Process-centric domain modeling for Division architecture.*

---

## The Two Mental Models

Most Event-Sourced systems approach aggregates from a **data-centric** perspective. Division Architecture approaches aggregates from a **process-centric** perspective.

### Data-Centric (Traditional)

```
Aggregate = object with current state
Events = mutations applied to the object
Question: "What IS this thing right now?"
```

```erlang
%% Traditional: Aggregate is a data structure
-record(capability_aggregate, {
    mri,
    status,        % current state
    announced_at,
    updated_at
}).

%% Events "apply" to mutate state
apply_event(Agg, #capability_announced_v1{} = E) ->
    Agg#capability_aggregate{
        mri = E#capability_announced_v1.mri,
        status = active,
        announced_at = E#capability_announced_v1.timestamp
    }.
```

The aggregate is a **thing**. Events happen **to** it.

### Process-Centric (Dossier)

```
Aggregate = the dossier itself (ordered event slips)
Desks = stations the dossier passes through
Question: "What has HAPPENED to this dossier?"
```

```
Dossier: capability-mri:capability:io.macula/weather
├── [slip] capability_announced_v1    ← added at announce_capability desk
├── [slip] capability_updated_v1      ← added at update_capability desk
├── [slip] capability_endorsed_v1     ← added at endorse_capability desk
└── [slip] capability_revoked_v1      ← added at revoke_capability desk
```

The dossier **is** its history. There is no separate "state object."

---

## The Dossier Metaphor

Imagine a physical folder (dossier) moving through an office:

1. **Dossier arrives** at a desk
2. **Clerk reviews** the slips inside (events so far)
3. **Clerk may add** a new slip (new event)
4. **Dossier moves on** to the next desk

Each desk has a specific responsibility:
- The `announce_capability` desk handles new announcements
- The `update_capability` desk handles modifications
- The `revoke_capability` desk handles revocations

The dossier accumulates slips as it moves through the process.

---

## Consequences of the Dossier Model

### 1. Stream ID = Dossier Identity

The event stream ID is the dossier's unique identifier:

```
capability-mri:capability:io.macula/weather
reputation-did:macula:agent123
subscription-{subscriber}-{publisher}
```

All slips (events) for a dossier share this stream ID.

### 2. Desks Are Verbs, Not Nouns

Desks represent **process steps**, not entities:

| ❌ Data-Centric (Noun) | ✅ Process-Centric (Verb) |
|------------------------|---------------------------|
| `capability/` | `announce_capability/` |
| `reputation/` | `track_rpc_call/` |
| `subscription/` | `subscribe_to_agent/` |

Each desk is a station where a specific action can occur.

### 3. "Rebuilding State" = Reading the Dossier

To understand current state, read the dossier front-to-back:

```erlang
%% Not "applying mutations" — just reading slips
get_current_status(DossierId) ->
    Slips = reckon_evoq:read_stream(DossierId),
    lists:foldl(fun interpret_slip/2, #{}, Slips).

interpret_slip(#capability_announced_v1{}, Acc) ->
    Acc#{status => active};
interpret_slip(#capability_revoked_v1{}, Acc) ->
    Acc#{status => revoked}.
```

### 4. Business Rules = "Can This Slip Be Added?"

Validation is about whether a new slip can be added to this dossier:

```erlang
%% Handler checks: given the slips so far, can we add this one?
maybe_announce_capability(Cmd, Dossier) ->
    case dossier_status(Dossier) of
        empty -> 
            %% New dossier, can add announcement slip
            {ok, capability_announced_v1:from_cmd(Cmd)};
        active -> 
            %% Already has announcement slip
            {error, already_announced};
        revoked -> 
            %% Has revocation slip
            {error, capability_revoked}
    end.
```

### 5. Projections = "Index Cards" for the Filing Cabinet

Projections create index cards (read models) so you can find dossiers without opening every one:

```
Filing Cabinet (Event Store)
├── Dossier: capability-mri:...weather
├── Dossier: capability-mri:...forecast
└── Dossier: capability-mri:...alerts

Index Cards (Projections/Read Models)
├── By Agent: [weather, forecast] → agent123
├── By Tag: [weather, alerts] → "weather"
└── By Status: [weather, forecast, alerts] → "active"
```

---

## Domain Design with Dossiers

When designing a domain, ask:

### 1. What dossiers exist?

What are the "things" that accumulate history?

- Capabilities (announced, updated, revoked)
- Reputations (calls tracked, disputes flagged)
- Subscriptions (created, cancelled)
- Identities (created, paired, updated)

### 2. What desks process each dossier?

What actions can happen to each dossier type?

```
Capability Dossier passes through:
├── announce_capability (creates dossier, adds first slip)
├── update_capability (adds update slip)
├── endorse_capability (adds endorsement slip)
└── revoke_capability (adds revocation slip)
```

### 3. What slips (events) can be added?

Each desk adds specific slip types:

| Desk | Slip (Event) |
|--------------|--------------|
| `announce_capability` | `capability_announced_v1` |
| `update_capability` | `capability_updated_v1` |
| `endorse_capability` | `capability_endorsed_v1` |
| `revoke_capability` | `capability_revoked_v1` |

### 4. What index cards (projections) do we need?

How will we find dossiers without opening each one?

- Find capabilities by agent
- Find capabilities by tag
- Find active capabilities
- List top-rated agents (reputation index)

---

## Example: Designing the Reputation Domain

### Dossier: Agent Reputation

Stream ID: `reputation-{agent_id}`

### Desks:

```
reputation dossier passes through:
├── track_rpc_call      → adds rpc_call_tracked_v1 slip
├── flag_dispute        → adds dispute_flagged_v1 slip
├── resolve_dispute     → adds dispute_resolved_v1 slip
└── award_badge         → adds badge_awarded_v1 slip
```

### Slips (Events):

```erlang
rpc_call_tracked_v1     % caller, callee, procedure, success, latency
dispute_flagged_v1      % reporter, reason, evidence
dispute_resolved_v1     % resolver, resolution, outcome
badge_awarded_v1        % badge_type, criteria_met
```

### Index Cards (Projections):

```sql
-- Reputation scores (aggregated from slips)
CREATE TABLE reputation (
    agent_id TEXT PRIMARY KEY,
    score REAL,
    total_calls INTEGER,
    success_rate REAL
);

-- Dispute history
CREATE TABLE disputes (
    id TEXT PRIMARY KEY,
    agent_id TEXT,
    status TEXT,
    ...
);

-- Badges earned
CREATE TABLE badges (
    agent_id TEXT,
    badge_type TEXT,
    awarded_at INTEGER
);
```

---

## The Dossier Checklist

When creating a new domain:

- [ ] **Identify the dossier** — What accumulates history?
- [ ] **Define the stream ID pattern** — How is each dossier uniquely identified?
- [ ] **List the desks** — What actions/process steps exist?
- [ ] **Define the slips (events)** — What gets added at each desk?
- [ ] **Design the index cards (projections)** — How will we query?
- [ ] **Map the process flow** — Which desks can the dossier visit, in what order?

---

## Why This Matters

The dossier model:

1. **Aligns with business reality** — Processes ARE dossiers moving through departments
2. **Makes desks obvious** — Each desk is a process step
3. **Clarifies validation** — "Can this slip be added to this dossier?"
4. **Simplifies projections** — "Index cards for the filing cabinet"
5. **Enables code generation** — The pattern is mechanical, not creative

---

---

## The Default Read Model and Data Flow

*The most important principle in event-sourced architecture.*

### The Default Read Model

Every process instance has a **default read model**: the aggregate state itself.

```
Default Read Model = Aggregate State = The Dossier's Current Content
```

When you replay all events for a single stream, you get the complete truth about that process instance. This IS the default read model. It is:

- The sum of everything that has happened to this dossier
- The authoritative source of truth within the bounded context
- What the `#license_state{}` or `#plugin_state{}` record represents

The PRJ department creates *derived* read models (index cards) for queries. But the *default* read model — the dossier itself — lives in the CMD department as the aggregate state.

### Event Payloads Are Subsets of the Default Read Model

When a desk adds a slip (event) to the dossier, the slip's payload is a **subset of the default read model**, possibly enriched with data from the command:

```
Event Payload = subset(Aggregate State) + new data from Command
```

For example, when `grant_license` runs:
- The aggregate state has `plugin_id`, `oci_image`, `plugin_name` (from initiation)
- The command brings `grant_reason` (new data)
- The event `license_granted_v1` carries both: fields echoed from state + new data from command

This means downstream consumers (PMs, projections) receive everything they need IN the event — because the event was built from the authoritative source (the dossier).

### Data Enters Through Commands Only

**Any data from outside the bounded context MUST enter the CMD service through an enriched command payload.**

```
                    BOUNDARY
                       │
  ┌─────────────┐      │      ┌──────────────────┐
  │ API Handler │      │      │   CMD Service     │
  │             │      │      │                   │
  │ Reads from  │──────┼─────>│ Command arrives   │
  │ read models │  CMD │      │ with ALL needed   │
  │ to build    │      │      │ data. Aggregate   │
  │ fat command │      │      │ processes it.     │
  │             │      │      │ Events echo from  │
  └─────────────┘      │      │ aggregate state.  │
                       │      │ PMs read events.  │
                       │      │ NO external       │
                       │      │ lookups.           │
                       │      └──────────────────┘
                       │
```

**Where read model access IS allowed — at the boundary:**

| Location | Read Model Access | Why |
|----------|-------------------|-----|
| API handler | YES | Building the command payload |
| Convenience endpoint | YES | Orchestrating user intent into commands |
| CLI handler | YES | Same as API handler |

**Where read model access is FORBIDDEN — inside the flow:**

| Location | Read Model Access | Why |
|----------|-------------------|-----|
| Aggregate execute | NEVER | Uses its own state (the dossier) |
| Event handler / PM | NEVER | Uses the event payload (a subset of the dossier) |
| Projection | NEVER* | Writes to its own read model, never reads from others |
| Emitter | NEVER | Publishes the event as-is |

*(\* A projection may read its own state to merge updates — e.g., `evoq_read_model:get(Id, RM)` — but never another projection's state.)*

### The Chain of Responsibility

If a PM needs data it doesn't have, the fix is NEVER to reach outside. Walk the chain backward:

```
PM needs field X
  └── Event doesn't carry X
       └── Handler didn't echo X from aggregate state
            └── Aggregate state doesn't have X
                 └── Command didn't bring X into the bounded context
                      └── API handler didn't enrich the command with X
```

**Fix it at the source.** Usually this means:
1. The command needs a new field (carry X from the boundary)
2. The handler needs to echo X from aggregate state into the event
3. The PM reads X from the event

### Why This Matters

This principle makes event-sourced systems **deterministic** and **replayable**:

- Events are self-contained facts — they carry everything needed to understand what happened
- PMs produce the same commands regardless of the current state of read models
- Replaying events produces the same results every time
- No race conditions between projections and PMs
- No coupling between CMD and PRJ departments

**If a PM reads from a read model, the system becomes non-deterministic.** The PM's behavior depends on whether the projection has caught up — a timing dependency that makes the system fragile, untestable, and unreplayable.

### The Practical Test

Before writing any event handler or PM, ask:

> *"Can I delete all read models, replay all events, and get the same result?"*

If the answer is no — because a PM reads from a read model that might be empty during replay — you have a Demon #41 violation. The event payload is too poor. Enrich it.

See: `skills/antipatterns/event_sourcing.md` — Demon #41

---

---

## Minimal Ceremony

*Don't add steps that exist only to transition between states.*

When designing business processes, every desk (command) should do **real work**. If a command exists only to "unlock" or "reopen" something so the real command can run, it's ceremony — remove it.

### The Test

> *"Does this command produce a business-meaningful event, or does it just change a flag so another command can run?"*

If the answer is the latter, merge the transition into the command that does the real work.

### Example: Vision Refinement

```
BAD (ceremony):
  submit_vision → SUBMITTED (locked)
  reopen_vision → SUBMITTED cleared (ceremony — does nothing but unlock)
  refine_vision → actually changes the vision

GOOD (minimal):
  submit_vision → SUBMITTED (locked)
  refine_vision → changes vision AND clears SUBMITTED
```

`refine_vision` IS the reopening. No separate unlock step needed.

### When Ceremony Is Justified

A dedicated transition command is NOT ceremony when:
- It carries its own business data (e.g., `shelve_planning` records a `shelve_reason`)
- It triggers side effects (e.g., a PM reacts to the transition event)
- It represents a distinct business decision that should be auditable in the event stream

### In Cyclic Processes

Cyclic processes (refine/submit, open/shelve/resume) should flow naturally:
- **Non-exit states** must never permanently block re-entry
- The command that does the work should handle the state transition itself
- Exit states (archived, concluded) are the only permanent locks

---

*The dossier is the aggregate. The slips are the events. The desks are the capabilities.*

*Pass the dossier. Add the slip. Move on.*
