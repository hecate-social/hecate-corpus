---
title: Plugin Scaffold Guide
layer: codegen
audience: [codegen]
stage: stable
---

# Plugin Scaffold Guide

How to create a new Hecate plugin from scratch. Reference implementation: `hecate-apps/hecate-app-snake-duel`.

---

## Repository Structure

Every plugin repo follows this layout:

```
hecate-app-{name}/
├── manifest.json                    # Single source of truth for metadata
├── CHANGELOG.md
├── LICENSE
├── README.md
├── .github/workflows/
│   ├── ci.yml                       # Compile + test + dialyzer
│   └── release.yml                  # Build tarball on git tags
├── hecate-app-{name}d/              # Erlang daemon
│   ├── rebar.config
│   ├── src/                         # Root OTP app
│   │   ├── hecate_app_{name}d.app.src
│   │   ├── hecate_app_{name}d_app.erl
│   │   ├── hecate_app_{name}d_sup.erl
│   │   ├── app_{name}.erl          # Callback module (hecate_plugin behaviour)
│   │   ├── app_{name}_sup.erl      # Real supervision tree
│   │   └── app_{name}d_api_utils.erl
│   ├── apps/                        # Domain apps (vertical slices)
│   │   ├── run_{name}/              # CMD app
│   │   └── query_{name}/            # QRY app (optional)
│   ├── priv/static/                 # Frontend assets (populated by CI)
│   └── scripts/
│       └── package.sh               # Plugin tarball builder
└── hecate-app-{name}w/              # Svelte 5 frontend
    ├── package.json
    ├── svelte.config.js
    ├── vite.config.ts
    ├── vite.config.lib.ts
    └── src/
        └── lib/
            └── {Name}Studio.svelte  # Root custom element
```

---

## Step 1: manifest.json

The single source of truth. Lives at repo root, copied into tarball by CI.

```json
{
  "name": "hecate-app-{name}",
  "display_name": "{Human Name}",
  "version": "0.1.0",
  "icon": "{emoji_shortcode}",
  "description": "{One-line description}",
  "tag": "{name}-studio",
  "callback_module": "app_{name}",
  "plugin_type": "in_vm",
  "trust_level": "untrusted",
  "permissions": [],
  "repository": "https://github.com/hecate-apps/hecate-app-{name}",
  "license": "Apache-2.0",
  "min_daemon_version": "0.14.0",
  "appstore": {
    "plugin_name": "{Human Name}",
    "org": "hecate-apps",
    "homepage": "https://github.com/hecate-apps/hecate-app-{name}",
    "tags": "{comma,separated,tags}",
    "license_type": "free",
    "fee_cents": 0,
    "fee_currency": "EUR",
    "node_limit": 0,
    "group_name": "{GROUP}",
    "group_icon": "{emoji_shortcode}"
  }
}
```

**Critical fields:**
- `name` = technical routing name (lowercase, hyphenated). Used for API paths: `/plugin/{name}/api/...`
- `display_name` = human-readable label. Shown in UI. NEVER used for routing.
- `version` = MUST match git tags. See antipatterns/plugin.md Demon #1.
- `callback_module` = Erlang module implementing `hecate_plugin` behaviour
- `plugin_type` = `"in_vm"` for BEAM plugins, `"container"` for OCI plugins
- `tag` = custom element tag name for the frontend component

---

## Step 2: Callback Module

The entry point. Implements `hecate_plugin` behaviour.

```erlang
-module(app_{name}).
-behaviour(hecate_plugin).
-include_lib("hecate_sdk/include/hecate_plugin.hrl").
-export([init/1, routes/0, store_config/0, static_dir/0, manifest/0, flag_maps/0]).

init(#{plugin_name := PluginName, data_dir := DataDir} = Config) ->
    logger:info("[app-{name}] Initializing ~s (data: ~s)", [PluginName, DataDir]),
    StoreId = maps:get(store_id, Config, none),
    persistent_term:put(app_{name}_config, #{
        plugin_name => PluginName,
        store_id => StoreId,
        data_dir => DataDir
    }),
    case app_{name}_sup:start_link() of
        {ok, Pid} -> {ok, #{sup_pid => Pid}};
        {error, {already_started, Pid}} -> {ok, #{sup_pid => Pid}};
        {error, Reason} -> {error, Reason}
    end.

routes() ->
    [
        %% {Path, Handler, Opts} — mounted under /plugin/{name}/api/
        %% e.g. {"/items", items_api, []} becomes /plugin/hecate-app-{name}/api/items
    ].

store_config() ->
    %% Return `none` if no event store needed (ephemeral plugin)
    %% Or configure a ReckonDB store:
    %% #hecate_store_config{
    %%     store_id = {name}_store,
    %%     dir_name = "{name}",
    %%     description = "{Name} event store"
    %% }.
    none.

static_dir() -> "priv/static".

manifest() ->
    #{
        name => <<"hecate-app-{name}">>,
        display_name => <<"{Human Name}">>,
        version => <<"0.1.0">>,
        description => <<"{Description}">>,
        icon => <<"{emoji}">>,
        tag => <<"{name}-studio">>,
        min_sdk_version => <<"0.1.0">>
    }.

flag_maps() -> #{}.
```

---

## Step 3: rebar.config

```erlang
{erl_opts, [debug_info, warnings_as_errors]}.

{deps, [
    {cowboy, "2.12.0"},
    {esqlite, "0.8.8"},       %% Only if using SQLite read models
    {hecate_sdk, "~> 0.1"}
]}.

{relx, [
    {release, {hecate_app_{name}d, "0.1.0"}, [
        hecate_app_{name}d,    %% Root app FIRST
        run_{name},            %% CMD app
        query_{name},          %% QRY app (if exists)
        sasl
    ]},
    {mode, dev},
    {dev_mode, true},
    {include_erts, false},
    {extended_start_script, true}
]}.

{profiles, [
    {prod, [{relx, [{mode, prod}, {dev_mode, false}, {include_erts, true}]}]},
    {test, [{deps, [{meck, "0.9.2"}]}]}
]}.

{project_plugins, [rebar3_ex_doc]}.
{ex_doc, [
    {source_url, <<"https://github.com/hecate-apps/hecate-app-{name}">>},
    {extras, [<<"README.md">>]},
    {main, <<"README.md">>}
]}.
{hex, [{doc, ex_doc}]}.

{dialyzer, [
    {warnings, [error_handling]},
    {plt_extra_apps, [cowboy, cowlib, ranch, esqlite, hecate_sdk]},
    {exclude_apps, [reckon_db, reckon_gater, khepri, ra, horus, aten, seshat, gen_batch_server]}
]}.
```

---

## Step 4: package.sh

Consolidates all .beam files + frontend + manifest into a flat tarball.

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
ROOT_DIR="$(cd "$SCRIPT_DIR/.." && pwd)"
BUILD_DIR="$ROOT_DIR/_build/plugin"
STAGING_DIR="$BUILD_DIR/staging"
PLUGIN_NAME="hecate-app-{name}"

echo "==> Compiling..."
cd "$ROOT_DIR"
rebar3 compile

echo "==> Preparing package..."
rm -rf "$STAGING_DIR"
mkdir -p "$STAGING_DIR/ebin"

## Consolidate all .beam files from root app + domain apps
## UPDATE THIS LIST when adding new umbrella apps
for ebin_dir in \
    "$ROOT_DIR/_build/default/lib/hecate_app_{name}d/ebin" \
    "$ROOT_DIR/_build/default/lib/run_{name}/ebin" \
    "$ROOT_DIR/_build/default/lib/query_{name}/ebin"; do
    if [ -d "$ebin_dir" ]; then
        cp "$ebin_dir"/*.beam "$STAGING_DIR/ebin/" 2>/dev/null || true
    fi
done

echo "  $(find "$STAGING_DIR/ebin" -name '*.beam' | wc -l) .beam files"

## Copy static assets
STATIC_DIR="$ROOT_DIR/priv/static"
if [ -d "$STATIC_DIR" ]; then
    mkdir -p "$STAGING_DIR/priv"
    cp -r "$STATIC_DIR" "$STAGING_DIR/priv/static"
    echo "  Static assets included"
else
    echo "  No static assets (run frontend build first)"
fi

## Copy manifest.json from repo root (single source of truth)
MANIFEST_SRC="$ROOT_DIR/../manifest.json"
if [ ! -f "$MANIFEST_SRC" ]; then
    echo "ERROR: manifest.json not found at $MANIFEST_SRC"
    exit 1
fi
cp "$MANIFEST_SRC" "$STAGING_DIR/manifest.json"

## Create tarball (flat structure)
cd "$STAGING_DIR"
tar czf "$BUILD_DIR/$PLUGIN_NAME.tar.gz" .

echo "==> Package created: $BUILD_DIR/$PLUGIN_NAME.tar.gz"
```

---

## Step 5: CI/CD Workflows

### ci.yml

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  frontend:
    name: Frontend (check + build)
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: hecate-app-{name}w
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
          cache-dependency-path: hecate-app-{name}w/package-lock.json
      - run: npm ci
      - run: npm run check
      - run: npm run build:lib
      - uses: actions/upload-artifact@v4
        with:
          name: frontend-dist
          path: hecate-app-{name}w/dist/
          retention-days: 1

  daemon:
    name: Daemon (compile + test)
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: hecate-app-{name}d
    steps:
      - uses: actions/checkout@v4
      - uses: erlef/setup-beam@v1
        with:
          otp-version: "27"
          rebar3-version: "3.24"
      - name: Cache rebar3 deps
        uses: actions/cache@v4
        with:
          path: |
            hecate-app-{name}d/_build/default/lib
            hecate-app-{name}d/_build/default/plugins
          key: rebar3-${{ runner.os }}-${{ hashFiles('hecate-app-{name}d/rebar.config') }}
          restore-keys: rebar3-${{ runner.os }}-
      - run: rebar3 compile
      - run: rebar3 eunit
      - run: rebar3 dialyzer
```

### release.yml

```yaml
name: Release
on:
  push:
    tags: ["v*"]

permissions:
  contents: write

jobs:
  package:
    name: Build plugin package
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: erlef/setup-beam@v1
        with:
          otp-version: "27"
          rebar3-version: "3.24"
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
          cache-dependency-path: hecate-app-{name}w/package-lock.json
      - name: Build frontend
        working-directory: hecate-app-{name}w
        run: npm ci && npx svelte-kit sync && npm run build:lib
      - name: Copy frontend assets
        run: |
          mkdir -p hecate-app-{name}d/priv/static
          cp -r hecate-app-{name}w/dist/* hecate-app-{name}d/priv/static/
      - name: Build plugin tarball
        run: bash hecate-app-{name}d/scripts/package.sh
      - name: Extract version
        id: version
        run: echo "version=${GITHUB_REF_NAME#v}" >> "$GITHUB_OUTPUT"
      - name: Extract changelog entry
        id: changelog
        run: |
          VERSION="${{ steps.version.outputs.version }}"
          BODY=$(sed -n "/^## \[${VERSION}\]/,/^## \[/p" CHANGELOG.md | sed '$d')
          echo "body<<EOF" >> "$GITHUB_OUTPUT"
          echo "${BODY}" >> "$GITHUB_OUTPUT"
          echo "EOF" >> "$GITHUB_OUTPUT"
      - uses: softprops/action-gh-release@v2
        with:
          body: ${{ steps.changelog.outputs.body }}
          generate_release_notes: false
          files: hecate-app-{name}d/_build/plugin/hecate-app-{name}.tar.gz
```

---

## Step 6: Frontend (Svelte 5)

### vite.config.lib.ts

Builds the frontend as a library (single JS file for the custom element):

```typescript
import { defineConfig } from 'vite';
import { svelte } from '@sveltejs/vite-plugin-svelte';
import tailwindcss from '@tailwindcss/vite';

export default defineConfig({
  plugins: [
    svelte({
      compilerOptions: {
        customElement: true
      }
    }),
    tailwindcss()
  ],
  build: {
    lib: {
      entry: 'src/lib/index.ts',
      formats: ['es'],
      fileName: 'component'
    },
    outDir: 'dist',
    emptyOutDir: true,
    rollupOptions: {
      output: {
        inlineDynamicImports: true
      }
    }
  }
});
```

### Custom element tag

The root Svelte component must register as a custom element with a tag matching `manifest.json`'s `tag` field:

```svelte
<svelte:options customElement="{name}-studio" />
```

The daemon host page loads the plugin via:
```html
<{name}-studio name="hecate-app-{name}"></{name}-studio>
```

---

## Step 7: Frontend API Communication

Plugins communicate with their daemon backend via the hecate-web proxy. All API calls go through the daemon's Unix socket.

### API path convention

Plugin routes are mounted at `/plugin/{name}/api/`. The frontend must use **relative paths**:

```typescript
// In the plugin's API client
async function apiCall(path: string, opts?: RequestInit) {
    const name = getPluginName();  // from custom element prop
    const resp = await fetch(`/plugin/${name}/api${path}`, opts);
    return resp.json();
}
```

### SSE (Server-Sent Events)

For real-time streaming (games, logs, status):

```typescript
function connectSSE(path: string): EventSource {
    const name = getPluginName();
    return new EventSource(`/plugin/${name}/api${path}`);
}
```

---

## Plugin Complexity Tiers

| Tier | Event Store | SQLite | Frontend | Example |
|------|-------------|--------|----------|---------|
| **Observation** | No | No | Read-only | MeshView |
| **Ephemeral** | No | Yes (history) | Interactive | Snake Duel |
| **Full CQRS** | Yes (ReckonDB) | Yes (read models) | Full app | Martha (ALC) |

Choose the simplest tier that meets your needs.

---

## Release Checklist

1. [ ] Bump version in `manifest.json`
2. [ ] Update `CHANGELOG.md` with `## [{version}]` entry
3. [ ] Commit: `chore: Release v{X.Y.Z}`
4. [ ] Tag: `git tag v{X.Y.Z}`
5. [ ] Push: `git push && git push --tags`
6. [ ] Verify CI builds green
7. [ ] Verify release tarball published on GitHub

**See `skills/antipatterns/plugin.md` for common pitfalls.**
