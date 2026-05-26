---
title: "ALC: Crafting -- Generate and Write Code"
layer: philosophy
audience: [agent, human]
stage: stable
---

# ALC: Crafting -- Generate and Write Code

*Process 3 of [HECATE_ALC](README.md)*

[Back to ALC Index](README.md)

---

## Purpose

Crafting is the creative, generative process. This is where designs and plans become working software.

Crafting covers:
- Scaffolding codebases from templates
- Generating desk implementations via CODEGEN
- Writing new business logic
- Building the walking skeleton

Crafting does **not** cover:
- Testing or verification (that is [Debugging](debugging.md))
- Building, compiling, or deploying artifacts (that is [Deployment](deployment.md))
- Restructuring existing code (that is [Refactoring](refactoring.md))

**Crafting produces code. Other processes verify and ship it.**

---

## Mindset

```
"I build the skeleton first -- fully operational from day one.
 I follow the templates strictly.
 I generate before I write.
 I commit after each desk, not after each file."
```

---

## Lifecycle

```
open_crafting     --> crafting_opened_v1       (begin writing code)
shelve_crafting   --> crafting_shelved_v1       (blocked, set aside)
resume_crafting   --> crafting_resumed_v1       (pick back up)
conclude_crafting --> crafting_concluded_v1     (hand off to debugging)
```

### State Machine

```
pending --> active --> paused --> completed
  |          |          |
  |       shelve     resume
  |          |          |
  open       +--paused--+
                         |
                      conclude
                         |
                      completed
```

---

## The Walking Skeleton Doctrine

**Before implementing features, establish a fully operational system.**

See: [HECATE_WALKING_SKELETON.md](../HECATE_WALKING_SKELETON.md)

> Don't build all the code, then add deployment.
> Build the deployment, then add the code.

### Day 1 Checklist

- [ ] Codebase scaffold (CMD, PRJ, QRY structure)
- [ ] CI/CD pipeline (build, test, deploy stages)
- [ ] GitOps repositories (TEST, STAGING, PROD)
- [ ] `initialize_{dossier}_v1` desk (end-to-end)
- [ ] Deployed to all environments
- [ ] Verified working

**Only after the skeleton walks do you add features.**

### Why the Skeleton Comes First

```
BIG BANG (wrong):     Build all desks locally --> "Now let's deploy" --> weeks of debugging infra
WALKING SKELETON:     Scaffold + CI/CD + GitOps + one desk --> deployed day 1 --> add desks one by one
```

Integration issues surface immediately. Infrastructure is validated early. Every desk after the skeleton follows the same proven path.

---

## Activities

### 1. Scaffold the Codebase

Create the domain structure with departments:

```
apps/{domain}/
  src/
    {domain}_app.erl
    {domain}_sup.erl
    {domain}_store.erl
    initialize_{dossier}/       <-- Walking skeleton desk
      initialize_{dossier}_v1.erl
      {dossier}_initialized_v1.erl
      maybe_initialize_{dossier}.erl
      {dossier}_initialized_to_{table}.erl

apps/query_{domain}/
  src/
    query_{domain}_app.erl
    query_{domain}_sup.erl
    query_{domain}_store.erl
    find_{dossier}/
      find_{dossier}.erl
```

**Use CODEGEN templates:** See [CODEGEN_ERLANG_TEMPLATES.md](../../skills/codegen/erlang/CODEGEN_ERLANG_TEMPLATES.md) and [CODEGEN_ERLANG_CHECKLISTS.md](../../skills/codegen/erlang/CODEGEN_ERLANG_CHECKLISTS.md)

---

### 2. Implement the Skeleton Desk

**`initialize_{dossier}_v1`** -- The simplest possible desk:

**CMD:**
```erlang
%% Command: Just an ID
initialize_{dossier}_v1:new(Id)

%% Event: Dossier initialized
{dossier}_initialized_v1
```

**PRJ:**
```erlang
%% Project to read model
{dossier}_initialized_to_{table}:project(Event)
```

**QRY:**
```erlang
%% Query by ID
find_{dossier}:execute(#{id => Id})
```

**Why this desk?**
- Proves the full CMD --> PRJ --> QRY flow
- No complex business logic
- Tests all infrastructure
- Safe to deploy to production

---

### 3. Implement Feature Desks

**The implementation loop -- one desk at a time:**

```
+----------------------------------------------+
|                                              |
|  +----------+    +----------+    +--------+  |
|  | Generate |    | Fill In  |    | Verify |  |
|  | Template | -> |  Logic   | -> | Locally|  |
|  +----------+    +----------+    +--------+  |
|       |                              |       |
|       |         +--------+           |       |
|       +---------| Commit |<----------+       |
|                 |  Push  |                   |
|                 +--------+                   |
|                     |                        |
|                     v                        |
|                Next Desk                     |
|                                              |
+----------------------------------------------+
```

For each desk in the plan:

1. Generate from CODEGEN template
2. Fill in business logic
3. Verify locally (compile, dialyzer)
4. Check against [antipatterns/INDEX.md](../../skills/antipatterns/INDEX.md)
5. Commit with clear message
6. Push and verify CI
7. Move to next desk

**Small commits. One desk at a time. Continuous verification.**

---

## Outputs

### Required

- [ ] **Working Code** -- All planned desks implemented
- [ ] **Codebase Scaffold** -- CMD, PRJ, QRY departments populated
- [ ] **Walking Skeleton** -- Deployed and operational before features

### Recommended

- [ ] **API Documentation** -- Endpoint descriptions
- [ ] **README Updates** -- Setup, running instructions

---

## Entry Checklist (Before Opening Crafting)

- [ ] [Planning](planning.md) concluded or sufficient desks planned
- [ ] Desks prioritized and sequenced
- [ ] Walking skeleton scope defined
- [ ] CI/CD and GitOps strategy decided
- [ ] CODEGEN templates available for the target language

---

## Exit Checklist (Before Concluding Crafting)

- [ ] Walking skeleton deployed and verified
- [ ] All planned desks implemented
- [ ] Code compiles cleanly
- [ ] Dialyzer clean (Erlang) or equivalent type checking passes
- [ ] Ready for [Debugging](debugging.md) to write and run tests

---

## Anti-Patterns

| Anti-Pattern | Problem | Instead |
|--------------|---------|---------|
| **Code first, deploy later** | Integration surprises at the end | Walking skeleton first |
| **Big commits** | Hard to review, risky to revert | One desk per commit |
| **Skipping templates** | Inconsistent desk structure | Generate from CODEGEN always |
| **Writing tests during crafting** | Conflates two processes | Craft the code, then debug/test it |
| **Horizontal structure** | services/, handlers/, utils/ | Vertical desks |
| **Complex skeleton** | Too much in first slice | Keep it minimal (initialize + archive only) |
| **Manual file creation** | Drift from conventions | Generate, then customize |

---

## Related

- [Walking Skeleton Doctrine](../HECATE_WALKING_SKELETON.md) -- Fully operational from day one
- [CODEGEN Templates](../../skills/codegen/erlang/CODEGEN_ERLANG_TEMPLATES.md) -- Code generation templates
- [CODEGEN Checklists](../../skills/codegen/erlang/CODEGEN_ERLANG_CHECKLISTS.md) -- Generation checklists
- [Vertical Slicing](../VERTICAL_SLICING.md) -- Features live together
- [Screaming Architecture](../SCREAMING_ARCHITECTURE.md) -- Names reveal intent

---

*Generate the skeleton. Fill in the desks. One at a time. The goddess crafts with precision.*
