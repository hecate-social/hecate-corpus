---
title: "ALC: Refactoring -- Improve Existing Structure"
layer: philosophy
audience: [agent, human]
stage: stable
---

# ALC: Refactoring -- Improve Existing Structure

*Process 4 of [HECATE_ALC](README.md)*

[Back to ALC Index](README.md)

---

## Purpose

Refactoring is planned, deliberate structural improvement of existing code. It is distinct from:

- **Crafting** -- which creates new code
- **Debugging** -- which finds and fixes defects
- **Rescue** -- which responds to emergencies

Refactoring improves the **shape** of working code without changing its behavior. The tests pass before you start. The tests pass when you finish. What changes is the structure underneath.

**Refactoring is never accidental cleanup buried in a feature branch. It is an explicit process with its own lifecycle.**

---

## Mindset

```
"I do not change behavior. I change structure.
 I write tests first if they don't exist.
 I scope tightly. I finish what I start.
 I never mix refactoring with feature work."
```

---

## Lifecycle

```
open_refactoring     --> refactoring_opened_v1       (begin structural improvement)
shelve_refactoring   --> refactoring_shelved_v1       (blocked, set aside)
resume_refactoring   --> refactoring_resumed_v1       (pick back up)
conclude_refactoring --> refactoring_concluded_v1     (structure improved, hand off)
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

## When to Refactor

Refactoring enters the ALC cycle wherever structural improvement is needed:

```
monitoring  ---> refactoring ---> crafting     (performance rot discovered)
debugging   ---> refactoring ---> crafting     (tangled code found during investigation)
design      ---> refactoring ---> crafting     (better model discovered)
code review ---> refactoring ---> crafting     (structural issues flagged)
```

### Triggers

| Trigger | Signal | Example |
|---------|--------|---------|
| **Debugging reveals structure problems** | Fixing one bug requires touching many files | Cross-cutting concern buried in desks |
| **Monitoring reveals performance issues** | Bottleneck caused by poor structure | N+1 queries in a projection |
| **Design review identifies drift** | Code no longer matches the domain model | Desk names drifted from business language |
| **Code review flags concerns** | Reviewer identifies structural debt | Horizontal patterns crept in (services/, utils/) |
| **New feature is hard to add** | Existing structure fights the change | Adding a desk requires modifying three others |

---

## Key Principle: Tests Before Refactoring

**Always have a safety net before changing structure.**

```
WRONG:  "I'll refactor this and fix the tests after"
RIGHT:  "I'll write tests that pin current behavior, then refactor"
```

If tests don't exist for the code you're about to restructure:
1. **Stop.** Write tests first.
2. Verify the tests pass against the current code.
3. Only then begin restructuring.
4. Run the tests after every change.

The tests are your contract: behavior does not change, only structure.

---

## Activities

### 1. Identify Scope

Define exactly what will be restructured and what will not be touched.

- [ ] Identify the desks, modules, or departments affected
- [ ] Document the current structure (before snapshot)
- [ ] Document the target structure (after snapshot)
- [ ] Confirm the scope is achievable in one refactoring session

**Scope creep is the enemy.** If you discover a second problem while fixing the first, open a new refactoring process for it later. Do not expand scope mid-flight.

### 2. Ensure Test Coverage

- [ ] Run existing tests -- they must pass
- [ ] Identify coverage gaps in the affected code
- [ ] Write pinning tests for any untested behavior
- [ ] Run tests again -- everything green

### 3. Restructure

Make structural changes incrementally:

- Rename modules to match screaming architecture conventions
- Move code into correct vertical slices
- Extract or inline modules as needed
- Update supervision trees if module locations change
- Update references across the codebase

**After every incremental change, run the tests.**

### 4. Verify

- [ ] All existing tests still pass
- [ ] New pinning tests still pass
- [ ] Code compiles cleanly
- [ ] Dialyzer clean
- [ ] No antipatterns introduced (check [antipatterns/INDEX.md](../../skills/antipatterns/INDEX.md))
- [ ] Documentation updated to reflect new structure

### 5. Commit and Conclude

- [ ] Commit with a clear message describing the structural change
- [ ] Update any architecture documentation that references the old structure
- [ ] Conclude refactoring and hand off to crafting or deployment

---

## Outputs

### Required

- [ ] **Restructured Code** -- Better organized, same behavior
- [ ] **Passing Tests** -- All tests green, no regressions
- [ ] **Updated Documentation** -- Architecture docs reflect new structure

### Recommended

- [ ] **Before/After Comparison** -- Document what changed and why
- [ ] **New Pinning Tests** -- Coverage added during the process

---

## Entry Checklist (Before Opening Refactoring)

- [ ] Structural problem identified and documented
- [ ] Scope defined (what changes, what doesn't)
- [ ] Current tests pass
- [ ] No active crafting in the same area (avoid merge conflicts)

---

## Exit Checklist (Before Concluding Refactoring)

- [ ] All tests pass (existing + new pinning tests)
- [ ] Code compiles cleanly and dialyzer is clean
- [ ] No new antipatterns introduced
- [ ] Architecture documentation updated
- [ ] Scope was not expanded (if it was, new refactoring opened for the rest)

---

## Anti-Patterns

| Anti-Pattern | Problem | Instead |
|--------------|---------|---------|
| **Refactoring without tests** | No safety net, behavior changes silently | Write pinning tests first |
| **Scope creep** | Started renaming modules, ended rewriting the domain | Define scope, stick to it |
| **Mixing with feature work** | "While I'm here, I'll also add..." | Separate processes: refactoring then crafting |
| **Big bang refactoring** | Rewrite everything at once | Incremental changes, tests between each |
| **Refactoring as excuse** | "I need to refactor before I can add this feature" (procrastination) | Only refactor when the structure genuinely blocks progress |
| **Invisible refactoring** | Buried in a feature commit, no one knows structure changed | Explicit process, dedicated commits |
| **Skipping documentation** | Code moved but docs still reference old locations | Update docs as part of the process |

---

## Related

- [Debugging](debugging.md) -- Often discovers the need for refactoring
- [Crafting](crafting.md) -- Often follows refactoring to add new features
- [Vertical Slicing](../VERTICAL_SLICING.md) -- The target structure for most refactoring
- [Screaming Architecture](../SCREAMING_ARCHITECTURE.md) -- The naming standard to refactor toward
- [antipatterns/INDEX.md](../../skills/antipatterns/INDEX.md) -- What not to introduce

---

*Reshape the clay. Don't add new clay. Don't break the pot. The goddess refines with intention.*
