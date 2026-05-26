---
title: Observation Plugin Pattern
layer: guide
audience: [agent, human]
stage: stable
---

# Observation Plugin Pattern

_How to build a read-only Hecate plugin that observes another daemon's state._

**Date:** 2026-03-01
**Status:** Active
**Reference Implementation:** `hecate-apps/hecate-app-meshview/`

---

## When to Use This Pattern

Use this instead of a full CQRS plugin when ALL of these are true:

1. **Read-only** -- you observe state, you never modify it
2. **No domain events** -- nothing happens worth recording
3. **No aggregates** -- no business process to model
4. **Data lives elsewhere** -- another daemon already owns the state

If any of these are false, use the full CQRS plugin pattern
(see [MARTHA_PLUGIN_ARCHITECTURE.md](MARTHA_PLUGIN_ARCHITECTURE.md)).

### Decision Table

| Need | Pattern | Reference |
|------|---------|-----------|
| Observe another daemon's state | Observation Plugin | MeshView |
| Own business process with events | Full CQRS Plugin | Martha |
| Simple CRUD (no events) | Observation Plugin (with writes via erpc) | -- |
| Cross-node data aggregation | Observation Plugin | MeshView |

---

## Architecture

The observation plugin has a **one-way dependency** on its data source.
It reads from the source; the source knows nothing about the observer.

```
observer-daemon ──erpc:call──> source-daemon
source-daemon ──knows nothing about──> observer-daemon
```

This is achieved through BEAM clustering. Both daemons share the same
Erlang cookie and discover each other via short names on the same host.

### What You Skip

| Full CQRS Plugin | Observation Plugin |
|-------------------|--------------------|
| ReckonDB store | Not needed |
| evoq/reckon_evoq | Not needed |
| Aggregates | Not needed |
| Commands/Events | Not needed |
| Projections | Not needed |
| SQLite read models | Not needed |
| Domain apps under `apps/` | Not needed |

### What You Keep

| Component | Purpose |
|-----------|---------|
| Cowboy on Unix socket | HTTP API for frontend |
| `/health` endpoint | Liveness check |
| `/manifest` endpoint | Plugin discovery by hecate-web |
| `/ui/[...]` static route | Frontend custom element |
| Plugin registrar | Register with hecate-daemon |
| Observer gen_server | Poll source daemon via erpc |

### Dependency Comparison

| | Full CQRS (Martha) | Observation (MeshView) |
|-|---------------------|------------------------|
| cowboy | Yes | Yes |
| reckon_db | Yes | No |
| evoq | Yes | No |
| reckon_evoq | Yes | No |
| esqlite | Yes | No |
| hackney | Yes | No |
| **Total deps** | 6 | 1 |

---

## The Observer Gen_Server

The core of this pattern is a gen_server that:

1. **Discovers** the source daemon's BEAM node
2. **Polls** via `erpc:call/4` on a timer (e.g. every 10s)
3. **Caches** the latest snapshot in gen_server state
4. **Exposes** cached state via synchronous API calls

### Discovery

```erlang
discover_daemon(State) ->
    Hostname = hostname(),
    DaemonNode = list_to_atom("hecate@" ++ Hostname),
    case net_adm:ping(DaemonNode) of
        pong -> {ok, DaemonNode, State#{daemon_node => DaemonNode}};
        pang -> {error, daemon_unreachable, State}
    end.
```

The observer resolves the daemon node name from its own hostname.
Both run on the same host with the same cookie, so `net_adm:ping/1`
establishes the connection.

### Polling via erpc

```erlang
safe_erpc(Node, Mod, Fun, Args) ->
    try
        erpc:call(Node, Mod, Fun, Args, 5000)
    catch
        error:{erpc, Reason} ->
            {error, Reason};
        Class:Reason ->
            {error, {Class, Reason}}
    end.
```

**Why erpc, not rpc:**
- `erpc:call/4` raises on timeout (catchable)
- `rpc:call/4` returns `{badrpc, timeout}` (easy to miss)
- `erpc` is the modern replacement (OTP 23+)

**Why poll, not subscribe:**
- Observer is read-only -- it does not need real-time push
- Polling is simpler and creates no coupling in the source
- The source daemon does not need any code to support the observer
- 10s poll interval is fine for topology/status dashboards

### Graceful Degradation

The observer NEVER crashes when the source is unavailable:

- If daemon node is unreachable, cache shows `daemon_connected: false`
- If erpc call fails, keep the last known state
- Retry discovery on the next poll interval
- Frontend shows "disconnected" state -- no error dialog

This means the observer can start before the source daemon,
survive source restarts, and handle network partitions.

---

## Directory Layout

Observation plugins have a minimal namespace:

```
~/.hecate/hecate-app-meshviewd/
    sockets/                    # Unix domain socket
        api.sock
    run/                        # PID and state files
```

No `sqlite/`, no `reckon-db/`, no `connectors/` -- they have no state
to persist. The entire state is a cached snapshot in gen_server memory.

### Paths Module

```erlang
-module(app_meshviewd_paths).
-export([base_dir/0, socket_dir/0, socket_path/1, run_dir/0, ensure_layout/0]).

%% Only two subdirectories needed
ensure_layout() ->
    lists:foreach(fun(Dir) -> ok = filelib:ensure_path(Dir) end,
                  [socket_dir(), run_dir()]).
```

---

## Frontend Pattern

The frontend follows the same custom element pattern as all plugins:

```
meshvieww/
    src/lib/
        MeshviewStudio.svelte       # <meshview-studio> custom element
        types.ts                    # PluginApi + domain types
        shared/api.ts               # setApi/getApi pattern
        mesh_status/                # Vertical slice
            mesh_status.ts          # Reactive store
            MeshStatus.svelte       # UI component
```

The store polls the daemon's `/api/mesh/status` endpoint.
The daemon returns the cached snapshot from the observer gen_server.
No WebSocket, no SSE -- just periodic fetch.

---

## Supervision Tree

```
hecate_app_meshviewd_sup (one_for_one)
    app_meshviewd_plugin_registrar (transient)
    mesh_observer (permanent)
```

Only two children. Compare to Martha which has a plugin registrar
plus multiple domain app supervisors.

---

## Repo Structure

```
hecate-app-meshview/
    .github/workflows/
        ci.yml                      # Frontend check + daemon compile/eunit/dialyzer
        docker.yml                  # Build + push to ghcr.io
    Dockerfile                      # 3-stage: node -> erlang -> alpine
    scripts/
        rebuild.sh                  # Local dev: build frontend + compile daemon
        bump-version.sh             # Bump version across all artifacts

    hecate-app-meshviewd/           # Daemon (Erlang/OTP)
        rebar.config                # deps: cowboy only
        config/
            sys.config
            vm.args                 # -sname, -setcookie hecate_cookie
        src/
            hecate_app_meshviewd.app.src
            hecate_app_meshviewd_app.erl
            hecate_app_meshviewd_sup.erl
            app_meshviewd_paths.erl
            app_meshviewd_health_api.erl
            app_meshviewd_manifest_api.erl
            app_meshviewd_api_utils.erl
            app_meshviewd_plugin_registrar.erl
            mesh_observer.erl       # The novel part
            mesh_status_api.erl
        priv/static/.gitkeep

    hecate-app-meshvieww/           # Frontend (SvelteKit)
        package.json
        vite.config.ts
        vite.config.lib.ts
        svelte.config.js
        tsconfig.json
        src/
            app.html
            app.css
            lib/
                MeshviewStudio.svelte
                types.ts
                shared/api.ts
                mesh_status/
                    mesh_status.ts
                    MeshStatus.svelte
            routes/
                +layout.ts
                +layout.svelte
                +page.svelte
```

---

## Checklist: Scaffolding an Observation Plugin

- [ ] Create repo with daemon (`*d`) and frontend (`*w`) directories
- [ ] `rebar.config` with cowboy as only dependency
- [ ] `vm.args` with `-setcookie hecate_cookie` (same as hecate-daemon)
- [ ] App module: `ensure_layout` then `start_cowboy` (no store creation)
- [ ] Supervisor with plugin_registrar (transient) + observer (permanent)
- [ ] Observer gen_server: discover, poll, cache, expose
- [ ] API endpoint returning cached snapshot
- [ ] Paths module with only `sockets/` and `run/`
- [ ] Frontend custom element with `setApi/getApi` pattern
- [ ] Dockerfile: 3-stage (node, erlang, alpine)
- [ ] CI/CD workflows (ci.yml + docker.yml)
- [ ] `scripts/bump-version.sh` for cross-artifact version sync

---

## Security Considerations

Observation plugins are the primary candidate for the **untrusted** trust
tier. They are read-only by design and should not need BEAM-level access
once the session-scoped cookie system is implemented.

Until then, they share `hecate_cookie` and use erpc for polling. When
session cookies arrive, observation plugins will switch to HTTP polling
against the daemon's REST API (e.g. `GET /api/mesh/status`) with a
session-scoped cookie for authentication.

See [PLUGIN_SECURITY_MODEL.md](../philosophy/PLUGIN_SECURITY_MODEL.md)
for the full tiered trust model.

---

## When to Graduate

If the observation plugin grows to need:

- **Persisted state** -- add SQLite, add `sqlite/` to paths
- **Business events** -- add ReckonDB + evoq, switch to full CQRS
- **Write operations** -- dispatch commands via erpc to the source daemon
  (still one-way dependency -- source processes commands, observer proxies)

Start simple. Graduate when the complexity is justified.
