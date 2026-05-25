# ANTIPATTERNS: Release, Testing & Packaging

*Demons about releases, versioning, hex packaging, testing, and plugin discovery.*

[Back to Index](ANTIPATTERNS.md)

---

## 🔥 Demon 27: Hardcoded User/Submitter IDs

**Date exorcised:** 2026-02-23
**Where it appeared:** hecate-app-appstored command modules
**Cost:** Every event in the store records the wrong actor — audit trail is useless

### The Lie

"Just use `<<"system">>` or a placeholder for the user ID — we'll fix it later."

### What Happened

Command modules hardcoded the submitter identity:

```erlang
%% WRONG — who actually did this?
Cmd = buy_license_v1:new(#{
    license_id => LicenseId,
    plugin_id => PluginId,
    user_id => <<"system">>   %% Hardcoded placeholder
}),
```

Every event stored in ReckonDB records `<<"system">>` as the actor. The audit trail — "who did what, when" — is destroyed. In an event-sourced system, events are immutable. You cannot retroactively fix the actor identity.

### Why It's Wrong

1. **Audit trail destroyed** — Event sourcing's primary value is a complete, truthful history. Hardcoded actors make the history a lie.
2. **Immutable damage** — Events cannot be amended. Once stored with `<<"system">>`, that event will always say "system did it."
3. **Security blind spot** — No way to trace actions back to actual users for access control, debugging, or compliance.
4. **Multi-user broken** — When two users buy licenses, both events say "system" — indistinguishable.

### The Rule

> **Commands MUST carry the real actor identity. The API handler extracts the user from the request context and passes it through to the command.**

### The Correct Pattern

```erlang
%% API handler extracts identity from request
handle_post(Req0, State) ->
    UserId = extract_user_id(Req0),  %% From auth token, session, etc.
    {ok, Params, Req1} = app_api_utils:read_json_body(Req0),
    Cmd = buy_license_v1:new(#{
        license_id => maps:get(<<"license_id">>, Params),
        plugin_id => maps:get(<<"plugin_id">>, Params),
        user_id => UserId   %% Real actor identity
    }),
    ...
```

For commands triggered by Policies or Listeners (no HTTP request), the actor is the **system process** that initiated the action — record it explicitly:

```erlang
%% Policy — actor is the policy itself
Cmd = remove_plugin_v1:new(#{
    license_id => LicenseId,
    initiated_by => <<"policy:on_license_revoked_v1_maybe_remove_plugin">>
}),
```

### Prevention

- Every command struct MUST have a `submitter_id` or `initiated_by` field
- API handlers MUST extract identity from request context
- Policies/Listeners MUST identify themselves as the actor
- Code review: reject any command with hardcoded `<<"system">>`, `<<"admin">>`, or `<<>>`

### The Lesson

> **Events are immutable history. Hardcoded actor IDs destroy that history permanently.**
> **The identity flows from the entry point (API, Policy, Listener) into the command. No exceptions.**

---

## 🔥 Demon 28: No Tests on Event-Sourced Domains

**Date exorcised:** 2026-02-23
**Where it appeared:** hecate-app-appstored — 0 tests across 6 CMD desks, 5 projections, 4 query handlers, 4 policies
**Cost:** Bugs found only by dialyzer or at runtime — no safety net for refactoring

### The Lie

"Dialyzer catches type errors, so we don't need tests."

### What Happened

The appstore daemon shipped with zero tests. Dialyzer caught the `row_to_map` tuple/list bug (Demon #19), but only because it was a type mismatch. Business logic bugs — wrong aggregate guards, incorrect projection SQL, broken policy chains — are invisible to dialyzer.

### What Dialyzer Cannot Catch

| Bug Type | Dialyzer? | Unit Test? |
|----------|-----------|------------|
| Wrong function argument types | Yes | Yes |
| Dead code branches | Yes | Yes |
| Wrong aggregate business rule (rejects valid command) | **No** | Yes |
| Projection writes wrong column value | **No** | Yes |
| Policy dispatches wrong command | **No** | Yes |
| Command validation too permissive | **No** | Yes |
| Event missing required field | **No** | Yes |
| Bit flag combination produces wrong status_label | **No** | Yes |

Dialyzer proves types align. Tests prove behavior is correct. Both are needed.

### Minimum Test Coverage for Event-Sourced Domains

| Component | What to Test | Priority |
|-----------|-------------|----------|
| **Aggregate** | Every command + every business rule guard | Critical |
| **Projection** | Each event type with real `#event{}` record input (Demon #23) | Critical |
| **Command struct** | `new/1` produces valid command, required fields enforced | High |
| **Event struct** | `new/N` produces valid event, `to_map/1` round-trips | High |
| **Policy** | Receives event, dispatches correct command | High |
| **Query API** | `row_to_map` works with actual esqlite3 output format | Medium |

### The Rule

> **Tests must be written BEFORE the code file (preferred) or IMMEDIATELY AFTER it. Never deferred to "before it ships" or attached to a CI/workflow gate.**

The "ship" framing is the lie. By the time something is "shipping" the code has been merged, the author has moved on, and the gap is now a backlog item that never closes. The test belongs alongside the code as a unit of work — not as a downstream chore.

### When to Write Tests

Two acceptable moments:

1. **Before** writing the `.erl` file — TDD style. Write a failing test that describes the slice's behaviour, then write the code that makes it pass. Best when the behaviour is well-understood (e.g. an aggregate guard rejecting an already-archived command).
2. **Immediately after** writing the `.erl` file — same commit, same task, same session. Acceptable when the behaviour took some exploration to land. The bar: the test file lands before the next desk/slice is started.

**Unacceptable:**

- "I'll add tests later" — later never arrives
- "Tests can come in a follow-up PR" — the PR rots in review
- "CI will run them when they exist" — a green CI on zero tests is a lie
- "Pre-commit hook blocks unverified commits" — hooks are local and bypassable, and they don't write tests for you

### Tests Are Not a Workflow Action

> **Test creation MUST NOT depend on any workflow gate — CI, pre-commit hooks, PR templates, release checklists, or any other downstream automation.**

Workflow gates check *whether tests passed*, not *whether tests exist*. A workflow can fail a build for a regression in an existing test. It cannot fail a build for a slice that ships with no test at all — because there's nothing to run, and "no tests" is indistinguishable from "all tests passed" to most green/red signals.

The discipline lives in the author's hands, in the same commit as the code. If the author skips the test, no later automation can recover the missed opportunity to verify behaviour at the moment the design decision was fresh.

### Minimum Viable Test Suite

For an aggregate with N commands:

```erlang
%% 1. Each command produces the right event
initiate_test() ->
    {ok, State} = my_aggregate:init(<<"agg-1">>),
    Cmd = #{command_type => <<"initiate_thing_v1">>, id => <<"t-1">>},
    {ok, [Event]} = my_aggregate:execute(State, Cmd),
    ?assertEqual(<<"thing_initiated_v1">>, maps:get(event_type, Event)).

%% 2. Business rules reject invalid commands
cannot_archive_already_archived_test() ->
    State = state_with_flags(?ARCHIVED),
    Cmd = #{command_type => <<"archive_thing_v1">>, id => <<"t-1">>},
    ?assertMatch({error, already_archived}, my_aggregate:execute(State, Cmd)).

%% 3. Projection handles real #event{} records
projection_with_record_test() ->
    Event = #event{
        event_type = <<"thing_initiated_v1">>,
        data = #{id => <<"t-1">>, name => <<"Test">>},
        stream_id = <<"thing-t-1">>, version = 0,
        event_id = <<"evt-1">>, metadata = #{},
        timestamp = 1000, epoch_us = 1000000
    },
    FlatMap = projection_event:to_map(Event),
    ok = thing_initiated_v1_to_sqlite_things:project(FlatMap).
```

### The Lesson

> **Dialyzer catches type bugs. Tests catch logic bugs. An event-sourced domain without tests is a domain you can't safely refactor.**
> **The appstore's tuple/list bug (Demon #19) was found by dialyzer. The next bug won't be.**
>
> **Write the test in the same session as the code, in the same commit if possible. Never push the responsibility onto CI, hooks, or "later".**

---

## 🔥 Missing `/ui/[...]` Cowboy Route in Plugin Daemon

**Demon #29** — 2026-02-24

### What Happened

The snake-duel daemon had a working `/manifest` endpoint and a healthy Unix socket. Its Dockerfile correctly built the SvelteKit frontend and copied `dist/` into `priv/static/`. But the plugin never appeared in hecate-web.

### The Bug

The cowboy route list in the daemon's `_app.erl` had no `/ui/[...]` route. The frontend assets were sitting in `priv/static/component.js` but cowboy never served them. When hecate-web fetched `/ui/component.js` through the Tauri socket proxy, it got a 404. The plugin loading code treats a 404 on the custom element as "plugin doesn't exist" and silently drops it.

### Why It's Insidious

- The daemon was running fine (health OK, manifest OK)
- The socket existed and responded to API calls
- The Dockerfile built and copied the frontend correctly
- The plugin discovery scan found the socket
- Zero errors in logs — the failure is a silent 404 in the browser

### The Fix

Every plugin daemon MUST include this route in its cowboy dispatch:

```erlang
{"/ui/[...]", cowboy_static, {dir, static_dir(), [{mimetypes, cow_mimetypes, all}]}}
```

With the helper:

```erlang
static_dir() ->
    PrivDir = code:priv_dir(my_plugin_app),
    filename:join(PrivDir, "static").
```

### Plugin Daemon Required Endpoints Checklist

| Endpoint | Purpose | Without it |
|----------|---------|-----------|
| `GET /health` | Health check | Plugin marked unhealthy |
| `GET /manifest` | Plugin metadata | Discovery fails with error |
| `GET /ui/[...]` | Frontend custom element | **Plugin silently invisible** |

### The Lesson

> **A plugin with a working daemon and manifest but no `/ui/[...]` route is invisible to hecate-web. The failure is completely silent. Always verify all three required endpoints when creating a new plugin daemon.**

---

## 🔥 Demon 30: Forgetting to Bump `.app.src` Versions Before Tagging

**Date exorcised:** 2026-02-24
**Where it appeared:** hecate-app-appstored — 4 `.app.src` files stuck at `"0.1.0"` while tagging `v0.2.0`
**Cost:** Had to delete the remote tag, bump versions, re-commit, and re-tag

### The Lie

"Just commit, tag, and push. The version takes care of itself."

### What Happened

A significant feature was implemented (schema extension, new endpoints, bug fixes), committed, tagged as `v0.2.0`, and pushed — but all 4 `.app.src` files still contained `{vsn, "0.1.0"}`. The OCI image built by CI would ship with the old version baked into the BEAM release, causing version mismatches between the git tag and the running application.

### Why It's Wrong

1. **BEAM release version comes from `.app.src`** — `application:get_key(App, vsn)` returns what's in the `.app.src`, not the git tag
2. **OCI images carry the wrong version** — Logs, health endpoints, and manifest responses report the old version
3. **Impossible to debug version mismatches** — "I deployed v0.2.0 but the daemon says 0.1.0"
4. **Tag deletion is destructive** — If CI already built on the tag, you have a phantom image with wrong metadata

### The Rule

> **When tagging a release, ALWAYS bump `{vsn, "X.Y.Z"}` in ALL `.app.src` files BEFORE committing and tagging.**

### Pre-Tag Checklist

Before running `git tag vX.Y.Z`:

1. [ ] **Root `.app.src`** — `src/{app_name}.app.src` bumped
2. [ ] **All umbrella app `.app.src` files** — `apps/*/src/*.app.src` bumped
3. [ ] **`rebar3 compile`** — still compiles clean
4. [ ] **Commit the version bump** — version change is IN the tagged commit
5. [ ] **Then tag and push**

### Where to Find `.app.src` Files

```bash
# Erlang umbrella — find all version files
grep -r '{vsn,' src/*.app.src apps/*/src/*.app.src
```

### For Other Ecosystems

| Ecosystem | Version File(s) | Same Rule |
|-----------|----------------|-----------|
| Erlang/OTP | `src/*.app.src`, `apps/*/src/*.app.src` | Yes |
| Tauri | `src-tauri/Cargo.toml` AND `src-tauri/tauri.conf.json` | Yes (see hecate-web incident) |
| Elixir | `mix.exs` | Yes |
| Node.js | `package.json` | Yes |

### The Lesson

> **The git tag is a label. The `.app.src` version is the truth. They must match.**
> **Bump versions FIRST, commit, THEN tag. Never the other way around.**

---

## 🔥 Hex Packages Without debug_info

**Date:** 2026-03-05
**Origin:** Dialyzer couldn't analyze reckon_gater beams from hex

### The Antipattern

Publishing a hex package with `no_debug_info` in the build profile:

```erlang
%% rebar.config
{profiles, [
    {prod, [
        {erl_opts, [
            no_debug_info,    %% Strips debug info from beams
            deterministic
        ]}
    ]}
]}.
```

### Why It's Wrong

Consumers need `debug_info` in beam files to run dialyzer. Without it, dialyzer reports "Could not get Core Erlang code" and silently skips the dependency, potentially missing type errors at the boundary.

### The Correct Pattern

```erlang
{prod, [
    {erl_opts, [
        debug_info,       %% KEEP for library packages
        deterministic
    ]}
]}
```

`no_debug_info` is appropriate for **release binaries** (final deployment artifacts), never for **library packages** published to hex.

**Rule:** Libraries on hex.pm MUST include `debug_info`. Only strip it from end-user release tarballs.

---

## 🔥 Containerized reckon_db With a Dynamic BEAM Node Name

**Date:** 2026-05-21
**Origin:** reckon-portal blog Division on reckon-db.org — brief 502 outage on the second redeploy

### The Antipattern

Running a containerized app that embeds **reckon_db** (Ra/Khepri) with the default release node name, `<app>@<container-id>`.

### Why It's Wrong

Ra persists the node name into its on-disk cluster state under the store's `data_dir`. The container hostname (hence the node name) changes on **every** `docker run`/recreate. So on the first boot everything works; on the next redeploy Ra reads its persisted state, sees a "leader" on a node that no longer exists, and waits for it forever:

```
[warning] Retry attempt 1 for store :blog_store after 106ms: :timeout
[warning] Retry attempt 2 ... :timeout      %% backs off to 30s, never recovers
```

The store never serves → the app hangs **unhealthy** → 502. With a persistent volume this bites on the *second* deploy (or the next watchtower auto-update), not the first.

### The Fix

Pin a stable node name in the container env:

```yaml
# docker-compose.yml — on the app service
RELEASE_DISTRIBUTION: name
RELEASE_NODE: reckon_portal@127.0.0.1   # any fixed name; IP avoids DNS
```

Persisted Ra state then stays valid across recreates: `1 record(s) recovered`, leader re-elected on the same node, healthy in ~10s.

### Why a Fresh-Volume Smoke Misses It

A container smoke that uses a **fresh** volume each run boots clean every time — there is no persisted node name to mismatch. **You must test the restart path:** `docker compose up -d --force-recreate` against an *existing* volume, and confirm it stays healthy. First-boot green is not enough for stateful stores.

### Recovery If Already Broken

The dead node name is baked into the volume. Pin the node name AND wipe the stale volume once: `docker compose rm -sf <svc>` (frees the volume), `docker volume rm <project>_<vol>`, then recreate. Safe only if there's no real data yet.

### The Rule

> **Stateful BEAM stores need a stable node identity.** If Ra/Khepri (or Mnesia) data outlives the container, the node name must too.

---

*We burned these demons so you don't have to. Keep the fire going.* 🔥🗝️🔥
