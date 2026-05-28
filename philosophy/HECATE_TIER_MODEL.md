---
title: The Hecate four-tier model
layer: philosophy
audience: [agent, human]
stage: stable
---

# The Hecate four-tier model

Adopted 2026-05-18. Codifies the cut between **per-user agent surface**
and **always-on realm infrastructure** that the Hecate stack drew
when `serve_llm` migrated out of `hecate-daemon` into the new
`hecate-services/*` family.

This document is shaping material. Future Claude sessions, future
contributors, and grant-reviewer audiences should be able to read it
in five minutes and answer "where does X belong?" without asking
anyone.

---

## The four tiers

```
Layer 4 — apps        hecate-app-martha, hecate-app-rag, hecate-app-scribe, …
                      User-facing plugins. Per-identity, per-session.
                      Live inside hecate-daemon's BEAM as in_vm plugins.

Layer 3 — session     hecate-daemon
                      One per human identity. Plugin host. Owns the
                      desktop-shell connection, SSE streams, and the
                      user's local Macula client pool.
                      Runs on user laptops + MaculaOS edge devices.

Layer 2 — services    hecate-services/hecate-{om, rag, dns, git, llm, …}
                      Always-on, multi-tenant, realm-bound.
                      Run on realm INFRASTRUCTURE NODES (BEAM cluster,
                      dedicated relay boxes, cooperative-contributed
                      service nodes). NOT on user laptops.

Layer 1 — identity    hecate-realm / macula-realm
                      Issues human realm-membership certs AND service
                      principal certs.

Layer 0 — kernel      macula-station
                      QUIC peering, DHT, SWIM liveness, routing.
                      Realm-agnostic. One per node, every node.
```

Every node runs Layer 0 (`macula-station`). Layer 1 (`hecate-realm`)
runs where the realm's stewards put it. Layer 2 services run on
infrastructure nodes. Layer 3 (`hecate-daemon`) runs per-user on
laptops. Layer 4 plugins live inside the daemon.

## Why this cut

Three forces pushed the line into existence:

1. **`hecate-daemon` was becoming a grab-bag.** It hosted
   `serve_llm` (heavy, multi-tenant, always-on), Martha (19-app
   agent runtime), and UI surfaces (Scribe, IRC, weather, …) in
   one BEAM. Different lifecycles, different resource shapes,
   different failure domains — wrong to mix.

2. **Mesh-as-infrastructure needs infrastructure SERVICES.** A
   federated mesh is not just plumbing for chat — it can host
   DNS, blob, git, LLM, RAG, build runners, archival. Those are
   capabilities the realm offers, not apps the user runs. They
   want their own deployment unit.

3. **The four workload classes in the strategic anchor**
   (conventional / LLM serving / federated AI / cooperative
   compute) are all Layer-2 or below. None are Layer-3 or 4.
   The tier model maps cleanly to the framing we already pitch.

## Cut criteria

When deciding where a new capability belongs, walk the list:

**Service (Layer 2)** if **any** of:

- Runs always-on without a logged-in user
- Heavy resource shape (>200 MB RAM resident, GPU, big disk)
- Multi-tenant (serves multiple humans on the same box)
- Maps to one of the four workload classes
- Holds shared mutable state that survives sessions
- Has its own external dependency (API keys, model weights, …)
- Translates an external data source (vendor MQTT, file watcher,
  smart-meter readout, IoT gateway, …) into mesh facts. These
  **ingestion adapters** are L2 by default — never L3 sidecars or
  L4 plugins. See "Offline operation via reckon-db" below.

**App (Layer 4)** if **all** of:

- User-facing UI surface, or a per-session event handler
- Per-session state only
- Cheap (kilobytes of memory, no heavy I/O)
- Coordinates rather than computes (calls Layer-2 services for
  the actual work)

If you can't decide, default to Layer 2 (be paranoid about the
grab-bag). It's easier to merge a small service into the daemon
later than to extract a heavy plugin under load.

### Format contracts belong in the protocol layer

The above criteria cut by *capability*. Cut criteria also exist for
*types* and *formats*: identifier shapes, envelope layouts, snapshot
serialization, subscription protocol, anything with a regex or schema
contract. These belong in the **protocol layer** (`reckon-gater`), not
the implementation layer (`reckon-db`).

The reason: an adapter that doesn't run the storage backend (e.g.
`reckon-evoq`) needs to validate ids without dragging the storage
backend in as a dep. A gateway client (e.g. `reckon-lazy`, or any future
write-only sender) needs to mint ids without depending on the storage.
Both are wrong if the format definition lives in the storage layer.

**Symptom of getting this wrong:** an upstream type module imports a
downstream storage module to do basic shape checks. If you see
`reckon_db_*` referenced from `reckon_evoq_*`'s code, the cut is in the
wrong place — the type moved out of layer.

**Worked example** (2026-05-26): the user-stream-id regex
`^[a-z]{1,32}-[a-f0-9]{32}$` and its `validate/1` + `new/1` helpers
lived in `reckon_db_stream_id` (inside the storage backend). Anyone
wanting to validate or mint stream ids without running reckon-db
couldn't. Relocated to `reckon_gater_stream_id` in reckon-gater 2.2.0;
`reckon-db` 3.0.0 calls into it. Both storage and the adapter (and any
future client) now reach the same module without coupling to a specific
implementation.

## The contract

Every Layer-2 service:

1. Lives at `codeberg.org/hecate-services/hecate-X`
2. Implements the `hecate_om_service` behaviour (six callbacks:
   `info/0`, `start/1`, `stop/1`, `health/0`, `capabilities/0`,
   `identity_spec/0`)
3. Ships as an OCI container to `ghcr.io/hecate-services/hecate-X`
4. Declares a Quadlet unit in `quadlet/` for system-wide
   systemd-managed Podman
5. Carries a `manifest.json` with `tenancy: realm`,
   `runs_on: infrastructure_node`, and the advertised capability list
6. Receives a realm-signed service-principal credential at install
   time, mounted at `/etc/hecate/secrets/service-cert.pem`
7. Connects to the local `macula-station` and advertises every
   capability via `macula:advertise/5`
8. Exposes `/health` on loopback (port 8470) for Podman's
   HEALTHCHECK; no externally-routable ports

The substrate library [`hecate-om`](https://codeberg.org/hecate-services/hecate-om)
provides 1, 7, and 8 for free. Services just implement the
behaviour and wire their `_mesh_rpc.erl` dispatch table.

## Offline operation via reckon-db

Layer-2 services are not assumed to be online all the time. The
mesh has connectivity gaps (relay restarts, fleet bounces, network
partitions, intermittent ISPs, sites that come online only when a
ship enters port). A service whose ingest path blocks on mesh
reachability is brittle.

The canonical pattern: **every L2 service that ingests external
data owns a reckon-db store**. Inbound data becomes a command →
domain event → local stream. A separate emitter slice drains the
stream to the mesh asynchronously, retrying forever. The ingest
path never observes mesh state.

```
external source ──► subscriber slice ──► command
                                            │
                                            ▼
                                  reckon-db (local stream)
                                            │
                                ┌───────────┴────────────┐
                                ▼                        ▼
                       on_event_to_mesh         other projections /
                       (emitter, async,         process managers
                        drains when station
                        is mesh-reachable)
                                │
                                ▼
                        macula:publish/3
```

The substrate already supports this directly. `hecate_om_service`
declares two optional callbacks:

```erlang
-callback store_id() -> atom().     %% the service's reckon-db store id
-callback data_dir() -> string().   %% on-disk root for the store
```

When both are exported, `hecate_om:boot/1` starts a `single`-mode
reckon-db store at `<data_dir>/<store_id>/` and an evoq subscription
**before** the service's own `start/1` fires. Producer-only services
(no event store) omit both callbacks and pay nothing.

**Consequences:**

- Boat goes offline two weeks → events accumulate locally → station
  reaches marina wifi → emitter drains backlog → realm subscribers
  catch up. Same code, no special branch.
- Restarting the service replays from the reckon-db stream. No data
  loss across restarts, container migrations, or host moves.
- Naive buffering / `try ... catch macula:publish` retries in handler
  code are an **antipattern**. The store IS the buffer.
- Ingestion adapters (Cut criteria item above) should ship with
  these callbacks declared from v0.1, not v0.2.

## Identity model

Services are **institutions** of the realm, not user-bound. Each
has its own keypair and a realm-signed credential. The metaphor:

> Citizens of a town carry citizen IDs. The town's library, post
> office, and water utility carry institutional badges — same
> issuer (the town clerk), narrower scope. The library doesn't
> borrow Alice's citizen ID to lend her a book.

In Hecate terms:

- **Citizens** (humans, hecate-daemon) carry realm-membership certs.
- **Institutions** (hecate-services/*) carry service-principal
  certs, also signed by the realm, but with narrower `actions`
  and `resources`.
- A user logging out doesn't take services with them. Services
  outlive sessions.

v1 implementation: long-lived service-principal certs provisioned
by a realm-admin script. v2 (when policy + UCAN delegation land):
short-lived UCANs auto-rotated against a `hecate-realm` HTTP
endpoint. The swap-in lives entirely behind `hecate_om_identity`;
consumers don't notice.

See `hecate-om/guides/identity_model.md` for the full
town/library walkthrough and the v1 / v2 trigger.

## Anti-patterns

Three things this model explicitly forbids:

1. **No user-bound services.** A `hecate-rag` running on Alice's
   laptop "for Alice" is wrong. Move it to a realm-owned
   infrastructure node; let Alice reach it across the mesh.
2. **No anonymity / self-rooted leaves.** Every service principal
   chains to a realm root. No Pubky-style ungoverned identities.
   (See `memory/feedback_no_anonymity_only_sovereignty`.)
3. **No "central" anything across services.** Each Layer-2
   service supervises its own state. There is no shared service
   bus, no central registry, no horizontal layer of "service
   plumbing". The realm coordinates through Macula RPC, not a
   service framework.
4. **No bridging L2-shaped work through `hecate-daemon`'s REST
   API.** The daemon's `/api/mesh/publish`, `/api/mesh/call`, etc.
   are for Layer-4 plugins inside the daemon's BEAM and for
   external integrations like `macula-mcp`. An L2 service uses
   the Macula SDK directly against its local `macula-station` —
   no HTTP middleman, no L3 dependency, correct identity. See
   `skills/antipatterns/integration.md` Demon 50.

## Lone-deployment exception

The default rule above ("NOT on user laptops") stands for every
multi-user, federated, or community context. For **genuinely
infrastructure-less** deployments — a boat at sea, an RV off-grid,
a single-household pilot before a cooperative exists — a Layer-2
service MAY be hosted on user-owned hardware (a laptop, a Cerbo, a
home NUC) provided **all** of:

1. The service has its own **service-principal credential**, not
   borrowed from the user's citizen cert. Even a household realm
   of one issues a separate badge for the service.
2. The service is **not user-session-keyed.** It runs whenever the
   host is up, not "when the user is logged in". `systemd --user`
   with linger-enabled, a Quadlet under root, or a Venus-OS runit
   slot all qualify; "starts when I open my laptop" does not.
3. The service principal still **chains to a realm root** — no
   self-rooted leaves, even in lone deployments. The realm may
   be small (one household, one boat); it must exist.
4. The manifest carries `deployment: lone` (or operational
   equivalent) so future operators know to migrate the service
   onto shared realm infrastructure when it becomes available.

This exception exists so that ingestion adapters and similar L2
services can run in scenarios that have **no community node yet**
(the lone phase of a future cooperative). It is not a license to
host community services on member laptops once a cooperative
infrastructure node exists. When in doubt, move it to the shared
node.

## How callers reach services

A plugin (Layer 4) or an agent inside the daemon (Layer 3) reaches
a Layer-2 service via Macula RPC:

```erlang
%% Unary
{ok, Result} = macula:call(
    LocalPool, Realm,
    <<"hecate-rag.answer_query">>,
    #{query => Q, top_k => 5},
    Timeout
).

%% Streaming
{ok, Stream} = macula:call_stream(
    LocalPool, Realm,
    <<"hecate-llm.stream_chat">>,
    #{model => Model, messages => Msgs},
    #{}
).
```

The local `macula-station` routes the call to whichever
infrastructure node is running the service. No HTTP, no DNS lookup,
no API keys in the caller. The service answers as itself with its
own credential; the realm verifies both sides.

## Naming

| Prefix | Means | Examples |
|--------|-------|----------|
| `macula-` | Realm-agnostic infrastructure (Layer 0–1) | `macula-station`, `macula-realm`, `macula-rag` (federation protocol — uses SDK), `macula-mcp` |
| `hecate-` | Realm-aware service or app (Layer 2–4) | `hecate-realm`, `hecate-rag`, `hecate-llm`, `hecate-daemon`, `hecate-app-martha` |

The `macula-X` prefix means *"depends on the Macula SDK"*, not
*"runs inside macula-station"*. `macula-rag` (the federation
library) runs in Layer 2 service consumers as a dep, alongside
`hecate-rag` itself.

## What lives in `hecate-services/` (as of 2026-05-18)

| Repo | Role |
|------|------|
| `hecate-om` | Substrate library — `hecate_om_service` behaviour, identity loader, capability advertiser, `/health` handler, container + Quadlet templates |
| `hecate-rag` | Retrieval-augmented generation over the realm's corpora |
| `hecate-dns` | DNS-over-mesh name resolution |
| `hecate-git` | Git-over-mesh repository server (companion to `git-remote-mesh`) |
| `hecate-llm` | LLM gateway (Anthropic / OpenAI / Google / Ollama) |

Future watchlist (not yet built):
- `hecate-blob` — content-addressed blob store
- `hecate-cron` — scheduled task runner
- `hecate-runner` — CI / build runner
- `hecate-faber` — federated neuroevolution
- `hecate-tools` — agent tool surfaces (web_fetch, web_search,
  synthesize_speech, transcribe_audio, …) — the bits dropped from
  `hecate-llm`'s extract on 2026-05-18
- `hecate-victron` — Victron Venus OS ingestion adapter
  (dbus-flashmq MQTT → mesh facts; reckon-db offline-first)
- `hecate-openems` — OpenEMS Edge ingestion adapter
  (JSON-RPC over WS → mesh facts; reckon-db offline-first)
- `hecate-shelly` — Shelly Pro local-MQTT ingestion adapter

## Reading order for a new contributor

1. This file — overall shape
2. `hecate-om/README.md` — the substrate
3. `hecate-om/guides/service_anatomy.md` — what every service looks like
4. `hecate-om/guides/identity_model.md` — town / library metaphor + v1/v2
5. `hecate-om/guides/container_deployment.md` — how a service lands on a node
6. Pick one shipped service (`hecate-rag` is the most fleshed-out)
   and read it end-to-end

Memory references for context:
- `[[project_hecate_services_tier]]` — the migration log
- `[[reference_codeberg_push_mirror_403]]` — bootstrap quirk you'll hit
- `[[feedback_stations_route_daemons_publish]]` — why Layer 0 is
  realm-agnostic
