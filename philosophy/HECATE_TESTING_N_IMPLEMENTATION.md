---
title: HECATE_TnI
layer: philosophy
audience: [agent, human]
stage: stable
---

# HECATE_TnI — Testing & Implementation

*Build it right.*

**Phase 3 of [HECATE_ALC](alc/README.md)**

---

## Purpose

Transform the design into working software:
- Scaffold the codebase
- Set up deployment infrastructure
- Implement desks following templates
- Test and verify continuously

**TnI is about disciplined execution.**

---

## Mindset

```
"I build the skeleton first — fully operational from day 1.
 I follow the templates strictly.
 I test as I build.
 I verify before committing."
```

---

## The Walking Skeleton Doctrine

**Before implementing features, establish a fully operational system.**

See: [HECATE_WALKING_SKELETON.md](HECATE_WALKING_SKELETON.md)

### Day 1 Checklist

- [ ] Codebase scaffold (CMD, PRJ, QRY structure)
- [ ] CI/CD pipeline (build, test, deploy stages)
- [ ] GitOps repositories (TEST, STAGING, PROD)
- [ ] `initialize_{dossier}_v1` desk (end-to-end)
- [ ] Deployed to all environments
- [ ] Verified working

**Only after the skeleton walks do you add features.**

---

## Activities

### 1. Scaffold the Codebase

**Create the domain structure:**

```
apps/{domain}/
├── src/
│   ├── {domain}_app.erl
│   ├── {domain}_sup.erl
│   ├── {domain}_store.erl
│   └── initialize_{dossier}/      ← Walking skeleton desk
│       └── ...
└── rebar.config
```

**Use CODEGEN templates:** See [CODEGEN_ERLANG_TEMPLATES.md](../skills/codegen/erlang/CODEGEN_ERLANG_TEMPLATES.md) and [CODEGEN_ERLANG_CHECKLISTS.md](../skills/codegen/erlang/CODEGEN_ERLANG_CHECKLISTS.md)

---

### 2. Set Up CI/CD Pipeline

**Pipeline stages:**

```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│  Build  │ →  │  Test   │ →  │ Package │ →  │ Deploy  │
└─────────┘    └─────────┘    └─────────┘    └─────────┘
```

**Build stage:**
- Compile
- Dialyzer (type checking)
- Linting

**Test stage:**
- Unit tests (eunit)
- Integration tests
- Coverage report

**Package stage:**
- Create release
- Build container image
- Tag with version

**Deploy stage:**
- Deploy to TEST (automatic on PR)
- Deploy to STAGING (automatic on main)
- Deploy to PROD (manual approval or tag)

---

### 3. Set Up GitOps

**Repository structure:**

```
gitops-{project}/
├── envs/
│   ├── test/
│   │   └── {app}/
│   │       ├── deployment.yaml
│   │       └── kustomization.yaml
│   ├── staging/
│   │   └── {app}/
│   │       └── ...
│   └── prod/
│       └── {app}/
│           └── ...
└── base/
    └── {app}/
        └── ...
```

**Flow:**
1. CI builds and pushes OCI image
2. Update image tag in ~/.hecate/gitops/ .container file
3. Reconciler detects change
4. systemd restarts the unit with new image

---

### 4. Implement the Skeleton Desk

**`initialize_{dossier}_v1`** — The simplest possible desk:

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
- Proves the full flow works
- No complex business logic
- Tests all infrastructure
- Safe to deploy to PROD

---

### 5. Verify the Skeleton

**Before adding features:**

```bash
# Local verification
rebar3 compile        # ✓ Compiles
rebar3 dialyzer       # ✓ Types check
rebar3 eunit          # ✓ Tests pass

# Pipeline verification
git push              # ✓ CI passes
                      # ✓ Deploys to TEST

# Smoke test
curl https://test.example.com/api/{dossier}/health  # ✓ 200 OK
```

**The skeleton must walk in all environments before proceeding.**

---

### 6. Implement Feature Desks

**For each desk in the plan:**

```
1. Generate from CODEGEN template
2. Fill in business logic
3. Write tests
4. Verify locally (compile, dialyzer, eunit)
5. Check against ANTIPATTERNS
6. Commit with clear message
7. Push and verify CI
8. Verify in TEST environment
```

**One desk at a time. Small commits. Continuous verification.**

---

### 7. Testing Strategy

**Test pyramid:**

```
        /\
       /  \      E2E (few)
      /----\
     /      \    Integration (some)
    /--------\
   /          \  Unit (many)
  --------------
```

**Unit tests:** Test handlers, projections, queries in isolation
**Integration tests:** Test desk interactions, store operations
**E2E tests:** Test full flows through API

**Test as you build, not after.**

---

### 8. Code Review Checklist

Before merging, verify:

- [ ] Follows vertical slicing (no services/, utils/)
- [ ] Names scream intent (no handler, manager, processor)
- [ ] Desk structure matches CODEGEN template
- [ ] Tests exist and pass
- [ ] Dialyzer clean
- [ ] No antipatterns (check [antipatterns/INDEX.md](../skills/antipatterns/INDEX.md))

---

## Outputs

### Required

- [ ] **Working Code** — All planned desks implemented
- [ ] **Test Suite** — Unit, integration, E2E tests
- [ ] **CI/CD Pipeline** — Automated build, test, deploy
- [ ] **GitOps Configuration** — All environments

### Recommended

- [ ] **API Documentation** — Endpoint descriptions
- [ ] **README Updates** — Setup, running, testing instructions
- [ ] **Architecture Decision Records** — Implementation choices

---

## The Implementation Loop

```
┌─────────────────────────────────────────────────┐
│                                                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│  │ Generate │ →  │  Fill In │ →  │   Test   │  │
│  │ Template │    │  Logic   │    │  Verify  │  │
│  └──────────┘    └──────────┘    └──────────┘  │
│       │                               │        │
│       │         ┌──────────┐          │        │
│       └─────────│  Commit  │←─────────┘        │
│                 │   Push   │                   │
│                 └──────────┘                   │
│                      │                         │
│                      ▼                         │
│                 Next Desk                     │
│                                                 │
└─────────────────────────────────────────────────┘
```

---

## Checklists

### Before Starting TnI

- [ ] PLAN_*.md complete
- [ ] Desks prioritized
- [ ] Walking skeleton scope defined
- [ ] CI/CD requirements known
- [ ] GitOps strategy decided

### Walking Skeleton Complete

- [ ] Codebase scaffolded
- [ ] CI/CD pipeline working
- [ ] GitOps repos configured
- [ ] `initialize_*` desk implemented
- [ ] Deployed to TEST, STAGING, PROD
- [ ] Smoke tests passing

### Before Leaving TnI

- [ ] All planned desks implemented
- [ ] All tests passing
- [ ] Dialyzer clean
- [ ] CI/CD green
- [ ] Deployed to STAGING
- [ ] Ready for production release

---

## Anti-Patterns

| Anti-Pattern | Problem | Instead |
|--------------|---------|---------|
| **Code first, deploy later** | Integration surprises | Walking skeleton first |
| **Big commits** | Hard to review, risky | Small, focused commits |
| **Test later** | Bugs accumulate | Test as you build |
| **Skipping dialyzer** | Type errors in prod | Run dialyzer always |
| **Manual deployments** | "Works on my machine" | GitOps everything |
| **Horizontal structure** | services/, handlers/ | Vertical desks |

---

## Transition to DnO

When TnI is complete:

1. All desks implemented and tested
2. CI/CD pipeline fully operational
3. STAGING deployment verified
4. Release prepared (version, changelog)
5. Proceed to [HECATE_DnO](HECATE_DEPLOYMENT_N_OPERATIONS.md)

---

*Build the skeleton. Add the flesh. Test every step.*
