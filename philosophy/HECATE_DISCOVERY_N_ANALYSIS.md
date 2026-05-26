---
title: HECATE_DnA
layer: philosophy
audience: [agent, human]
stage: stable
---

# HECATE_DnA — Discovery & Analysis

*Understand the problem before solving it.*

**Phase 1 of [HECATE_ALC](alc/README.md)**

---

## Purpose

Before writing any code, understand:
- What problem are we solving?
- Who are we solving it for?
- What constraints exist?
- What could go wrong?

**DnA is about questions, not answers.**

---

## Mindset

```
"I resist the urge to jump to solutions.
 I explore the problem space fully.
 I document what I learn.
 I identify what I don't know."
```

---

## Activities

### 1. Requirements Gathering

**What does the user/system need?**

- Functional requirements: What should it DO?
- Non-functional requirements: How should it PERFORM?
- User stories: Who needs what, and why?
- Acceptance criteria: How do we know it's done?

**Questions to ask:**
- What problem is the user trying to solve?
- What happens if we don't build this?
- What's the simplest thing that could work?
- What would "done" look like?

---

### 2. Domain Exploration

**What concepts exist in this domain?**

- Identify domain terms (ubiquitous language)
- Map relationships between concepts
- Find existing domain knowledge
- Talk to domain experts (if available)

**The Dossier Questions:**
- What "things" accumulate history over time? (dossiers)
- What actions can happen to these things? (commands/desks)
- What events occur? (slips)
- How do we query the current state? (projections)

See: [DDD.md](DDD.md)

---

### 3. Constraint Identification

**What limits our solution space?**

| Constraint Type | Examples |
|-----------------|----------|
| Technical | Language, framework, platform, existing systems |
| Business | Budget, timeline, regulations, policies |
| Operational | Scale, availability, latency requirements |
| Team | Skills, availability, timezone |

**Document constraints explicitly.** They shape every decision.

---

### 4. Risk Assessment

**What could go wrong?**

| Risk Category | Questions |
|---------------|-----------|
| Technical | Can we build it? Do we have the skills? |
| Schedule | Can we build it in time? |
| Requirements | Do we understand what's needed? |
| Dependencies | What do we depend on? What depends on us? |
| Unknown unknowns | What don't we know that we don't know? |

**For each risk:**
- Likelihood (low/medium/high)
- Impact (low/medium/high)
- Mitigation strategy

---

### 5. Prior Art Research

**What exists already?**

- Similar systems (open source, commercial)
- Patterns that apply
- Anti-patterns to avoid
- Lessons learned by others

**Don't reinvent wheels.** Learn from existing solutions.

---

### 6. Stakeholder Mapping

**Who cares about this?**

| Stakeholder | Interest | Influence |
|-------------|----------|-----------|
| Users | Use the system | Feedback |
| Operators | Run the system | Requirements |
| Developers | Build the system | Implementation |
| Business | Pay for the system | Priorities |

**Understand their needs.** They may conflict.

---

## Outputs

### Required

- [ ] **Problem Statement** — Clear description of what we're solving
- [ ] **Requirements List** — Functional and non-functional
- [ ] **Domain Glossary** — Key terms and definitions
- [ ] **Constraints Document** — Technical, business, operational limits

### Recommended

- [ ] **Risk Register** — Identified risks with mitigation
- [ ] **Research Notes** — Prior art, patterns, learnings
- [ ] **Stakeholder Map** — Who cares and why

---

## Checklists

### Before Starting DnA

- [ ] Scope is roughly defined
- [ ] Time allocated for discovery
- [ ] Access to stakeholders/domain experts (if needed)

### Before Leaving DnA

- [ ] Problem is clearly stated
- [ ] Requirements are documented
- [ ] Domain concepts are understood
- [ ] Constraints are identified
- [ ] Major risks are known
- [ ] Ready to design a solution

---

## Anti-Patterns

| Anti-Pattern | Problem | Instead |
|--------------|---------|---------|
| **Solution jumping** | Designing before understanding | Stay in problem space |
| **Assumption blindness** | Treating assumptions as facts | Document and validate assumptions |
| **Scope creep** | Requirements growing endlessly | Timebox discovery, defer to next cycle |
| **Analysis paralysis** | Never finishing discovery | Set clear exit criteria |
| **Lone wolf** | Not talking to stakeholders | Engage early and often |

---

## Templates

### Problem Statement Template

```markdown
## Problem Statement

**Context:** [Background and current situation]

**Problem:** [What's wrong or missing]

**Impact:** [Consequences of not solving this]

**Success Criteria:** [How we'll know it's solved]
```

### Requirement Template

```markdown
## Requirement: [REQ-001]

**Type:** Functional / Non-functional

**Description:** [What is needed]

**Rationale:** [Why it's needed]

**Acceptance Criteria:**
- [ ] [Criterion 1]
- [ ] [Criterion 2]

**Priority:** Must / Should / Could / Won't
```

---

## Transition to AnP

When DnA is complete:

1. Review outputs with stakeholders
2. Confirm understanding is correct
3. Identify first iteration scope
4. Proceed to [HECATE_AnP](HECATE_ARCHITECTURE_N_PLANNING.md)

---

*Understand first. Solve second.*
