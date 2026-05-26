---
title: "ALC: Rescue -- Respond and Recover"
layer: philosophy
audience: [agent, human]
stage: stable
---

# ALC: Rescue -- Respond and Recover

*Process 8 of [HECATE_ALC](README.md)*

[Back to ALC Index](README.md)

---

## Purpose

When monitoring detects an incident or production breaks, rescue opens. This is incident response: detect, triage, mitigate, diagnose, resolve, review. Rescue is also where the ALC becomes circular -- when rescue reveals a design flaw, it escalates back to design, and the wheel turns again.

**Rescue is not failure. It is the system learning.**

---

## Mindset

```
"I respond quickly, not recklessly.
 I mitigate first, diagnose second.
 I document everything.
 I never blame -- I learn."
```

---

## Lifecycle

| Command | Event | Transition |
|---------|-------|------------|
| `open_rescue` | `rescue_opened_v1` | pending --> active |
| `shelve_rescue` | `rescue_shelved_v1` | active --> paused |
| `resume_rescue` | `rescue_resumed_v1` | paused --> active |
| `conclude_rescue` | `rescue_concluded_v1` | active --> completed |

A rescue is opened when an incident is detected -- by monitoring alerts, user reports, or direct observation. It is shelved when waiting for external input (vendor response, dependent team). It is concluded when the incident is resolved, the post-mortem is written, and follow-up actions are routed to the appropriate processes.

---

## Activities

### 1. The Incident Response Flow

Every incident follows this sequence. Do not skip steps.

```
1. DETECT    -- Alert fires, user reports, or monitoring flags anomaly
2. TRIAGE    -- Assess severity and blast radius
3. MITIGATE  -- Stop the bleeding (rollback, scale, disable feature)
4. DIAGNOSE  -- Find root cause (logs, metrics, traces)
5. RESOLVE   -- Fix the issue (hotfix, config change, data repair)
6. REVIEW    -- Post-incident analysis (blameless, learning-focused)
```

**Mitigate before diagnosing.** The instinct is to understand first -- resist it. If the system is down, restore service first (rollback, disable, scale), then investigate at leisure. Users do not care about root cause while they are waiting.

---

### 2. Severity Levels

Not all incidents are equal. Severity determines response urgency and communication.

| Severity | Impact | Response Time | Who Responds |
|----------|--------|---------------|--------------|
| **SEV1** | System down, all users affected | Immediate | All hands -- drop everything |
| **SEV2** | Major feature broken, many users affected | < 1 hour | Owning team + escalation |
| **SEV3** | Minor feature broken, workaround exists | Next business day | Owning team |
| **SEV4** | Cosmetic issue, low impact | Backlog | Whoever has time |

**SEV1 and SEV2 always open a rescue process.** SEV3 and SEV4 may be handled as normal debugging work without a formal rescue.

**Severity can change.** A SEV3 that turns out to affect more users than expected escalates. A SEV1 that has a simple fix may be reclassified. Re-triage as new information arrives.

---

### 3. Mitigation Strategies

The goal of mitigation is to restore service, not to fix the bug. These are temporary measures.

| Strategy | When to Use | Trade-off |
|----------|-------------|-----------|
| **Rollback** | New deployment caused the issue | Reverts all changes, including good ones |
| **Feature disable** | One feature is broken, rest is fine | Partial service degradation |
| **Scale up** | Resource exhaustion under load | Buys time, costs resources |
| **Traffic redirect** | One node/region affected | Requires multi-region setup |
| **Hotfix** | Root cause is known and fix is small | Risky if rushed -- prefer rollback |

**Rollback is always the safest first response.** See [Deployment](deployment.md) for rollback mechanics -- it is a forward GitOps action, not a special procedure.

---

### 4. Diagnosis

Once service is restored (mitigation complete), investigate the root cause.

**Investigation checklist:**

- [ ] What changed recently? (deployments, config, traffic patterns)
- [ ] What do the logs say? (filter by correlation ID, error patterns)
- [ ] What do the metrics show? (four golden signals around incident time)
- [ ] Can the issue be reproduced? (staging, local, test)
- [ ] Is this a known issue? (search previous incident reports)

**Common root causes:**

| Category | Examples |
|----------|---------|
| **Deployment** | Bad config, missing migration, version mismatch |
| **Capacity** | Traffic spike, resource leak, connection exhaustion |
| **Dependency** | External service down, network partition, DNS failure |
| **Data** | Corrupted state, unexpected input, schema mismatch |
| **Code** | Logic error, race condition, unhandled edge case |

---

### 5. Resolution

Resolution is the permanent fix. It may be a code change, a configuration update, a data repair, or an architectural change.

- **Small fixes** go through the normal cycle: crafting --> debugging --> deployment
- **Hotfixes** follow an expedited path but still go through GitOps -- no ad-hoc podman run
- **Architectural issues** escalate to design (see Escalation below)

Every resolution must be verified in production after deployment. "It works in staging" is not sufficient.

---

### 6. Post-Incident Review

Every SEV1 and SEV2 gets a blameless post-mortem. SEV3 gets a brief write-up.

**Post-mortem template:**

```markdown
## Incident: {Title}

**Severity:** SEV{N}
**Duration:** {start} to {end} ({total time})
**Impact:** {who was affected, how}

### Timeline
- HH:MM -- {event}
- HH:MM -- {event}
- ...

### Root Cause
{What actually went wrong, technically}

### Mitigation
{What was done to restore service}

### Resolution
{What was done to permanently fix the issue}

### Lessons Learned
- What went well
- What went poorly
- Where we got lucky

### Action Items
- [ ] {action} -- owner: {who}, due: {when}
- [ ] {action} -- owner: {who}, due: {when}
```

**Blameless means blameless.** The question is "how did the system allow this to happen?" not "who caused this?" If a human made an error, the system should have prevented it.

---

## Escalation: The Feedback Loop

Rescue is where the ALC becomes circular. When rescue reveals that the issue is not a bug but a design flaw, it escalates backward through the cycle.

```
rescue detects architectural flaw
    |
    v
conclude_rescue (with escalation findings)
    |
    v
open_design (new cycle begins)
    |
    v
design --> planning --> crafting --> debugging --> deployment --> monitoring
```

**Escalation triggers:**

| Finding | Escalate To |
|---------|-------------|
| Fundamental domain modeling error | Design |
| Missing desk or capability | Planning |
| Structural code rot | Refactoring |
| Performance architecture issue | Design |
| Recurring incidents from same subsystem | Design |

**Escalation is not failure -- it is the system working as intended.** The ALC is designed to cycle. A rescue that escalates to design is evidence that the feedback loop is functioning.

---

## Runbook Template

Every division in production should have a runbook. Rescue references it during incidents.

```markdown
# Runbook: {Division Name}

## Overview
{What this division does, in one paragraph}

## Architecture
{Key components, dependencies, data flow}

## Common Operations

### Check health
{Command to verify the division is healthy}

### View logs
{Command to tail structured logs}

### Restart
{GitOps procedure -- never manual restart}

### Scale
{How to adjust capacity}

## Troubleshooting

### High latency
1. Check resource utilization
2. Check dependent services
3. Check recent deployments

### Error rate spike
1. Check logs for error patterns
2. Check recent deployments
3. Consider rollback

### Health check failure
1. Check pod status
2. Check resource limits
3. Check dependency health

## Rollback Procedure
1. Identify last known good version
2. Update GitOps manifest
3. Push and reconcile
4. Verify smoke tests

## Contacts
- Owning team: ...
- Escalation: ...
```

---

## Entry Checklist (Before Opening Rescue)

- [ ] Incident detected (alert, user report, or observation)
- [ ] Severity assessed (SEV1-4)
- [ ] Communication started (team notified for SEV1/2)
- [ ] Runbook consulted (if available)

---

## Exit Checklist (Before Concluding Rescue)

- [ ] Service restored (mitigation applied)
- [ ] Root cause identified
- [ ] Permanent fix deployed (or escalated to another process)
- [ ] Post-incident review completed (SEV1/2 mandatory)
- [ ] Action items created and assigned
- [ ] Findings routed: bugs to debugging, design flaws to design, performance issues to refactoring
- [ ] Runbook updated with new knowledge

---

## Transition

**Inbound:** From [Monitoring](monitoring.md) (process 7) when an incident is detected. Can also be triggered by user reports or external alerts.

**Outbound:** Back to [Monitoring](monitoring.md) when the incident is resolved and production returns to normal. To [Design](design.md) when rescue reveals architectural flaws (the cycle restarts). To [Refactoring](refactoring.md) when rescue reveals structural rot.

**The cycle:** Rescue is process 8, but it points back to process 1. This is not a pipeline -- it is a wheel. The goddess turns it as many times as needed.

---

*Detect. Triage. Mitigate. Diagnose. Resolve. Learn. Turn the wheel.*
