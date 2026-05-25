# ANTIPATTERNS.md — Demons We've Exorcised

*Mistakes we've made and corrected. Read this. Don't repeat them.*

---

## Demon Index

| # | Name | One-Line Summary | Date |
|---|------|-----------------|------|
| 1 | Technical Names Don't Scream | Slice names describe HOW not WHAT | 2026-02-04 |
| 2 | Parallel Domain Infrastructure | Duplicate infrastructure across domains | 2026-02-04 |
| 3 | Incomplete Desks / Flat Workers | Missing desk supervisors and components | 2026-02-04 |
| 4 | Missing or Wrong "Birth" Event | Must use `{agg}_initiated_v1` | 2026-02-08 |
| 5 | Auto-Creating Child Aggregates | Parent IDENTIFIES, child INITIATES | 2026-02-08 |
| 6 | ~~Listeners as Separate Desks~~ (REVERSED 2026-05-24 → see Demon 18) | PMs/listeners ARE their own slices; this demon is reversed | 2026-02-08 |
| 7 | Using Mesh for Internal Integration | Use pg internally, mesh for WAN only | 2026-02-08 |
| 8 | Centralized Listener Supervisors | No central supervisor for all listeners | 2026-02-08 |
| 9 | Direct Creation Endpoints for Child Aggregates | Children created through parents only | 2026-02-09 |
| 10 | Wrong Aggregate Callback Argument Order | State first, Payload second | 2026-02-09 |
| 11 | Side Effects Based on Hope Acknowledgment | React to facts, not acknowledgments | 2026-02-09 |
| 12 | Read-Time Status Enrichment | Compute at projection time, not query time | 2026-02-10 |
| 13 | Ambiguous Query Module Names | Vague query names hide scaling dangers | 2026-02-10 |
| 14 | God Module API Handlers | All endpoints in one file | 2026-02-10 |
| 15 | Consumer-Generated Command IDs | Framework owns command_id generation | 2026-02-11 |
| 16 | Parameterized Phase Lifecycle | Generic `start_phase/pause_phase` hides intent | 2026-02-12 |
| 17 | CRUD Verbs in Event-Sourced Commands | "update" is the U in CRUD | 2026-02-12 |
| 18 | Process Managers Inside Desks (reversed) | PMs are sibling slices in target CMD app, NOT nested inside desks | 2026-02-12 (rev 2026-03-12) |
| 19 | esqlite3 Returns Lists, Not Tuples | `row_to_map({A,B})` silently fails — use `[A,B]` | 2026-02-12 |
| 20 | Eager Default in `maps:get/3` | `maps:get(k, M, f())` evaluates `f()` even when key exists | 2026-02-12 |
| 21 | esqlite3 Argument Order (Db First) | `esqlite3:exec(Db, SQL)` not `exec(SQL, Db)` | 2026-02-12 |
| 22 | Manual Event Emission from API Handlers | Emitters subscribe via evoq, not called from handlers | 2026-02-13 |
| 23 | Raw #event{} Records in Projections | Projections receive records, call maps:find on tuples | 2026-02-13 |
| 24 | Silent Subscription Pipeline Failures | POST succeeds, GET returns [] — three bugs, zero errors | 2026-02-13 |
| 25 | Centralized Route Registration Files | One file per app listing all routes — horizontal grouping | 2026-02-16 |
| 26 | PG Emitters "Dead Code" Without Subscribers | Pub/sub publishers are infrastructure — no subscribers ≠ dead code | 2026-02-23 |
| 27 | Hardcoded User/Submitter IDs | Commands must carry the real actor identity, not placeholders | 2026-02-23 |
| 28 | No Tests on Event-Sourced Domains | Tests written BEFORE (preferred) or IMMEDIATELY AFTER the code file, same session — never deferred to CI/workflow gates | 2026-02-23 |
| 29 | Missing `/ui/[...]` Cowboy Route in Plugin Daemon | Plugin has manifest + socket but no static file route — hecate-web silently drops it | 2026-02-24 |
| 30 | Forgetting to Bump `.app.src` Versions Before Tagging | Tag a release without bumping vsn in .app.src files — OCI image ships old version | 2026-02-24 |
| 31 | Inline Projections After Command Dispatch | LiveViews writing to read models after evoq dispatch — bypasses PRJ department | 2026-03-02 |
| 32 | Consolidating PRJ and QRY Into One Department | PRJ (event→write) and QRY (read→response) are ALWAYS separate departments | 2026-03-02 |
| 33 | Evoq Without ReckonDB ("In-Memory" Event Sourcing) | Evoq REQUIRES ReckonDB — "in-memory only" loses all events on restart | 2026-03-02 |
| 34 | Binary Keys in Event `to_map/1` | Atom vs binary keys — silently stores `event_type = undefined` | 2026-03-05 |
| 35 | gen_server Self-Call Deadlock | `gen_server:call` from within the same server deadlocks | 2026-03-05 |
| 36 | Hex Packages Without debug_info | `no_debug_info` breaks consumers' dialyzer | 2026-03-05 |
| 37 | Flattening Event Envelopes Into Business Data | `maps:merge(Data, Envelope)` silently overwrites business fields | 2026-03-06 |
| 38 | Emoji Literals in esqlite3 SQL Strings | 4-byte UTF-8 emojis in SQL cause `badarg` crash in esqlite3 NIF | 2026-03-06 |
| 39 | Bypassing Evoq Behaviours with Raw gen_servers | Raw gen_server + evoq_subscriptions instead of evoq_event_handler/evoq_projection/evoq_process_manager | 2026-03-07 |
| 40 | Reading Business Fields from Event Envelope | `get_field(license_id, Event)` in handler — silently returns `undefined`, business data is in `Event.data` | 2026-03-08 |
| **41** | **🔥🔥🔥 Reading from Read Models During Event Flow** | **THE cardinal sin: PMs/projections/handlers reading from read models instead of using event data — causes race conditions, tight coupling, and silent failures** | **2026-03-08** |
| 42 | Silent Catch-All in Message Handlers | Catch-all discards unknown actions without logging — hides payload mismatches | 2026-03-26 |
| 43 | Dual Subscription Registries | Gateway streams and peer handlers are separate — delivery must check both | 2026-03-26 |
| 44 | Callback Payload Format Assumptions | Library wraps payload in `#{topic, payload}` but consumer expected raw JSON | 2026-03-26 |
| 45 | Fire-Once Publishing Over Unreliable Transport | Mesh announce fired once — lost on relay restart or late subscribers | 2026-03-26 |
| 46 | No Subscription Replay on Reconnect | QUIC reconnect doesn't re-send SUBSCRIBE — gateway loses all subscriptions | 2026-03-26 |
| 47 | Eager Peer Connection Explosion | Persistent peer_system per discovered peer → 692 handlers on 4 nodes | 2026-03-26 |
| **48** | **DEBUG Logging on Critical Failure Paths** | **"Found 0 subscribers" at DEBUG level — failures invisible in production** | **2026-03-26** |

---

## Debugging Rule

**In case of a bug: write a test BEFORE going wild in unguided reasoning.**

1. Reproduce the bug in a test (make it fail)
2. Fix the code (make the test pass)
3. THEN explore if more instances exist

Unguided runtime debugging (checking processes, logs, curl) without a failing test is junior behavior. Tests are deterministic, reproducible, and document the bug for posterity.

---

## Detail Files by Topic

### [ANTIPATTERNS_NAMING.md](ANTIPATTERNS_NAMING.md) — Names That Don't Scream

Demons #1, #13, #16, #17. Naming violations where module, event, or command names fail to communicate business intent.

### [ANTIPATTERNS_STRUCTURE.md](ANTIPATTERNS_STRUCTURE.md) — Code Organization Violations

Demons #3, #6, #8, #14, #18, #25. Structural mistakes where code is organized by technical concern instead of business capability.

### [ANTIPATTERNS_DOMAIN.md](ANTIPATTERNS_DOMAIN.md) — Domain Modeling Mistakes

Demons #2, #4, #5, #9. Errors in modeling aggregate lifecycles, parent-child relationships, and domain boundaries.

### [ANTIPATTERNS_EVENT_SOURCING.md](ANTIPATTERNS_EVENT_SOURCING.md) — Aggregates, Events, Envelopes

Demons #10, #23, #33, #34, #37, #40, **#41**. Aggregate callback order, event record handling, evoq+reckondb requirements, map key types, envelope flattening, envelope field extraction, and **reading from read models during event flow (THE cardinal sin)**.

### [ANTIPATTERNS_PROJECTIONS.md](ANTIPATTERNS_PROJECTIONS.md) — Projections & Read Models

Demons #12, #22, #31, #32. Projection timing, manual event emission, inline projections, and PRJ/QRY separation.

### [ANTIPATTERNS_INTEGRATION.md](ANTIPATTERNS_INTEGRATION.md) — Subscriptions, Messaging, Process Managers

Demons #7, #11, #15, #24, #26, #39. pg vs mesh, hope acknowledgments, command IDs, subscription pipeline failures, emitter lifecycle, and bypassing evoq behaviours.

### [ANTIPATTERNS_ERLANG.md](ANTIPATTERNS_ERLANG.md) — Erlang/OTP Gotchas

Demons #19, #20, #21, #35, #38. esqlite3 return types and argument order, eager map defaults, gen_server self-call deadlocks, and emoji literals in SQL.

### [ANTIPATTERNS_RELEASE.md](ANTIPATTERNS_RELEASE.md) — Release, Testing & Packaging

Demons #27, #28, #29, #30, #36. Hardcoded IDs, missing tests, plugin discovery routes, version bumping, and hex packaging.

### [ANTIPATTERNS_MESH_PUBSUB.md](ANTIPATTERNS_MESH_PUBSUB.md) — Mesh Pub/Sub: The 13-Bug Marathon

Demons #42, #43, #44, #45, #46, #47, #48. Silent catch-alls, dual registries, payload wrapper assumptions, fire-once publishing, missing subscription replay, eager connection explosion, and invisible DEBUG logging. All from a single debugging session where one game announcement needed 13 fixes to cross the mesh.

---

*We burned these demons so you don't have to. Keep the fire going.*
