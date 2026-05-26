---
title: HECATE_WALKING_SKELETON
layer: philosophy
audience: [agent, human]
stage: stable
---

# HECATE_WALKING_SKELETON — Fully Operational from Day 1

*Build the thinnest possible slice through all layers, deployed to all environments, before adding features.*

**A doctrine for [HECATE_TnI](HECATE_TESTING_N_IMPLEMENTATION.md)**

---

## The Principle

> **Don't build all the code, then add deployment.**
> **Build the deployment, then add the code.**

A Walking Skeleton is:
- The **thinnest possible end-to-end slice**
- Through **all architectural layers** (CMD → PRJ → QRY)
- Deployed to **all environments** (TEST → STAGING → PROD)
- With **full CI/CD operational**
- **Before** implementing business features

---

## Why Walking Skeleton?

### The Big Bang Anti-Pattern

```
❌ BIG BANG INTEGRATION

Week 1-4: Build all desks locally
Week 5:   "Now let's add CI/CD"
Week 6:   "Why doesn't it deploy?"
Week 7:   "GitOps is broken"
Week 8:   "Works on my machine..."
Week 9:   "Production is different"
Week 10:  Still debugging infrastructure
```

**Problems:**
- Integration issues discovered late
- Infrastructure assumptions invalidated
- Long feedback loops
- High risk at the end

### The Walking Skeleton Approach

```
✅ WALKING SKELETON

Day 1:  Scaffold + CI/CD + GitOps + initialize_* desk
Day 2:  Deploy to PROD (empty but operational)
Day 3+: Add desks one by one, each deploys immediately
```

**Benefits:**
- Integration issues discovered immediately
- Infrastructure validated early
- Short feedback loops
- Low risk throughout

---

## The Skeleton Components

### 1. Codebase Scaffold

Create the domain structure with empty departments:

```
apps/{domain}/
├── src/
│   ├── {domain}_app.erl
│   ├── {domain}_sup.erl
│   ├── {domain}_store.erl
│   │
│   └── initialize_{dossier}/    ← Skeleton desk
│       ├── initialize_{dossier}_desk_sup.erl
│       ├── initialize_{dossier}_v1.erl
│       ├── {dossier}_initialized_v1.erl
│       ├── maybe_initialize_{dossier}.erl
│       └── {dossier}_initialized_to_{table}.erl
│
└── rebar.config

apps/query_{domain}/
├── src/
│   ├── query_{domain}_app.erl
│   ├── query_{domain}_sup.erl
│   ├── query_{domain}_store.erl
│   │
│   └── find_{dossier}/
│       └── find_{dossier}.erl
│
└── rebar.config
```

---

### 2. CI/CD Pipeline

```yaml
# .github/workflows/ci.yml or equivalent

stages:
  - build
  - test
  - package
  - deploy

build:
  - rebar3 compile
  - rebar3 dialyzer

test:
  - rebar3 eunit
  - rebar3 ct (if applicable)

package:
  - rebar3 release
  - docker build
  - docker push

deploy:
  - update GitOps repo
  - (reconciler applies systemd units automatically)
```

**The pipeline must work end-to-end before adding features.**

---

### 3. GitOps Configuration

```
gitops-{project}/
├── base/
│   └── {app}/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── kustomization.yaml
│
└── envs/
    ├── test/
    │   └── {app}/
    │       └── kustomization.yaml    # image: :test-{sha}
    ├── staging/
    │   └── {app}/
    │       └── kustomization.yaml    # image: :staging-{sha}
    └── prod/
        └── {app}/
            └── kustomization.yaml    # image: :v1.0.0
```

**All environments configured before any features.**

---

### 4. The Skeleton Desk

**`initialize_{dossier}_v1`** — The simplest possible desk.

**Purpose:** Create a dossier with minimal data.

**CMD:**
```erlang
%% initialize_capability_v1.erl
-record(initialize_capability_v1, {
    mri :: binary()
}).

new(MRI) ->
    #initialize_capability_v1{mri = MRI}.
```

```erlang
%% capability_initialized_v1.erl
-record(capability_initialized_v1, {
    mri :: binary(),
    initialized_at :: integer()
}).
```

```erlang
%% maybe_initialize_capability.erl
handle(Cmd) ->
    Event = capability_initialized_v1:from_command(Cmd),
    {ok, Event}.
```

**PRJ:**
```erlang
%% capability_initialized_to_capabilities.erl
project(Event) ->
    Row = #{
        mri => Event#capability_initialized_v1.mri,
        status => <<"initialized">>,
        created_at => Event#capability_initialized_v1.initialized_at
    },
    query_capabilities_store:upsert(capabilities, Row).
```

**QRY:**
```erlang
%% find_capability.erl
execute(#{mri := MRI}) ->
    query_capabilities_store:get(capabilities, MRI).
```

**Why this desk?**
- No business logic (just creates empty dossier)
- Tests full CMD → PRJ → QRY flow
- Safe to deploy to production
- Proves infrastructure works

---

## The Skeleton Checklist

### Day 1 Tasks

- [ ] Create codebase scaffold
- [ ] Set up CI/CD pipeline
- [ ] Configure GitOps repos
- [ ] Implement `initialize_{dossier}_v1` desk
- [ ] Implement projection to read model
- [ ] Implement simple query

### Day 1 Verifications

- [ ] `rebar3 compile` passes
- [ ] `rebar3 dialyzer` clean
- [ ] `rebar3 eunit` passes
- [ ] CI pipeline runs successfully
- [ ] Deployed to TEST environment
- [ ] Deployed to STAGING environment
- [ ] Deployed to PROD environment

### Smoke Tests (All Environments)

```bash
# Health check
curl https://{env}.example.com/health
# → 200 OK

# Initialize a dossier
curl -X POST https://{env}.example.com/api/{dossier}/init \
  -d '{"mri": "test-001"}'
# → 201 Created

# Query the dossier
curl https://{env}.example.com/api/{dossier}/test-001
# → 200 OK, {"mri": "test-001", "status": "initialized"}
```

---

## When the Skeleton Walks

**You know the skeleton works when:**

1. Code compiles and passes all checks
2. CI/CD pipeline is green
3. GitOps syncs to all environments
4. Health checks pass in all environments
5. You can create and query a dossier in PROD

**Only then do you add features.**

---

## Adding Features

Once the skeleton walks, each feature follows the same path:

```
1. Implement desk locally
2. Write tests
3. Verify (compile, dialyzer, eunit)
4. Push to trigger CI
5. CI deploys to TEST
6. Verify in TEST
7. Merge to main
8. CI deploys to STAGING
9. Verify in STAGING
10. Tag for PROD release
11. CI deploys to PROD
12. Verify in PROD
```

**Every feature walks the same infrastructure the skeleton proved.**

---

## Common Mistakes

| Mistake | Problem | Solution |
|---------|---------|----------|
| Skipping PROD | "We'll deploy later" | Deploy skeleton to PROD on day 1 |
| Complex skeleton | Too much in first slice | Keep it minimal (initialize only) |
| Manual deploys | "Just this once" | GitOps from the start |
| Skipping tests | "Skeleton is trivial" | Test infrastructure matters |
| Separate skeleton branch | Never merges | Skeleton goes to main |

---

## The Doctrine

> **If you can't deploy an empty system to production on day 1, you have infrastructure problems.**
>
> **Find them now, not after you've built features.**

The Walking Skeleton is not optional. It is the **first task** of every project.

---

## See Also

- [HECATE_TnI](HECATE_TESTING_N_IMPLEMENTATION.md) — Testing & Implementation phase
- [HECATE_ALC](alc/README.md) — The full lifecycle
- [CODEGEN_ERLANG_TEMPLATES.md](../skills/codegen/erlang/CODEGEN_ERLANG_TEMPLATES.md) — Templates for desks
- [CODEGEN_ERLANG_CHECKLISTS.md](../skills/codegen/erlang/CODEGEN_ERLANG_CHECKLISTS.md) — Generation checklists

---

*Build the skeleton. Make it walk. Then add the flesh.*
