---
title: Plugin Security Model
layer: philosophy
audience: [agent, human]
stage: stable
---

# Plugin Security Model

_Tiered trust levels and session-scoped cookies for Hecate plugins._

**Date:** 2026-03-01
**Status:** Target Architecture (session cookies not yet implemented)

---

## Trust Tiers

Every plugin declares a trust level in its `manifest.json`. The daemon
enforces access based on this declaration.

| Tier | Access | Example |
|------|--------|---------|
| **core** | Full BEAM clustering, erpc, shared cookie | hecate-daemon itself |
| **trusted** | BEAM clustering with restricted erpc targets | hecate-martha (AI agent) |
| **untrusted** | HTTP only, no BEAM access, session-scoped cookie | meshview, third-party plugins |

### Why Three Tiers

- **core** is the daemon. It owns all state and exposes APIs.
- **trusted** plugins need BEAM-level access for performance (erpc polling,
  pg group membership) but should not call arbitrary modules.
- **untrusted** plugins interact only through HTTP. They cannot crash the
  daemon, leak internal state, or call arbitrary functions. This is the
  default for all new plugins.

---

## Current State: Shared Cookie

Today, all plugins share `hecate_cookie` and connect via BEAM clustering.
This means any plugin can `erpc:call/4` any function on the daemon node.

This is acceptable during early development because:
- All current plugins are first-party
- The cluster is local (same host, Unix socket)
- There is no third-party plugin ecosystem yet

---

## Target Architecture: Session-Scoped Cookies

When untrusted plugins arrive, the daemon will issue per-session Erlang
cookies. Each plugin gets a unique cookie at registration time, valid only
for that session.

### How It Works

```
Plugin starts
    |
    v
POST /api/plugins/register (over Unix socket)
    |
    v
Daemon validates manifest.json
    |
    +--> trust_level: "untrusted"
    |        |
    |        v
    |    Generate ephemeral cookie
    |    Return: { cookie: "abc123", endpoints: ["/api/mesh/status"] }
    |    Plugin uses cookie for HTTP auth (not BEAM clustering)
    |
    +--> trust_level: "trusted"
             |
             v
         Return shared cookie + allowed erpc targets
         Plugin joins BEAM cluster with restrictions
```

### Session Cookie Properties

- **Ephemeral**: Generated per plugin session, not persisted
- **Scoped**: Grants access only to declared permissions
- **Revocable**: Daemon can invalidate at any time
- **Rotatable**: Cookie changes on plugin restart

### Untrusted Plugin HTTP Flow

Untrusted plugins never join the BEAM cluster. They communicate over HTTP:

```
meshview-daemon
    |
    | GET /api/mesh/status (cookie in header)
    v
hecate-daemon (validates cookie, returns JSON)
```

No erpc, no pg groups, no shared BEAM state.

---

## manifest.json Specification

The `manifest.json` file is the plugin's public identity. It serves two
purposes: security (trust + permissions) and appstore pre-filling.

### Full Example

```json
{
  "name": "meshview",
  "version": "0.1.0",
  "icon": "🌐",
  "description": "Macula Mesh Observer",
  "tag": "meshview-studio",
  "trust_level": "untrusted",
  "permissions": ["read:mesh_status"],
  "repository": "https://github.com/hecate-apps/hecate-app-meshview",
  "license": "Apache-2.0",
  "appstore": {
    "plugin_name": "Mesh Observer",
    "org": "hecate-apps",
    "homepage": "https://github.com/hecate-apps/hecate-app-meshview",
    "tags": "mesh,network,monitoring,observation",
    "license_type": "free",
    "fee_cents": 0,
    "fee_currency": "USD",
    "node_limit": 0,
    "oci_image": "ghcr.io/hecate-apps/hecate-app-meshviewd",
    "min_daemon_version": "0.14.0"
  }
}
```

### Top-Level Fields

| Field | Required | Purpose |
|-------|----------|---------|
| `name` | Yes | Plugin identifier (unique within a daemon) |
| `version` | Yes | Semantic version (overridden at runtime by app vsn) |
| `icon` | Yes | Emoji or icon URL for UI display |
| `description` | Yes | Short description for plugin discovery |
| `tag` | Yes | Custom element tag name (e.g. `meshview-studio`) |
| `trust_level` | Yes | `core`, `trusted`, or `untrusted` |
| `permissions` | Yes | Declared capabilities the plugin needs |
| `repository` | Yes | Source code URL for auditability |
| `license` | Yes | SPDX license identifier |
| `appstore` | No | Appstore license pre-fill data (see below) |

### appstore Section

The `appstore` object pre-fills fields in the `initiate_license_v1`
command. When a seller lists this plugin in the appstore, these values
populate the license form so the seller only needs to review and confirm.

| Field | Type | Purpose |
|-------|------|---------|
| `plugin_name` | string | Display name in appstore catalog |
| `org` | string | Publisher organization |
| `homepage` | string | Plugin homepage URL |
| `tags` | string | Comma-separated search/category tags |
| `license_type` | string | `free`, `paid`, or `subscription` |
| `fee_cents` | integer | Default price in cents (0 for free) |
| `fee_currency` | string | ISO 4217 currency code |
| `duration_days` | integer | License validity in days (0 = perpetual) |
| `node_limit` | integer | Max nodes covered (0 = unlimited) |
| `oci_image` | string | OCI image URL (without tag) |
| `min_daemon_version` | string | Minimum hecate-daemon version required |

**Fields NOT in appstore section** (set by the system at runtime):
- `plugin_id` -- derived from `name`
- `seller_id` -- from authenticated seller context
- `license_id` -- auto-generated as `license-{seller_id}-{plugin_id}`
- `version` -- from top-level `version` field
- `description` -- from top-level `description` field
- `icon` -- from top-level `icon` field
- `github_repo` -- from top-level `repository` field

### Permission Strings

Permissions follow `action:resource` format:

| Permission | Grants |
|------------|--------|
| `read:mesh_status` | GET /api/mesh/status |
| `read:health` | GET /health |
| `read:llm_status` | GET /api/llm/status |
| `write:rpc_call` | POST /api/rpc/call |

The daemon checks declared permissions against the session cookie scope
before allowing access to each endpoint.

---

## Migration Path

1. **Now**: Shared cookie, all plugins trusted, HTTP endpoints open
2. **Next**: manifest.json in all plugins (auditability without enforcement)
3. **Later**: Session cookies, permission checking, untrusted tier enforced

Step 2 is happening now. Steps 1 and 3 coexist during transition --
plugins that declare `untrusted` still use the shared cookie until the
session cookie system is built.

---

## What This Does NOT Cover

- **Code signing**: Verifying plugin binaries are authentic
- **Sandboxing**: OS-level isolation (containers already provide this)
- **Network policies**: Firewall rules between plugin containers
- **Audit logging**: Recording which plugins accessed which endpoints

These are future concerns, not current priorities.
