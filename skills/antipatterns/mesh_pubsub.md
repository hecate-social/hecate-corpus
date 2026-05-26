---
title: "ANTIPATTERNS: Mesh Pub/Sub"
layer: skill
audience: [agent, human]
stage: stable
---

# ANTIPATTERNS: Mesh Pub/Sub — The 13-Bug Marathon

*Demons from a single debugging session (2026-03-26) where sending one game announcement across the mesh required 13 fixes. Every link in the chain failed silently.*

[Back to Index](INDEX.md)

---

## The Meta-Lesson

> **"ok" without observability is a lie.**
>
> Every component in the pubsub chain returned `ok`. The sender said "sent." The relay said "delivered." The receiver said "callback invoked." But no challenger joined.
>
> 13 fixes later — across 3 repos (macula, hecate-daemon, macula-boot) — the announcement finally reached the seeker. The root cause wasn't one bug. It was a chain of silent failures, each hidden by the one before it.

---

## 🔥 Demon 42: Silent Catch-All in Message Handlers

**Date exorcised:** 2026-03-26
**Where it appeared:** `mpong_lobby_seeker.erl` — `handle_mesh_action/5`
**Cost:** Final bug in a 13-fix chain. Seeker received announcements but silently dropped them.

### The Antipattern

A pattern-match catch-all that silently discards unrecognized messages:

```erlang
%% WRONG — silent discard
handle_mesh_action(<<"hosted">>, ...) -> try_mesh_join(...);
handle_mesh_action(_, _, _, _, State) ->
    {noreply, State}.  %% Unknown action? Shrug.
```

When `Action` was `undefined` (due to payload format mismatch — Demon #44), this catch-all silently ate every message. No log, no warning, no trace.

### The Rule

> **Catch-all clauses in message handlers MUST log at WARNING level.**
> **"I received something I don't understand" is always worth knowing.**

### The Correct Pattern

```erlang
handle_mesh_action(<<"hosted">>, ...) -> try_mesh_join(...);
handle_mesh_action(<<"withdrawn">>, ...) -> handle_withdrawal(...);
handle_mesh_action(Action, _HostNodeId, _OurNode, _Msg, State) ->
    logger:warning("[mpong_seeker] Unknown mesh action: ~p", [Action]),
    {noreply, State}.
```

### Prevention

Before writing ANY catch-all clause, ask:
1. **Can this match in production?** If yes → log it
2. **Is "do nothing" the correct response?** If no → crash or log error
3. **Would I want to know this happened during debugging?** → Then log it

---

## 🔥 Demon 43: Dual Subscription Registries

**Date exorcised:** 2026-03-26
**Where it appeared:** macula gateway — `handle_pubsub_route_deliver` only checked gateway streams
**Cost:** pubsub_route messages arrived at beam nodes but were never delivered to application callbacks

### The Antipattern

Two separate subscription registries existing in the same system without either knowing about the other:

1. **Gateway pubsub** (`macula_gateway_pubsub`) — stream-based subscriptions from remote clients
2. **Peer pubsub_handler** (`macula_pubsub_handler`) — callback-based subscriptions from `macula:subscribe()`

When a `pubsub_route` arrived at the gateway, `deliver_local` only checked registry #1. Application subscriptions (registry #2) were invisible.

### Why It's Deadly

- `_mesh.peer.connected` worked because the gateway auto-subscribes streams to it
- `games.available` didn't because it was only in the peer's pubsub_handler
- No error, no warning — `deliver_local` found 0 subscribers and returned `ok`

### The Rule

> **When adding a new delivery path, grep for ALL places that answer "who is subscribed?"**
> **If there are two registries, delivery must check both.**

### The Fix

`handle_pubsub_route_deliver` now delivers to gateway streams AND to all local `macula_pubsub_handler` processes via gproc lookup.

---

## 🔥 Demon 44: Callback Payload Format Assumptions

**Date exorcised:** 2026-03-26
**Where it appeared:** `mpong_lobby_seeker.erl` — expected raw JSON, got macula wrapper
**Cost:** Seeker received every announcement but couldn't read any of them

### The Antipattern

Assuming the format of a callback payload without checking the library's contract:

```erlang
%% The subscription callback
fun(Msg) -> Self ! {mesh_lobby, Msg}, ok end

%% What the seeker expected Msg to be:
#{<<"action">> => <<"hosted">>, <<"game_id">> => ...}

%% What macula actually delivers:
#{topic => <<"io.macula...">>, matched_pattern => ..., payload => #{<<"action">> => ...}}
```

The actual game data was INSIDE the `payload` field. `maps:get(<<"action">>, Msg, undefined)` returned `undefined` because `<<"action">>` doesn't exist at the top level.

### Why It's Insidious

- No crash — `maps:get/3` with a default silently returns `undefined`
- No type error — both the wrapper and the payload are maps
- The callback IS invoked — logging shows "Found 1 subscription(s)"
- The silent catch-all (Demon #42) ate the `undefined` action

### The Rule

> **Callback payload format is an API contract. Document it. Test it.**
> **Never assume — read the library code or write a test that prints the payload.**

### The Correct Pattern

```erlang
%% Defensive extraction — handle both raw and wrapped payloads
handle_mesh_lobby(PublishData, State) ->
    RawPayload = case PublishData of
        #{payload := P} -> P;  %% macula wrapper
        _ -> PublishData        %% raw (future-proof)
    end,
    Msg = decode_payload(RawPayload),
    ...
```

### Prevention

When subscribing to ANY pub/sub system:
1. **Log the raw callback payload once** — `logger:info("Received: ~p", [Msg])`
2. **Check the library's invoke_callbacks** — what wrapper does it add?
3. **Write a test** that publishes and asserts the callback receives the expected format

---

## 🔥 Demon 45: Fire-Once Publishing Over Unreliable Transport

**Date exorcised:** 2026-03-26
**Where it appeared:** `mpong_lobby_server.erl` — mesh announce fired once
**Cost:** Game announcement lost when boot.macula.io restarted

### The Antipattern

Publishing a message once and assuming it will reach all subscribers:

```erlang
%% WRONG — fire and forget
handle_info({init_mesh, GameId, HostNode, MaxPlayers}, State) ->
    MeshAdvRef = advertise_join_rpc(GameId),
    advertise_game:announce(#{game_id => GameId, ...}),
    {noreply, State}.
    %% If boot restarts, subscribers join late, or network blips: announcement gone forever
```

The LAN path already had a repeat timer (`broadcast_lobby` every 2s). The mesh path didn't.

### The Rule

> **Mesh pub/sub is unreliable by nature. Publishers must repeat until the condition expires.**
> **"Sent ok" ≠ "received." The transport may lose the message, the relay may restart, subscribers may join late.**

### The Correct Pattern

```erlang
%% Repeat announcement while lobby is waiting
handle_info({init_mesh, ...}, State) ->
    advertise_game:announce(...),
    erlang:send_after(?BROADCAST_MS, self(), broadcast_mesh),  %% Keep announcing
    {noreply, State};

handle_info(broadcast_mesh, #lobby{state = waiting} = State) ->
    advertise_game:announce(...),
    erlang:send_after(?BROADCAST_MS, self(), broadcast_mesh),
    {noreply, State};

handle_info(broadcast_mesh, State) ->
    {noreply, State}.  %% No longer waiting — stop
```

---

## 🔥 Demon 46: No Subscription Replay on Reconnect

**Date exorcised:** 2026-03-26
**Where it appeared:** `macula_connection.erl` — reconnect didn't re-send SUBSCRIBE messages
**Cost:** boot.macula.io had 1 subscriber instead of 5 after restart

### The Antipattern

Reconnecting a QUIC connection without replaying application-level session state:

```
T=0: connect → SUBSCRIBE "games.available" → gateway stores stream subscription
T=5: boot restarts → QUIC connection drops
T=6: auto-reconnect → new QUIC stream → gateway has NO subscriptions
T=7: publish "games.available" → gateway deliver_local finds 0 subscribers
```

The connection reconnected. The QUIC transport was healthy. But the gateway's subscription state was on the OLD stream — gone forever.

### The Rule

> **QUIC reconnection ≠ session resumption. Application state (subscriptions, registrations) must be explicitly replayed after reconnect.**

### The Correct Pattern

```erlang
complete_connection_setup(Conn, Stream, Host, Port, State) ->
    ...
    %% After successful reconnect, replay subscriptions
    replay_peer_subscriptions(State).

replay_peer_subscriptions(#state{realm = Realm, peer_id = PeerId}) ->
    case gproc:lookup_local_name({pubsub_handler, Realm, PeerId}) of
        undefined -> ok;
        PubSubPid -> gen_server:cast(PubSubPid, replay_subscriptions)
    end.
```

---

## 🔥 Demon 47: Eager Peer Connection Explosion

**Date exorcised:** 2026-03-26
**Where it appeared:** `macula_peer_discovery.erl` — created persistent connections to every discovered peer
**Cost:** 692 pubsub_handlers on a 4-node cluster, reconnect storms on boot restart

### The Antipattern

Creating a persistent connection (with full supervisor tree) to every peer discovered via DHT:

```erlang
%% WRONG — persistent connection to every discovered peer
connect_to_peer(PeerInfo, State) ->
    ...
    macula_peers_sup:start_peer(PeerUrl, #{realm => Realm, node_id => MyNodeID}).
    %% Creates: macula_peer_system supervisor
    %%   ├── macula_connection (persistent QUIC)
    %%   ├── macula_pubsub_handler
    %%   ├── macula_rpc_handler
    %%   └── macula_advertisement_manager
```

With 4 nodes × multiple connections each = hundreds of handlers. When boot.macula.io restarted, ALL reconnected simultaneously → thundering herd.

### The Rule

> **DHT discovery populates the routing table. Actual connections are JIT.**
> **Only ONE persistent connection should exist: the application's client connection to bootstrap.**
> **`macula_peer_connector` already handles JIT QUIC with connection pooling (94.5% hit rate).**

### The Correct Pattern

```erlang
%% Discovery only updates routing table — no persistent connections
connect_to_peer(PeerInfo, State) ->
    PeerNodeID = get_peer_field(PeerInfo, node_id, <<"node_id">>),
    PeerUrl = build_url(PeerInfo),
    ?LOG_INFO("Discovered peer ~s at ~s (routing table only)", [...]),
    ok.  %% JIT connections via peer_connector handle actual delivery
```

### Connection Budget

| What | Count | Why |
|------|-------|-----|
| Application → bootstrap | 1 | Stream-based relay for pub/sub |
| Discovery → peers | 0 persistent | JIT via peer_connector |
| Incoming from peers | N (gateway accepts) | Gateway handles incoming QUIC |

### Prevention

Monitor connection count as a health metric. If you can't explain why there are N connections, something is wrong.

---

## 🔥 Demon 48: DEBUG Logging on Critical Failure Paths

**Date exorcised:** 2026-03-26
**Where it appeared:** `send_to_subscriber`, `deliver_local`, `send_via_pool` — all at DEBUG level
**Cost:** Failures invisible in production logs (default level: INFO)

### The Antipattern

Logging failures and unexpected states at DEBUG level:

```erlang
%% WRONG — failure invisible at INFO log level
send_to_subscriber(...) ->
    case macula_peer_connector:send_message(...) of
        ok -> ?LOG_DEBUG("Sent pubsub_route...");    %% Success at DEBUG — invisible
        {error, R} -> ?LOG_DEBUG("Direct send failed: ~p", [R])  %% Failure at DEBUG — invisible
    end.

deliver_local(Topic, ...) ->
    Subscribers = find_matching_subscribers(Topic, State),
    ?LOG_DEBUG("Found ~p subscribers", [length(Subscribers)]).  %% 0 subscribers at DEBUG — invisible
```

### The Rule

> **Unexpected-but-not-crashing states MUST log at WARNING.**
> **"Found 0 subscribers" is not a DEBUG fact — it's a WARNING that delivery will silently fail.**

### Logging Level Guide for Pub/Sub

| Situation | Level | Why |
|-----------|-------|-----|
| Sent message successfully | INFO | Confirms delivery path works |
| Found N subscribers (N > 0) | INFO | Confirms subscription health |
| Found 0 subscribers | WARNING | Delivery will silently fail |
| Send failed, trying fallback | WARNING | Degraded path |
| Send failed, no fallback | ERROR | Message lost |
| Unknown action in handler | WARNING | Payload mismatch or new action |

---

## The Chain: How 13 Bugs Hid Behind Each Other

```
Bug 1: Node_id divergence          → Fixed → revealed...
Bug 2: Quicer 3-tuple crash        → Fixed → revealed...
Bug 3: DHT zombie subscriptions    → Fixed → revealed...
Bug 4: Binary key mismatch         → Fixed → revealed...
Bug 5: pubsub_route not delivered   → Fixed → revealed...
Bug 6: DEBUG logging hid failures  → Fixed → revealed...
Bug 7: Gateway didn't notify peer handlers  → Fixed → revealed...
Bug 8: No visibility into peer PUBLISH      → Fixed → revealed...
Bug 9: 692 handlers from eager connections  → Fixed → revealed...
Bug 10: Wrong default realm                 → Fixed → revealed...
Bug 11: No subscription replay on reconnect → Fixed → revealed...
Bug 12: Fire-once announcement              → Fixed → revealed...
Bug 13: Payload wrapper mismatch in seeker  → Fixed → IT WORKS
```

### The Prevention

**One integration test would have caught bugs 7, 11, 12, and 13 on day one:**

```erlang
mesh_pubsub_e2e_test() ->
    %% Start two nodes, publish on A, verify callback on B
    {ok, ClientA} = macula:connect(BootUrl, #{realm => Realm}),
    {ok, ClientB} = macula:connect(BootUrl, #{realm => Realm}),

    Self = self(),
    {ok, _} = macula:subscribe(ClientB, <<"test.topic">>,
        fun(Msg) -> Self ! {received, Msg}, ok end),

    timer:sleep(1000),  %% Let subscription propagate

    ok = macula:publish(ClientA, <<"test.topic">>, <<"hello">>),

    receive
        {received, #{payload := <<"hello">>}} -> ok
    after 5000 ->
        error(pubsub_delivery_failed)
    end.
```

**This test did not exist.**

---

*13 demons exorcised in one session. Each one hiding behind the last. The lesson isn't "fix bugs faster" — it's "test the chain, not the links."*
