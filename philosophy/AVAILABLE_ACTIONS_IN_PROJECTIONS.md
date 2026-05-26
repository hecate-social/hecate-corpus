---
title: Frontends Are Pure Views
layer: philosophy
audience: [agent, human]
stage: stable
---

# Frontends Are Pure Views

*Projections provide available actions, not just status labels. Frontends render --- they never decide.*

---

## The Principle

**Available actions MUST be computed at projection time and stored alongside the status in the read model. The frontend MUST use the pre-computed action list to determine which buttons and controls to render --- never evaluate state machine logic client-side.**

This extends [STATUS_LABELS_IN_PROJECTIONS.md](STATUS_LABELS_IN_PROJECTIONS.md). That document established that status labels belong in projections. This document establishes the same for available actions. Together they make the frontend a pure renderer with zero domain knowledge.

---

## Why

1. **The aggregate owns the state machine** --- Only the aggregate (and its projection) knows what transitions are valid from a given state. This knowledge MUST NOT leak to the frontend.

2. **Single source of truth** --- If valid transitions are encoded in both the projection and the frontend, they will drift. One will allow actions the other doesn't. We've seen this with status labels already.

3. **Frontend deploys become unnecessary** --- When the state machine changes (new states, different transitions, additional guards), only the projection updates. The frontend doesn't need to know that "shelve" is now available from the "reviewing" state --- it just renders whatever actions the backend provides.

4. **Read models serve the view** --- CQRS says read models should be shaped for the view. If the UI needs to know which buttons to show, the read model should contain that information. The projection is the right place to compute it.

5. **Testable at the backend** --- Available actions can be unit tested against known status values. No browser tests needed to verify state machine correctness.

---

## The Pattern

### Projection (Erlang)

Every projection that writes a `status` field MUST also write `status_label` AND `available_actions`:

```erlang
%% In the projection module
handle_event(Event, State) ->
    NewStatus = compute_new_status(Event, State),
    Label = evoq_bit_flags:to_string(NewStatus, ?FLAG_MAP),
    Actions = available_actions(NewStatus),

    Updated = State#{
        status => NewStatus,
        status_label => Label,
        available_actions => Actions
    },
    {ok, Updated}.

available_actions(Status) ->
    case evoq_bit_flags:has(Status, ?PHASE_OPEN) of
        true -> [<<"shelve">>, <<"conclude">>];
        false ->
            case evoq_bit_flags:has(Status, ?PHASE_SHELVED) of
                true -> [<<"resume">>];
                false ->
                    case evoq_bit_flags:has(Status, ?PHASE_INITIATED) of
                        true -> [<<"open">>];
                        false -> []
                    end
            end
    end.
```

The `available_actions/1` function mirrors the aggregate's state machine. It answers: "given this status, what can the user do next?"

### Query (Erlang)

Queries return the actions alongside other fields. Already computed --- just pass through:

```erlang
row_to_map([DivisionId, Status, StatusLabel, AvailableActions | _]) ->
    #{
        division_id => DivisionId,
        status => Status,
        status_label => StatusLabel,
        available_actions => AvailableActions  %% Pre-computed list from projection
    }.
```

### Frontend (TypeScript/Svelte)

The frontend renders the actions. It does NOT evaluate state:

```typescript
// CORRECT: Render whatever the backend says is available
{#each division.available_actions as action}
    <button on:click={() => dispatch(action, division.id)}>
        {actionLabel(action)}
    </button>
{/each}
```

The `actionLabel()` function is a pure display concern (mapping `"shelve"` to `"Shelve Division"`) --- it contains zero domain logic.

---

## The Complete Picture

Status labels and available actions together make the frontend entirely passive:

```
Aggregate (state machine owner)
    |
    v
Event (status changed)
    |
    v
Projection (computes EVERYTHING the view needs)
    |-- status: integer (bit flags)
    |-- status_label: "Open"
    |-- available_actions: ["shelve", "conclude"]
    |
    v
Read Model (pre-computed, shaped for the view)
    |
    v
Query (passes through, no computation)
    |
    v
Frontend (pure renderer, zero domain logic)
    |-- Shows "Open" label
    |-- Shows [Shelve] [Conclude] buttons
    |-- No flag constants, no state machine knowledge
```

---

## Anti-Patterns

| Anti-Pattern | Why It's Wrong |
|-------------|----------------|
| Frontend bit flag constants | Copy of server truth that drifts |
| Client-side `hasFlag()` for button visibility | Duplicates state machine logic |
| Frontend state machine (`if status === OPEN, show shelve`) | Domain logic in the wrong layer |
| Fetching `/valid-transitions` endpoint on demand | Read model should already contain this |
| Hardcoded button visibility per "page" | Breaks when state machine changes |

### The Worst Anti-Pattern

```typescript
// WRONG: Frontend duplicates the entire state machine
import { PHASE_OPEN, PHASE_SHELVED, PHASE_INITIATED } from './constants';

function getAvailableActions(status: number): string[] {
    if (hasFlag(status, PHASE_OPEN)) return ['shelve', 'conclude'];
    if (hasFlag(status, PHASE_SHELVED)) return ['resume'];
    if (hasFlag(status, PHASE_INITIATED)) return ['open'];
    return [];
}
```

This is the state machine, duplicated in TypeScript. When the backend adds a new state or changes a transition, this code is silently wrong until someone notices buttons are missing or present when they shouldn't be.

### The Correct Pattern

```typescript
// RIGHT: Frontend is a pure view
function renderActions(actions: string[]) {
    return actions.map(action => (
        <button on:click={() => dispatch(action)}>{labels[action]}</button>
    ));
}
```

No imports. No constants. No flag knowledge. The backend said these actions are valid --- render them.

---

## Status Labels Are for Display Only --- Never for Logic

**`status_label` is a human-readable string for display. It is opaque to the frontend. The frontend MUST render it, never parse it.**

Labels can change at any time --- renamed, translated, reformatted, reworded for clarity. If the frontend branches on label content, a label rename breaks application behavior. This is unacceptable.

### The Rule

- **Display decisions** (which buttons to show, which actions are valid): use `available_actions`
- **Visual styling** (icon, color, badge shape): use `status_class` from the backend, or derive from `available_actions` presence/absence
- **Label rendering**: render the string as-is, treat it as opaque

### Anti-Pattern: Branching on Label Content

```typescript
// WRONG: Coupling display text to logic
if (label.includes('concluded')) showCheckmark();
if (label.toLowerCase().includes('open')) showPulsingDot();
if (label === 'Shelved') disableActions();
```

This breaks when:
- "Concluded" is renamed to "Completed"
- "Open" is translated to "Ouvert"
- The label format changes from "Shelved" to "On Hold"

Every label rename becomes a frontend bug hunt.

### Correct Pattern: Backend Provides Everything

The backend provides `status_class` for visual hints, or the frontend derives visuals from `available_actions`:

```typescript
// RIGHT: Backend provides status_class for visual styling
<span class={division.storming_status_class}>{division.storming_status_label}</span>

// RIGHT: Derive visuals from available_actions
const isActive = actions.length > 0;
const isDone = actions.length === 0 && label !== '';
const icon = isActive ? 'pulse' : isDone ? 'check' : 'pending';
```

No string matching. No substring checks. No case-insensitive comparisons. The label is rendered, never interrogated.

### Projection Responsibilities

The projection MUST provide sufficient structured data so the frontend never needs to parse labels:

| Frontend Need | Correct Source | Wrong Source |
|---------------|---------------|--------------|
| What to display | `status_label` (render as-is) | --- |
| Which buttons to show | `available_actions` | parsing `status_label` |
| Icon/color/badge | `status_class` or derived from `available_actions` | parsing `status_label` |
| Whether something is "done" | `available_actions` is empty | `label.includes('concluded')` |
| Whether something is "active" | `available_actions` is non-empty | `label.includes('open')` |

If the frontend finds itself parsing a label, the projection is missing a field. Fix the projection, not the frontend.

---

## Generality

This principle applies to ANY domain, not just phases:

| Domain | Status | Available Actions |
|--------|--------|-------------------|
| Division Phase | `PHASE_OPEN` | `["shelve", "conclude"]` |
| Plugin Lifecycle | `INSTALLED` | `["activate", "uninstall"]` |
| License | `ACTIVE` | `["suspend", "revoke"]` |
| User Account | `VERIFIED` | `["promote", "suspend"]` |
| Domain | `DISCOVERING` | `["pause", "advance_to_design"]` |

Every domain with a state machine benefits. The pattern is universal.

---

## When the State Machine Changes

Consider what happens when we add a new "reviewing" state between "open" and "concluded":

| Layer | Without This Principle | With This Principle |
|-------|----------------------|---------------------|
| Aggregate | Update state machine | Update state machine |
| Projection | Update status label | Update status label AND available_actions |
| Frontend | Update flag constants, add new `hasFlag()` checks, deploy | **Nothing. Zero changes.** |

The frontend already renders whatever actions arrive. A new action name appears in the list, and the frontend renders it. If a display label is needed, add one entry to the label map --- still no domain logic.

---

## Relationship to Other Principles

- **STATUS_LABELS_IN_PROJECTIONS** --- This document extends it. Labels are the "what is it?" question. Actions are the "what can I do?" question. Both answers belong in the projection.
- **Labels are opaque** --- The "Status Labels Are for Display Only" section above closes the loop. Labels answer "what is it?" for the user's eyes. Actions answer "what can I do?" for the frontend's logic. Visual styling comes from `status_class` or is derived from actions. The frontend never parses labels.
- **DDD / The Dossier Principle** --- The aggregate is the authority on valid transitions. Projections derive view data from that authority. Frontends consume, never interpret.
- **VERTICAL_SLICING** --- The available_actions function lives in the projection module, inside the slice. Not in a shared `utils/state_machine.ex` helper.

---

*Projections provide available actions, not just status labels. Frontends render --- they never decide.*
