---
title: "ALC: Deployment -- Ship to Environments"
layer: philosophy
audience: [agent, human]
stage: stable
---

# ALC: Deployment -- Ship to Environments

*Process 6 of [HECATE_ALC](README.md)*

[Back to ALC Index](README.md)

---

## Purpose

Get the division's software from a tested artifact into running environments. Deployment is not "push and pray" -- it is a deliberate, repeatable, auditable process where every version is traceable, every change flows through GitOps, and every rollback is a forward action.

**Deployment is where discipline meets production.**

---

## Mindset

```
"I deploy small and often.
 I never build locally for production.
 I tag everything.
 Every deployment is traceable to a commit."
```

---

## Lifecycle

| Command | Event | Transition |
|---------|-------|------------|
| `open_deployment` | `deployment_opened_v1` | pending --> active |
| `shelve_deployment` | `deployment_shelved_v1` | active --> paused |
| `resume_deployment` | `deployment_resumed_v1` | paused --> active |
| `conclude_deployment` | `deployment_concluded_v1` | active --> completed |

A deployment is opened when a debugged, tested artifact is ready to ship. It is shelved when blocked (e.g., environment unavailable, dependency not ready). It is concluded when the artifact is running on the target node and smoke tests pass.

---

## Activities

### 1. Release Preparation

Before deploying, the artifact must be tagged and documented.

**Release checklist:**

- [ ] Version number assigned (semver: major.minor.patch)
- [ ] CHANGELOG updated with version entry
- [ ] Release notes written
- [ ] Breaking changes documented
- [ ] Migration steps documented (if any)
- [ ] Rollback plan ready

**Release template:**

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

### 2. The GitOps Flow

**The Golden Rule: Code Repo and GitOps Directory are separate.**

```
+----------------------------------------------------------------+
|  CODE REPO (e.g., hecate-daemon)                               |
|  - Source code, tests, Dockerfile                              |
|  - Semantic versioning in app.src / mix.exs                    |
|  - Git tags for releases (v0.7.3)                              |
|  - CI builds OCI images on tag push                            |
+----------------------------------------------------------------+
                              |
                              v
+----------------------------------------------------------------+
|  CONTAINER REGISTRY (ghcr.io)                                  |
|  - CI pushes BOTH :latest AND semver tag (v0.7.3)             |
|  - Semver tags are immutable, :latest is mutable              |
+----------------------------------------------------------------+
                              |
                              v
+----------------------------------------------------------------+
|  GITOPS DIR (~/.hecate/gitops/)                                 |
|  - Podman Quadlet units (.container files)                     |
|  - Image=app:latest + AutoUpdate=registry                      |
|  - Reconciler manages unit lifecycle (add/remove)              |
+----------------------------------------------------------------+
                              |
                              v
+----------------------------------------------------------------+
|  NODE (systemd + podman auto-update)                            |
|  - podman auto-update detects new :latest digest               |
|  - Automatically pulls and restarts the container              |
+----------------------------------------------------------------+
```

---

### 3. Complete Deployment Flow (Example)

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
# .container files use Image=app:latest + AutoUpdate=registry.
# podman auto-update detects new :latest digest, pulls and restarts.
# No manual .container file edits needed per release.

# STEP 5: VERIFY
curl --unix-socket ~/.hecate/hecate-daemon/sockets/api.sock \
  http://localhost/api/health
# Check version matches expected release
```

**Rollback (pin to specific version):**

```bash
# To "rollback" to v0.7.2:
# Edit ~/.hecate/gitops/hecate-daemon.container:
#   Image=ghcr.io/hecate-social/hecate-daemon:v0.7.2
git -C ~/.hecate/gitops commit -am "fix: Pin hecate-daemon to v0.7.2 due to {reason}"
# Reconciler restarts systemd unit with the pinned image.
# After fixing: revert to :latest to resume auto-updates.
```

Rollback pins to a specific semver tag. After the fix ships, revert to `:latest` to resume auto-updates. There is no separate rollback mechanism. There is no ad-hoc `podman run`. There is no SSH-and-restart.

---

### 4. Deployment Strategies

| Strategy | Use When | Risk | How |
|----------|----------|------|-----|
| **In-place** | Single-node updates, quick iteration | Medium | Update .container file, reconciler restarts the unit |
| **Node-by-node** | Multiple beam cluster nodes | Low | Update one node at a time, verify health before proceeding |
| **Recreate** | Breaking changes requiring clean state | High (downtime) | Stop unit, clean up volumes/state, start with new version |

For most Hecate divisions running on individual nodes, **in-place** is the default. When deploying across the beam cluster (beam00-03), update **one node at a time** and verify health before proceeding to the next. Blue/green and canary are not applicable to per-node systemd deployments -- node-by-node sequencing provides the same risk reduction.

---

### 5. Smoke Testing

**Immediately after deployment, verify the artifact is alive and functional.**

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
# Whatever is most important to the division -- verify it works.
```

Automated smoke tests should run on every deployment. If a smoke test fails, the deployment is not concluded -- it is shelved or the version is rolled back.

---

## Anti-Patterns

| Anti-Pattern | Problem | Instead |
|--------------|---------|---------|
| `Image=app:main` | Same tag, different content, no semver | Use `:latest` + semver tags from CI |
| Pinned version without `AutoUpdate` | Must manually edit .container per release | Use `:latest` + `AutoUpdate=registry` |
| `podman run` ad-hoc | Bypasses GitOps, causes state drift | Update .container file in gitops |
| Editing units in ~/.config/containers/systemd/ | Bypasses gitops, lost on next reconcile | Edit in ~/.hecate/gitops/ only |
| Skipping the version bump | Cannot tell what is deployed | Always bump, always tag |
| Building images locally | Unreproducible, no audit trail | Let CI build from the git tag |
| Deploy and forget | Issues go unnoticed until users complain | Hand off to [Monitoring](monitoring.md) |
| No rollback plan | Stuck when things break | Always document the rollback path |
| Manual deployments | Inconsistent, error-prone | GitOps always |

---

## Entry Checklist (Before Opening Deployment)

- [ ] Debugging concluded -- tests pass, quality verified
- [ ] Version number assigned (semver)
- [ ] CHANGELOG updated
- [ ] Release notes written
- [ ] Breaking changes documented
- [ ] Rollback plan documented
- [ ] CI/CD pipeline green
- [ ] STAGING verified (if applicable)
- [ ] Team notified

---

## Exit Checklist (Before Concluding Deployment)

- [ ] Artifact deployed to target node
- [ ] Health checks passing
- [ ] Smoke tests passing
- [ ] No error spikes in initial observation window
- [ ] Version traceable on node (correct image tag running: `podman ps`)
- [ ] Monitoring process ready to open (hand off to [Monitoring](monitoring.md))
- [ ] Users unaffected or notified of changes

---

## Transition

**Inbound:** From [Debugging](debugging.md) (process 5). Debugging concluded, artifact is tested and ready.

**Outbound:** To [Monitoring](monitoring.md) (process 7). Deployment concluded, artifact is running -- now observe it.

**Escalation:** If deployment reveals issues that cannot be fixed by rollback, open [Rescue](rescue.md).

---

*Tag it. Ship it. Verify it. Move on.*
