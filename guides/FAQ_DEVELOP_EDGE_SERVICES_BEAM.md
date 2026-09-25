---
title: "FAQ: Developing Macula Edge Services in Erlang, Elixir, Gleam"
layer: guide
audience: [agent, human]
stage: stable
---

# FAQ: How Do I Develop Macula/Hecate Edge Services in BEAM Languages?

[Back to FAQ index](FAQ.md) · [Back to corpus index](../INDEX.md)

For Go/Rust/C#/F#/PHP, see
[Go](FAQ_DEVELOP_EDGE_SERVICES_GO.md),
[Rust](FAQ_DEVELOP_EDGE_SERVICES_RUST.md),
[C#/F# (.NET)](FAQ_DEVELOP_EDGE_SERVICES_DOTNET.md), or
[PHP](FAQ_DEVELOP_EDGE_SERVICES_PHP.md)
instead.

There are two starting points, and which one you want depends on what
you're building:

- **Raw `macula` SDK** ([`macula-io/macula`](https://github.com/macula-io/macula),
  Erlang, 96 modules at 10.17.0) — the low-level primitives: connect, publish,
  subscribe, advertise, call. Use this for a standalone tool or a small
  daemon that only needs one or two mesh operations.
- **`hecate_om_service` behaviour** (`hecate-services/hecate-om`) — the
  scaffold-generated shape every production hecate-service in this
  workspace actually uses. Six callbacks (`info/0`, `start/1`, `stop/1`,
  `health/0`, `capabilities/0`, `identity_spec/0`) get you generic
  capability advertisement, TTL management, org-scoped registration, and
  `/health` wiring for free, centrally maintained — see
  [`skills/antipatterns/structure.md`, Demon 59](../skills/antipatterns/structure.md)
  for exactly what goes wrong when a service reinvents this instead. **This
  is the recommended default** for anything meant to run as a real
  hecate-service, not just a script.

Since Erlang, Elixir, and Gleam all run on the same BEAM VM and share the
same module/function calling convention, all three call the *identical*
Erlang SDK modules — there's no separate Elixir or Gleam package, by
design (see this workspace's own "no Elixir wrappers" rule: call Erlang
directly with `:module.function()` / `module:function()` syntax).

---

## Install: `macula` is a real, published Hex package

**`macula`** is published on [hex.pm](https://hex.pm/packages/macula) —
current version **10.17.0** — not something you pull via a git
dependency. Depend on it with a loose `~>` constraint (this workspace's
own convention — exact pins block coordinated library updates), the same
shape every real consumer in this workspace uses:

```erlang
%% rebar.config
{deps, [
    {macula, "~> 10.0"}
]}.
```

```elixir
# mix.exs
defp deps do
  [
    {:macula, "~> 10.1"}
  ]
end
```

Docs: [`macula.hexdocs.pm`](https://macula.hexdocs.pm/). The polyglot
ports (`macula-go`, `macula-rust`, `macula-dotnet`, `macula-php`) are
separate, much younger packages on their own 0.x version lines — don't
confuse this Erlang package's 10.x maturity with theirs.

## Erlang — raw SDK

PubSub (from `macula`'s own `docs/guides/shared/CONNECTING_GUIDE.md`):

```erlang
Seeds = [<<"quic://relay-1.example.com:4433">>, <<"quic://relay-2.example.com:4433">>],
{ok, Pool} = macula:connect(Seeds, #{}),
ok         = macula:publish(Pool, Realm, Topic, Payload),
{ok, _Sub} = macula:subscribe(Pool, Realm, Topic, self()),
ok = macula:close(Pool).
```

Inbound events arrive as messages: `{macula_event, SubRef, Topic, Payload, Meta}`.

RPC provider (from `macula`'s `docs/guides/rpc/RPC_GUIDE.md`):

```erlang
-module(math_service).
-behaviour(macula_response).
-export([init/1, handle_request/2]).

init(_Args) -> {ok, []}.
handle_request(#{<<"a">> := A, <<"b">> := B}, State) ->
    {reply, A + B, State}.
```
```erlang
{ok, _Sup} = macula_response:advertise(Pool, Realm, Procedure, math_service, []).
```

The SDK's own guides cover every primitive beyond this in depth:
`rpc/RPC_PROTOCOL.md`, `pubsub/PUBSUB_GUIDE.md`, `streaming/STREAMING_GUIDE.md`,
`content/CONTENT_GUIDE.md`, `shared/MRI_GUIDE.md`,
`shared/AUTHORIZATION_GUIDE.md`.

**A genuinely minimal real "hello world"**, if the guides above feel too
low-level to start from: [`hecate-social/hecate-stub`](https://github.com/hecate-social/hecate-stub) —
"Connects to a Macula relay, announces geo identity, and serves a health
endpoint. That's it." Real `Dockerfile`, a real `docker run` one-liner
(`MACULA_RELAYS`, `HECATE_MESH_REALM` default `io.macula`,
`HECATE_GEO_CITY`/`COUNTRY`/`LAT`/`LNG`, `HEALTH_PORT` default `8080`), and
a real `rebar3 shell` local-run path. Small enough to read start to finish
in a sitting — a better first read than a full multi-app production
service.

## Erlang — `hecate_om_service` (the recommended path for a real service)

```bash
rebar3 new hecate_service
```

scaffolds the standard shape. The shortest real, currently-deployed
example — one capability, no pubsub authority — is
`hecate-services/hecate-stations/apps/hecate_stations/src/hecate_stations_service.erl`:

```erlang
-module(hecate_stations_service).
-behaviour(hecate_om_service).

-export([info/0, start/1, stop/1, health/0, capabilities/0, identity_spec/0]).
%% Two more exports beyond the six required callbacks, for its
%% barrel_docdb read model (opt-in, not part of the behaviour itself):
-export([read_model_id/0, data_dir/0]).

info() ->
    #{name => <<"hecate-stations">>,
      version => <<"0.1.0">>,
      description => <<"Live, filterable directory of macula stations: geo, "
                        "health, and direct-dial IP, so clients never "
                        "hand-maintain a station list">>}.

start(_Opts) -> hecate_stations_sup:start_link().
stop(_State) -> ok.
health() -> ok.

capabilities() ->
    [#{name    => <<"hecate_stations.list_stations">>,
       version => 1,
       handler => {list_stations, []}}].

identity_spec() ->
    #{scope => <<"hecate-stations">>, actions => [], resources => [], ttl_days => 30}.

read_model_id() -> <<"hecate_stations">>.
data_dir() -> os:getenv("HECATE_DATA_DIR", "/var/lib/hecate-stations").
```

Declaring `handler` here is the whole story — `hecate_om:boot/1` handles
wiring the mesh pool, publishing the signed `procedure_advertisement` DHT
record, periodic re-advertisement, and TTL, generically, for every service
that uses this path. See
[FAQ: How do I deploy my own hecate service?](FAQ_DEPLOY_HECATE_SERVICES.md)
for what happens after `rebar3 eunit` passes locally.

## Elixir

Real precedent exists across several current Elixir hecate-services in
this workspace (`hecate-services/hecate-whiteboard`, `macula-realm`,
`macula-portal`) — Elixir calls the Erlang SDK's modules directly, exactly
as the "no wrapper" convention prescribes. Two small, complete, real
examples from `hecate-whiteboard`:

Publisher (`guide_board_lifecycle/lib/guide_board_lifecycle/mesh_publisher.ex`,
the full file):
```elixir
defmodule GuideBoardLifecycle.MeshPublisher do
  # Trivial fire-and-forget :macula_publisher callback shared by every
  # mesh-fact emitter in this app -- none of them need to react to the
  # publish outcome, they just want the supervised pid/mesh-fact
  # machinery macula_publisher already provides around a bare
  # macula:publish/4. Mirrors hecate-tube's tube_mesh_publisher.erl.
  @behaviour :macula_publisher

  @impl true
  def init(_args), do: {:ok, nil}

  @impl true
  def handle_published(result, state) do
    require Logger
    Logger.info("[MeshPublisher] outcome: #{inspect(result)}")
    {:stop, :normal, state}
  end
end
```
called as:
```elixir
:macula_publisher.start_link(GuideBoardLifecycle.MeshPublisher, pool, realm, topic, fact, [])
```

Subscriber, started under a `DynamicSupervisor` — this `spec`/`start_child`
call itself is real, but it's only ever reached from inside a ~60-line
retry-loop `GenServer`
(`track_presence/lib/track_presence/peer_departed_mesh_subscriber_starter.ex`)
that waits for `:hecate_om.mesh_handles()` to succeed first, same reason
as the Phoenix LiveView FAQ's "Starting the subscriber" section — a naive
one-shot call here races the mesh pool's async init and loses:
```elixir
spec = %{
  id: TrackPresence.PeerDepartedMeshSubscriber,
  start: {:macula_subscriber, :start_link,
    [TrackPresence.PeerDepartedMeshSubscriber, pool, realm,
     TrackPresence.PeerDepartedMeshSubscriber.topic(), [], %{}]},
  restart: :permanent
}
DynamicSupervisor.start_child(TrackPresence.MeshSubscriberSupervisor, spec)
```

Both patterns repeat throughout these codebases: an Elixir module
implementing `:macula_publisher`/`:macula_subscriber`'s Erlang behaviour
callbacks (`init/1`, `handle_published/2` or the subscriber equivalent),
started via `:module.start_link(...)` with the pool/realm/topic obtained
from `:hecate_om.mesh_handles()`. `macula-energy-mesh-poc` (an older
proof-of-concept) has its own per-app Elixir wrapper module around the
Erlang client — that predates the current no-wrapper convention; treat it
as historical, not a pattern to copy.

## Gleam

The first real Gleam edge service in this workspace is
`macula-services/mcl-bookclub-gleam` (2026-09-25): the Bookclub-on-Mesh
twin ported to Gleam, deployed beside the Erlang and Elixir clubs. It is
the reference to copy from — full domain (CMD/PRJ/QRY desks), the
`mcl_om_service` contract, mesh fact emitters, a cowboy LAN admin, 91
tests, all green. The constructed example that used to live here is
replaced below by the binding shapes that actually work.

Gleam calls the Erlang SDK directly (`@external`), exactly like Elixir —
the "no wrapper" convention holds. The real binding layer
(`src/mcl_bookclub_gleam/internal/`):

```gleam
// internal/mesh.gleam -- the mesh edge, in the shape that ran in
// production. Gleam's Result IS Erlang's {ok, V} | {error, E}.
@external(erlang, "mcl_om", "boot")
pub fn boot(service_module: dynamic.Dynamic) -> Result(dynamic.Dynamic, dynamic.Dynamic)

@external(erlang, "macula_topic", "app_fact")
pub fn app_fact_topic(
  realm_name: String,
  org: String,
  app: String,
  domain: String,
  name: String,
  version: Int,
) -> String

@external(erlang, "mcl_om_wire", "field")
pub fn field(key: dynamic.Dynamic, payload: Payload) -> dynamic.Dynamic
```

The lessons this first build paid for, each pinned by a test or comment in
mcl-bookclub-gleam:

- **A Gleam project is ONE OTP app** (no umbrella). The many-app division
  layout becomes screaming-architecture FOLDERS inside the package; the
  division boundary is enforced by tests (source-scanning bans), exactly
  as the Erlang twin does it. Boot order that the twins get from their
  application order is done in the app's `start/2`: start the division
  supervisors, THEN `mcl_om:boot/1` — its evoq replay must find the
  `deliver`-policy projections registered.
- **gleam_stdlib 1.x has no `dynamic.from/1`** — only typed constructors
  (`dynamic.string`, `dynamic.int`, `dynamic.properties`, ...). Arbitrary
  terms (atoms, pids, tuples) need an identity FFI helper.
- **gleam_erlang's `charlist` module stores a BINARY.** A real charlist
  (the twins' `data_dir/0` contract; esqlite paths) comes from
  `binary_to_list` in an FFI module. Relatedly, `esqlite3:open/1` REJECTS
  a binary path outright.
- **`process.new_name/1` always appends a unique suffix** — a fixed-name
  registration (health-pinged stores) needs the exact atom via an FFI.
  gleam actors also receive only through their subject envelope
  (`{Name, Message}` for named actors), never raw messages.
- **Many Erlang APIs return a bare `ok` atom** (`filelib:ensure_dir`,
  `macula:publish`, `reckon_gater_stream_id:validate`) or three-tuples
  (`evoq_command_router:dispatch`). Neither shapes Gleam's `Result`; one
  small FFI module reshapes them all.
- **The release: rebar3's relx assembles `gleam build` output** (Gleam
  owns compilation; `{project_app_dirs, []}` makes rebar3 skip its own).
  The staged `.app` needs its `mod` entry patched in, and every beam
  unioned into the modules list — `gleam_otp`/`gleam_erlang` ship beams
  their own `.app` files omit, which is an `undef` at boot otherwise.

See also: [FAQ: How do I add event sourcing to a new hecate service?](FAQ_ADD_EVENT_SOURCING.md)

## See also

- [FAQ: Developing Edge Services in Go](FAQ_DEVELOP_EDGE_SERVICES_GO.md)
- [FAQ: Developing Edge Services in Rust](FAQ_DEVELOP_EDGE_SERVICES_RUST.md)
- [FAQ: Developing Edge Services in C#/F# (.NET)](FAQ_DEVELOP_EDGE_SERVICES_DOTNET.md)
- [FAQ: Developing Edge Services in PHP](FAQ_DEVELOP_EDGE_SERVICES_PHP.md)
- [FAQ: How do I deploy my own hecate service?](FAQ_DEPLOY_HECATE_SERVICES.md)
- [FAQ: How do I add event sourcing to a new hecate service?](FAQ_ADD_EVENT_SOURCING.md) — the CMD department, once a raw `hecate_om_service` scaffold isn't enough
- [FAQ: How do I authorize a procedure or topic with UCAN?](FAQ_AUTHORIZE_WITH_UCAN.md) — gating a served procedure, including the Erlang reference implementation
- [`skills/antipatterns/structure.md`, Demon 59](../skills/antipatterns/structure.md) — why `hecate_om_service.capabilities/0` beats hand-rolling mesh advertisement
