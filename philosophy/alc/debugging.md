---
title: "ALC: Debugging -- Test and Verify"
layer: philosophy
audience: [agent, human]
stage: stable
---

# ALC: Debugging -- Test and Verify

*Process 5 of [HECATE_ALC](README.md)*

[Back to ALC Index](README.md)

---

## Purpose

Debugging is the process of testing, investigating failures, and verifying correctness. It covers both **writing tests** and **investigating bugs**.

Debugging covers:
- Writing unit, integration, and end-to-end tests
- Running test suites and analyzing results
- Investigating failures and unexpected behavior
- Verifying acceptance criteria are met
- Code review for correctness

Debugging does **not** cover:
- Writing new business logic (that is [Crafting](crafting.md))
- Restructuring code (that is [Refactoring](refactoring.md))
- Responding to production incidents (that is [Rescue](rescue.md))

**Debugging proves the code works. If it doesn't, debugging finds out why.**

---

## Mindset

```
"I test as I verify, not after.
 I follow the pyramid: many units, some integrations, few E2E.
 I investigate root causes, not symptoms.
 I do not ship what I cannot prove."
```

---

## Lifecycle

```
open_debugging     --> debugging_opened_v1       (begin testing and investigation)
shelve_debugging   --> debugging_shelved_v1       (blocked, set aside)
resume_debugging   --> debugging_resumed_v1       (pick back up)
conclude_debugging --> debugging_concluded_v1     (verified, hand off to deployment)
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

## The Test Pyramid

```
        /\
       /  \       E2E (few)
      /----\      Full flows through API
     /      \
    /--------\    Integration (some)
   /          \   Desk interactions, store operations
  /------------\
 /              \ Unit (many)
/________________\ Handlers, projections, queries in isolation
```

### Unit Tests

**What:** Test individual modules in isolation.

- Handler logic (maybe_register_user receives command, returns correct event)
- Projection logic (user_registered_to_users transforms event to row correctly)
- Query logic (find_user_by_id returns correct data shape)
- Aggregate apply logic (state updates correctly from event)

**How many:** Many. Every handler, projection, and query gets unit tests.

**Tool:** `rebar3 eunit` (Erlang)

### Integration Tests

**What:** Test desk interactions and store operations.

- Command dispatch through the full CMD stack (dispatch --> handler --> event stored)
- Projection receives event and updates read model
- Query reads from populated read model
- Emitters publish events to correct channels

**How many:** Some. One per desk, covering the happy path and key error paths.

**Tool:** `rebar3 ct` (Erlang common_test) or integration test suites

### End-to-End Tests

**What:** Test full flows through the API boundary.

- HTTP request --> command dispatched --> event stored --> projection updates --> query returns
- Multi-desk flows (initialize --> update --> archive)
- Error scenarios that span departments

**How many:** Few. Cover the critical business flows only.

**Tool:** API test suites, smoke tests against deployed environments

---

## Activities

### 1. Write Tests for Crafted Code

After [Crafting](crafting.md) produces desks, debugging writes the tests:

For each desk:
- [ ] Unit tests for handler (happy path + error cases)
- [ ] Unit tests for projection (event transforms correctly)
- [ ] Unit tests for query (returns correct shape)
- [ ] Integration test for full desk flow (dispatch --> query)

### 2. Run Test Suites

```bash
# Full verification
rebar3 compile        # Compiles cleanly
rebar3 dialyzer       # Types check
rebar3 eunit          # Unit tests pass
rebar3 ct             # Integration tests pass (if applicable)
```

**All four must pass. No exceptions. No skipping dialyzer.**

### 3. Investigate Failures

When tests fail or unexpected behavior is found:

1. **Reproduce** -- Write a failing test that captures the bug
2. **Isolate** -- Narrow down to the smallest failing case
3. **Trace** -- Follow the execution path to the root cause
4. **Fix** -- Fix at the source, not with a workaround
5. **Verify** -- Failing test now passes, all other tests still pass

**Fix bugs at the source.** Do not add defensive code to handle "impossible" states. Do not wrap errors to make them go away. Find the root cause and fix it there.

If the root cause is structural (tangled code, wrong abstraction), open a [Refactoring](refactoring.md) process.

### 4. Verify Acceptance Criteria

For each desk, verify against the plan:

- [ ] Does the desk handle the documented command correctly?
- [ ] Does the desk emit the documented event?
- [ ] Does the desk reject invalid input with clear errors?
- [ ] Does the projection populate the read model correctly?
- [ ] Does the query return the expected shape?

### 5. Code Review

Before concluding debugging, review the code for correctness:

- [ ] Follows vertical slicing (no services/, utils/)
- [ ] Names scream intent (no handler, manager, processor)
- [ ] Desk structure matches CODEGEN template
- [ ] Tests exist and pass
- [ ] Dialyzer clean
- [ ] No antipatterns (check [antipatterns/INDEX.md](../../skills/antipatterns/INDEX.md))
- [ ] Evoq callback argument order correct (State first, Payload second)
- [ ] Event names use business verbs, not CRUD verbs
- [ ] Bit flags used for aggregate status fields

---

## Outputs

### Required

- [ ] **Passing Test Suite** -- Unit, integration, and E2E tests all green
- [ ] **Dialyzer Clean** -- No type errors or warnings
- [ ] **Verified Builds** -- Code compiles and all checks pass
- [ ] **Code Review Completed** -- Checklist verified

### Recommended

- [ ] **Coverage Report** -- Identify untested paths
- [ ] **Bug Report** -- Document any defects found and how they were fixed
- [ ] **Regression Tests** -- Tests added for every bug fixed

---

## Entry Checklist (Before Opening Debugging)

- [ ] [Crafting](crafting.md) concluded or sufficient desks implemented to test
- [ ] Code compiles cleanly (debugging tests code, not syntax errors)
- [ ] Test infrastructure available (test databases, mocks, fixtures)

---

## Exit Checklist (Before Concluding Debugging)

- [ ] All unit tests pass
- [ ] All integration tests pass
- [ ] All E2E tests pass (for critical flows)
- [ ] Dialyzer clean
- [ ] Code review checklist completed
- [ ] No known defects remaining (or explicitly deferred with documentation)
- [ ] Ready for [Deployment](deployment.md)

---

## Anti-Patterns

| Anti-Pattern | Problem | Instead |
|--------------|---------|---------|
| **Test after shipping** | Bugs discovered in production | Debug before deploying |
| **Testing only the happy path** | Error cases explode in production | Test error paths explicitly |
| **Skipping dialyzer** | Type errors surface as runtime crashes | Run dialyzer always |
| **Inverting the pyramid** | Many E2E, few unit tests; slow and brittle | Many unit, some integration, few E2E |
| **Fixing symptoms** | Wrapping errors, adding defensive code | Trace to root cause and fix there |
| **Testing implementation** | Tests break on every refactor | Test behavior, not structure |
| **Ignoring flaky tests** | "It passes most of the time" | Fix the flake or remove the test |
| **Skipping code review** | Antipatterns slip through | Review checklist before concluding |

---

## Related

- [Crafting](crafting.md) -- Produces the code that debugging verifies
- [Refactoring](refactoring.md) -- When debugging reveals structural problems
- [Deployment](deployment.md) -- Receives verified code from debugging
- [antipatterns/INDEX.md](../../skills/antipatterns/INDEX.md) -- What to look for during code review
- [Walking Skeleton Doctrine](../HECATE_WALKING_SKELETON.md) -- Verify the skeleton walks before adding features

---

*Test every desk. Trace every failure to its root. The goddess does not ship what she cannot prove.*
