---
title: "ALC: Monitoring -- Observe and Detect"
layer: philosophy
audience: [agent, human]
stage: superseded
---

# ALC: Monitoring -- Observe and Detect

*Process 7 of [HECATE_ALC](README.md)*

[Back to ALC Index](README.md)

---

## Purpose

Observe production health, collect metrics, and detect issues before they become incidents. Monitoring is not passive -- it is the continuous act of watching, measuring, and understanding what the division's software is doing in the real world.

**Monitoring is where you learn the truth about your software.**

---

## Mindset

```
"I observe before assuming.
 I measure before optimizing.
 I alert on symptoms, investigate causes.
 I feed everything back into the cycle."
```

---

## Lifecycle

| Command | Event | Transition |
|---------|-------|------------|
| `open_monitoring` | `monitoring_opened_v1` | pending --> active |
| `shelve_monitoring` | `monitoring_shelved_v1` | active --> paused |
| `resume_monitoring` | `monitoring_resumed_v1` | paused --> active |
| `conclude_monitoring` | `monitoring_concluded_v1` | active --> completed |

Monitoring opens when a deployment is concluded and the artifact is running. It remains active as long as the division is in production. It is shelved only when the environment is intentionally taken offline. It is concluded when the division is superseded or decommissioned.

In practice, monitoring is the longest-running process in the ALC. It outlives most other processes.

---

## Activities

### 1. The Four Golden Signals

These are the foundational metrics. Every division in production must track all four.

| Signal | What to Monitor | Alert When |
|--------|-----------------|------------|
| **Latency** | Response time p50, p95, p99 | Exceeds threshold for the endpoint |
| **Traffic** | Requests per second, messages processed | Unusual spike or drop |
| **Errors** | Error rate, error types, error distribution | Exceeds threshold or new error type appears |
| **Saturation** | CPU, memory, connections, disk, queue depth | Exceeds 80% capacity |

Do not invent custom signal categories until these four are fully instrumented. They cover the vast majority of production health.

---

### 2. Health Checks

Every deployed division exposes health endpoints. These are not optional.

```bash
# Liveness: is the process alive?
curl --unix-socket /run/hecate/daemon.sock http://localhost/health
# Expected: {"status": "ok", "version": "0.7.3"}

# Readiness: can it serve traffic?
curl --unix-socket /run/hecate/daemon.sock http://localhost/ready
# Expected: {"status": "ready"} or {"status": "not_ready", "reason": "..."}
```

systemd uses these for container lifecycle management. If liveness fails, the unit restarts. If readiness fails, dependent units wait.

---

### 3. Structured Logging

Logs are not free-text diaries. They are structured events.

```erlang
?LOG_INFO(#{
    event => capability_announced,
    capability_mri => MRI,
    agent_id => AgentId,
    duration_ms => Duration
})
```

**Log levels:**

| Level | Use For | Example |
|-------|---------|---------|
| `debug` | Detailed troubleshooting (noisy, off by default) | State transitions, message payloads |
| `info` | Normal operations worth recording | Deployment completed, config loaded |
| `warning` | Unexpected but handled situations | Retry succeeded, fallback activated |
| `error` | Failures requiring attention | Connection lost, command rejected |

**Correlation IDs:** Every request/command that enters the system gets a correlation ID. Pass it through all log entries so you can trace a single operation across desks and departments.

---

### 4. Dashboards

Build dashboards that answer questions, not dashboards that look impressive.

**Essential dashboards:**

| Dashboard | Shows | Audience |
|-----------|-------|----------|
| **System Health Overview** | The four golden signals at a glance | Everyone |
| **Request/Response Metrics** | Latency distribution, throughput, error rates | Developers |
| **Error Breakdown** | Error types, frequency, affected endpoints | Developers |
| **Resource Utilization** | CPU, memory, disk, connections per node | Operations |

A dashboard nobody looks at is worse than no dashboard -- it creates false confidence.

---

### 5. Alerts

Alerts are for situations that require human attention. Not everything that is unusual requires an alert.

**Alert design principles:**

- Alert on **symptoms**, not causes (high latency, not "CPU at 75%")
- Every alert must have a **runbook link** or action description
- If an alert fires and nobody needs to act, **delete it**
- Tune thresholds based on actual baselines, not guesses

**Essential alerts:**

| Alert | Condition | Action |
|-------|-----------|--------|
| Error rate spike | Error rate > 2x baseline for 5 min | Investigate, consider rollback |
| Latency degradation | p95 > threshold for 10 min | Check dependencies, recent deploys |
| Resource exhaustion | Memory/CPU > 90% for 5 min | Scale or investigate leak |
| Health check failure | Liveness fails for 3 consecutive checks | Pod will auto-restart; investigate if recurring |

**Alert fatigue is a monitoring anti-pattern.** If your team ignores alerts, your monitoring is broken.

---

### 6. Feedback Collection

Monitoring is not only machine metrics. It includes all signals about how the division is performing.

**Sources of feedback:**

| Source | What You Learn |
|--------|----------------|
| Metrics | System behavior, performance trends |
| Logs | Errors, unusual patterns, edge cases |
| User reports | Pain points, bugs, confusion |
| Support tickets | Common issues, recurring problems |
| Usage analytics | Feature adoption, dead code, hot paths |

**Questions monitoring should answer:**

- Is the division performing as expected?
- Are users successful in their tasks?
- What errors are occurring and how often?
- What is confusing or broken?
- What changed since the last deployment?

---

### 7. Feeding Back Into the Cycle

Monitoring closes the loop. Production insights flow backward through the ALC.

```
Production observation
    |
    v
Categorize findings
    |
    +---> Bugs: fix now, open rescue if severe
    +---> Improvements: feed into next design cycle
    +---> Features: add to domain backlog
    +---> Performance rot: open refactoring
    +---> Architectural flaw: escalate to design
```

This is how the ALC becomes a cycle instead of a waterfall. Monitoring observes, categorizes, and routes findings to the appropriate process.

---

## Entry Checklist (Before Opening Monitoring)

- [ ] Deployment concluded -- artifact running in target environment
- [ ] Health check endpoints exposed and responding
- [ ] Structured logging configured
- [ ] Four golden signals instrumented
- [ ] Baseline metrics documented (what "normal" looks like)
- [ ] Alert thresholds set based on baseline

---

## Exit Checklist (Before Concluding Monitoring)

Monitoring rarely concludes. When it does:

- [ ] Division is being decommissioned or superseded
- [ ] All active alerts resolved or transferred
- [ ] Final metrics snapshot documented
- [ ] Feedback log handed off to relevant processes
- [ ] Runbook archived

---

## Transition

**Inbound:** From [Deployment](deployment.md) (process 6). Deployment concluded, artifact is running.

**Outbound:** To [Rescue](rescue.md) (process 8) when monitoring detects an incident. To [Design](design.md) when monitoring reveals architectural issues. To [Refactoring](refactoring.md) when monitoring reveals performance rot.

**Note:** Monitoring is typically the longest-running ALC process. It does not conclude just because other processes open. A division can have active monitoring *and* active design simultaneously.

---

*Watch it. Measure it. Learn from it. Feed it back.*
