---
title: HECATE_DnO
layer: philosophy
audience: [agent, human]
stage: stable
---

# HECATE_DnO — Deployment & Operations

*Ship it and keep it running.*

**Phase 4 of [HECATE_ALC](alc/README.md)**

---

## Purpose

Get the software into production and keep it healthy:
- Deploy releases safely
- Monitor system health
- Respond to incidents
- Collect feedback for the next cycle

**DnO is where software meets reality.**

---

## Mindset

```
"I deploy small and often.
 I observe before assuming.
 I respond quickly to issues.
 I learn from production."
```

---

## Activities

### 1. Release Preparation

**Before deploying:**

- [ ] Version number assigned (semver)
- [ ] CHANGELOG updated
- [ ] Release notes written
- [ ] Breaking changes documented
- [ ] Rollback plan ready

**Release checklist:**

```markdown
## Release v{X.Y.Z}

### Changes
- feat: ...
- fix: ...

### Breaking Changes
- None / List them

### Migration Steps
- None / List them

### Rollback Plan
- Revert to v{previous}
- Data migration rollback: ...
```

---

### 2. Deployment

**GitOps flow:**

```
1. Merge to main (or push tag)
2. CI builds OCI image, pushes :latest + semver tag to ghcr.io
3. podman auto-update detects new :latest, pulls and restarts
4. Health checks pass
5. Deployed
```

`.container` files use `:latest` with `AutoUpdate=registry`. No manual file edits per release. Semver tags (`v0.10.3`) remain on ghcr.io for rollback.

---

### 2a. GitOps Deployment Principles

**The Golden Rule: Code Repo =/= GitOps Directory**

```
+----------------------------------------------------------------+
|  CODE REPO (hecate-daemon)                                      |
|  - Source code, tests, Dockerfile                               |
|  - Semantic versioning in app.src/mix.exs                       |
|  - Git tags for releases (v0.7.3)                               |
|  - CI builds OCI images                                         |
+----------------------------------------------------------------+
                              |
                              v
+----------------------------------------------------------------+
|  CONTAINER REGISTRY (ghcr.io)                                   |
|  - Images tagged with version (ghcr.io/org/app:v0.7.3)         |
|  - Immutable once pushed                                        |
+----------------------------------------------------------------+
                              |
                              v
+----------------------------------------------------------------+
|  GITOPS DIR (~/.hecate/gitops/)                                  |
|  - Podman Quadlet units (.container files)                      |
|  - References specific image tags                               |
|  - Local reconciler watches this directory                      |
+----------------------------------------------------------------+
                              |
                              v
+----------------------------------------------------------------+
|  NODE (systemd)                                                  |
|  - Reconciler applies Quadlet units to systemd --user           |
|  - Pulls images from registry via podman                        |
+----------------------------------------------------------------+
```

**Complete Deployment Flow (Example):**

```bash
# STEP 1: CODE REPO — Bump version
cd ~/work/github.com/hecate-social/hecate-daemon
# Edit src/hecate.app.src: {vsn, "0.7.3"}

# STEP 2: CODE REPO — Commit, tag, push
git add -A && git commit -m "chore: Bump version to 0.7.3"
git tag v0.7.3
git push origin main
git push origin v0.7.3

# STEP 3: CI BUILDS AUTOMATICALLY
# GitHub Actions triggers on tag push:
# - .github/workflows/docker.yml builds multi-arch image
# - Pushes BOTH ghcr.io/hecate-social/hecate-daemon:0.7.3 AND :latest
# Monitor: gh run list --repo hecate-social/hecate-daemon
# NEVER build docker images locally for production!

# STEP 4: AUTOMATIC UPDATE
# podman auto-update detects new :latest digest, pulls and restarts.
# .container files use Image=app:latest + AutoUpdate=registry.
# No manual .container file edits needed per release.

# STEP 5: VERIFY
curl --unix-socket ~/.hecate/hecate-daemon/sockets/api.sock \
  http://localhost/api/health
# Check version matches expected release
```

**Image Tagging Strategy:**

| What | Tag | Purpose |
|------|-----|---------|
| `.container` files | `:latest` | `AutoUpdate=registry` pulls automatically |
| CI pushes | `:latest` + `:v0.7.3` | Latest for auto-update, semver for rollback |
| Rollback | Pin to `:v0.7.2` | Temporarily override `:latest` in `.container` |

| Practice | Benefit |
|----------|---------|
| `AutoUpdate=registry` | Zero-touch deployments after CI |
| Semver tags on ghcr.io | Rollback to any known-good version |
| `.container` files in gitops | Single source of truth per node |
| CI-only builds | Reproducible, auditable |

**Rollback (Pin to Specific Version):**

```bash
# To "rollback" to v0.7.2:
# Edit ~/.hecate/gitops/hecate-daemon.container:
#   Image=ghcr.io/hecate-social/hecate-daemon:v0.7.2
git -C ~/.hecate/gitops commit -am "fix: Pin hecate-daemon to v0.7.2 due to {reason}"
# Reconciler restarts systemd unit with the pinned image.
# After fixing: revert to :latest to resume auto-updates.
```

Rollback pins to a specific semver tag. After the fix is released, revert to `:latest` to resume auto-updates.

---

**Deployment strategies:**

| Strategy | Use When | Risk |
|----------|----------|------|
| **Node-by-node** | Standard deploys across multiple nodes | Low |
| **In-place** | Single-node updates, quick iteration | Medium |
| **Recreate** | Breaking changes requiring clean state | High (downtime) |

For most Hecate divisions deployed to individual nodes, **in-place** is the default. When deploying across multiple beam cluster nodes, update **one node at a time** and verify health before proceeding to the next.

---

### 3. Smoke Testing

**Immediately after deployment:**

```bash
# Health check via Unix socket
curl --unix-socket ~/.hecate/hecate-daemon/sockets/api.sock \
  http://localhost/health
# Expected: {"status": "ok", "version": "0.7.3"}

# Basic API functionality
curl --unix-socket ~/.hecate/hecate-daemon/sockets/api.sock \
  http://localhost/api/{resource}
# Expected: 200 OK

# Critical path test
# Whatever is most important -- verify it works.
```

**Automated smoke tests should run on every deployment.**

---

### 4. Monitoring Setup

**The four golden signals:**

| Signal | What to Monitor | Alert When |
|--------|-----------------|------------|
| **Latency** | Response time p50, p95, p99 | > threshold |
| **Traffic** | Requests per second | Unusual spike/drop |
| **Errors** | Error rate, error types | > threshold |
| **Saturation** | CPU, memory, connections | > 80% |

**Dashboards:**

- System health overview
- Request/response metrics
- Error breakdown
- Resource utilization

**Alerts:**

- Error rate spike
- Latency degradation
- Resource exhaustion
- Health check failures

---

### 5. Logging

**Structured logging:**

```erlang
?LOG_INFO(#{
    event => capability_announced,
    capability_mri => MRI,
    agent_id => AgentId,
    duration_ms => Duration
})
```

**Log levels:**

| Level | Use For |
|-------|---------|
| `debug` | Detailed troubleshooting |
| `info` | Normal operations |
| `warning` | Unexpected but handled |
| `error` | Failures requiring attention |

**Correlation IDs:** Trace requests across services.

**Viewing logs on a node:**

```bash
# Follow logs for a specific unit
journalctl --user -u hecate-daemon -f

# Logs since last boot
journalctl --user -u hecate-daemon -b

# Logs from a specific time
journalctl --user -u hecate-daemon --since "2026-02-18 10:00:00"
```

---

### 6. Incident Response

**When something goes wrong:**

```
1. DETECT   -- Alert fires or user reports
2. TRIAGE   -- Assess severity and impact
3. MITIGATE -- Stop the bleeding (rollback, restart, disable)
4. DIAGNOSE -- Find root cause
5. RESOLVE  -- Fix the issue
6. REVIEW   -- Post-incident analysis
```

**Severity levels:**

| Severity | Impact | Response |
|----------|--------|----------|
| **SEV1** | System down | All hands, immediate |
| **SEV2** | Major feature broken | Team, < 1 hour |
| **SEV3** | Minor issue | Next business day |
| **SEV4** | Cosmetic/low impact | Backlog |

---

### 7. Feedback Collection

**Sources of feedback:**

| Source | What You Learn |
|--------|----------------|
| Metrics | System behavior, performance |
| Logs | Errors, unusual patterns |
| User reports | Pain points, bugs |
| Support tickets | Common issues |
| Usage analytics | Feature adoption |

**Questions to answer:**

- Is it performing as expected?
- Are users successful?
- What errors are occurring?
- What's confusing or broken?

---

### 8. Iteration Planning

**Feed back into DnA:**

```
Production feedback
    |
Prioritize issues/improvements
    |
Add to backlog
    |
Next DnA cycle
```

**Categories:**

- **Bugs:** Things that don't work -- Fix in current cycle
- **Improvements:** Things that could be better -- Next cycle
- **Features:** New capabilities needed -- Backlog
- **Tech debt:** Internal quality -- Scheduled maintenance

---

## Outputs

### Required

- [ ] **Deployed Release** -- Running in production
- [ ] **Monitoring** -- Dashboards and alerts configured
- [ ] **Runbook** -- How to operate the system

### Recommended

- [ ] **Incident Reports** -- Post-mortems for issues
- [ ] **Feedback Log** -- Collected feedback organized
- [ ] **Performance Baseline** -- Normal metrics documented

---

## Runbook Template

```markdown
# Runbook: {System Name}

## Overview
{What this system does}

## Architecture
{Key components, dependencies}

## Common Operations

### Restart the service
```bash
systemctl --user restart {unit-name}
```

### Check status
```bash
systemctl --user status {unit-name}
```

### Check logs
```bash
journalctl --user -u {unit-name} -f
```

### Inspect running container
```bash
podman ps --filter name={container-name}
podman inspect {container-name}
```

## Troubleshooting

### High latency
1. Check resource utilization: podman stats {container-name}
2. Check dependent services
3. Check recent deployments: git -C ~/.hecate/gitops log --oneline -5

### Error rate spike
1. Check logs for error patterns
2. Check recent deployments
3. Consider rollback (update .container file to previous version)

## Rollback Procedure
1. Edit ~/.hecate/gitops/{unit}.container -- set Image to previous version
2. Commit: git -C ~/.hecate/gitops commit -am "fix: Rollback {unit} to vX.Y.Z"
3. Reconciler restarts unit (or: systemctl --user restart {unit-name})
4. Verify health via smoke test

## Contacts
- Team: ...
- Escalation: ...
```

---

## Checklists

### Before Deployment

- [ ] CI/CD green
- [ ] STAGING verified
- [ ] Release notes ready
- [ ] Rollback plan documented
- [ ] Team notified

### After Deployment

- [ ] Health checks passing
- [ ] Smoke tests passing
- [ ] Monitoring shows healthy
- [ ] No error spikes
- [ ] Users unaffected or notified

### Ongoing Operations

- [ ] Alerts configured
- [ ] Dashboards reviewed regularly
- [ ] Incidents documented
- [ ] Feedback collected
- [ ] Backlog updated

---

## Anti-Patterns

| Anti-Pattern | Problem | Instead |
|--------------|---------|---------|
| **Deploy and forget** | Issues go unnoticed | Monitor actively |
| **No rollback plan** | Stuck when things break | Always have a way back |
| **Alert fatigue** | Real issues ignored | Tune alerts, reduce noise |
| **Manual deployments** | Inconsistent, error-prone | GitOps always |
| **Blame culture** | People hide mistakes | Blameless post-mortems |
| **Ignoring feedback** | Same issues recur | Feed back into planning |
| **`:main` branch tags** | Same tag, different content, no semver | Use `:latest` + semver tags from CI |
| **`podman run` ad-hoc** | Bypasses GitOps, state drift | Update .container file in gitops |
| **Pinned version without `AutoUpdate`** | Must manually edit .container per release | Use `:latest` + `AutoUpdate=registry` |
| **Skipping the version bump** | Cannot tell what is deployed | Always bump, always tag |
| **Building images locally** | Unreproducible, no audit trail | Let CI build from tag |
| **Editing units directly in ~/.config/containers/systemd/** | Bypasses gitops, changes lost on next reconcile | Edit in ~/.hecate/gitops/ only |

---

## Transition to DnA (Next Cycle)

After DnO stabilizes:

1. Production is stable and monitored
2. Feedback collected and organized
3. Issues triaged and prioritized
4. Next iteration scope identified
5. Return to [HECATE_DnA](HECATE_DISCOVERY_N_ANALYSIS.md)

---

*Ship it. Watch it. Learn from it. Improve it.*
