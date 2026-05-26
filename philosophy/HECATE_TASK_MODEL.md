---
title: The Task Model
layer: philosophy
audience: [agent, human]
stage: draft
---

# The Task Model — Work as First-Class Citizen

_How Hecate surfaces software development as a list of actionable tasks with AI assistance._

**Date:** 2026-02-11
**Status:** Draft — proposed replacement for phase-centric navigation
**Origin:** Design conversation on task-centric vs phase-centric UX

---

## The Insight

The Domain Lifecycle (HECATE_DOMAIN_LIFECYCLE.md) defines **10 processes** organized by phase. This is the correct domain model — processes are real, phases are real, divisions are real.

But in the **user experience**, phases are not what the user works with. The user works with **tasks**. A task is a concrete action: "Refine the vision," "Design the Auth Service," "Generate code for the Payment Gateway." The phase is just a property of the task — metadata that determines which AI role assists.

**The shift:**

| Aspect | Phase-Centric (old UX) | Task-Centric (new UX) |
|--------|----------------------|---------------------|
| Navigation | Domain → Phase → Division → act | Domain → Task List → pick a task |
| First-class citizen | The phase | The task |
| Phase role | Container to enter | Tag on a task |
| Division role | Context to switch to | Grouping header in task list |
| AI assistance | Manual role switching | Auto-attached to task type |
| What's blocked | Implicit (user guesses) | Explicit (task shows prerequisites) |

**The domain model doesn't change.** The 10 processes, the domain hierarchy, the CMD/QRY/PRJ departments — all unchanged. What changes is how the Dev Studio presents work to the user.

---

## Task Anatomy

Every task has:

```
┌─────────────────────────────────────────────────┐
│ Task                                            │
├─────────────────────────────────────────────────┤
│ verb        : "design"                          │
│ scope       : domain | division                │
│ subject     : "Auth Service" (division name)    │
│ phase       : AnP                               │
│ ai_role     : AnP                               │
│ state       : pending | active | paused | running | done │
│ blocked_by  : ["submit-vision"]                          │
│ has_chat    : true                                       │
│ has_ui      : true (dedicated task UI)                   │
└──────────────────────────────────────────────────────────┘
```

### Task Lifecycle (Event-Sourced)

Every task follows the same lifecycle protocol. Each transition is an **event** stored in the event stream, giving full audit trail and tracking.

```
                    ┌─────────┐
                    │ pending │
                    └────┬────┘
                         │ start
                         ▼
              ┌──── ┌────────┐ ────┐
              │     │ active │     │
              │     └────┬───┘     │
          resume         │      pause
              │          │         │
              │          │         ▼
              │          │    ┌────────┐
              └──────────┼─── │ paused │
                         │    └────────┘
                         │ complete
                         ▼
                   ┌───────────┐
                   │ completed │
                   └───────────┘
```

**State machine:** `pending → active ⇄ paused → completed`

Each transition emits a versioned event:

| Transition | Event | Example |
|-----------|-------|---------|
| start | `{task}_started_v1` | `refine_vision_started_v1` |
| pause | `{task}_paused_v1` | `design_division_paused_v1` |
| resume | `{task}_resumed_v1` | `refine_vision_resumed_v1` |
| complete | `{task}_completed_v1` | `test_division_completed_v1` |

### Task States

| State | Symbol | Meaning |
|-------|--------|---------|
| **blocked** | `○` (dim) | Prerequisites not met |
| **pending** | `○` | Ready to start, prerequisites met |
| **active** | `●` | User is currently working on this |
| **paused** | `◑` | Started but paused (user can resume) |
| **running** | `◐` | AI is working in background |
| **done** | `✓` | Completed |

A task can be **active** (user is interacting) or **running** (AI background work). Multiple tasks can be **running** simultaneously. **Paused** means the user explicitly stepped away — the event stream records when and for how long.

### Desks per Task (Lifecycle)

Each lifecycle transition is its own **desk** (vertical slice). Every task type gets 4 lifecycle desks:

```
refine_vision/
├── start_refine_vision/       → emits refine_vision_started_v1
├── pause_refine_vision/       → emits refine_vision_paused_v1
├── resume_refine_vision/      → emits refine_vision_resumed_v1
├── complete_refine_vision/    → emits refine_vision_completed_v1

design_division/
├── start_design_division/     → emits design_division_started_v1
├── pause_design_division/     → emits design_division_paused_v1
├── resume_design_division/    → emits design_division_resumed_v1
├── complete_design_division/  → emits design_division_completed_v1
```

Task-specific action desks (chat, confirm, etc.) are separate and only valid when the task is **active**.

---

## The Task Catalog

### Domain-Scoped Tasks

These tasks exist once per domain.

| Task | Verb | AI Role | Prerequisites | Ongoing? |
|------|------|---------|---------------|----------|
| **Initiate Domain** | `initiate-domain` | — | None | No |
| **Refine Vision** | `refine-vision` | DnA | Domain initiated | Yes |
| **Submit Vision** | `submit-vision` | — | Vision refined | No |
| **Refine Divisions** | `refine-divisions` | DnA | Vision submitted | Yes |

**Initiate Domain** — Creates the domain. Name, brief description, scaffolding. This is the birth event. No AI needed — it's a form.

**Refine Vision** — The user and AI (DnA role) collaborate to shape the domain's vision. What are we building? Why? For whom? What are the constraints? The chat IS the work — the AI asks questions, challenges assumptions, helps crystallize the vision. This task is **ongoing**: the user can return to refine further.

**Submit Vision** — Locks the vision. An explicit confirmation action (button, not a task UI). After submission, the vision document becomes the foundation for division discovery.

**Refine Divisions** — The user and AI (DnA role) explore the business domain together. Through conversation, the AI gradually proposes divisions (bounded contexts). Each proposed division appears with a **[Confirm]** action. Confirming a division spawns its sub-tasks. This task is **ongoing**: the user can always come back to discover more divisions or rethink existing ones.

### Division-Scoped Tasks

These tasks are created when a division is **confirmed** during "Refine Divisions." One set per division.

| Task | Verb | AI Role | Prerequisites | Ongoing? |
|------|------|---------|---------------|----------|
| **Design Division** | `design-division` | AnP | Division confirmed | Yes |
| **Plan Division** | `plan-division` | AnP | Design reviewed | Yes |
| **Generate Division** | `generate-division` | TnI | Plan approved | No (background) |
| **Test Division** | `test-division` | TnI | Code generated | No (background) |
| **Deploy Division** | `deploy-division` | DnO | Tests passing | No |
| **Monitor Division** | `monitor-division` | DnO | Deployed | Yes |
| **Rescue Division** | `rescue-division` | DnO | Issue detected | Yes |

**Design Division** — The user and AI (AnP role) collaborate on event storming, aggregate design, desk inventory. The task UI shows the emerging domain model alongside the chat. Ongoing — design evolves.

**Plan Division** — The user and AI (AnP role) break down the design into an implementation plan. Sequencing, priorities, dependencies between desks. Ongoing — plans adapt.

**Generate Division** — AI (TnI role) generates skeleton code from templates. This runs in the **background** — the user can work on other tasks while code is being generated. The task shows progress and generated files.

**Test Division** — AI (TnI role) runs tests and verifies the generated code. Also background. Shows test results, coverage, issues found.

**Deploy Division** — Ship it. The user reviews and confirms deployment. Could show deployment status, canary results.

**Monitor Division** — Observe production health. Ongoing — this task stays open as long as the division is deployed. Shows metrics, logs, alerts.

**Rescue Division** — Something broke. Diagnose, intervene, fix. Opens when an issue is detected. The AI (DnO role) helps diagnose. Can escalate back to Design if the fix requires architectural changes.

---

## The Task List (Dev Studio)

The Dev Studio's primary view is the task list. It's a single, scrollable list with visual grouping.

### Layout

```
Domain: MyProject
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 ✓  Initiate Domain
 ✓  Refine Vision                 🤖 DnA
 ✓  Submit Vision
 ●  Refine Divisions              🤖 DnA
─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
 ▸ Auth Service
    ✓  Design                     🤖 AnP
    ✓  Plan                       🤖 AnP
    ◐  Generate                   🤖 TnI
    ○  Test
    ○  Deploy
 ▸ Payment Gateway
    ●  Design                     🤖 AnP
    ○  Plan
    ○  Generate
    ○  Test
    ○  Deploy
 ▸ Notification Service
    ○  Design                     (blocked)
    ○  ...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 ↑/↓ navigate  Enter: open  q: back
```

### Behaviors

- **Collapsible groups**: Division groups can be collapsed/expanded with `Tab` or arrow keys.
- **Status at a glance**: Each task shows its state symbol. Background tasks show progress.
- **Blocked tasks are visible but dimmed**: The user can see what's coming but can't open blocked tasks.
- **Navigate with j/k or arrows**: Standard vim-style navigation in Normal mode.
- **Enter opens the task UI**: Each task type has its own dedicated interface.
- **Division independence**: Tasks in one division don't block tasks in another division.

### Dynamic Growth

The task list grows as the user works:

1. **`venture_initiated_v1`** → "Initiate Domain" completes, "Refine Vision" unlocks
2. **`refine_vision_started_v1`** → task becomes active, AI greets with DnA role
3. **`vision_submitted_v1`** → "Refine Divisions" unlocks
4. **`refine_divisions_started_v1`** → user and AI explore the domain
5. **`division_confirmed_v1`** → Division group appears with 7 sub-tasks
6. **`design_division_started_v1`** → user opens a division task, AI assists with AnP role
7. **`design_division_paused_v1`** → user steps away, timestamp recorded
8. **`design_division_resumed_v1`** → user returns, picks up where they left off

---

## Task UIs

Each task type has a dedicated UI. Most AI-assisted tasks share a common layout:

### AI-Assisted Task Layout

```
┌─────────────────────────────────────────────────┐
│ Design Division: Auth Service          🤖 AnP   │
├─────────────────────────────────────────────────┤
│                                                 │
│  [Task-specific content panel]                  │
│  (domain model, file tree, test results, etc.)  │
│                                                 │
├─────────────────────────────────────────────────┤
│  AI: What authentication patterns are you       │
│  considering? OAuth2, session-based, or JWT?    │
│                                                 │
│  You: We need OAuth2 for external clients and   │
│  session-based for the web dashboard.           │
│                                                 │
│  AI: Good split. I'd suggest two desks:         │
│  - authenticate_oauth_client (external)         │
│  - authenticate_web_session (internal)          │
│  Each with their own aggregate...               │
│                                                 │
├─────────────────────────────────────────────────┤
│ > Type here...                                  │
└─────────────────────────────────────────────────┘
```

The chat component is embedded in the task UI. It's both the AI interaction surface AND the primary text input. The AI role is auto-selected based on the task type.

### Task-Specific Content Panels

| Task | Content Panel |
|------|--------------|
| Refine Vision | Vision document (evolving markdown) |
| Refine Divisions | Proposed divisions list with [Confirm] / [Reject] actions |
| Design Division | Domain model: aggregates, events, commands |
| Plan Division | Task breakdown, sequencing, dependency graph |
| Generate Division | File tree, generation progress, generated code preview |
| Test Division | Test results, coverage, failing tests |
| Deploy Division | Deployment status, rollout progress |
| Monitor Division | Metrics dashboard, recent logs, alerts |
| Rescue Division | Incident timeline, diagnosis, suggested fixes |

### Non-AI Tasks

Some tasks don't need AI assistance:

- **Initiate Domain** — A form (name, brief, path). Completes immediately.
- **Submit Vision** — A confirmation button within the Refine Vision task.
- **Deploy Division** — May be a confirmation + status view (the deployment itself is automated).

---

## The DAG (Prerequisites)

Tasks form a directed acyclic graph. The task engine enforces prerequisites.

### Domain-Level DAG

```
initiate-domain
       │
       ▼
 refine-vision
       │
       ▼
 submit-vision
       │
       ▼
refine-divisions ──────┐
       │               │
       ▼               ▼
  [confirm div A]  [confirm div B]  ...
       │               │
       ▼               ▼
  design-div-A    design-div-B
       │               │
       ...             ...
```

### Division-Level DAG

```
design-division
       │
       ▼
 plan-division
       │
       ▼
generate-division
       │
       ▼
 test-division
       │
       ▼
deploy-division
       │
       ▼
monitor-division
       │
       ├──── (issue detected) ────▶ rescue-division
       │                                   │
       │                                   ▼
       │                           (may escalate to design)
       │
       ▼
    (ongoing)
```

### Cross-Division Independence

Division-level DAGs are completely independent. Auth Service can be deploying while Payment Gateway is still being designed. The only shared prerequisite is that a division must be confirmed before any of its tasks unlock.

---

## Verb-Based API

The daemon API is organized around verbs — each endpoint IS a workflow action. Every task has lifecycle endpoints (start, pause, resume, complete) and task-specific action endpoints.

### API Pattern

Every task follows the same lifecycle pattern:

```
POST /api/start-{task}/:id         → {task}_started_v1
POST /api/pause-{task}/:id         → {task}_paused_v1
POST /api/resume-{task}/:id        → {task}_resumed_v1
POST /api/complete-{task}/:id      → {task}_completed_v1
```

Task-specific actions only work when the task is **active**:

```
POST /api/{task}/:id/{action}      → (task-specific event)
```

### Domain Lifecycle

```
# Initiate (no lifecycle — it's the birth event)
POST   /api/initiate-domain
  Body: { name, brief, path }
  Returns: { venture_id, tasks: [...] }
  Event: venture_initiated_v1

# Refine Vision
POST   /api/start-refine-vision/:venture_id      → refine_vision_started_v1
POST   /api/pause-refine-vision/:venture_id       → refine_vision_paused_v1
POST   /api/resume-refine-vision/:venture_id      → refine_vision_resumed_v1
POST   /api/complete-refine-vision/:venture_id    → refine_vision_completed_v1

# Refine Vision actions (only when active):
POST   /api/refine-vision/:venture_id/chat
  Body: { message }
  Returns: { response, vision_snapshot }

# Submit Vision (confirmation action, not a lifecycle task)
POST   /api/submit-vision/:venture_id
  Returns: { ok, tasks_unlocked: ["refine-divisions"] }
  Event: vision_submitted_v1

# Refine Divisions
POST   /api/start-refine-divisions/:venture_id    → refine_divisions_started_v1
POST   /api/pause-refine-divisions/:venture_id     → refine_divisions_paused_v1
POST   /api/resume-refine-divisions/:venture_id    → refine_divisions_resumed_v1
POST   /api/complete-refine-divisions/:venture_id  → refine_divisions_completed_v1

# Refine Divisions actions (only when active):
POST   /api/refine-divisions/:venture_id/chat
  Body: { message }
  Returns: { response, proposed_divisions: [...] }

POST   /api/refine-divisions/:venture_id/confirm
  Body: { division_name, division_brief }
  Returns: { division_id, tasks_created: [...] }
  Event: division_confirmed_v1
```

### Division Lifecycle

Each division task follows the same lifecycle pattern. `:id` is the division ID.

```
# Design
POST   /api/start-design-division/:id             → design_division_started_v1
POST   /api/pause-design-division/:id              → design_division_paused_v1
POST   /api/resume-design-division/:id             → design_division_resumed_v1
POST   /api/complete-design-division/:id           → design_division_completed_v1

POST   /api/design-division/:id/chat               (only when active)
  Body: { message }
  Returns: { response, model_snapshot }

# Plan
POST   /api/start-plan-division/:id               → plan_division_started_v1
POST   /api/pause-plan-division/:id                → plan_division_paused_v1
POST   /api/resume-plan-division/:id               → plan_division_resumed_v1
POST   /api/complete-plan-division/:id             → plan_division_completed_v1

POST   /api/plan-division/:id/chat                 (only when active)

# Generate (background — starts AI work)
POST   /api/start-generate-division/:id            → generate_division_started_v1
POST   /api/pause-generate-division/:id            → generate_division_paused_v1
POST   /api/resume-generate-division/:id           → generate_division_resumed_v1
POST   /api/complete-generate-division/:id         → generate_division_completed_v1

GET    /api/generate-division/:id/status
  Returns: { progress, files_generated }

# Test (background)
POST   /api/start-test-division/:id               → test_division_started_v1
POST   /api/complete-test-division/:id             → test_division_completed_v1

GET    /api/test-division/:id/status
  Returns: { results, coverage, failures }

# Deploy
POST   /api/start-deploy-division/:id             → deploy_division_started_v1
POST   /api/complete-deploy-division/:id           → deploy_division_completed_v1

POST   /api/deploy-division/:id/confirm
  Body: { target_env }
  Event: deployment_confirmed_v1

# Monitor (ongoing)
POST   /api/start-monitor-division/:id            → monitor_division_started_v1
POST   /api/pause-monitor-division/:id             → monitor_division_paused_v1
POST   /api/resume-monitor-division/:id            → monitor_division_resumed_v1

GET    /api/monitor-division/:id/status
  Returns: { metrics, alerts, health }

# Rescue (opens on incident)
POST   /api/start-rescue-division/:id             → rescue_division_started_v1
POST   /api/complete-rescue-division/:id           → rescue_division_completed_v1

POST   /api/rescue-division/:id/chat               (only when active)
```

### Task List

```
GET    /api/domain/:venture_id/tasks
  Returns: {
    domain: { id, name, vision_status },
    tasks: [
      {
        verb, scope, subject, phase, ai_role,
        state,         -- pending | active | paused | running | completed
        started_at,    -- from *_started_v1 event
        paused_at,     -- from *_paused_v1 event (null if active)
        completed_at,  -- from *_completed_v1 event
        blocked_by     -- task verbs that must complete first
      },
      ...
    ],
    divisions: [
      { id, name, confirmed_at, tasks: [...] },
      ...
    ]
  }
```

This single endpoint powers the Dev Studio's task list view. The frontend subscribes via the `hecate://` protocol for real-time updates. Timestamps from lifecycle events enable tracking ("Design paused 2 days ago", "Generation took 45 minutes").

---

## AI Role Auto-Selection

Each task type knows which AI role it needs. The Dev Studio doesn't ask the user to switch roles — it's automatic.

| Task Type | AI Role | Personality Aspect |
|-----------|---------|-------------------|
| Refine Vision | DnA | Curious, probing, analytical |
| Refine Divisions | DnA | Domain explorer, boundary finder |
| Design Division | AnP | Architect, pattern matcher |
| Plan Division | AnP | Planner, sequencer, prioritizer |
| Generate Division | TnI | Code generator, template engine |
| Test Division | TnI | Quality verifier, edge case finder |
| Deploy Division | DnO | Ops engineer, safety checker |
| Monitor Division | DnO | Observer, metric interpreter |
| Rescue Division | DnO | Diagnostician, firefighter |

The system prompt for each AI interaction is:

```
[PERSONALITY.md]   — Hecate's core personality
---
[role file]        — The phase-specific role (DnA, AnP, TnI, DnO)
---
[task context]     — Domain name, division name, current state, history
---
[task prompt]      — Task-specific instructions (what the AI should focus on)
```

---

## Ongoing vs Completable Tasks

Not all tasks "finish" the same way.

### Completable Tasks

These have a clear end state:

- **Initiate Domain** — Done when domain exists
- **Submit Vision** — Done when confirmed
- **Generate Division** — Done when code is generated
- **Test Division** — Done when tests pass
- **Deploy Division** — Done when deployed

### Ongoing Tasks

These can be revisited indefinitely:

- **Refine Vision** — Vision can always be refined (but must be submitted to proceed)
- **Refine Divisions** — New divisions can always be discovered
- **Design Division** — Design evolves as understanding deepens
- **Plan Division** — Plans adapt to reality
- **Monitor Division** — Always watching
- **Rescue Division** — Opens on incidents, closes when resolved, can reopen

Ongoing tasks show as `✓` when their minimum criteria are met, but the user can always reopen them. They might show a secondary indicator: `✓↻` (done but revisitable).

---

## Impact on Daemon Architecture

### What Changes

1. **New: Task Engine** — A service that tracks task states, prerequisites, and transitions. The task engine reads lifecycle events to derive current state and enforces the DAG (e.g., rejects `start-design-division` if vision is not submitted).

2. **New: Verb-based API routes** — Each workflow action gets its own endpoint. Replaces noun-based CRUD endpoints. Each endpoint maps to a desk.

3. **New: Background job tracking** — Generation and testing run as background jobs with progress reporting via SSE.

4. **Evolves: CMD apps** — The existing process-centric CMD apps (`design_division`, etc.) gain lifecycle desks. Each CMD app has 4 lifecycle desks + N action desks:

```
apps/design_division/src/
├── start_design_division/
│   ├── start_design_division_v1.erl        (command)
│   ├── design_division_started_v1.erl      (event)
│   └── maybe_start_design_division.erl     (handler)
├── pause_design_division/
│   ├── pause_design_division_v1.erl
│   ├── design_division_paused_v1.erl
│   └── maybe_pause_design_division.erl
├── resume_design_division/
│   └── ...
├── complete_design_division/
│   └── ...
├── chat_design_division/                    (task-specific action)
│   ├── chat_design_division_v1.erl
│   ├── design_division_chatted_v1.erl
│   └── maybe_chat_design_division.erl
└── design_division_aggregate.erl            (enforces lifecycle state machine)
```

5. **Evolves: Fact stream** — Lifecycle events emit facts via pg so the Dev Studio updates in real-time. "Design Division started", "Generation paused", etc.

### What Doesn't Change

- **Event sourcing** — Still the foundation. Lifecycle transitions ARE events.
- **CMD/QRY/PRJ departments** — Still the internal architecture.
- **Vertical slicing** — Each desk owns its command, event, and handler.
- **10 processes** — Still the domain model. Tasks map 1:1 to processes.
- **Aggregate pattern** — The aggregate enforces the state machine (can't pause a pending task, can't chat on a paused task).

---

## Impact on Dev Studio

### Dev Studio Structure

The Dev Studio becomes a task-driven workspace:

1. **Task List View** (default) — Shows all tasks, grouped by division, with state indicators.
2. **Task UI View** (on Enter) — Opens the dedicated UI for the selected task.
3. **Back to list** (Escape/q) — Returns to the task list.

The Dev Studio does NOT share the LLM Studio's chat. Each AI-assisted task has its own embedded chat component with the appropriate AI role.

### Key Components

- **TaskList** — The scrollable, collapsible task list with vim navigation
- **TaskChat** — Embedded chat component (reusable across task UIs)
- **VisionEditor** — Refine Vision task UI
- **DivisionExplorer** — Refine Divisions task UI (chat + proposed list)
- **DomainModeler** — Design Division task UI (chat + model view)
- **PlanBuilder** — Plan Division task UI (chat + task breakdown)
- **GenerationMonitor** — Generate Division task UI (progress + file preview)
- **TestRunner** — Test Division task UI (results + coverage)
- **DeployDashboard** — Deploy Division task UI (status + rollout)
- **MonitorDashboard** — Monitor Division task UI (metrics + alerts)
- **RescueConsole** — Rescue Division task UI (diagnosis + chat)

---

## Relationship to Existing Documents

| Document | Relationship |
|----------|-------------|
| `HECATE_DOMAIN_LIFECYCLE.md` | **Foundation** — The 10 processes and domain hierarchy are unchanged. This document adds the UX layer on top. |
| `alc/README.md` | **Evolves** — ALC phases become AI role selectors, not navigation contexts. The four phases (DnA, AnP, TnI, DnO) map to task groups. |
| `DDD.md` | **Unchanged** — The Dossier Principle still applies. Each task works on a dossier. |
| `CARTWHEEL.md` | **Unchanged** — CMD/QRY/PRJ departments are internal architecture, not user-facing. |

---

_Tasks are the work. Phases are the weather. The goddess guides both._ 🔥🗝️🔥
