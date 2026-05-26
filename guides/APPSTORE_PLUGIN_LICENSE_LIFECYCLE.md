---
title: Appstore License Lifecycle
layer: guide
audience: [agent, human]
stage: stable
---

# Appstore License Lifecycle

_Reference architecture for the hecate-app-appstore plugin._

---

## Overview

The appstore manages **licenses** through a seller/buyer lifecycle.
The catalog is populated by events — NOT hardcoded seed data.

**hecate-app-appstore replaces hecate-marketplace.** The `hecate-app-*` convention
is the standard for all plugins.

---

## Bounded Context Split (Decision: 2026-02-24)

The plugin lifecycle spans **two bounded contexts** with distinct responsibilities:

| Bounded Context | Repo | Aggregate | Owns |
|----------------|------|-----------|------|
| **Appstore** | `hecate-app-appstore` | `license` | Commercial lifecycle — licensing, trading |
| **Node** | `hecate-app-node` | `plugin` (TBD) | Operational lifecycle — installing, upgrading, removing artifacts |

### Why the Split

There is a 3-fold tension in the domain:

1. **Appstore** — how the user experiences it (browse, search, buy)
2. **License** — the link between the object (plugin) and the subject (license to use it)
3. **License vs Plugin** — the license is what is *traded*, the plugin (app) is what is *installed*

The verbs make this obvious:

| Verb | Acts on | Bounded context |
|------|---------|----------------|
| initiate, announce, publish | license (the product listing) | Appstore |
| buy, revoke, archive | license (the commercial contract) | Appstore |
| install, upgrade, remove | plugin (the running artifact) | Node |

Forcing operational verbs (install/upgrade/remove) into the license aggregate
makes the names dishonest: "install license" doesn't scream intent,
"install plugin" does.

### Why `license` (not `plugin_license`)

Within the appstore bounded context, "license" is unambiguous — the store only
sells plugin licenses. The `plugin_` qualifier answers "license for what?" but
within this context, the answer is always "a plugin."

The buyer-side naturally speaks of licenses (buy a license, revoke a license).
The seller-side naturally speaks of licenses too (initiate a license offering,
publish a license). No disambiguation needed.

If other license types emerge, they would live in different bounded contexts.

### Cross-Context Integration

The appstore and node domains communicate via **process managers**, not direct dispatch:

```
BUYING TRIGGERS INSTALL:

  Appstore                              Node
  ─────────────────────────             ─────────────────────────
  buy_license_v1
  → license_bought_v1
  → license_bought_v1_to_pg ──────────→ PM: on_license_bought_v1_
                                             maybe_install_plugin
                                        → install_plugin_v1
                                        → plugin_installed_v1

REVOKING TRIGGERS REMOVAL:

  Appstore                              Node
  ─────────────────────────             ─────────────────────────
  revoke_license_v1
  → license_revoked_v1
  → license_revoked_v1_to_pg ─────────→ PM: on_license_revoked_v1_
                                             maybe_remove_plugin
                                        → remove_plugin_v1
                                        → plugin_removed_v1
```

The PMs live in the **node domain** (the target). They subscribe to appstore
events via pg and dispatch node commands.

---

## Aggregate: license_aggregate

The appstore aggregate tracks the **commercial lifecycle** only.

### Status Bit Flags

| Flag | Value | Meaning |
|------|-------|---------|
| `LIC_INITIATED` | 1 (2^0) | Dossier born |
| `LIC_ANNOUNCED` | 2 (2^1) | Pre-published, visible to seller |
| `LIC_PUBLISHED` | 4 (2^2) | Available in catalog |
| `LIC_LICENSED` | 8 (2^3) | Buyer has purchased a license |
| `LIC_REVOKED` | 16 (2^4) | License revoked |
| `LIC_ARCHIVED` | 32 (2^5) | Soft-deleted (walking skeleton) |

Note: `INSTALLED` and `OUTDATED` are **not** appstore concerns — they belong
to the node domain's plugin aggregate.

### Full Event Lifecycle

```
SELLER SIDE:
  initiate_license  → license_initiated_v1   (birth event)
  announce_license  → license_announced_v1   (pre-publish, marketing)
  publish_license   → license_published_v1   (available for purchase)

BUYER SIDE (commercial only):
  buy_license       → license_bought_v1
  revoke_license    → license_revoked_v1
  archive_license   → license_archived_v1
```

### Seller Desks (CMD)

| Desk | Command | Event | Entry Point |
|------|---------|-------|-------------|
| `initiate_license/` | `initiate_license_v1` | `license_initiated_v1` | API |
| `announce_license/` | `announce_license_v1` | `license_announced_v1` | API |
| `publish_license/` | `publish_license_v1` | `license_published_v1` | API + Mesh Emitter |

### Buyer Desks (CMD — commercial)

| Desk | Command | Event | Entry Point |
|------|---------|-------|-------------|
| `buy_license/` | `buy_license_v1` | `license_bought_v1` | API |
| `revoke_license/` | `revoke_license_v1` | `license_revoked_v1` | API |
| `archive_license/` | `archive_license_v1` | `license_archived_v1` | API |

---

## Catalog Population — Two Sources

The catalog read model is populated from two entry points:

### 1. Local Seller Publishes

```
Seller fills form → initiate_license → ... → publish_license
  → license_published_v1 (event in local store)
  → Projection writes to catalog SQLite
  → Mesh Emitter publishes FACT to mesh
```

### 2. Remote Seller Publishes (via Mesh)

```
FACT arrives from mesh (license_published)
  → Listener: on_license_published_maybe_register_catalog_entry
  → Command: register_catalog_entry_v1
  → Event: catalog_entry_registered_v1
  → Projection writes to catalog SQLite
```

Both paths produce events in the local store, projected to the same catalog read model.
The catalog does not distinguish local vs remote — all published plugins appear the same.

---

## App Structure

```
hecate-app-appstored/
├── apps/
│   ├── guide_license_lifecycle/       # CMD — seller + buyer (commercial)
│   │   ├── include/
│   │   │   ├── license_state.hrl
│   │   │   └── license_status.hrl
│   │   └── src/
│   │       ├── license_aggregate.erl
│   │       ├── initiate_license/          # Seller — birth
│   │       ├── announce_license/          # Seller — pre-publish
│   │       ├── publish_license/           # Seller — catalog + mesh
│   │       ├── buy_license/               # Buyer — purchase
│   │       ├── revoke_license/            # Buyer — revoke
│   │       └── archive_license/           # Buyer — soft-delete
│   │
│   ├── project_appstore/              # PRJ — projections + SQLite store
│   │   └── src/
│   │       ├── project_appstore_store.erl
│   │       ├── license_initiated_v1_to_catalog/
│   │       ├── license_announced_v1_to_catalog/
│   │       ├── license_published_v1_to_catalog/
│   │       ├── license_bought_v1_to_licenses/
│   │       ├── license_revoked_v1_to_licenses/
│   │       └── license_archived_v1_to_licenses/
│   │
│   └── query_appstore/                # QRY — API handlers, reads only
│       └── src/
│           ├── browse_catalog/
│           ├── get_plugin_details/
│           ├── list_installed/
│           └── list_licenses/
```

---

## Node Domain (hecate-app-node) — Operational Lifecycle

**Repo:** `hecate-social/hecate-app-node` (to be created)

The node domain owns plugin **installation and runtime** on a specific host.

### Aggregate: plugin_aggregate (TBD)

```
OPERATIONAL:
  install_plugin     → plugin_installed_v1
  upgrade_plugin     → plugin_upgraded_v1
  remove_plugin      → plugin_removed_v1
```

### Process Managers (react to appstore events)

| PM | Trigger (from appstore via pg) | Dispatches |
|----|-------------------------------|------------|
| `on_license_bought_v1_maybe_install_plugin` | `license_bought_v1` | `install_plugin_v1` |
| `on_license_revoked_v1_maybe_remove_plugin` | `license_revoked_v1` | `remove_plugin_v1` |

### Side Effects (react to own events)

| PM | Trigger | Action |
|----|---------|--------|
| `on_plugin_installed_v1_provision_container` | `plugin_installed_v1` | Pull OCI image, create Quadlet unit |
| `on_plugin_upgraded_v1_update_container` | `plugin_upgraded_v1` | Update OCI image tag, restart |
| `on_plugin_removed_v1_deprovision_container` | `plugin_removed_v1` | Remove Quadlet unit, stop container |

---

_The appstore trades licenses. The node installs plugins. Process managers connect them._
