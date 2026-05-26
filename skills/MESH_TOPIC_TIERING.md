---
title: Mesh Topic Tiering
layer: skill
audience: [agent, human]
stage: stable
---

# Mesh Topic Tiering — Hecate Practical Guide

Companion to the authoritative spec at `macula-io/macula/docs/guides/TOPIC_NAMING_GUIDE.md`.

This guide is for developers writing publishers, subscribers, and RPC procedures inside Hecate-org repos (`hecate-daemon`, `hecate-app-trader`, `hecate-app-martha`, future plugins). It assumes you have read the spec and focuses on **how to pick the right tier** in real Hecate situations.

## TL;DR

Every mesh topic in a Hecate-org repo MUST be built by calling `hecate_topics` (or the equivalent wrapper for your app's repo). Choose one of three builders:

- `realm_fact / realm_hope` — concept owned by the realm authority
- `org_fact / org_hope` — concept shared across multiple beam-campus apps
- `app_fact / app_hope` — concept that lives entirely inside this app

If you find yourself building a topic with `<<...>>` or string interpolation, **stop**. That's a forbidden pattern (see Anti-pattern 1 below).

## How to pick a tier — five questions

Walk through these in order. The first "yes" wins.

1. **Could the realm authority publish or revoke this fact at will?** (membership, ban, identity-key revocation, realm-wide announcements)
   → **realm tier**

2. **Is the schema something every realm member needs to verify or trust, regardless of which app they run?** (cryptographic identity, capability grants signed by the realm)
   → **realm tier**

3. **Is this a concept beam-campus owns that crosses multiple beam-campus apps?** (commercial licensing, org-wide billing, shared settings, cross-app catalog)
   → **org tier**

4. **Will any future beam-campus app outside hecate need to subscribe to this exact topic shape?**
   → **org tier** (otherwise you'll regret it the day you ship the second app)

5. **Is this internal to one app — game state, RPC procedures it advertises, app-specific events?**
   → **app tier**

If you answered "no" to all five, you probably haven't characterized the topic yet. Talk to a reviewer before writing the publisher.

## Worked examples from the audit (2026-04-23)

These are the actual classifications applied during the topic-tiering audit.

### Realm tier (4)

| Topic | Why realm |
|---|---|
| `membership/revoked` | Realm authority revokes a member. The fact's authority IS the realm. |
| `membership/resigned` | The member announces to the realm. Same realm-owned concept, opposite direction. |
| `identity/public_key_announced` | Every realm participant must verify signatures from every DID. The key directory is realm-wide. |
| `identity/public_key_revoked` | Symmetric to the announcement. |

### Org tier (3)

| Topic | Why org |
|---|---|
| `licenses/issued_batch` | beam-campus commercial licensing. Spans hecate today, will span hecate-trader and hecate-martha tomorrow. |
| `licenses/rewrapped_batch` | Same pipeline. |
| `licenses/revoked` | Same pipeline. |

### App tier (9)

| Topic | Why app |
|---|---|
| `mpong/game_advertised`, `mpong/state_broadcast`, `mpong/paddle_moved` | Multiplayer-pong demo. No other beam-campus app will ever care. |
| `mpong/join_game` (hope) | Same. |
| `site/node_announced` | Hecate clustering primitive. App-internal. |
| `llm/chat_to_model`, `llm/check_health`, `llm/list_models` (all hopes) | Capabilities advertised by hecate's `serve_llm`. App-specific RPC surface. |
| `{domain}/replay_events` (hope) | Hecate-internal catch-up RPC. |

## How `hecate_topics` is structured

Inside `hecate-daemon`, `apps/shared/src/hecate_topics.erl` is the wrapper. It pre-fills the realm and the `(org, app)` constants so call-sites stay short.

```erlang
-module(hecate_topics).

-define(ORG, <<"beam-campus">>).
-define(APP, <<"hecate">>).

-export([
    realm_fact/3, realm_hope/3,
    org_fact/3,   org_hope/3,
    app_fact/3,   app_hope/3,
    realm/0
]).

realm_fact(Domain, Name, V) -> macula_topic:realm_fact(realm(), Domain, Name, V).
org_fact(Domain, Name, V)   -> macula_topic:org_fact(realm(), ?ORG, Domain, Name, V).
app_fact(Domain, Name, V)   -> macula_topic:app_fact(realm(), ?ORG, ?APP, Domain, Name, V).

realm_hope(Domain, Name, V) -> macula_topic:realm_hope(realm(), Domain, Name, V).
org_hope(Domain, Name, V)   -> macula_topic:org_hope(realm(), ?ORG, Domain, Name, V).
app_hope(Domain, Name, V)   -> macula_topic:app_hope(realm(), ?ORG, ?APP, Domain, Name, V).

realm() -> application:get_env(hecate, realm, <<"io.macula">>).
```

Future Hecate apps (`hecate-app-trader`, `hecate-app-martha`) ship their own `*_topics` module with their own `?APP` constant. The org constant stays `beam-campus`.

## Call-site discipline

Match the tier of the topic to the builder name. The wrapper makes this trivial:

```erlang
%% Realm-owned: a daemon resigns its realm membership
Topic = hecate_topics:realm_fact(<<"membership">>, <<"resigned">>, 1),

%% Org-owned: a beam-campus license batch is issued
Topic = hecate_topics:org_fact(<<"licenses">>, <<"issued_batch">>, 1),

%% App-owned: hecate's pong game broadcasts state
Topic = hecate_topics:app_fact(<<"mpong">>, <<"state_broadcast">>, 1),

%% App-owned RPC: hecate's serve_llm advertises chat
Topic = hecate_topics:app_hope(<<"llm">>, <<"chat_to_model">>, 1),
```

Reviewers grep for tier mismatches by builder name alone. If a license-related publisher calls `app_fact`, it stands out. No need to mentally parse the resulting string.

## Anti-patterns

### 1. Inline topic strings

```erlang
%% NO — inline string, bypasses validator, drifts silently
Topic = <<Realm/binary, "/_realm/membership/revoked_v1">>.

%% NO — Elixir interpolation, same problem on the other side
topic = "#{realm}.membership.revoked"
```

Always call the builder. CI greps for these patterns and fails the build.

### 2. Wrong-tier publisher

```erlang
%% NO — membership_revoked is realm-owned. Don't publish it under app namespace.
Topic = hecate_topics:app_fact(<<"membership">>, <<"revoked">>, 1),

%% YES
Topic = hecate_topics:realm_fact(<<"membership">>, <<"revoked">>, 1),
```

This is the original smell that motivated the tiering work: the daemon listened on `io.macula/beam-campus/hecate/membership/revoked_v1` while the realm published on `io.macula.membership.revoked`. Two different topic shapes, two different worlds, never met.

### 3. Cross-app subscriber on app-tier topic

If you find yourself wanting to subscribe to another app's topic from your app, the topic is probably mis-tiered. Either:
- The other app should publish at org tier (cross-app concept owned by the org)
- Or there should be a parallel realm-tier topic for the broader announcement

Don't reach across app namespaces. That makes the cross-app coupling invisible.

### 4. Subscribing before you have authority

Subscribing to a realm-tier topic with an `anonymous` MRI is meaningless. The pre-join hecate-daemon currently does this for `membership/revoked` and the listener never has a real DID to match against. The fix is a process-manager that gates subscription on `realm_joined_v1`. See the related task in MEMORY.md.

### 5. Promoting an app-tier topic to org or realm tier without coordination

Tier promotion is a wire break. Every existing subscriber stops receiving messages until they update. If you're promoting:

1. Land the new tier publisher first (without removing the old)
2. Update every subscriber in the same release window
3. Remove the old tier publisher in the next release

Don't do step 1 and forget step 2. Don't combine them in one PR — split for reviewability.

## Versioning rules

- Every topic name carries `_v{N}`. The version is part of the topic, not just the schema.
- Backward-compatible payload additions don't bump the version. A new field that's optional is safe.
- Removing a field, renaming a field, changing a type → bump the version.
- Realm-tier version bumps require coordinated upgrades across all daemons. Plan accordingly.
- Org-tier bumps require coordination across the org's apps.
- App-tier bumps are local and don't need cross-team coordination.

## When in doubt

Ask in `#hecate-architecture`. Better to spend 5 minutes confirming the tier than to discover six months later that the integration has been silently dead because the publisher and subscriber chose different tiers for the same concept.

## Related

- `macula-io/macula/docs/guides/TOPIC_NAMING_GUIDE.md` — authoritative SDK spec
- `macula-io/macula/docs/guides/PUBSUB_GUIDE.md` — pub/sub usage
- `macula-io/macula/docs/guides/RPC_GUIDE.md` — RPC usage
- `hecate-corpus/philosophy/HOPE_FACT_SIDE_EFFECTS.md` — when a fact vs a hope; side-effect semantics
- `hecate-corpus/skills/antipatterns/integration.md` — broader integration anti-patterns
- `hecate-corpus/skills/antipatterns/mesh_pubsub.md` — mesh pub/sub specifics
