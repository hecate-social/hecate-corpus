# hecate-agents

*Shaping material for Hecate agent runtimes.*

This repository contains the philosophical foundations, mental models, skills, and guardrails that shape how Hecate agents think and work.

---

## Structure

```
hecate-agents/
├── SOUL.md                           # Identity, values, personality
├── PERSONALITY.md                    # Goddess traits
│
├── philosophy/                       # Mental models (WHY)
│   ├── DDD.md                        # The Dossier Principle
│   ├── CARTWHEEL.md                  # Division Architecture (Cartwheel) overview
│   ├── VERTICAL_SLICING.md           # Features live together
│   ├── SCREAMING_ARCHITECTURE.md     # Names that reveal intent
│   ├── INTEGRATION_TRANSPORTS.md     # pg vs mesh
│   ├── HECATE_VENTURE_LIFECYCLE.md    # Process-centric architecture (venture/division model)
│   ├── PROCESS_MANAGERS.md           # Cross-domain coordination
│   ├── PARENT_CHILD_AGGREGATES.md    # Parent identifies, child initiates (superseded by Venture Lifecycle)
│   ├── HECATE_ALC.md                 # Application Lifecycle
│   ├── HECATE_DISCOVERY_N_ANALYSIS.md
│   ├── HECATE_ARCHITECTURE_N_PLANNING.md
│   ├── HECATE_TESTING_N_IMPLEMENTATION.md
│   ├── HECATE_DEPLOYMENT_N_OPERATIONS.md
│   └── HECATE_WALKING_SKELETON.md
│
├── guides/                           # Deep dives (HOW concepts work)
│   ├── CARTWHEEL_OVERVIEW.md
│   ├── CARTWHEEL_COMPANY_MODEL.md
│   ├── CARTWHEEL_WRITE_SEQUENCE.md
│   ├── CARTWHEEL_PROJECTION_SEQUENCE.md
│   ├── CARTWHEEL_QUERY_SEQUENCE.md
│   ├── HECATE_PLUGIN_DIRECTORY_CONVENTION.md
│   ├── MARTHA_PLUGIN_ARCHITECTURE.md  # Full CQRS plugin reference
│   ├── OBSERVATION_PLUGIN_PATTERN.md  # Read-only plugin (erpc, no CQRS)
│   └── APPSTORE_PLUGIN_LICENSE_LIFECYCLE.md
│
├── skills/                           # Executable knowledge (HOW to do things)
│   ├── ANTIPATTERNS.md               # What NOT to do (guardrails)
│   ├── TESTING.md                    # Testing patterns
│   ├── CODE_QUALITY.md               # Quality standards
│   ├── HOPE_FACT_SIDE_EFFECTS.md     # Mesh message patterns
│   ├── NAMING_CONVENTIONS.md         # Consolidated naming quick-reference
│   └── codegen/erlang/
│       ├── CODEGEN_ERLANG_CHECKLISTS.md   # Generation checklists
│       ├── CODEGEN_ERLANG_TEMPLATES.md    # Code templates
│       ├── CODEGEN_ERLANG_NAMING.md       # Naming rules (detailed)
│       └── BIT_FLAGS_STATUS_PROJECTION.md # Status projection patterns
│
├── examples/                         # Concrete code examples
│   ├── PROJECTIONS.md
│   ├── BIT_FLAGS_STATUS.md
│   ├── MESH_INTEGRATION.md
│   ├── VERTICAL_API_HANDLERS.md
│   └── ERPC_OBSERVER_PATTERN.md       # Cross-node observation via erpc
│
├── templates/                        # Parameterized templates for codegen
│   ├── CHANGELOG.md.tmpl             # Venture scaffolding
│   ├── README.md.tmpl                # Venture scaffolding
│   ├── VISION.md.tmpl                # Venture scaffolding
│   ├── erlang/                       # Erlang desk templates
│   │   ├── cmd_spoke.erl.tmpl        # Command desk (cmd + event + handler)
│   │   ├── cmd_api.erl.tmpl          # Command API handler
│   │   ├── qry_page_spoke.erl.tmpl   # Paged query desk
│   │   ├── qry_byid_spoke.erl.tmpl   # By-ID query desk
│   │   ├── qry_page_api.erl.tmpl     # Paged query API handler
│   │   ├── qry_byid_api.erl.tmpl     # By-ID query API handler
│   │   ├── projection.erl.tmpl       # Projection worker
│   │   ├── listener.erl.tmpl         # Mesh listener
│   │   ├── emitter.erl.tmpl          # Mesh emitter
│   │   ├── process_manager.erl.tmpl  # Process manager
│   │   ├── app_sup.erl.tmpl          # App + supervisor
│   │   ├── app_src.erl.tmpl          # .app.src
│   │   └── rebar_config.tmpl         # rebar.config
│   └── routes/
│       └── route_entry.erl.tmpl      # Route entry
│
├── plans/                            # Active plans
│   └── PLAN_ALC_UX.md
│
└── assets/                           # Images
```

---

## Layers

| Layer | Purpose | Files |
|-------|---------|-------|
| **Soul** | Identity, personality, values | `SOUL.md`, `PERSONALITY.md` |
| **Philosophy** | Mental models, principles | `philosophy/*.md` |
| **Skills** | Executable knowledge, templates | `skills/**/*.md` |
| **Guardrails** | What NOT to do | `skills/ANTIPATTERNS.md` |
| **Templates** | Parameterized code generation | `templates/**/*.tmpl` |

---

## Usage

### For Apprentices (Claude/AI assistants)

Reference these docs in your workspace `CLAUDE.md`:

```bash
cat ~/work/codeberg.org/hecate-social/hecate-agents/philosophy/DDD.md
cat ~/work/codeberg.org/hecate-social/hecate-agents/skills/ANTIPATTERNS.md
```

### For Hecate Martha (AI Agent)

Skills are injected contextually based on the current task:
- Architecture work → Load `philosophy/DDD.md`, `philosophy/CARTWHEEL.md`
- Code generation → Load `skills/NAMING_CONVENTIONS.md` + `skills/codegen/erlang/CODEGEN_ERLANG_TEMPLATES.md`
- Code review → Load `skills/ANTIPATTERNS.md`

### For TnI Codegen (Tier 3)

The LLM reads ONLY:
1. `skills/NAMING_CONVENTIONS.md` — derivation rules
2. The relevant `templates/erlang/*.tmpl` — parameterized template

This is enough for mechanical code generation. No creativity required.

---

## Contributing

These documents shape how agents think. Changes should be deliberate.

- **Philosophy** changes affect mental models
- **Skills** changes affect code output
- **Guardrails** changes affect quality control
- **Templates** changes affect generated code structure

---

*The goddess shapes her servants.*
