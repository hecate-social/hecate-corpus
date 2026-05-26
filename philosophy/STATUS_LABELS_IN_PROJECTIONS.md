---
title: Status Labels Belong in Projections
layer: philosophy
audience: [agent, human]
stage: stable
---

# Status Labels Belong in Projections

*Status labels are projections. Projections belong in projections.*

---

## The Principle

**Status labels MUST be computed at projection time and stored alongside the integer status in the read model. The frontend MUST use the pre-computed label --- never decode bit flags client-side.**

---

## Why

1. **Single source of truth** --- The flag map lives in one place (the Erlang `.hrl` file). If labels change, only the projection logic updates. The frontend doesn't need to know about flag definitions.

2. **No duplication** --- Client-side bit flag constants are a copy of server-side truth. Copies drift. We've already found a bug where frontend domain lifecycle flags (`VL_DISCOVERING=1`) didn't match backend values (`VL_DISCOVERING=8`).

3. **Read models serve the view** --- CQRS says read models should be shaped for the view that consumes them. If the UI needs a label, the read model should contain the label. The projection is the right place to compute it.

4. **Projections own the interpretation** --- `evoq_bit_flags:to_string(Status, FLAG_MAP)` is the canonical way to produce a human-readable label. This belongs in the projection, next to the status update, not scattered across frontend components.

---

## The Pattern

### Projection (Erlang)

Every projection that writes a `status` field MUST also write a `status_label` field:

```erlang
%% In the projection module
NewStatus = evoq_bit_flags:set(CurrentStatus, ?PLANNING_OPEN),
StatusLabel = evoq_bit_flags:to_string(NewStatus, ?PLANNING_FLAG_MAP),

project_division_plannings_store:execute(Db,
    "UPDATE division_plannings SET status = ?1, status_label = ?2 WHERE division_id = ?3",
    [NewStatus, StatusLabel, DivisionId]).
```

### Query (Erlang)

Queries return both fields. The label is already computed --- just pass it through:

```erlang
row_to_map([DivisionId, Status, StatusLabel | _]) ->
    #{
        division_id => DivisionId,
        status => Status,
        status_label => StatusLabel  %% Pre-computed, from projection
    }.
```

### Frontend (TypeScript)

The frontend displays the label. It does NOT decode the integer:

```typescript
// CORRECT: Use the label from the backend
<span>{division.status_label}</span>

// WRONG: Decode bit flags client-side
<span>{hasFlag(division.status, PHASE_OPEN) ? 'Open' : 'Pending'}</span>
```

The integer `status` field is still available for programmatic checks (e.g., "is this archived?"), but the human-readable label always comes from the backend.

---

## Composite Views

When a view needs status from multiple domains (e.g., a division list showing storming + planning + kanban + crafting statuses), the read model projection should be enriched to include all needed statuses and labels.

**Do NOT** have the frontend fetch multiple endpoints and stitch together. Shape the read model for the view.

```
Incorrect:
  Frontend -> GET /divisions         (no status)
  Frontend -> GET /stormings/:id     (storming status)
  Frontend -> GET /plannings/:id     (planning status)
  Frontend -> GET /kanbans/:id       (kanban status)
  Frontend -> GET /craftings/:id     (crafting status)
  ... stitch together client-side

Correct:
  Projection listens to phase lifecycle events -> enriches division read model
  Frontend -> GET /divisions          (includes all phase statuses + labels)
```

---

## Anti-Patterns

| Anti-Pattern | Why It's Wrong |
|-------------|----------------|
| Frontend bit flag constants | Copy of server truth that drifts |
| Client-side `hasFlag()` for labels | Duplicates projection logic |
| `phaseLabel()` in TypeScript | Should come from backend |
| Frontend fetching `/flag-maps` to decode | Read model should already contain the label |
| Query handler computing labels | Too late --- projection should have done it |

---

## Exceptions

The frontend MAY use the integer `status` for **boolean guards** (e.g., "show archive button only if not already archived") when the specific flag value is well-known and stable. But labels for display always come from the backend.

---

*Status labels are projections. Projections belong in projections.*
