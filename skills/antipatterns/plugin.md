---
title: Plugin Antipatterns
layer: skill
audience: [agent, human]
stage: stable
---

# Plugin Antipatterns

Demons exorcised from real plugin development. Every entry here caused a real bug.

---

## Demon #1: Manifest Version Drift

**Symptom:** Plugin re-publish downloads old tarball. New code never reaches the daemon.

**Root cause:** `manifest.json` version was not bumped before tagging. The daemon derives `package_url` from the manifest version:

```
https://github.com/{org}/{repo}/releases/download/v{VERSION}/{name}.tar.gz
```

If `manifest.json` says `0.2.2` but the git tag is `v0.3.0`, the daemon constructs a URL pointing at the OLD `v0.2.2` tarball.

**Fix:** The version in `manifest.json` is the single source of truth. ALWAYS bump it before tagging.

**Release checklist:**
1. Bump `manifest.json` version
2. Commit: `chore: Release v{X.Y.Z}`
3. Tag: `git tag v{X.Y.Z}`
4. Push: `git push && git push --tags`

**Never tag without bumping the manifest first.**

---

## Demon #2: Stale Tarball Cache

**Symptom:** Plugin update not reflected after re-install.

**Root cause:** `maybe_extract_plugin_package.erl` skips download if the tarball file already exists on disk:

```erlang
ensure_tarball(_Url, TarPath, true) ->
    logger:info("[extract] Tarball already present: ~s", [TarPath]),
    ok;
```

If the filename is the same (e.g., `hecate-app-snake-duel.tar.gz`), the old tarball is reused even though the URL points to a new release.

**Mitigation:** Delete `~/.hecate*/plugins/{name}/` before re-installing. Or use `--clear` flag on `dev-all.sh`.

**Future fix:** Include version or checksum in tarball filename, or always re-download when version changes.

---

## Demon #3: name vs display_name Contamination

**Symptom:** Plugin API routing breaks — daemon logs show `plugin=Snake Duel` instead of `plugin=hecate-app-snake-duel` in request paths.

**Root cause:** The `publish_from_url_api.erl` stored the human-readable `display_name` from the manifest as `plugin_name`, which propagated through the entire offering/license/plugin event chain. The process manager `on_license_granted_install_plugin` used `plugin_name` to derive the routing `name`.

**The rule:** `name` = technical routing name (e.g., `hecate-app-snake-duel`). `display_name` = human-readable label (e.g., `Snake Duel`). These are ALWAYS separate fields. Never conflate them.

**In manifest.json:**
```json
{
  "name": "hecate-app-snake-duel",
  "display_name": "Snake Duel"
}
```

**In events/commands:** Both fields must propagate independently through the full CQRS chain.

---

## Demon #4: Frontend Served from Wrong Path

**Symptom:** Custom element loads but API calls return 404/505.

**Root cause:** The plugin web component used the `name` prop (which was the display name from the sidebar store) to construct API URLs like `/plugin/Snake Duel/api/...`. The daemon expects `/plugin/hecate-app-snake-duel/api/...`.

**The rule:** Frontend components must derive the technical routing name from `plugin_id`, never from the display name.

```typescript
// Derive technical name from plugin_id
const id = plugin.plugin_id;  // "hecate-apps/hecate-app-snake-duel"
const techName = id.substring(id.lastIndexOf('/') + 1);  // "hecate-app-snake-duel"
```

---

## Demon #5: Hardcoded Plugin Names in CI

**Symptom:** Package script breaks when app names change, or packages wrong set of .beam files.

**Root cause:** `package.sh` and `release.yml` hardcode app directory names for .beam file consolidation.

**The rule:** When adding or renaming apps in the umbrella, update ALL of:
1. `package.sh` — beam file consolidation loop
2. `release.yml` — the package step's cp commands
3. `rebar.config` — relx release app list

---

## Demon #6: Missing `withGlobalTauri` for Plugin Web Components

**Symptom:** Plugin web component throws `Tauri API not available` when loaded inside hecate-web.

**Root cause:** Tauri v2 only exposes `window.__TAURI__` when `withGlobalTauri: true` is set in `tauri.conf.json`. Plugin web components loaded as custom elements need this global.

**Fix:** In hecate-web's `src-tauri/tauri.conf.json`:
```json
{
  "app": {
    "withGlobalTauri": true
  }
}
```

This is a one-time fix in hecate-web, not per-plugin.

---

## Prevention Checklist

Before releasing any plugin version:

- [ ] `manifest.json` version matches the tag you're about to create
- [ ] `name` field is the technical name (lowercase, hyphenated)
- [ ] `display_name` field is the human-readable name
- [ ] `callback_module` matches the actual module name in code
- [ ] Frontend build outputs to the correct location (`priv/static/`)
- [ ] All umbrella apps are listed in `package.sh` and `release.yml`
- [ ] Tarball contains: `ebin/*.beam`, `priv/static/*`, `manifest.json`
