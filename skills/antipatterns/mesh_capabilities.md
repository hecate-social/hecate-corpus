---
title: Mesh Capability Advertisement
layer: skill
audience: [agent, human]
stage: reversed
---

# antipatterns/mesh_capabilities.md — Mesh Capability Advertisement

Demons about advertising and resolving capabilities on the mesh:
procedure_advertisement records, the station-side advertise registry,
the org namespace, and the caller's pin.

---

## 🔥🔥🔥 Demon 69: Two Providers, One Procedure, One Station — the Registry Holds One

**Date exorcised:** 2026-09-25
**Where it appeared:** the first two-provider fleet deployment — the Erlang
`mcl-bookclub` and its Elixir twin `mcl-bookclub-phoenix`, one org
(`mcl-bookclub`), one procedure (`get_bookclub_by_id`), both on the same
beam03 station set. Diagnosed through macula-services/mcl-om issue #5.
**Cost:** The many-club scoreboard showed `bookclub_verified => no` for the
second club on live fleet, and the issue's first diagnosis chased a
nonexistent "resolve drops the org-key record" for a full session, because
the caller-side error was a lie.

### The Lie

"One wire contract, many clubs: every club publishes its own signed
`procedure_advertisement` under the same org key, the DHT holds all of them
(signer-deduped multiset), and the `advertiser` pin dials the named club —
so a thousand clubs on one procedure are each addressable."

### What Happened

The DHT half of that is true. The ROUTING half is not: a station's
`macula_remote_advertise_registry` keys on `(realm, procedure)` and holds
**one advertiser per key** — "Single-provider invariant: re-advertising
replaces the prior entry"; direct ADVERTISE frames trump gossip ("Direct
trumps anything — replace"). Both clubs' records named the SAME serving
station, because the SDK's `publish_advertisement` names the pool's
first-connected link, and the link order is map-term order over the seed
set — **alphabetical host name, not the seed list order** (rotating the
seeds in the deploy env does nothing). So both records said falkenstein,
the station routed every `get_bookclub_by_id` CALL to whichever club had
re-advertised last, the other club's link answered it anyway (the inbound
CALL handler dispatches on `{realm, procedure}` with NO target check —
only STREAM_OPEN checks `for_this_node`), the caller refused the
mis-signed reply (`responded_by ≠ target` → `not_the_target`), the call
timed out — and mcl_om's failover collapsed the timeout into
`{error, no_provider}`. The same answer as a stale pin. The scoreboard
said "no"; the diagnosis went into the resolve path, where the bug was
not.

The live proof of last-wins: triggering the Phoenix node's `publish/0`
flipped the failing club from Phoenix to Erlang and back with the next
republish tick.

### The Fix

Three mechanisms, one per layer of the lie:

1. **Spread co-org providers across serving stations** (mcl_om 0.29.0):
   advertise passes `advertise_direct` a `publish_advertisement` opt that
   names the station `choose_serving_station/2` picks for THAT node
   (`phash2(node_id)` over the SORTED connected stations) — deterministic
   per node, stable across republish ticks, distinct whenever there are
   stations to spare. Unit-tested with the two clubs' REAL node ids as
   the regression: they must land apart.
2. **`no_provider` means exactly "nothing to dial"** (mcl_om 0.29.0): a
   provider that was dialed reports its own failure (`{error, timeout}`,
   `{error, {station_endpoint, _}}`, …). A stale pin still fails closed
   as `no_provider` — the two can never be confused again.
3. **The fixture that would have caught it**:
   `mcl_om_capabilities_two_providers_tests` (live, real station): two
   providers under one org — both records resolve, the pin dials the
   named (most recent) advertiser, a stale pin fails closed, and the
   displaced provider fails with a real dial error, NEVER `no_provider`.

### The Rule

> **A station's registry holds ONE advertiser per `(realm, procedure)`;
> "one procedure, many providers" is true only across stations.**
> **Pick the serving station per node, not per pool — and never let a
> dial failure answer like a resolve miss, or the diagnosis follows the
> lie.**

---

*We burned these demons so you don't have to. Keep the fire going.*
