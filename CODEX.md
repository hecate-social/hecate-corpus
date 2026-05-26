---
title: The Hecate Codex
layer: codex
audience: [human, agent]
stage: draft
---

# The Hecate Codex

*A practitioner's manual for process-centric software development.*

---

## What this is

This is the distilled book form of the Hecate Corpus. The corpus is a working repository of philosophy, skills, and guardrails; this codex is its narrative companion. You can ship Hecate-style systems by reading only the corpus, but the codex is the thing to hand to a new collaborator, a funder, a colleague at another shop, or your future self after a long break.

The corpus answers "what do we do here?" The codex answers "why, in what order, and how does it hang together?"

## Who this is for

- Senior engineers and architects who feel that mainstream "clean architecture" patterns leak business intent through technical layers.
- Teams adopting event sourcing who want a process-centric mental model, not a data-centric one.
- New hires at any shop that ships Hecate-style work — the codex is the onboarding artifact.
- Funders, advisors, and reviewers who need a single document explaining the doctrine end-to-end.

If you read four chapters and disagree with the central claim that *process beats data*, the rest of the book will frustrate you. Stop early; we will not have been a good fit.

## How to read this

Five parts. Designed to be read in order; designed to survive being read out of order.

| Part | Title | Pages | Topic |
|------|-------|-------|-------|
| I | Soul | 8 | Identity, voice, the four crossroad questions |
| II | Mental Models | 30 | The 5D Hierarchy, Dossier Principle, Screaming, Vertical Slicing |
| III | Architecture | 50 | CMD/PRJ/QRY, Process Managers, Integration, Tier model, Plugins |
| IV | Process | 25 | The 2-process ALC, Walking Skeleton, Storming |
| V | Practice | 40 | Naming, the 20 demons, Command Pipelines, Code Quality |
| App | Appendix | 15 | Glossary, codegen templates summary, citation index |

Total target: ~170 pages, ~50,000 words.

Each chapter follows the same shape:
1. **The Claim** — one sentence stating what the chapter says.
2. **Why it matters** — the failure mode this chapter prevents.
3. **The Pattern** — the actual content.
4. **What it replaces** — the wrong pattern this displaces.
5. **Read also** — pointers to corpus files for depth.

---

## Table of Contents

### Part I — Soul

- **Chapter 1.** [Identity at the Crossroads](#chapter-1-identity-at-the-crossroads)
- **Chapter 2.** [Five Values](#chapter-2-five-values)
- **Chapter 3.** [The Four Crossroad Questions](#chapter-3-the-four-crossroad-questions)

### Part II — Mental Models

- **Chapter 4.** [The 5D Hierarchy](#chapter-4-the-5d-hierarchy)  ← *cornerstone*
- **Chapter 5.** The Dossier Principle (`philosophy/DDD.md`)
- **Chapter 6.** Vertical Slicing (`philosophy/VERTICAL_SLICING.md`)
- **Chapter 7.** Screaming Architecture (`philosophy/SCREAMING_ARCHITECTURE.md`)
- **Chapter 8.** The Company Model (`guides/CARTWHEEL_COMPANY_MODEL.md`)
- **Chapter 9.** Events Are Sacred (distilled from SOUL.md + DDD.md)

### Part III — Architecture

- **Chapter 10.** Division Architecture (`philosophy/CARTWHEEL.md`)
- **Chapter 11.** CMD: Write Sequence (`guides/CARTWHEEL_WRITE_SEQUENCE.md`)
- **Chapter 12.** PRJ: Projection Sequence (`guides/CARTWHEEL_PROJECTION_SEQUENCE.md`)
- **Chapter 13.** QRY: Query Sequence (`guides/CARTWHEEL_QUERY_SEQUENCE.md`)
- **Chapter 14.** Process Managers (`philosophy/PROCESS_MANAGERS.md`)
- **Chapter 15.** Integration Transports: HOPE / FACT / FEEDBACK (`philosophy/INTEGRATION_TRANSPORTS.md`)
- **Chapter 16.** Parent-Child Coordination (`philosophy/PARENT_CHILD_AGGREGATES.md`)
- **Chapter 17.** The Tier Model (L0-L4) (`philosophy/HECATE_TIER_MODEL.md`)
- **Chapter 18.** Plugin Architecture (`guides/MARTHA_PLUGIN_ARCHITECTURE.md`, `guides/HECATE_PLUGIN_DIRECTORY_CONVENTION.md`)

### Part IV — Process

- **Chapter 19.** The Domain Lifecycle (`philosophy/HECATE_DOMAIN_LIFECYCLE.md`)
- **Chapter 20.** The 2-Process ALC: Planning + Crafting (`philosophy/alc/README.md`)
- **Chapter 21.** The Walking Skeleton (`philosophy/HECATE_WALKING_SKELETON.md`)
- **Chapter 22.** EventStorming, Hecate-Style (`roles/storm/*.md`, `agents/storm_*/`)

### Part V — Practice

- **Chapter 23.** Naming Conventions (`skills/NAMING_CONVENTIONS.md`)
- **Chapter 24.** The 20 Demons We've Exorcised (`skills/antipatterns/INDEX.md` + each demon file)
- **Chapter 25.** Command Pipelines, Demon 41's Cure (`philosophy/COMMAND_PIPELINES.md`)
- **Chapter 26.** Status as Bit Flags (`skills/codegen/erlang/BIT_FLAGS_STATUS_PROJECTION.md`, `examples/BIT_FLAGS_STATUS.md`)
- **Chapter 27.** Code Quality, Hecate-Flavored (`skills/CODE_QUALITY.md`)

### Appendix

- **A.** Glossary (`GLOSSARY.md`)
- **B.** Codegen Templates Summary (`skills/codegen/erlang/*.md`)
- **C.** Citation Index — corpus file → codex chapter cross-reference

---

# Part I — Soul

*Eight pages. Read in one sitting. Re-read every time you join a new project.*

---

## Chapter 1. Identity at the Crossroads

### The Claim

Hecate agents work at thresholds. Between intent and implementation, between human thought and machine execution, between code and architecture. The name is the brief.

### Why it matters

Software work fails when the worker is unclear about which side of a threshold they are on. The frontend developer pretending to be a domain modeller produces a leaky read model. The backend engineer pretending to be a product designer produces forms with eleven optional fields. The architect pretending to be a coder produces diagrams that no codebase reflects.

Hecate is a deliberate stance at the threshold. We are the ones who pass between sides without pretending to be either. We hold the keys, not the territory.

### The Pattern

Three thresholds, three keys.

| Threshold | Hecate's role | The key |
|-----------|---------------|---------|
| Code ↔ Architecture | Ensure code looks like the architecture intends | Names that reveal intent |
| Intention ↔ Implementation | Ensure intent survives to running code | Vertical slices that group everything for one capability |
| Human ↔ Machine | Ensure humans control gates and machines do mechanical work | Strict patterns templatable into codegen |

We are not the architect deciding the topology. We are not the coder writing the function body. We are the one ensuring that, when you stand at the threshold and look both ways, the code and the architecture are the same thing.

### What it replaces

The "full-stack senior" pretending to be every role. The "architect" who hands down diagrams to "developers." The "AI assistant" treating code as autocomplete and architecture as text. All three collapse the threshold into one side and lose the other.

### Read also

- `SOUL.md` — the goddess Hecate framing in full
- `PERSONALITY.md` — the voice (confident, witty, direct, warm)

---

## Chapter 2. Five Values

### The Claim

Five values, in priority order, settle every architectural argument inside Hecate.

### Why it matters

Without a value priority, every design conversation devolves into "I prefer this." With a value priority, the conversation becomes "which value applies here?" — which is a tractable question because the priority is fixed.

### The Pattern

```
1. Process Over Data            — Ask "what has happened?" not "what is?"
2. Vertical Over Horizontal     — Features live together; never group by technical concern
3. Strict Over Creative         — When the pattern is tight enough to template, template
4. Explicit Over Clever         — Name for intent. No `poller`, no `handler`. Business verbs.
5. Events Are Sacred            — Immutable. Append-only. Never mutate. Never delete.
```

When two values conflict, the higher-priority one wins. When you face a tie, read SOUL.md until you remember why these are in this order.

#### How to use the priority

- *Should I add a `services/` folder for cross-cutting code?* — Value 2 wins. No.
- *Should I rename `OrderCreated` to `OrderPlaced`?* — Value 1 wins. Yes.
- *Should I add a clever metaprogramming trick to reduce boilerplate?* — Value 3 + 4 win. No; use the template.
- *Should I mutate an event to fix a typo in production?* — Value 5 wins. Never. Add a corrective event.
- *Should we extract a beautiful abstraction over these three Desks?* — Value 3 wins, Value 4 wins. No; copy three times.

### What it replaces

The "It depends" architecture conversation. With a priority, it depends *on* something specific that can be looked up.

### Read also

- `SOUL.md`, "Values" section, for the longer form
- `skills/antipatterns/INDEX.md` for what happens when the priority is ignored

---

## Chapter 3. The Four Crossroad Questions

### The Claim

Before any decision involving code structure, ask four questions. If you cannot answer all four with yes, you are on the wrong path.

### Why it matters

Most architectural mistakes are *not* mistakes of knowledge — the people making them know the rules. They are mistakes of *attention*. The four questions are a checklist that converts attention into a small, predictable ritual.

### The Pattern

```
1. Does this scream its intent?       (naming)
2. Does this belong with its feature?  (vertical slicing)
3. Am I thinking in processes, not objects?  (Dossier Principle)
4. Is this strict enough to template?  (discipline)
```

Run the four against the artifact you are about to create. A file. A directory. A function. A schema. A test.

- Question 1 fails → rename until it passes.
- Question 2 fails → move it to where it belongs.
- Question 3 fails → restate the work in terms of events that happen, not state that is.
- Question 4 fails → either tighten the pattern until it can be templated, or accept that this is genuinely creative work that deserves a name and a doc.

If three of four pass and one fails persistently, you may have found a legitimate exception. Write it down. Add it to the corpus.

### What it replaces

"Looks fine, ship it." The four questions cost about ninety seconds. They prevent the multi-week regret of code that scattered a feature across seven directories.

### Read also

- `SOUL.md`, "The Crossroads" section
- `skills/SLICE_AUDIT.md` for the automated form of question 2

---

# Part II — Mental Models

*Thirty pages. The chapters in this part are the only ones a new collaborator must read in full. The rest can be skimmed and looked up. Not these.*

---

## Chapter 4. The 5D Hierarchy

*The cornerstone chapter. If you read only one chapter, read this one.*

### The Claim

Hecate organizes work into a five-level hierarchy of containers, each named with a word starting with D. Four are nested containers. One is the artifact that flows through them. Every file, function, repo, and conversation lives somewhere in this hierarchy.

```
Domain                    — the whole problem space
  └── Division            — one bounded context per Division
       └── Department     — CMD, PRJ, or QRY
            └── Desk      — one capability

       Dossier flows through Desks, accumulating Slips.
```

### Why it matters

Hierarchies in software fail in two opposite ways:

1. **Too flat.** Everything lives in `src/`. New readers cannot find anything. The "ubiquitous language" of DDD becomes "the ubiquitous folder of Things."
2. **Too rigid.** Layers proliferate (services, repositories, controllers, models, utils, helpers). The cost of touching one feature is reading seven directories.

The 5D Hierarchy solves both by being:
- **Strictly nested** — every artifact has exactly one home at each level.
- **Strictly limited in number** — four container levels, no more.
- **Memorable** — five Ds, said aloud in a single breath.

A new collaborator who internalizes the 5D Hierarchy can navigate any Hecate codebase, predict where any feature will live, and propose a new feature's location without asking.

### The Pattern

#### The four containers

##### Domain

The overall business endeavor. The whole problem space the venture is concerned with. One per business intent.

| Aspect | Detail |
|--------|--------|
| Cardinality | 1 per endeavor |
| Lifecycle | Setup → Discovery → ongoing |
| Owns | Brand, the list of Divisions, strategic intent |
| Concrete examples | "Macula," "Parksim," "BEAM Campus" |

> ⚠ "Domain" is a heavily-overloaded word in Domain-Driven Design. Hecate Domain = Eric Evans' senior usage (the whole problem space), never the junior usage (= bounded context). The bounded-context concept lives at the **Division** level. See the Glossary's Domain override notice for the full disambiguation.

##### Division

A cohesive piece of software responsible for exactly one bounded context. The unit of repo, or of umbrella app under a shared codebase.

| Aspect | Detail |
|--------|--------|
| Cardinality | N per Domain |
| Lifecycle | Discovery → Planning → Crafting → Operations |
| Owns | One bounded context, three Departments |
| Concrete examples | `parksim-entry2exit`, `parksim-lot`, `parksim-pricing`, `hecate-trader` |

A Division has its own ubiquitous language. Two Divisions in the same Domain do not share schemas, do not share databases, do not call each other directly. They communicate over the mesh as integration FACTs.

##### Department

A CQRS role inside a Division. Exactly three Departments per Division: CMD, PRJ, QRY.

| Department | Role | Naming |
|------------|------|--------|
| CMD | Receives intents, produces events | `{process_verb}_{subject}` (e.g., `manage_capabilities`, `design_division`) |
| PRJ | Subscribes to events, writes read models | `project_{read_model_plural}` (e.g., `project_capabilities`) |
| QRY | Reads from read models, serves queries | `query_{read_model_plural}` (e.g., `query_capabilities`) |

CMD is process-centric (verb-named). PRJ and QRY are data-centric (noun-named). The asymmetry is intentional: the work side is named for what it *does*; the read side is named for what it *holds*.

##### Desk

A single capability inside a Department. One folder, one supervisor, one vertical slice. The atomic unit of work in Hecate.

A CMD Desk contains: a command record, an event record, a handler, optionally a responder/emitter/listener. Everything that makes the capability work lives in that one folder.

| Department | Desk naming |
|------------|-------------|
| CMD | `{verb}_{noun}/` (e.g., `announce_capability/`) |
| PRJ | `{event}_to_{table}/` (e.g., `capability_announced_v1_to_capabilities/`) |
| QRY | `get_{noun}_by_id/` or `get_{nouns}_page/` |

A Desk is the unit at which code generation produces a useful artifact. A Desk is the unit at which a new feature is added.

#### The flowing artifact

##### Dossier

The aggregate, in Hecate's vocabulary. A folder of event slips that accumulates as the Dossier passes through Desks.

The Dossier is not a data structure. The Dossier is its history. The stream of Slips *is* the Dossier. When you "rebuild state," you read the Dossier front to back. When validating a command, you ask: *given the Slips accumulated so far, can this Slip be added?*

The Dossier flows. It enters at one Desk (`initiate_*`), passes through other Desks (each adding its Slip), and eventually exits (`conclude_*` or `archive_*`). At any moment in its life, the Dossier's current state is fully determined by the Slips it carries.

> The choice to make the Dossier the fifth D rather than a sixth-level container is deliberate. The four containers are *places* in the codebase; the Dossier is the *thing that moves through them*. The mnemonic captures this asymmetry: "five Ds, four nested, one flowing."

#### Where the 5D shows in a file path

```
/                                                  ← Domain (the repo/org)
  parksim-pricing/                                 ← Division (one repo per bounded context)
    apps/
      manage_pricing_policies/                     ← Department: CMD
        src/
          set_pricing_policy/                      ← Desk
            set_pricing_policy_v1.erl              ← Command
            pricing_policy_set_v1.erl              ← Slip (event)
            maybe_set_pricing_policy.erl           ← Handler
      project_pricing_policies/                    ← Department: PRJ
        src/
          pricing_policy_set_v1_to_pricing_policies/  ← Desk
            ...
      query_pricing_policies/                      ← Department: QRY
        src/
          get_pricing_policies_page/               ← Desk
            ...
```

The Dossier in this example is the stream `pricing-policy-{lot-id}`. It is not a directory or a file — it is the *event stream* identified by that stream ID, holding all Slips ever added at all Desks.

#### Scale invariance

The 5D Hierarchy works at every scale.

- A one-person Hecate Domain might have 1 Division with 2 Desks per Department: six Desks total.
- A fifty-person Hecate Domain might have 15 Divisions, each with 10 Desks per Department: 450 Desks total.
- The structure is identical. The hierarchy does not need to "evolve" as the Domain grows.

This is the property that makes the 5D Hierarchy worth memorizing. You learn it once. It applies forever.

### What it replaces

The traditional "layered architecture" model — Presentation, Application, Domain, Infrastructure — fails the four-questions test. Layers do not scream. Layers scatter features. Layers think in objects, not processes. Layers are clever, not strict.

The 5D Hierarchy is the alternative. Containers, not layers. Process, not object. Verb-named at the CMD level. Strictly templatable at the Desk level. The four crossroad questions all answer yes.

### Read also

- `GLOSSARY.md` — full definitions, including the Domain override notice
- `philosophy/CARTWHEEL.md` — Department detail (CMD/PRJ/QRY structure)
- `philosophy/HECATE_DOMAIN_LIFECYCLE.md` — Domain → Division relationship and process model
- `guides/CARTWHEEL_COMPANY_MODEL.md` — the company-metaphor variant of the 5D model

---

## Chapter 5. The Dossier Principle

> **Stub.** Distilled in full from `philosophy/DDD.md`. The chapter establishes that aggregates ARE their event histories, not state objects events apply to. Five consequences: stream ID = Dossier identity; Desks are verbs; "rebuilding state" = reading the Dossier; business rules = "can this Slip be added?"; projections are "index cards for the filing cabinet."

## Chapter 6. Vertical Slicing

> **Stub.** Distilled from `philosophy/VERTICAL_SLICING.md`. The chapter argues that horizontal layers (`services/`, `utils/`, `helpers/`) are graves where understanding dies, and that every feature must own its full vertical: command, event, handler, projection, listener.

## Chapter 7. Screaming Architecture

> **Stub.** Distilled from `philosophy/SCREAMING_ARCHITECTURE.md`. The chapter requires that folder and file names reveal business intent, not technical mechanism. A reader looking at `apps/` should learn the business domain from folder names alone.

## Chapter 8. The Company Model

> **Stub.** Distilled from `guides/CARTWHEEL_COMPANY_MODEL.md`. The chapter offers an alternative framing of the Division as a small company specializing in one process, with frontdesks (mesh-facing), backoffice desks (internal), tube mail (pg/mesh), and filing cabinets (read models).

## Chapter 9. Events Are Sacred

> **Stub.** Synthesizes SOUL.md Value 5 + the consequences laid out in DDD.md. The chapter establishes immutability, append-only-ness, and the never-mutate-never-delete rule. Cites the "rebuild state by replaying" guarantee and Demon 41 as enforcement.

---

# Part III — Architecture

> **Outline only.** Eighteen chapters covering Division Architecture (10), CMD/PRJ/QRY sequences in detail (11-13), Process Managers (14), Integration Transports (15), Parent-Child coordination (16), the Tier Model (17), and Plugin Architecture (18). Each chapter distilled from one or two corpus files listed in the TOC above.

# Part IV — Process

> **Outline only.** Four chapters: the Domain Lifecycle (19), the 2-process ALC (20), the Walking Skeleton (21), and EventStorming Hecate-style (22). Distilled from `philosophy/HECATE_DOMAIN_LIFECYCLE.md`, `philosophy/alc/README.md`, `philosophy/HECATE_WALKING_SKELETON.md`, and `roles/storm/*.md`.

# Part V — Practice

> **Outline only.** Five chapters: Naming (23), the 20 demons (24), Command Pipelines (25), Bit Flags (26), Code Quality (27). Distilled from `skills/NAMING_CONVENTIONS.md`, `skills/antipatterns/`, `philosophy/COMMAND_PIPELINES.md`, `skills/codegen/erlang/BIT_FLAGS_STATUS_PROJECTION.md`, and `skills/CODE_QUALITY.md`.

# Appendix

- **A. Glossary** — see `GLOSSARY.md`
- **B. Codegen Templates Summary** — see `skills/codegen/erlang/`
- **C. Citation Index** — each corpus file mapped to its codex chapter; pending generation

---

## Draft status

| Part | Status | Drafted | Outline |
|------|--------|---------|---------|
| I — Soul | Drafted | Chapters 1, 2, 3 | — |
| II — Mental Models | Cornerstone drafted | Chapter 4 | Chapters 5-9 stubbed |
| III — Architecture | Outline | — | Chapters 10-18 |
| IV — Process | Outline | — | Chapters 19-22 |
| V — Practice | Outline | — | Chapters 23-27 |
| App | Outline | — | A, B, C |

**Next session targets** (in priority order):
1. Draft Chapter 5 (Dossier Principle) — distill from `philosophy/DDD.md`
2. Draft Chapter 6 (Vertical Slicing)
3. Draft Chapter 10 (Division Architecture) — the second cornerstone
4. Then expand Part III sequentially.

---

*The goddess shapes her servants. Five Ds. One way.*
