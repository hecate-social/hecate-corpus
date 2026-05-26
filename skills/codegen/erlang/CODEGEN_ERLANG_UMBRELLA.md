---
title: Erlang Umbrella Project Layout
layer: codegen
audience: [codegen]
stage: stable
---

# CODEGEN_ERLANG_UMBRELLA.md — Erlang Umbrella Project Layout

_Correct directory structure for Erlang/OTP umbrella applications._

**Target:** Erlang/OTP with rebar3

**Related files:**
- [CODEGEN_ERLANG_CHECKLISTS.md](CODEGEN_ERLANG_CHECKLISTS.md) — Generation checklists
- [CODEGEN_ERLANG_TEMPLATES.md](CODEGEN_ERLANG_TEMPLATES.md) — Code templates
- [CODEGEN_ERLANG_NAMING.md](CODEGEN_ERLANG_NAMING.md) — Naming conventions

---

## The Rule

In a rebar3 umbrella, the **root project IS the shell application**. It has its own `src/` at the root level. Domain apps live under `apps/`.

**The root application is NOT nested under `apps/`.**

---

## Correct Layout

```
my_daemon/
├── rebar.config              # Umbrella: deps, relx listing all apps
├── config/
│   ├── sys.config
│   └── vm.args
├── src/                      # ROOT APP (shell) — lives at project root
│   ├── my_daemon.app.src
│   ├── my_daemon_app.erl     # Starts cowboy, ensures paths
│   ├── my_daemon_sup.erl     # Supervises plugin infra only
│   ├── my_daemon_paths.erl   # Path resolution
│   └── my_daemon_*.erl       # HTTP infra (health, manifest, api_utils)
├── apps/                     # DOMAIN APPS — only CMD/QRY/PRJ here
│   ├── run_something/        # CMD app
│   │   ├── src/
│   │   ├── include/
│   │   └── test/
│   └── query_something/      # QRY app
│       └── src/
├── priv/
└── test/                     # Root app tests (if any)
```

## Wrong Layout

```
my_daemon/
├── rebar.config
├── apps/
│   ├── my_daemon/            # WRONG — root app nested under apps/
│   │   └── src/
│   ├── run_something/
│   └── query_something/
```

**Why this is wrong:** The root project in a rebar3 umbrella already IS an OTP application. Nesting it under `apps/` creates a redundant wrapper — rebar3 treats both the root and everything under `apps/` as apps. The root app should use its natural `src/` directory.

---

## What Goes Where

### Root App (`src/`)

The shell application. Owns infrastructure that spans all domain apps:

| Module | Purpose |
|--------|---------|
| `*_app.erl` | Application callback — starts cowboy, ensures directory layout |
| `*_sup.erl` | Supervises shell-level workers only (e.g. plugin registrar) |
| `*_paths.erl` | Path resolution (base_dir, sqlite_dir, socket_dir, etc.) |
| `*_api_utils.erl` | Shared HTTP response helpers (json_response, json_error) |
| `*_health_api.erl` | `GET /health` endpoint |
| `*_manifest_api.erl` | `GET /manifest` endpoint |
| `*_plugin_registrar.erl` | Registers with hecate-daemon |

The root app's cowboy routes reference handler modules from ALL apps (root + domain apps). This is fine — modules are globally visible within a release.

**The root app does NOT supervise domain workers.** Each domain app owns its own supervisor tree.

### CMD App (`apps/run_*/` or `apps/{verb}_{noun}/`)

Owns the write side — commands, events, handlers, process managers, game engines, etc.

- Has its own `_app.erl`, `_sup.erl`, `.app.src`
- Supervises its own workers (e.g. `duel_proc_sup`)
- Starts any infrastructure it needs (pg scope, ReckonDB store)
- Tests live in `apps/run_*/test/`

### QRY App (`apps/query_*/`)

Owns the read side — SQLite store, query handlers, projections.

- Has its own `_app.erl`, `_sup.erl`, `.app.src`
- Supervises its own workers (e.g. `query_*_store`)
- Owns `esqlite` as a dependency in its `.app.src`

---

## `.app.src` Dependencies

```
Root app:
  applications: [kernel, stdlib, crypto, cowboy, run_something, query_something]
                 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                 External deps                    Domain apps (started first)

CMD app:
  applications: [kernel, stdlib, crypto, query_something]
                                         ^^^^^^^^^^^^^^^^
                                         CMD depends on QRY (records results)

QRY app:
  applications: [kernel, stdlib, esqlite]
                                 ^^^^^^^
                                 Owns the SQLite dependency
```

---

## `rebar.config`

The root `rebar.config` declares ALL external dependencies and lists ALL apps in the release:

```erlang
{deps, [
    {cowboy, "2.12.0"},
    {reckon_db, "1.3.2"},
    {evoq, "1.3.1"},
    {reckon_evoq, "1.1.4"},
    {esqlite, "0.8.8"}
]}.

{relx, [
    {release, {my_daemon, "0.1.0"}, [
        my_daemon,              %% Root/shell app
        run_something,          %% CMD domain app
        query_something,        %% QRY domain app
        reckon_db, evoq, reckon_evoq,
        sasl
    ]},
    ...
]}.
```

**No `src_dirs` needed** — rebar3 automatically discovers `src/` at root and all apps under `apps/`. Sub-directories within `src/` (desk directories like `start_duel/`, `get_leaderboard/`) are discovered automatically by rebar3 in umbrella apps.

---

## Reference Implementations

| Daemon | Repo | Root App | Domain Apps |
|--------|------|----------|-------------|
| hecate-marthad | `hecate-martha/hecate-marthad/` | `hecate_marthad` | `guide_venture_lifecycle`, `query_venture_lifecycle`, `guide_division_alc`, `query_division_alc` |
| snake-dueld | `hecate-app-snake-duel/hecate-app-snake-dueld/` | `hecate_app_snake_dueld` | `run_snake_duel`, `query_snake_duel` |

---

_The root project is the shell. Domain apps live under apps/. Never nest the shell under apps/._
