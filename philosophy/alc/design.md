---
title: "ALC: Design -- Discover and Model the Domain"
layer: philosophy
audience: [agent, human]
stage: stable
---

# ALC: Design -- Discover and Model the Domain

*Process 1 of [HECATE_ALC](README.md)*

[Back to ALC Index](README.md)

---

## Purpose

Before writing any code, understand the problem space fully. Design is about **questions**, not answers.

- What problem are we solving? For whom?
- What domain concepts exist? What accumulates history over time?
- What constraints limit the solution space?
- What could go wrong?

Design transforms a vague idea into a concrete domain model: aggregates, events, stream patterns, status flags. It is the foundation on which all other ALC processes build.

---

## Lifecycle

Design is a **long-lived** process. It uses the standard ALC lifecycle protocol with screaming verbs:

```
Command:  open_design_v1       -->  Event: design_opened_v1
Command:  shelve_design_v1     -->  Event: design_shelved_v1
Command:  resume_design_v1     -->  Event: design_resumed_v1
Command:  conclude_design_v1   -->  Event: design_concluded_v1
```

**State transitions:**

```
            open
 pending ----------> active
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
- Domain-specific commands (`design_aggregate_v1`, `design_event_v1`) only work when state is `active`.
- `shelve_design` records a reason (blocked, waiting for domain expert, external dependency).
- `resume_design` clears the shelve reason and reactivates.
- `conclude_design` signals handoff -- facts flow to `plan_division` and `guide_venture`.

---

## Aggregate and Stream

| Attribute | Value |
|-----------|-------|
| **CMD App** | `design_division` |
| **QRY+PRJ App** | `query_designs` |
| **Aggregate** | `design` |
| **Stream Pattern** | `design-{division_id}` |
| **Cardinality** | N per domain (one per division) |

---

## Activities

### 1. Requirements Gathering

What does the user or system need?

- **Functional requirements** -- What should it DO?
- **Non-functional requirements** -- How should it PERFORM?
- **User stories** -- Who needs what, and why?
- **Acceptance criteria** -- How do we know it is done?

Questions to ask:
- What problem is the user trying to solve?
- What happens if we do not build this?
- What is the simplest thing that could work?
- What would "done" look like?

### 2. Domain Exploration (The Dossier Questions)

Apply the [Dossier Principle](../DDD.md). For each domain concept, ask:

| Question | Reveals |
|----------|---------|
| What "things" accumulate history over time? | Dossiers (aggregates) |
| What actions can happen to these things? | Commands / desks |
| What events record that something happened? | Event slips |
| How do we query the current state? | Projections / index cards |

Identify the **ubiquitous language** -- domain terms that everyone uses. Map relationships between concepts. Consult domain experts when available.

### 3. Constraint Identification

What limits the solution space?

| Constraint Type | Examples |
|-----------------|----------|
| Technical | Language, framework, platform, existing systems |
| Business | Budget, timeline, regulations, policies |
| Operational | Scale, availability, latency requirements |
| Team | Skills, availability, timezone |

Document constraints explicitly. They shape every decision that follows.

### 4. Risk Assessment

What could go wrong?

| Risk Category | Questions |
|---------------|-----------|
| Technical | Can we build it? Do we have the skills? |
| Schedule | Can we build it in time? |
| Requirements | Do we understand what is needed? |
| Dependencies | What do we depend on? What depends on us? |
| Unknown unknowns | What don't we know that we don't know? |

For each risk: assess likelihood (low/medium/high), impact (low/medium/high), and define a mitigation strategy.

### 5. Event Storming

The primary design activity. Walk through the domain timeline:

1. **Brainstorm events** -- What happened? (past tense, business language)
2. **Identify commands** -- What caused each event? (imperative, user intent)
3. **Group by aggregate** -- Which events belong together?
4. **Define stream patterns** -- How are aggregate instances identified?
5. **Design status flags** -- What states does each aggregate pass through?

### 6. Aggregate Design

For each aggregate discovered during event storming:

```
Command:  design_aggregate_v1
          { design_id, division_id, aggregate_name, stream_pattern,
            status_flags, description }

Command:  design_event_v1
          { design_id, division_id, event_name, aggregate_name,
            payload_fields, description }

Events:   aggregate_designed_v1
          event_designed_v1
```

Status fields are **bit flags** (see `evoq_bit_flags`). Define each flag as a power of 2:

```erlang
-define(INITIATED,  1).   %% 2^0
-define(ACTIVE,     2).   %% 2^1
-define(PAUSED,     4).   %% 2^2
-define(COMPLETED,  8).   %% 2^3
-define(ARCHIVED,  16).   %% 2^4
```

---

## Outputs

### Required

- [ ] **Problem Statement** -- Clear description of what we are solving
- [ ] **Domain Model** -- Dossiers, events, commands mapped
- [ ] **Aggregate Definitions** -- Names, stream patterns, status flags
- [ ] **Event Catalog** -- All events with payload fields
- [ ] **Constraints Document** -- Technical, business, operational limits

### Recommended

- [ ] **Domain Glossary** -- Key terms and definitions (ubiquitous language)
- [ ] **Risk Register** -- Identified risks with mitigation strategies
- [ ] **Prior Art Notes** -- Similar systems, patterns, lessons learned

---

## Fact Flow

Design produces facts consumed by downstream processes:

```
design_division  --fact-->  plan_division    (aggregates + events as input to desk planning)
design_division  --fact-->  guide_venture    (progress tracking)
```

Design also **receives** facts when rescue escalates:

```
rescue_division  --fact-->  design_division  (redesign escalation)
```

This makes the ALC cycle circular, not linear. A design flaw discovered in production flows back to design through rescue.

---

## Entry Checklist (Before Opening Design)

- [ ] Domain exists (`venture_setup_v1` recorded)
- [ ] Division discovered (`division_discovered_v1` recorded)
- [ ] Scope is roughly defined
- [ ] Access to stakeholders or domain experts (if needed)
- [ ] Time allocated for discovery work

## Exit Checklist (Before Concluding Design)

- [ ] Problem is clearly stated
- [ ] All aggregates named with stream patterns
- [ ] All events cataloged with payload fields
- [ ] Status flags defined for each aggregate (bit flags)
- [ ] Constraints identified and documented
- [ ] Major risks known with mitigations
- [ ] Domain glossary written
- [ ] Ready to hand off to planning

---

## Anti-Patterns

| Anti-Pattern | Problem | Instead |
|--------------|---------|---------|
| **Solution jumping** | Designing desks before understanding the domain | Stay in the problem space -- model first, plan desks second |
| **Assumption blindness** | Treating assumptions as facts | Document and validate every assumption |
| **CRUD events** | `user_created`, `order_updated` | Business verbs: `user_registered`, `order_confirmed` |
| **Horizontal thinking** | Designing `services/`, `handlers/`, `utils/` | Design aggregates and events -- planning will derive desks |
| **Scope creep** | Requirements growing endlessly | Timebox discovery, defer to next design cycle |
| **Analysis paralysis** | Never finishing discovery | Set clear exit criteria, conclude when checklist is met |
| **Lone wolf** | Not talking to stakeholders | Engage early and often |
| **Skipping design** | Jumping straight to code | Wrong assumptions cascade into every downstream process |

---

## The Guided Conversation

Design is driven by a guided conversation that produces structured artifacts, not prose.

1. **Frame a decision** -- Ask a clear, bounded question
2. **Present options** -- Show tradeoffs as a table with pros AND cons
3. **User decides** -- They own the choice
4. **Record the decision** -- It constrains all future decisions
5. **Build forward** -- Each decision narrows the next

**Design conversation produces:** aggregates, events, stream patterns, status flags.

These outputs become the input to [alc/planning.md](planning.md).

---

*Understand first. Model second. The domain tells you what to build -- listen to it.*
