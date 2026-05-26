# NAMING_CONVENTIONS.md — Consolidated Quick Reference

_Single-page reference for all naming conventions in the Venture/Division/Desk architecture._
_An LLM doing TnI codegen reads ONLY this file + the relevant template._

---

## Module Naming Rules (All Desk Types)

| Component | Pattern | Example |
|-----------|---------|---------|
| **CMD app** | `{verb}_{aggregate_plural}` | `process_orders` |
| **PRJ app** | `project_{read_model_plural}` | `project_orders` |
| **QRY app** | `query_{read_model_plural}` | `query_orders` |
| **CMD Desk** (directory) | `{verb_present}_{subject}/` | `initiate_order/` |
| **PRJ Desk** (directory) | `{event}/` (version-agnostic) | `order_initiated/` |
| **Command** | `{verb_present}_{subject}_v1` | `initiate_order_v1` |
| **Event** | `{subject}_{verb_past}_v1` | `order_initiated_v1` |
| **Handler** | `maybe_{verb_present}_{subject}` | `maybe_initiate_order` |
| **CMD API** | `{command}_api` | `initiate_order_api` |
| **CMD Desk Supervisor** | `{command}_desk_sup` | `initiate_order_desk_sup` |
| **PRJ Desk Supervisor** | `{event}_sup` | `order_initiated_sup` |
| **Responder** | `{command}_responder_v1` | `initiate_order_responder_v1` |
| **Emitter (mesh)** | `{event}_to_mesh` | `order_initiated_to_mesh` |
| **Emitter (pg)** | `{event}_to_pg` | `order_initiated_to_pg` |
| **Aggregate** | `{noun}_aggregate` | `order_aggregate` |
| **Projection** | `{event}_to_{read_store}` | `order_initiated_to_orders` |
| **Policy** | `on_{event}_maybe_{command}` | `on_license_revoked_v1_maybe_remove_plugin` |
| **Listener** | `on_{fact}_maybe_{command}` | `on_app_available_maybe_install_plugin` |
| **Query (by PK)** | `get_{aggregate}_by_id` | `get_order_by_id` |
| **Query (by field)** | `get_{aggregate}_by_{field}` | `get_order_by_number` |
| **Query (paged)** | `get_{aggregates}_page` | `get_orders_page` |
| **Query (filtered)** | `get_{aggregates}_by_{field}` | `get_orders_by_customer` |
| **QRY API** | `{query_module}_api` | `get_orders_page_api` |
| **Tests** | `{module}_tests` | `order_aggregate_tests` |
| **Status Header** | `{aggregate}_status.hrl` | `order_status.hrl` |

---

## Event Flows from the Store

Every event stored in ReckonDB can trigger one of 5 flow types. Each has a distinct naming pattern and role:

| # | Flow | Module Pattern | Role |
|---|------|---------------|------|
| 1 | Event Store → Read Model | `on_{event}_to_sqlite_{table}` / `on_{event}_to_couchdb_{table}` | **Projection** |
| 2 | Event Store → PubSub | `{event}_to_pg` | **Integration Emitter** |
| 3 | Event Store → Mesh | `{event}_to_mesh` | **Mesh Emitter** |
| 4 | Event Store → own Aggregate | `on_{event}_maybe_{command}` | **Policy** |
| 5 | Other Domain/Mesh → own Aggregate | `on_{fact}_maybe_{command}` | **Listener** |

All 5 subscribe to events via `reckon_evoq_adapter:subscribe/5`. The difference is what they DO with the event:

- **Projection** (1): Writes to a read model (SQLite, CouchDB). Lives in the PRJ app.
- **Integration Emitter** (2): Broadcasts to OTP `pg` groups for inter-domain consumption. Lives in CMD desk.
- **Mesh Emitter** (3): Publishes to Macula mesh for WAN/cross-network consumption. Lives in CMD desk.
- **Policy** (4): Reacts to an internal domain event, dispatches a command to own aggregate. Lives as a sibling slice in the target CMD app under `apps/{cmd_app}/src/on_{event}_{action}_{target}/`.
- **Listener** (5): Reacts to a FACT from outside (another domain via pg, or mesh), dispatches a local command. Lives as a sibling slice in the target CMD app under `apps/{cmd_app}/src/on_{fact}_{action}_{target}/`. (Reversed 2026-03-12 — was originally "inside the target desk"; see Demon 18.)

### Policy vs Listener

Both dispatch commands. The difference is the SOURCE of the trigger:

| Name | Source | Example |
|------|--------|---------|
| **Policy** | Internal domain event (own event store) | `on_license_revoked_v1_maybe_remove_plugin` |
| **Listener** | External fact (pg from another domain, or mesh) | `on_app_available_maybe_install_plugin` |

## Command Entry Points

A command is NOT always triggered by an API call. There are three entry points:

| Entry Point | Pattern | Name | Source |
|-------------|---------|------|--------|
| API call (HOPE) | `{command}_api.erl` | **API Handler** | User/frontend via HTTP |
| Internal domain event | `on_{event}_maybe_{command}.erl` | **Policy** | Own event store |
| External fact | `on_{fact}_maybe_{command}.erl` | **Listener** | Another domain via pg/mesh |

A command without an API handler is NOT dead code — it may only be triggered by policies or listeners. The audit question is: "does every command have at least one entry point?"

### Department Ownership

| Flow | Lives in | Department |
|------|----------|------------|
| Projection | `apps/project_{plural}/src/{event}/` | PRJ |
| Integration Emitter | `apps/{cmd_app}/src/{desk}/` | CMD |
| Mesh Emitter | `apps/{cmd_app}/src/{desk}/` | CMD |
| Policy | `apps/{cmd_app}/src/on_{event}_{action}_{target}/` (sibling of desks) | CMD |
| Listener | `apps/{cmd_app}/src/on_{fact}_{action}_{target}/` (sibling of desks) | CMD |

---

## Division App Structure (CMD / PRJ / QRY)

Every division with event sourcing has three separate apps:

| Department | Naming Pattern | Example | Responsibility |
|-----------|---------------|---------|---------------|
| **CMD** | `guide_{aggregate}_lifecycle` or `{process_verb}_{subject}` | `guide_license_lifecycle` | Commands, events, aggregates, emitters, PMs |
| **PRJ** | `project_{domain}` or `project_{read_model_plural}` | `project_appstore` | Projections, SQLite store, event subscriptions |
| **QRY** | `query_{domain}` or `query_{read_model_plural}` | `query_appstore` | API handlers, reads from SQLite |

PRJ and QRY are SEPARATE apps. PRJ writes read models, QRY reads them. Different responsibilities.

---

## Derivation Rules (Input -> Convention -> Output)

Given a dossier/app name, all other names are **deterministic**:

### From App Name

| Input | Convention | Output |
|-------|-----------|--------|
| `process_orders` | strip prefix | aggregate noun: `order` |
| `process_orders` | strip prefix, keep plural | aggregate plural: `orders` |
| `process_orders` | pair with query | query app: `query_orders` |

### From Command Desk Name

| Input | Convention | Output |
|-------|-----------|--------|
| `initiate_order` | `_v1` suffix | command module: `initiate_order_v1` |
| `initiate_order` | past tense + `_v1` | event module: `order_initiated_v1` |
| `initiate_order` | `maybe_` prefix | handler: `maybe_initiate_order` |
| `initiate_order` | `_api` suffix | API handler: `initiate_order_api` |
| `initiate_order` | `_desk_sup` suffix | supervisor: `initiate_order_desk_sup` |
| `initiate_order` | `_responder_v1` suffix | responder: `initiate_order_responder_v1` |
| `initiate_order` | event `_to_mesh` | emitter: `order_initiated_to_mesh` |
| `initiate_order` | event `_to_pg` | pg emitter: `order_initiated_to_pg` |

### From PRJ Desk Name (Event-Based)

PRJ desk directories are named after the **triggering event** (version-agnostic).
Modules inside keep their full `{event}_v{N}_to_{target}` names.

| Input | Convention | Output |
|-------|-----------|--------|
| `order_initiated` | directory name | desk dir: `order_initiated/` |
| `order_initiated` | `_sup` suffix | supervisor: `order_initiated_sup` |
| `order_initiated` | `_v1_to_{target}` | projection module: `order_initiated_v1_to_orders` |
| `order_initiated` | `_v2_to_{target}` | projection module: `order_initiated_v2_to_orders` |

Multiple versions and targets coexist in the same desk directory.

### From Query Desk Name

| Input | Convention | Output |
|-------|-----------|--------|
| `get_order_by_id` | `_api` suffix | API handler: `get_order_by_id_api` |
| `get_orders_page` | `_api` suffix | API handler: `get_orders_page_api` |

---

## Route Pattern Rules

| Desk Type | Route Pattern | Example |
|-----------|--------------|---------|
| CMD (create) | `POST /api/{plural}/{verb}` | `POST /api/orders/initiate` |
| CMD (on existing) | `POST /api/{plural}/:id/{verb}` | `POST /api/orders/:id/archive` |
| QRY (paged list) | `GET /api/{plural}` | `GET /api/orders` |
| QRY (by ID) | `GET /api/{plural}/:id` | `GET /api/orders/:order_id` |
| QRY (filtered) | `GET /api/{plural}?{filter}=value` | `GET /api/orders?customer_id=X` |

---

## Event Naming Rules

1. **Past tense**: Events describe what happened: `order_initiated`, not `initiate_order`
2. **Version suffix**: Always `_v1`: `order_initiated_v1`
3. **Business verbs only**: No CRUD (`created`, `updated`, `deleted`)
4. **Subject first**: `{subject}_{verb_past}`, not `{verb_past}_{subject}`

### Event Verb Examples

| Action | Event Name | NOT |
|--------|-----------|-----|
| Birth | `{noun}_initiated_v1` | `{noun}_created_v1` |
| Soft delete | `{noun}_archived_v1` | `{noun}_deleted_v1` |
| Phase change | `{noun}_submitted_v1` | `{noun}_updated_v1` |
| Refinement | `{noun}_refined_v1` | `{noun}_modified_v1` |
| Discovery | `{noun}_identified_v1` | `{noun}_found_v1` |

---

## Stream ID Conventions

| Aggregate | Stream Pattern | Example |
|-----------|---------------|---------|
| Venture | `venture-{venture_id}` | `venture-019e642a1acf7019a433f0634a035877` |
| Division | `division-{division_id}` | `division-019e643ba9937926b14648f21ec23fc9` |
| Generic | `{aggregate_noun}-{id}` | `order-019e644a8b1d70291a456b5e9c3df8a2` |

**Format contract** (enforced by `reckon_gater_stream_id:validate/1`):

```
^[a-z]{1,32}-[a-f0-9]{32}$
```

Lowercase letters for the prefix (1–32 chars), single hyphen, exactly
32 lowercase hex chars in the suffix. The suffix carries 128 bits of
entropy — one UUID-worth — at a predictable length.

**Use `reckon_gater_stream_id:new(Prefix)` to mint fresh ids.** The
helper uses `reckon_gater_uuid:v7/0` under the hood, so the leading
48 bits of the hex suffix are `unix_ts_ms` — `ORDER BY stream_id`
returns aggregates in birth order without an explicit time column.
Useful for replay tooling, projection grouping, and chronological log
inspection.

```erlang
SessionId = reckon_gater_stream_id:new(<<"sess">>).
%% => <<"sess-019e642a1acf7019a433f0634a035877">>

LotId = reckon_gater_stream_id:new(parking_lot).  %% atom prefix also accepted
%% => <<"parking_lot-019e643ba9937926b14648f21ec23fc9">>
```

**Do NOT hand-roll** with `crypto:strong_rand_bytes/16` + a UUID-v4
formatter. UUID-v4 string output (`9D9D16B6-F56E-...`) fails the
contract: uppercase + internal hyphens. The dispatch returns
`{error, {invalid_stream_id, malformed_user_id, ...}}` — see Demon 49
in `ANTIPATTERNS_EVENT_SOURCING.md` for the entire failure mode.

---

## rebar.config src_dirs Convention

Every desk directory MUST be listed in `src_dirs`:

```erlang
%% CMD app (process_orders)
{src_dirs, [
    "src",
    "src/initiate_order",
    "src/archive_order",
    "src/refine_order"
]}.

%% PRJ app (project_orders) — desk dirs named after the event, version-agnostic
{src_dirs, [
    "src",
    "src/order_initiated",
    "src/order_archived"
]}.

%% QRY app (query_orders)
{src_dirs, [
    "src",
    "src/get_order_by_id",
    "src/get_orders_page"
]}.
```

---

## Status Bit Flags Convention

| Flag | Value | Purpose |
|------|-------|---------|
| `STATUS_INITIATED` | `1` (2^0) | Aggregate born |
| `STATUS_ARCHIVED` | varies | Soft-deleted, hidden from queries |
| Domain flags | powers of 2 | Domain-specific states |

Macro naming: `?{AGGREGATE_UPPER}_{FLAG}` (e.g., `?VENTURE_INITIATED`, `?ORDER_ARCHIVED`)

---

## Banned Names

**NEVER use these patterns:**

| Pattern | Why | Correct Alternative |
|---------|-----|-------------------|
| `list_{aggregates}` | Unbounded | `get_{aggregates}_page` |
| `get_all_{aggregates}` | Unbounded | `get_{aggregates}_page` |
| `get_{aggregate}` (raw) | Ambiguous | `get_{aggregate}_by_id` |
| `{noun}_created_v1` | CRUD | `{noun}_initiated_v1` |
| `{noun}_updated_v1` | CRUD | Use business verb |
| `{noun}_deleted_v1` | CRUD | `{noun}_archived_v1` |
| `*_service` | Horizontal layer | Vertical slice |
| `*_handler` | Technical noise | `maybe_{command}` |
| `*_manager` | God module | Domain-specific module |
| `*_helper` / `*_util` | Junk drawer | Put in the desk |

---

## Quick Derivation Example

Given: dossier `process_orders`, desk `initiate_order` (command type)

```
CMD app:         process_orders
PRJ app:         project_orders
QRY app:         query_orders
Aggregate:       order_aggregate
Desk dir:        src/initiate_order/
Command module:  initiate_order_v1
Event module:    order_initiated_v1
Handler:         maybe_initiate_order
API handler:     initiate_order_api
Supervisor:      initiate_order_desk_sup
Responder:       initiate_order_responder_v1
Mesh emitter:    order_initiated_v1_to_mesh
pg emitter:      order_initiated_v1_to_pg
PRJ desk dir:    src/order_initiated/                   (in PRJ app)
PRJ desk sup:    order_initiated_sup                    (in PRJ app)
Projection:      order_initiated_v1_to_sqlite_orders    (in PRJ app, inside order_initiated/)
QRY handler:     get_orders_page_api                   (in QRY app)
Route (CMD):     POST /api/orders/initiate
Route (QRY):     GET /api/orders
Test:            order_aggregate_tests
Status header:   order_status.hrl
```

---

## [PROPOSED] Plugin App Naming Template

> **Status: PROPOSED — discuss before adopting universally.**
> This template emerged from the appstore plugin and fits lifecycle-centric domains well.
> Not all plugin apps may fit this paradigm — evaluate case by case.

### The Template

For plugin daemons (`hecate-app-{name}d`) that manage an aggregate through a lifecycle:

```
hecate-app-{name}d/
├── apps/
│   ├── guide_{aggregate}_lifecycle/   ← CMD department
│   ├── project_{aggregate}/           ← PRJ department
│   └── query_{aggregate_plural}/      ← QRY department
```

| Department | Pattern | Intent | Example |
|-----------|---------|--------|---------|
| **CMD** | `guide_{aggregate}_lifecycle` | Shepherds the aggregate through its business process | `guide_license_lifecycle` |
| **PRJ** | `project_{aggregate}` | Projects domain events into read models | `project_appstore` |
| **QRY** | `query_{aggregate_plural}` | Serves read models via API | `query_appstore` |

### Why `guide_`?

- "Guide" screams intent — the app **guides** something through a process
- Plugin domains are inherently lifecycle managers (initiate → use → archive)
- The `_lifecycle` suffix distinguishes from hecate core `guide_*` orchestrators (e.g., `guide_venture`)

### When This Template Fits

- The plugin manages a **single aggregate** through a defined lifecycle
- The lifecycle has clear phases (buy, install, upgrade, remove, archive)
- Commands map to lifecycle transitions

### When It Might NOT Fit

- The plugin has **multiple independent aggregates** with separate lifecycles
- The domain is not lifecycle-centric (e.g., a real-time data stream processor)
- The CMD app is better described by a specific process verb (e.g., `trade_assets`)

### Derivation Example

Given: plugin `appstore`, aggregate `license`

```
CMD app:    guide_license_lifecycle
PRJ app:    project_appstore           (domain-oriented — acceptable when PRJ serves
                                        multiple read models like catalog + licenses)
QRY app:    query_appstore             (same reasoning)
Aggregate:  license_aggregate
```

Note: PRJ/QRY apps may use the **domain name** instead of the aggregate name
when the bounded context has multiple read models (e.g., `plugin_catalog` +
`licenses` tables both served by `query_appstore`). The aggregate-oriented
naming (`project_licenses` / `query_licenses`) is preferred when the domain
has a single aggregate with a single read model.

---

_Every name is deterministic. Conventions are the compiler._
