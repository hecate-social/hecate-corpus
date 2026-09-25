---
title: Demons We've Exorcised
layer: skill
audience: [agent, human]
stage: reversed
---

# antipatterns/INDEX.md — Demons We've Exorcised

*Mistakes we've made and corrected. Read this. Don't repeat them.*

> ⚠ **`hecate-daemon`, `hecate-web`, and `hecate-gitops` have been
> REMOVED (2026-09-05) and no longer exist anywhere in this workspace.**
> Several demons below name them as where a bug was originally found —
> that provenance is accurate history and untouched. But do not treat
> any of the three as live infrastructure to read, extend, deploy
> through, or register a new component with. See the authoritative
> note in [`../../philosophy/HECATE_TIER_MODEL.md`](../../philosophy/HECATE_TIER_MODEL.md)
> (2026-09-05 amendment) for what replaced them and why.

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
| **49** | **Discarding `evoq_dispatcher:dispatch/2`'s Return Value** | **`_ = dispatch(...)` throws away the only error channel — events vanish silently** | **2026-05-26** |
| **50** | **Daemon-as-Mesh-Middleman** | **L2-shaped work bridged through `hecate-daemon`'s `/api/mesh/publish` — wrong identity, wrong layer dependency, defeats reckon-db** | **2026-05-28** |
| **51** | **🔥🔥🔥 Human-Readable Aggregate IDs as Reckon Stream IDs** | **evoq AggregateId IS the reckon stream id (`^[a-z]{1,32}-[a-f0-9]{32}$`); a human slug fails the regex and a bare `catch` eats the rejection — empty store, no errors** | **2026-05-31** |
| **52** | **Duplicate `{profiles}` Tuple in rebar.config** | **rebar3 keeps only the LAST top-level `{profiles}` — second one shadows `prod`, drops `include_erts` → ERTS-less release → exit-127 crash loop** | **2026-05-31** |
| **53** | **🔥🔥🔥 Comments That State Intent as Fact** | **The comment records the design you meant to write; the code does something else; both land in the same commit. Four in one day.** | **2026-08-07** |
| 54 | A Test for the Name and None for the Fold | The untested part is the part you were confident about — and every fault sits at a seam no test crosses | 2026-08-07 |
| **55** | **🔥🔥🔥 Believing That Writing It Down Prevents It** | **Demon 23 was in this index, correctly described, and got repeated in full five months later. The index is prose too.** | **2026-08-07** |
| 56 | Correct Behaviour With No Reporting | A graceful fallback that never says so, whose healthy and failed states publish identical numbers | 2026-08-07 |
| **57** | **🔥🔥🔥 The Silent No-Op Edit** | **A scripted replacement whose anchor does not match changes nothing and reports success. Three times in one day: a red-check that was never red, a broken build, and every chart on a live page blank.** | **2026-08-08** |
| 58 | History Narration in Operational Docs | A PLAN/guide/role file says "corrected 2026-09-01, first pass said X, second pass says Y" instead of just stating Y — and the moment a third pass happens, the narration is stale too, on top of being noise a RAG-fed reader never needed | 2026-09-01 |
| 59 | Hand-Rolled Mesh Capability Advertising | A service calls `macula_response:advertise_direct` itself instead of declaring the capability in `hecate_om_service:capabilities/0` — library fixes (TTL, `reuse_sup`) never reach the duplicate; hit twice, five months apart, on the same file | 2026-09-01 |
| **61** | **🔥🔥🔥 The Store That Stores Without Indexing** | **A barrel write with no vector, and a read-modify-write that drops the one it had (`get_doc` never returns `_embedding`) — both return `ok`, both silently remove the document from semantic search while exact fetch keeps working perfectly** | **2026-09-02** |
| 60 | Mesh RPC Payloads Arrive Atom-Keyed AND CBOR-Value-Wrapped | Two compounding hazards, fixed a day apart: macula's frame decoder atomizes an inbound payload's keys (a hard `#{<<"key">> := V}` match silently never matches), AND a CBOR text-string value decodes to `{text, binary()}`, not a bare binary — fixing the keys alone still didn't resolve the live symptom until a diagnostic log found the second one | 2026-09-01 |
| 62 | A Test That Passes For a Different Layer's Reason | A TTL-config-passthrough test slept past `expires_at` and asserted the doc was gone — passed even with the passthrough completely stubbed out, because barrel_docdb's own reads treat expired docs as gone unconditionally, sweep config or not. Caught by stubbing the fix and demanding RED first | 2026-09-05 |
| 63 | Existence Mistaken for Freshness in a Build Cache | `build-nifs.sh` skipped recompiling a NIF whenever the `.so` already existed — no comparison against source mtime, so every "clean eunit" that day silently tested a stale binary, once for a real security fix. A force-rebuild env var was also dead code because the existence check ran before it was ever consulted | 2026-09-05 |
| 64 | A Split Name Doesn't Mean a Split Namespace | A rename to fix a PyPI collision kept the old import name, citing beautifulsoup4/bs4 as precedent — but bs4 is a name nothing else uses, while the kept name WAS the colliding package's own real import namespace. A human's one question caught it; nobody checked before that | 2026-09-05 |
| **65** | **Mesh Pubsub Facts Arrive `{text, Bin}`-Keyed** | **The frame decoder does NOT atomize pubsub payload keys — Demon 60's atomization was RPC args. A hand-rolled `maps:get` skips every fact while the mesh delivers everything; the observer recorded nothing with zero errors** | **2026-09-25** |
| **66** | **An Unadmitted Service Subscribes Successfully and Receives Nothing** | **Client-side subscribe succeeds (sub_ref held) while the realm refuses routing until the provider grant is issued — held refs, zero facts, zero errors, green health** | **2026-09-25** |
| **67** | **An Invalid Stream Id Raises in the Store Client** | **Dispatch to a bad stream id crashes the aggregate into a restart loop and hangs the registry's synchronous start call — the error never returns; the desk must validate BEFORE dispatch, not only inside the aggregate** | **2026-09-25** |
| **68** | **The Flag→Name Map That Exists Nowhere** | **Status flags live in the CMD status module and the readable names live as SQL literals scattered across N projections, with nothing mapping one to the other — evoq_bit_flags:to_string/2 plus a flag map is the bridge; a projection spelling a status literal is the relapse, refused by a source-grep test. And: when one division needs another's pure functions, depend on the module, never on the app — booting the CMD app into a projection test env drags its mesh emitters in and silently stalls $all delivery** | **2026-09-25** |
| **69** | **Two Providers, One Procedure, One Station — the Registry Holds One** | **A station's advertise registry keys on `(realm, procedure)` and holds ONE advertiser (last direct ADVERTISE wins), so two providers of one procedure that name the same serving station are not both dialable — the SDK names the pool's first-connected link (map-term order = alphabetical host name, not seed order, so seed rotation cannot steer it); the displaced club's CALL is answered by the other club (no target check on inbound CALLs), the caller refuses the mis-signed reply, and a failover that collapsed the timeout into `no_provider` sent the diagnosis into the resolve path. Fix: per-node serving-station choice, wire registration ONLY at the serving station, `no_provider` reserved for "nothing to dial", and a two-provider live fixture** | **2026-09-25** |

---

## ⚠ Before You Add a Demon Here

**Name the mechanism that will refuse it.** A type, a failing test, a lint rule.
This index held Demon 23 for five months and the same bug was written again in
another repository, because a line in a document does not bite. See
[verification.md](verification.md), Demon 55.

---

## Debugging Rule

**In case of a bug: write a test BEFORE going wild in unguided reasoning.**

1. Reproduce the bug in a test (make it fail)
2. Fix the code (make the test pass)
3. THEN explore if more instances exist

Unguided runtime debugging (checking processes, logs, curl) without a failing test is junior behavior. Tests are deterministic, reproducible, and document the bug for posterity.

---

## Detail Files by Topic

### [antipatterns/naming.md](naming.md) — Names That Don't Scream

Demons #1, #13, #16, #17. Naming violations where module, event, or command names fail to communicate business intent.

### [antipatterns/structure.md](structure.md) — Code Organization Violations

Demons #3, #6, #8, #14, #18, #25, **#59**. Structural mistakes where code is organized by technical concern instead of business capability, and **a service hand-rolling shared mesh-advertising infrastructure instead of declaring it through `hecate_om_service:capabilities/0`**.

### [antipatterns/domain.md](domain.md) — Domain Modeling Mistakes

Demons #2, #4, #5, #9. Errors in modeling aggregate lifecycles, parent-child relationships, and domain boundaries.

### [antipatterns/event_sourcing.md](event_sourcing.md) — Aggregates, Events, Envelopes

Demons #10, #23, #33, #34, #37, #40, **#41**, #49, **#51**, **#67**. Aggregate callback order, event record handling, evoq+reckondb requirements, map key types, envelope flattening, envelope field extraction, **reading from read models during event flow (THE cardinal sin)**, discarding `evoq_dispatcher:dispatch/2`'s return value (the only error channel), **human-readable aggregate ids that fail the reckon stream-id regex while a bare `catch` hides the rejection**, and **an invalid stream id that RAISES in the store client — the aggregate crash-loops and the registry call hangs, so the desk must validate before dispatch, not only inside the aggregate**.

### [antipatterns/projections.md](projections.md) — Projections & Read Models

Demons #12, #22, #31, #32, **#68**, **#70**. Projection timing, manual event emission, inline projections, PRJ/QRY separation, and **the flag→name map that exists nowhere: status bits in the CMD status module, readable names as literals scattered across the projections, evoq_bit_flags:to_string/2 + a flag map as the unread bridge — and the trap where "depending on the CMD app" (not just its pure modules) boots mesh emitters into a projection test env and silently stalls $all delivery**.

### [antipatterns/integration.md](integration.md) — Subscriptions, Messaging, Process Managers

Demons #7, #11, #15, #24, #26, #39, **#50**. pg vs mesh, hope acknowledgments, command IDs, subscription pipeline failures, emitter lifecycle, bypassing evoq behaviours, and **bridging L2-shaped work through hecate-daemon's REST API instead of using the Macula SDK directly**.

### [antipatterns/erlang.md](erlang.md) — Erlang/OTP Gotchas

Demons #19, #20, #21, #35, #38, **#60**, **#61**, **#65**. esqlite3 return types and argument order, eager map defaults, gen_server self-call deadlocks, emoji literals in SQL, **a mesh RPC payload whose keys arrive as atoms AND whose CBOR text-string values arrive wrapped in a `{text, Bin}` tuple**, **barrel writes that store a document without indexing it — including the ordinary read-modify-write, since `get_doc` never hands back the `_embedding` it needs to keep**, and **a mesh pubsub payload whose keys arrive as `{text, Bin}` tuples — Demon 60's atomization was RPC, not pubsub, so a hand-rolled `maps:get` skips every fact while the mesh delivers**.

### [antipatterns/release.md](release.md) — Release, Testing & Packaging

Demons #27, #28, #29, #30, #36, #52, #63, **#64**. Hardcoded IDs, missing tests, plugin discovery routes, version bumping, hex packaging, the containerized-reckon_db dynamic-node-name 502, the duplicate `{profiles}` rebar.config tuple that ships an ERTS-less release, a NIF build cache that checks existence instead of freshness, and a rename that kept a name someone else's package already imports.

### [antipatterns/verification.md](verification.md) — What We Wrote Down vs What We Built

Demons #53, #54, #55, #56, #57, **#62**. The gap between a repository's prose and its behaviour.
Every other file here catalogues a technical mistake; this one catalogues the
mistake of believing the technical mistakes were caught. **Read it before adding a
demon to this index**, because it is about why adding one is not enough.

### [antipatterns/mesh_pubsub.md](mesh_pubsub.md) — Mesh Pub/Sub: The 13-Bug Marathon

Demons #42, #43, #44, #45, #46, #47, #48, **#66**. Silent catch-alls, dual registries, payload wrapper assumptions, fire-once publishing, missing subscription replay, eager connection explosion, and invisible DEBUG logging. All from a single debugging session where one game announcement needed 13 fixes to cross the mesh — plus **the unadmitted service that subscribed successfully and received nothing: client-side subscribe succeeds while the realm gates routing until the provider grant is issued**.

### [antipatterns/mesh_capabilities.md](mesh_capabilities.md) — Mesh Capability Advertisement

Demon **#69**. Advertising and resolving capabilities: the station's advertise
registry holds ONE advertiser per `(realm, procedure)` — last direct ADVERTISE
wins — so two providers of one procedure must name different serving stations;
the SDK's first-connected-link choice is map-term order (alphabetical host
name, not seed order), the pin reaches only the most recent advertiser, and a
failover that folds the dial timeout into `no_provider` sends the diagnosis to
the wrong layer. Per-node serving-station choice, `no_provider` reserved for
"nothing to dial", and a two-provider live fixture are the cure.

### [antipatterns/documentation.md](documentation.md) — History Narration in Operational Docs

Demon #58. PLANs, guides, and role files that narrate their own revision history — "corrected on this date, here's what changed and why" — instead of stating current fact. The CHANGELOG is the only place for history.

---

*We burned these demons so you don't have to. Keep the fire going.*
