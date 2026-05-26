---
title: Hecate Corpus — Index
layer: index
audience: [agent, human, codegen]
stage: stable
---

# Hecate Corpus

*Shaping material for Hecate agent runtimes — philosophy, skills, guides, guardrails.*

This index is the entry point. Three reading paths are defined below; each loads a different subset of the corpus for a different consumer.

---

## What lives in this repo

| Layer | Directory | Purpose |
|-------|-----------|---------|
| **Soul** | `SOUL.md`, `PERSONALITY.md` | Identity, voice, values |
| **Philosophy** | `philosophy/` | Mental models — the WHY |
| **Guides** | `guides/` | Walkthroughs — the HOW |
| **Skills** | `skills/` | Executable knowledge, guardrails, codegen |
| **Examples** | `examples/` | Concrete patterns |
| **Roles** | `roles/`, `agents/` | Personas |
| **Templates** | `templates/`, `hecate-app-template/` | Parameterized codegen |
| **Glossary** | `GLOSSARY.md` | Canonical vocabulary, 5D hierarchy |
| **Codex** | `CODEX.md` | Narrative booklet distilled from the corpus (draft) |

---

## Reading Path 1 — Apprentice (humans, first contact)

A linear reading path. Stop when you have enough to ship.

### Foundations (~30 min)
1. [`SOUL.md`](SOUL.md) — what Hecate IS
2. [`PERSONALITY.md`](PERSONALITY.md) — how Hecate communicates
3. [`GLOSSARY.md`](GLOSSARY.md) — the 5D hierarchy, the canonical terms
4. [`philosophy/DDD.md`](philosophy/DDD.md) — the Dossier Principle (process over data)
5. [`philosophy/VERTICAL_SLICING.md`](philosophy/VERTICAL_SLICING.md) — features live together
6. [`philosophy/SCREAMING_ARCHITECTURE.md`](philosophy/SCREAMING_ARCHITECTURE.md) — names that reveal intent

### Architecture (~45 min)
7. [`philosophy/CARTWHEEL.md`](philosophy/CARTWHEEL.md) — Division Architecture (CMD/PRJ/QRY)
8. [`guides/CARTWHEEL_COMPANY_MODEL.md`](guides/CARTWHEEL_COMPANY_MODEL.md) — the company metaphor (desks, slips, filing cabinets)
9. [`guides/CARTWHEEL_WRITE_SEQUENCE.md`](guides/CARTWHEEL_WRITE_SEQUENCE.md) — CMD path in detail
10. [`guides/CARTWHEEL_PROJECTION_SEQUENCE.md`](guides/CARTWHEEL_PROJECTION_SEQUENCE.md) — PRJ path
11. [`guides/CARTWHEEL_QUERY_SEQUENCE.md`](guides/CARTWHEEL_QUERY_SEQUENCE.md) — QRY path
12. [`philosophy/PARENT_CHILD_AGGREGATES.md`](philosophy/PARENT_CHILD_AGGREGATES.md) — coordination, not hierarchy
13. [`philosophy/PROCESS_MANAGERS.md`](philosophy/PROCESS_MANAGERS.md) — cross-Division integration

### Process (~30 min)
14. [`philosophy/HECATE_DOMAIN_LIFECYCLE.md`](philosophy/HECATE_DOMAIN_LIFECYCLE.md) — Domain → Division → Process model
15. [`philosophy/alc/README.md`](philosophy/alc/README.md) — Planning + Crafting (the 2-process ALC)
16. [`philosophy/HECATE_WALKING_SKELETON.md`](philosophy/HECATE_WALKING_SKELETON.md) — minimum vertical implementation

### Discipline (~45 min)
17. [`skills/NAMING_CONVENTIONS.md`](skills/NAMING_CONVENTIONS.md) — the strict naming rules
18. [`skills/antipatterns/INDEX.md`](skills/antipatterns/INDEX.md) — demon index (skim first)
19. [`philosophy/COMMAND_PIPELINES.md`](philosophy/COMMAND_PIPELINES.md) — Demon 41's cure
20. [`philosophy/CONSISTENCY_BOUNDARIES.md`](philosophy/CONSISTENCY_BOUNDARIES.md) — Hecate's position vs aggregateless ES / DCB
21. [`philosophy/INTEGRATION_TRANSPORTS.md`](philosophy/INTEGRATION_TRANSPORTS.md) — HOPE / FACT / FEEDBACK

### Production (read when relevant)
21. [`guides/HECATE_PLUGIN_DIRECTORY_CONVENTION.md`](guides/HECATE_PLUGIN_DIRECTORY_CONVENTION.md) — plugin layout
22. [`guides/MARTHA_PLUGIN_ARCHITECTURE.md`](guides/MARTHA_PLUGIN_ARCHITECTURE.md) — full CQRS plugin reference
23. [`philosophy/HECATE_TIER_MODEL.md`](philosophy/HECATE_TIER_MODEL.md) — L0-L4 runtime layers

---

## Reading Path 2 — Agent runtime (Martha, Apprentices, code review)

Skills are loaded contextually. The retrieval system should hold these always-on as system context, then layer on topic-specific docs.

### Always-on system context (~500 lines)
- `SOUL.md`
- `GLOSSARY.md`
- `philosophy/DDD.md` — the Dossier Principle
- `philosophy/VERTICAL_SLICING.md`
- `skills/antipatterns/INDEX.md` — demon index only (link-out for details)

### Topic-based retrieval

| User intent | Load |
|-------------|------|
| "Design a new Division" | `philosophy/CARTWHEEL.md`, `philosophy/HECATE_DOMAIN_LIFECYCLE.md`, `philosophy/DDD.md` |
| "Add a Desk" | `philosophy/CARTWHEEL.md`, `skills/NAMING_CONVENTIONS.md`, `examples/VERTICAL_API_HANDLERS.md` |
| "Write a Process Manager" | `philosophy/PROCESS_MANAGERS.md`, `examples/PROJECTIONS.md` (similar pattern) |
| "Set up integration" | `philosophy/INTEGRATION_TRANSPORTS.md`, `skills/HOPE_FACT_SIDE_EFFECTS.md`, `examples/MESH_INTEGRATION.md` |
| "Code review" | `skills/antipatterns/INDEX.md` + relevant `skills/antipatterns/*.md` |
| "Naming" | `skills/NAMING_CONVENTIONS.md`, `skills/codegen/erlang/CODEGEN_ERLANG_NAMING.md` |
| "Test something" | `skills/TESTING.md` |
| "Plugin work" | `guides/HECATE_PLUGIN_DIRECTORY_CONVENTION.md`, `guides/MARTHA_PLUGIN_ARCHITECTURE.md` |

---

## Reading Path 3 — Codegen (Tier 3, deterministic)

The Tier 3 LLM reads **only** these. No creativity required.

| Step | File |
|------|------|
| Naming derivation | [`skills/codegen/erlang/CODEGEN_ERLANG_NAMING.md`](skills/codegen/erlang/CODEGEN_ERLANG_NAMING.md) |
| Generation checklist | [`skills/codegen/erlang/CODEGEN_ERLANG_CHECKLISTS.md`](skills/codegen/erlang/CODEGEN_ERLANG_CHECKLISTS.md) |
| Templates | [`skills/codegen/erlang/CODEGEN_ERLANG_TEMPLATES.md`](skills/codegen/erlang/CODEGEN_ERLANG_TEMPLATES.md) |
| Scaffolding | [`skills/codegen/erlang/CODEGEN_PLUGIN_SCAFFOLD.md`](skills/codegen/erlang/CODEGEN_PLUGIN_SCAFFOLD.md) |
| Templates source | `templates/erlang/*.tmpl`, `templates/routes/*.tmpl` |

The Tier 3 path is intentionally bare. Philosophy is not needed for mechanical code generation.

---

## The 5D Hierarchy at a Glance

```
Domain (problem space)
  └── Division (bounded context)
       └── Department (CMD | PRJ | QRY)
            └── Desk (single capability)

       Dossier flows through Desks, accumulating Slips
```

See [`GLOSSARY.md`](GLOSSARY.md) for full definitions.

---

## When in doubt

Read [`SOUL.md`](SOUL.md). Then ask:

1. Does this scream its intent? (naming)
2. Does this belong with its feature? (vertical slicing)
3. Am I thinking in processes or objects? (Dossier Principle)
4. Is this strict enough to template? (discipline)

If you can't answer all four with "yes," you're on the wrong path.

---

*The goddess shapes her servants. Five Ds. One way.*
