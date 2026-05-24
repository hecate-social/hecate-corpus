# SLICE_AUDIT.md — Auditing CMD & QRY Apps

_A repeatable workflow for reviewing vertical slices in any Hecate CMD or QRY app._

**Date:** 2026-02-12
**Origin:** guide_venture_lifecycle / guide_division_alc / guide_node_lifecycle audit session

---

## When to Run This Audit

- After creating a new CMD or QRY app
- After consolidating multiple apps into one
- During DnA phase of a division
- Periodically, to catch drift

---

## The Workflow

### Step 1: Inventory Slices

For each CMD app, list every slice directory under `src/`:

```
apps/{cmd_app}/src/
├── {slice_1}/
├── {slice_2}/
├── ...
└── infrastructure files (aggregate, sup, routes, app)
```

For each QRY app, list projections, queries, and listeners:

```
apps/{qry_app}/src/
├── {event}_to_{table}/          (projections)
├── get_{entity}_page/           (queries)
├── {source}_{subject}_listener/ (mesh listeners)
└── infrastructure files (store, sup, subscriber, routes, app)
```

### Step 2: Evaluate Each Slice Name

For every slice, ask:

| Question | Pass? | Action if Fail |
|----------|-------|---------------|
| **Does the name scream business intent?** | | Rename to a verb phrase that says WHAT, not HOW |
| **Is it a business verb (not CRUD)?** | | Replace `create/update/delete` with business verb |
| **Does a stranger understand what it does?** | | The folder name alone should tell the story |
| **Is it consistent with sibling slices?** | | Naming should be consistent across siblings |

### Step 3: Check Lifecycle Protocol

**Every long-lived sub-process needs its own lifecycle commands.**

The test: "Can a human at this desk get blocked, step away, and come back later?"
If yes → it's a long-lived process → it needs lifecycle verbs.

#### Identifying Sub-Processes

Look at the aggregate's command routing. Group commands by the sub-process they serve:

```
Sub-process: vision refinement
  Commands: refine_vision, submit_vision
  Is it long-lived? YES — investigation, iteration, blocking
  Has lifecycle? open_vision / shelve_vision / resume_vision / submit_vision
```

#### The Lifecycle Verb Rule

> **Lifecycle verbs MUST scream what's happening at the desk.**
> Generic `start_phase` / `pause_phase` / `complete_phase` is FORBIDDEN.
> Each sub-process gets its own screaming lifecycle verbs.

| Generic (BAD) | Screaming (GOOD) | Why |
|--------------|-----------------|-----|
| `start_phase(inception)` | `open_vision` | You're opening an investigation |
| `pause_phase(inception)` | `shelve_vision` | You're putting it on the shelf |
| `resume_phase(inception)` | `resume_vision` | You're picking it back up |
| `complete_phase(inception)` | `submit_vision` | You're handing it off |
| `start_phase(discovery)` | `open_discovery` | You're beginning to explore |
| `pause_phase(discovery)` | `shelve_discovery` | You're setting it aside |
| `complete_phase(discovery)` | `conclude_discovery` | You've reached a conclusion |
| `start_phase(dna)` | `open_design` | You're opening the design workshop |
| `pause_phase(dna)` | `shelve_design` | Blocked, parking the design work |
| `complete_phase(dna)` | `conclude_design` | Design is done, hand off |

**The pattern:**

| Lifecycle Step | Verb | Meaning |
|---------------|------|---------|
| **Start** | `open_{process}` | Open the investigation/workshop/work |
| **Pause** | `shelve_{process}` | Put it on the shelf (with reason) |
| **Resume** | `resume_{process}` | Pick it back up |
| **Complete** | `conclude_{process}` or domain-specific | Hand off to next process |

Some processes have a natural completion verb that's better than `conclude_`:
- Vision refinement → `submit_vision` (submitting for review)
- Discovery → `conclude_discovery` (reaching a conclusion)
- Design → `conclude_design` or `finalize_design`
- Deployment → `confirm_deployment` (confirming it's live)

**Use the natural verb when one exists. Fall back to `conclude_` when nothing better fits.**

### Step 4: Check Walking Skeleton

Every aggregate needs:

| Slice | Purpose | Missing = Problem |
|-------|---------|------------------|
| `initiate_{entity}` | Birth event | No lifecycle, no aggregate |
| `archive_{entity}` | Soft-delete | Test data pollutes, no undo, unbounded lists |

If archive is missing, flag it. This was learned the hard way (test venture with no archive desk).

### Step 5: Check Projection Coverage

For every event emitted by a CMD app, there MUST be a corresponding projection in the QRY app:

```
CMD emits:           QRY projects:           Gap?
vision_refined_v1    (nothing)               ← MISSING
vision_submitted_v1  (nothing)               ← MISSING
discovery_started_v1 discovery_started_v1_to_ventures  ← OK
```

**Every CMD event needs a QRY projection.** If data isn't projected, the read model is stale and queries return wrong information.

### Step 6: Check Query Coverage

For every table in the QRY store, there should be at least one query desk:

```
Store table:          Query desk:              Gap?
ventures              get_ventures_page        ← OK
discovered_divisions  get_discovered_divisions ← OK
designed_aggregates   (nothing)                ← MISSING
planned_desks         (nothing)                ← MISSING
```

Data projected but never queryable is dead weight.

### Step 7: Check Desk Completeness

Every desk is a complete capability with three aspects:

| Aspect | What It Is | Example |
|--------|-----------|---------|
| **Inboxes** | Internal (pg) and external (mesh) topics this desk listens for | `discovery_completed_v1` from venture lifecycle |
| **Policies** | Decision rules: when event/fact arrives → dispatch command? | "When discovery completes, declare expertise with discovered domains" |
| **Emitters** | Facts this desk publishes to the mesh after success | `expertise_declared_v1` to mesh topic |

**A PM IS its own slice.** A process manager is a sibling of desks in the target CMD app, with its own directory, supervisor, and gen_server. This was reversed from the original (2026-02-12) guidance — see [ANTIPATTERNS_STRUCTURE.md Demon 18](ANTIPATTERNS_STRUCTURE.md#-demon-18-process-managers-inside-desks).

```
apps/manage_capabilities/src/
├── declare_expertise/                                  # the desk
│   ├── declare_expertise_v1.erl                        # Command
│   ├── expertise_declared_v1.erl                       # Event
│   ├── maybe_declare_expertise.erl                     # Handler
│   └── declare_expertise_api.erl                       # HTTP entry
│
└── on_discovery_completed_declare_expertise/           # PM sibling slice
    ├── on_discovery_completed_declare_expertise_sup.erl
    └── on_discovery_completed_declare_expertise.erl    # pg:join + dispatch
```

**If you see `on_*` logic nested inside a desk directory** → it should be lifted out into its own sibling slice. `on_*` directories at the top level of `src/` are the discoverability anchor for cross-domain integration.

### Step 8: Reconstruct the Business Process

From the slices, draw the process flow:

1. Find the birth event (initiate/register/submit)
2. Find the sub-processes and their lifecycle commands
3. Find the domain actions within each sub-process
4. Find the terminal event (archive/conclude)
5. Find desk policies that react to cross-domain events

The reconstructed flow should read like a story. If it doesn't, the naming is wrong.

---

## Checklist (Copy-Paste Per App)

```markdown
### Audit: {app_name}

**Aggregate:** {name} | **Stream:** {pattern}

- [ ] All slice names scream business intent (no technical noise)
- [ ] No CRUD verbs (create/update/delete)
- [ ] Consistent naming across sibling slices
- [ ] Every long-lived sub-process has lifecycle (open/shelve/resume/conclude)
- [ ] Lifecycle verbs are process-specific (no generic start_phase/pause_phase)
- [ ] Walking skeleton: initiate + archive both present
- [ ] PMs ARE standalone sibling slices in target CMD app, named `on_{src_event}_{action}_{target}/` — never nested inside desks (Demon 18, reversed)
- [ ] Every desk has complete capability: Inboxes, Policies, Emitters
- [ ] Every CMD event has a QRY projection
- [ ] Every QRY table has at least one query desk
- [ ] Process flow reconstructs into a readable story
- [ ] No leftover/orphan files from pre-consolidation
```

---

## Demon: Parameterized Phase Lifecycle

**Date:** 2026-02-12
**Origin:** guide_division_alc audit

### The Antipattern

Using generic, parameterized lifecycle commands instead of process-specific verbs:

```erlang
%% BAD — generic lifecycle that doesn't scream
execute(State, #{command_type := <<"start_phase">>, phase := <<"dna">>})
execute(State, #{command_type := <<"pause_phase">>, phase := <<"dna">>})
execute(State, #{command_type := <<"complete_phase">>, phase := <<"dna">>})
```

### Why It's Wrong

1. **Doesn't scream** — `start_phase(dna)` tells you nothing about what's happening
2. **Horizontal in disguise** — it's a generic lifecycle manager, not a business action
3. **Hides intent** — a stranger reading the code has no idea what "dna" means or what starting it entails
4. **API noise** — `POST /api/divisions/:id/phases/start` is infrastructure, not business

### The Fix

Each sub-process gets its own screaming lifecycle verbs:

```erlang
%% GOOD — each process screams what it does
execute(State, #{command_type := <<"open_design">>})
execute(State, #{command_type := <<"shelve_design">>})
execute(State, #{command_type := <<"resume_design">>})
execute(State, #{command_type := <<"conclude_design">>})

execute(State, #{command_type := <<"open_planning">>})
execute(State, #{command_type := <<"shelve_planning">>})
execute(State, #{command_type := <<"resume_planning">>})
execute(State, #{command_type := <<"conclude_planning">>})
```

### The Test

> "If I read only the slice directory names, can I understand what this app does?"
>
> `start_phase/` → NO. Start what? Which phase? Why?
> `open_design/` → YES. Someone opens a design workshop.

---

## Demon: CRUD Verbs in Event Sourcing

**Date:** 2026-02-12
**Origin:** guide_node_lifecycle audit

### The Antipattern

Using `update` as a verb in commands and events:

```
update_identity    → identity_updated_v1      ← CRUD
update_capability  → capability_updated_v1    ← CRUD
```

### The Fix

Replace with a business verb that describes WHAT changed:

| CRUD (BAD) | Business Verb (GOOD) | Why |
|-----------|---------------------|-----|
| `update_identity` | `amend_identity` | Metadata amendment — you're amending a record |
| `update_capability` | `amend_capability` | Capability details changed — amended |

The event becomes `identity_amended_v1`, `capability_amended_v1`.

### When "amend" Fits

`amend` means "to make minor changes to improve or correct." It fits when:
- You're changing metadata, not the identity of the entity
- The change is a refinement, not a replacement
- The entity already exists and you're adjusting it

### When Something Else Fits Better

| Action | Better Verb | Example |
|--------|------------|---------|
| Changing status | `promote`, `suspend`, `activate` | `promote_user` |
| Correcting an error | `correct`, `rectify` | `correct_address` |
| Adding information | `enrich`, `supplement` | `enrich_profile` |
| Replacing entirely | `replace`, `supersede` | `supersede_policy` |

---

_This workflow was developed during the first full audit of hecate-daemon's CMD and QRY apps. It catches naming violations, missing lifecycle management, projection gaps, and architectural drift._
