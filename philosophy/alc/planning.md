---
title: "ALC: Planning -- Design the Desk Capabilities"
layer: philosophy
audience: [agent, human]
stage: stable
---

# ALC: Planning -- Design the Desk Capabilities

*Process 2 of [HECATE_ALC](README.md)*

[Back to ALC Index](README.md)

---

## Purpose

Design gave us the domain model -- aggregates, events, stream patterns. Planning transforms that model into a **buildable plan**: which desks exist, what each desk does, and in what order they are built.

Planning answers:
- What desks does this division need?
- What does each desk listen to, decide, and emit?
- What is the implementation sequence?
- What is the Walking Skeleton?

**Planning is where vertical slicing becomes concrete.** Each desk is inventoried with its inboxes, policies, and emitters -- the full capability model.

---

## Lifecycle

Planning is a **long-lived** process. It uses the standard ALC lifecycle protocol with screaming verbs:

```
Command:  open_planning_v1       -->  Event: planning_opened_v1
Command:  shelve_planning_v1     -->  Event: planning_shelved_v1
Command:  resume_planning_v1     -->  Event: planning_resumed_v1
Command:  conclude_planning_v1   -->  Event: planning_concluded_v1
```

**State transitions:**

```
             open
 pending -----------> active
                       | ^
                 shelve | | resume
                       v |
                     paused
                       |
              conclude |
                       v
                   completed
```

**Rules:**
- Domain-specific commands (`inventory_desk_v1`, `sequence_desks_v1`) only work when state is `active`.
- `shelve_planning` records a reason (waiting for design clarification, blocked on dependency).
- `resume_planning` clears the shelve reason and reactivates.
- `conclude_planning` signals handoff -- facts flow to `craft_division`, `deploy_division`, and `guide_venture`.

---

## Aggregate and Stream

| Attribute | Value |
|-----------|-------|
| **CMD App** | `plan_division` |
| **QRY+PRJ App** | `query_plans` |
| **Aggregate** | `plan` |
| **Stream Pattern** | `plan-{division_id}` |
| **Cardinality** | N per domain (one per division) |

---

## Activities

### 1. Desk Inventory

List every desk the division needs. Each desk is a single capability within a department:

```
Command:  inventory_desk_v1
          { plan_id, division_id, desk_name, desk_type, priority, description }

Event:    desk_inventoried_v1
          { plan_id, division_id, desk_id, desk_name, desk_type,
            priority, description, inventoried_at }
```

**Desk types map to departments:**

| Department | Desk Type | Naming Pattern | Example |
|------------|-----------|----------------|---------|
| CMD | command desk | `{verb}_{subject}/` | `register_user/` |
| PRJ | projection desk | `{event}_to_{table}/` | `user_registered_v1_to_users/` |
| QRY | query desk | `get_{subject}_by_{key}/` | `get_user_by_id/` |

**Priority levels:**

| Priority | Meaning |
|----------|---------|
| P0 | Walking Skeleton -- minimum viable end-to-end |
| P1 | Core feature -- primary business value |
| P2 | Supporting -- enhances core |
| P3 | Nice-to-have -- defer if needed |

### 2. Capability Planning (The Desk Capability Model)

Each desk is more than a name. It has a full **capability definition**: what it listens to, what decisions it makes, and what it announces.

#### Inboxes

Where work arrives at the desk. Two sources:

| Source | Mechanism | Example |
|--------|-----------|---------|
| **pg** (internal) | OTP process groups, same BEAM VM | `pg:join(announce_capability, self())` |
| **mesh** (external) | Macula mesh topics, cross-daemon | `subscribe("marketplace.app_available")` |

A desk may have zero, one, or both inbox types. Command desks typically receive from pg (dispatched commands). Process manager desks subscribe to pg or mesh events from other domains.

#### Policies

The decision logic within the desk. A policy reacts to an inbox message and decides what to do:

- **Validate** -- Is this command allowed in the current aggregate state?
- **Transform** -- Map external data to internal event shape
- **Decide** -- Apply business rules, emit events or reject

Policies are the `maybe_{verb}_{subject}` handlers. They embody the business rules.

#### Emitters

What the desk announces after processing:

| Target | Mechanism | Example |
|--------|-----------|---------|
| **pg** (internal) | Broadcast to local BEAM processes | `{event}_to_pg.erl` |
| **mesh** (external) | Publish integration facts to mesh | `{event}_to_mesh.erl` |

Not every desk emits externally. Internal events stay in pg. Only explicitly chosen facts go to the mesh -- this is the domain events vs integration facts boundary.

#### Example: Full Desk Capability

```
Desk: register_user/
  Inboxes:
    - pg: register_user_commands (dispatched from API)
  Policies:
    - maybe_register_user.erl (validate email unique, check constraints)
  Emitters:
    - user_registered_v1_to_pg.erl (internal broadcast)
    - user_registered_v1_to_mesh.erl (integration fact for other divisions)
```

### 3. Sequencing

Order the desks into implementation phases:

```
Command:  sequence_desks_v1
          { plan_id, division_id, phase_number, desk_ids }

Event:    desks_sequenced_v1
          { plan_id, division_id, phase_number, desk_ids, sequenced_at }
```

**Standard sequencing pattern:**

| Phase | Content | Purpose |
|-------|---------|---------|
| Phase 0 | Walking Skeleton | Scaffold, CI/CD, GitOps, `initiate_*` + `archive_*` end-to-end |
| Phase 1 | Core desks (P1) | Primary business capabilities |
| Phase N | Supporting desks | Additional features, integrations |

The Walking Skeleton always includes both `initiate_{aggregate}` and `archive_{aggregate}` desks -- no aggregate exists without both its birth and its soft-delete.

### 4. Interface Design

Define the contracts that desks expose:

**Commands (Input):**
```erlang
#{
    command_type := register_user_v1,
    user_id := binary(),
    email := binary(),
    display_name := binary()
}
```

**Events (Internal):**
```erlang
#{
    event_type := user_registered_v1,
    user_id := binary(),
    email := binary(),
    registered_at := integer()
}
```

**Integration Facts (Mesh Output):**
```erlang
#{
    mri := binary(),
    data := map(),
    published_at := integer()
}
```

**Query APIs:**
```
GET /api/{resource}/:id
GET /api/{resource}?filter=value
```

---

## The Desk Capability Model (Summary)

Every desk in the plan must answer three questions:

```
+------------------+
|      DESK        |
|                  |
|  Inboxes:        |  <-- Where does work arrive? (pg, mesh, or both)
|  Policies:       |  <-- What decisions are made? (validate, transform, decide)
|  Emitters:       |  <-- What is announced? (pg, mesh, or both)
|                  |
+------------------+
```

This model ensures no desk is a black box. The plan documents exactly what flows in, what logic applies, and what flows out.

---

## Outputs

### Required

- [ ] **Desk Inventory** -- All desks with names, types, priorities, and capabilities
- [ ] **Capability Definitions** -- Inboxes, policies, emitters for each desk
- [ ] **Implementation Sequence** -- Phased plan with Walking Skeleton as Phase 0
- [ ] **PLAN_*.md** -- The plan document (see template below)

### Recommended

- [ ] **Interface Specifications** -- Command, event, and fact schemas
- [ ] **Technology Decisions** -- Choices with rationale
- [ ] **Dependency Map** -- Which desks depend on which

---

## PLAN_*.md Template

```markdown
# PLAN: {Division Name}

**Status:** Planning | In Progress | Complete
**Created:** {date}

## Overview

{Brief description of what this division does}

## Domain Model (from Design)

### Dossiers

| Dossier | Stream Pattern | Description |
|---------|---------------|-------------|
| ... | ... | ... |

## Desk Inventory

### CMD Desks

| Desk | Events | Priority | Inboxes | Emitters |
|------|--------|----------|---------|----------|
| ... | ... | ... | ... | ... |

### PRJ Desks

| Projection | Source Event | Target Table |
|------------|-------------|--------------|
| ... | ... | ... |

### QRY Desks

| Query | Parameters | Returns |
|-------|------------|---------|
| ... | ... | ... |

## Implementation Phases

### Phase 0: Walking Skeleton
- [ ] Scaffold division
- [ ] CI/CD pipeline
- [ ] GitOps manifests
- [ ] `initiate_*` + `archive_*` desks end-to-end

### Phase 1: Core
- [ ] ...

## Open Questions
- ...

## Decisions Log

| Date | Decision | Rationale |
|------|----------|-----------|
| ... | ... | ... |
```

---

## Fact Flow

Planning is a **living document**. It produces facts and receives feedback:

**Outbound facts:**
```
plan_division  --fact-->  craft_division     (desk inventory as generation input)
plan_division  --fact-->  deploy_division    (release scope)
plan_division  --fact-->  guide_venture      (progress tracking)
```

**Inbound facts (feedback loop):**
```
craft_division    --fact-->  plan_division   (desk_generated_v1)
debug_division    --fact-->  plan_division   (testing_completed_v1)
deploy_division   --fact-->  plan_division   (release_deployed_v1)
monitor_division  --fact-->  plan_division   (incident_raised_v1)
rescue_division   --fact-->  plan_division   (rescue_completed_v1)
```

**Acknowledgement events** (plan records that it received feedback):
```
generation_acknowledged_v1
testing_acknowledged_v1
deployment_acknowledged_v1
incident_acknowledged_v1
rescue_acknowledged_v1
```

The plan knows what has been designed, crafted, tested, deployed, and rescued. It is the division's memory of its own progress.

---

## Entry Checklist (Before Opening Planning)

- [ ] Design concluded (`design_concluded_v1` recorded for this division)
- [ ] Aggregates defined with stream patterns
- [ ] Events cataloged with payload fields
- [ ] Status flags defined (bit flags)
- [ ] Domain glossary available
- [ ] Constraints documented

## Exit Checklist (Before Concluding Planning)

- [ ] All desks inventoried with names, types, priorities
- [ ] Capability definitions complete (inboxes, policies, emitters) for each desk
- [ ] Walking Skeleton identified (Phase 0 desks)
- [ ] Implementation sequence defined (phases with desk assignments)
- [ ] PLAN_*.md written and reviewed
- [ ] Interface specifications drafted (commands, events, facts)
- [ ] Ready to hand off to crafting

---

## Anti-Patterns

| Anti-Pattern | Problem | Instead |
|--------------|---------|---------|
| **Big design up front** | Planning every desk in perfect detail | Plan Phase 0 + Phase 1 in detail, sketch the rest |
| **Horizontal thinking** | Planning `services/`, `handlers/`, `utils/` layers | Plan vertical desks -- each desk owns its full stack |
| **Skipping the skeleton** | Jumping to core features without end-to-end scaffolding | Walking Skeleton is always Phase 0 |
| **Implicit decisions** | "We just knew" why we chose this | Document decisions in the PLAN_*.md decisions log |
| **Ignoring constraints** | Beautiful plan that cannot be built | Plan within the constraints identified during design |
| **Desks without capabilities** | Inventory says "register_user" but no inbox/policy/emitter detail | Every desk must have its full capability model defined |
| **Missing archive desk** | Aggregate has `initiate_*` but no `archive_*` | Both birth and soft-delete desks are mandatory from Phase 0 |
| **CRUD desk names** | `create_user/`, `update_order/`, `delete_item/` | Business verbs: `register_user/`, `confirm_order/`, `archive_item/` |

---

## The Guided Conversation

Planning is driven by a guided conversation that produces the desk inventory and implementation sequence.

1. **Review design outputs** -- Aggregates, events, stream patterns from design
2. **Inventory desks** -- For each aggregate, what desks are needed? CMD, PRJ, QRY.
3. **Define capabilities** -- For each desk, what are its inboxes, policies, emitters?
4. **Prioritize** -- P0 (skeleton), P1 (core), P2 (supporting), P3 (nice-to-have)
5. **Sequence** -- Group into phases, Walking Skeleton first
6. **Document** -- Write the PLAN_*.md

**Planning conversation produces:** desk inventory with capabilities, implementation sequence, PLAN_*.md.

These outputs become the input to [alc/crafting.md](crafting.md).

---

*The domain model tells you what exists. The plan tells you what to build, and when. Every desk has a name, a purpose, and a voice.*
