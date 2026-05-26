---
title: "Example: Projection Patterns"
layer: example
audience: [agent, human]
stage: stable
---

# Example: Projection Patterns

*Canonical example: Transforming domain events into optimized read models*

---

## The Pattern

A **projection** transforms domain events from the event store into read models optimized for queries. Projections are the "write" side of the query service in CQRS.

```
Event Store (ReckonDB)
    ↓
    ↓ subscription
    ↓
Projection Module
    ↓
    ↓ INSERT/UPDATE
    ↓
Read Model (SQLite)
    ↓
    ↓ SELECT
    ↓
Query Handler
```

**Key principle:** The event store is the single source of truth. Projections derive all read model data from events. Never query external systems in a projection.

---

## Naming Convention

```
{event_type}_to_{read_model}
```

| Event | Read Model | Projection Module |
|-------|------------|-------------------|
| `agent_followed_v1` | followers | `agent_followed_v1_to_followers` |
| `capability_announced_v1` | capabilities | `capability_announced_v1_to_capabilities` |
| `finding_recorded_v1` | findings | `finding_recorded_v1_to_findings` |
| `identity_registered_v1` | identities | `identity_registered_v1_to_identities` |

---

## Wrong Way

### Projection queries external source

```erlang
%%% @doc WRONG: Projection queries external service
-module(user_registered_v1_to_users).

project(#{user_id := UserId} = Event) ->
    %% ❌ WRONG: Calling external service from projection
    {ok, Profile} = external_profile_service:get(UserId),

    %% ❌ WRONG: Mixing event data with external data
    users_store:insert(#{
        user_id => UserId,
        name => maps:get(name, Event),
        avatar_url => maps:get(avatar_url, Profile),  %% External!
        bio => maps:get(bio, Profile)                 %% External!
    }).
```

**Why wrong:**
- Projection now depends on external service availability
- Replay from event store will fail if service is down
- External data may have changed since event was stored
- Breaks single source of truth principle

### Complex joins at read time

```erlang
%%% @doc WRONG: Store normalized, join at read time
-module(get_order_details).

execute(OrderId) ->
    %% ❌ WRONG: Multiple queries and joins at read time
    {ok, [Order]} = orders_store:query(
        "SELECT * FROM orders WHERE id = ?", [OrderId]),

    {ok, Items} = orders_store:query(
        "SELECT * FROM order_items WHERE order_id = ?", [OrderId]),

    {ok, [Customer]} = customers_store:query(
        "SELECT * FROM customers WHERE id = ?",
        [element(3, Order)]),

    %% ❌ WRONG: Assembling data at query time
    {ok, #{
        order => Order,
        items => Items,
        customer => Customer
    }}.
```

**Why wrong:**
- Multiple database queries per read
- Cross-table (even cross-database) joins are slow
- Read models should be pre-computed, not assembled on demand
- Query complexity increases with data model complexity

---

## Correct Way

### Projection pre-computes everything

```erlang
%%% @doc Projection: finding_recorded_v1 -> findings table + counter update
-module(finding_recorded_v1_to_findings).

-export([project/1]).

%% @doc Project finding_recorded_v1 event.
%% Pre-computes all derived data and updates denormalized counters.
-spec project(map()) -> ok | {error, term()}.
project(#{finding_id := FId, division_id := PId, category := Cat, title := Title} = E) ->
    %% Insert the finding itself
    InsertSql = "INSERT OR REPLACE INTO findings "
                "(finding_id, division_id, category, title, content, priority, recorded_at) "
                "VALUES (?1, ?2, ?3, ?4, ?5, ?6, ?7)",
    ok = query_divisions_store:execute(InsertSql, [
        FId,
        PId,
        Cat,
        Title,
        maps:get(content, E, undefined),
        maps:get(priority, E, <<"should">>),
        maps:get(recorded_at, E, erlang:system_time(millisecond))
    ]),

    %% Update denormalized counter on parent (no join needed at query time)
    CountSql = "UPDATE divisions SET finding_count = finding_count + 1 "
               "WHERE division_id = ?1",
    query_divisions_store:execute(CountSql, [PId]).
```

### Store denormalized for fast reads

```erlang
%%% @doc Projection: identity_registered_v1 -> identities table
-module(identity_registered_v1_to_identities).

-export([project/1]).

%% @doc Project identity_registered_v1 event.
%% Stores all data needed for identity queries in one row.
-spec project(map()) -> ok | {error, term()}.
project(#{
    event_type := <<"identity_registered_v1">>,
    mri := MRI,
    public_key := PublicKey,
    key_type := KeyType,
    metadata := Metadata,
    registered_at := RegisteredAt
}) ->
    %% Serialize nested data as JSON (no joins needed)
    MetadataJson = json:encode(Metadata),

    Sql = io_lib:format(
        "INSERT OR REPLACE INTO identities "
        "(mri, public_key, key_type, metadata, registered_at) "
        "VALUES ('~s', '~s', '~s', '~s', ~B)",
        [escape_sql(MRI), escape_sql(PublicKey), escape_sql(KeyType),
         escape_sql(MetadataJson), RegisteredAt]
    ),

    case query_identities_store:execute(iolist_to_binary(Sql)) of
        ok -> ok;
        {error, Reason} -> {error, Reason}
    end;

project(_OtherEvent) ->
    ok.  %% Ignore events we don't care about

%% Internal functions

escape_sql(Binary) when is_binary(Binary) ->
    binary:replace(Binary, <<"'">>, <<"''">>, [global]).
```

---

## Complete Architecture

### Event Flow

```
Command Service (manage_social)
    ↓
    ↓ aggregate produces events
    ↓
ReckonDB Event Store (manage_social_db)
    ↓
    ↓ subscription pushes events
    ↓
Query Subscriber (query_social_subscriber)
    ↓
    ↓ routes by event_type
    ↓
Projection (agent_followed_v1_to_followers)
    ↓
    ↓ INSERT/UPDATE
    ↓
SQLite Read Model (query_social.db)
```

### Subscriber Pattern

```erlang
%%% @doc Event subscriber for query_social
%%% Subscribes to social events and triggers projections.
-module(query_social_subscriber).
-behaviour(gen_server).

-export([start_link/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2, terminate/2]).

-include_lib("evoq/include/evoq_types.hrl").

-record(state, {
    subscriptions :: [binary()],
    store_id :: atom(),
    event_count = 0 :: non_neg_integer()
}).

-define(SOCIAL_EVENT_TYPES, [
    <<"agent_followed_v1">>,
    <<"agent_unfollowed_v1">>,
    <<"capability_endorsed_v1">>,
    <<"endorsement_revoked_v1">>
]).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

init([]) ->
    StoreId = manage_social_db,
    SubscriptionPrefix = <<"query_social_">>,

    %% Subscribe to each event type we care about
    Subscriptions = lists:filtermap(fun(EventType) ->
        SubName = <<SubscriptionPrefix/binary, EventType/binary>>,
        case reckon_evoq_adapter:subscribe(
            StoreId,
            event_type,
            EventType,
            SubName,
            #{start_from => 0, subscriber_pid => self()}
        ) of
            {ok, SubId} ->
                {true, SubId};
            {error, _Reason} ->
                false
        end
    end, ?SOCIAL_EVENT_TYPES),

    {ok, #state{subscriptions = Subscriptions, store_id = StoreId}}.

%% Events arrive as messages
handle_info({event, #evoq_event{event_type = EventType, data = EventData} = Event}, State) ->
    #state{store_id = StoreId, event_count = Count} = State,

    ProjectionResult = project_event(EventType, EventData),

    case ProjectionResult of
        ok ->
            %% Acknowledge the event so we don't receive it again
            SubName = <<"query_social_", EventType/binary>>,
            reckon_evoq_adapter:ack(StoreId, SubName,
                                    Event#evoq_event.stream_id, Event#evoq_event.version),
            {noreply, State#state{event_count = Count + 1}};
        {error, Reason} ->
            %% Log error but don't ack - event will be retried
            logger:error("Projection error for ~s: ~p", [EventType, Reason]),
            {noreply, State}
    end;

handle_info(_Info, State) ->
    {noreply, State}.

%% Route events to appropriate projections
project_event(<<"agent_followed_v1">>, EventData) ->
    agent_followed_v1_to_followers:project(EventData);
project_event(<<"agent_unfollowed_v1">>, EventData) ->
    agent_unfollowed_v1_to_followers:project(EventData);
project_event(<<"capability_endorsed_v1">>, EventData) ->
    capability_endorsed_v1_to_endorsements:project(EventData);
project_event(<<"endorsement_revoked_v1">>, EventData) ->
    endorsement_revoked_v1_to_endorsements:project(EventData);
project_event(_UnknownType, _EventData) ->
    {error, unknown_event_type}.
```

### Projection Module Template

```erlang
%%% @doc Projection: division_initiated_v1 -> divisions table
-module(division_initiated_v1_to_divisions).

-export([project/1]).

%% @doc Project division_initiated_v1 event to divisions table.
%% All calculations done here, read model is denormalized.
-spec project(map()) -> ok | {error, term()}.
project(#{
    division_id := DivisionId,
    venture_id := VentureId,
    context_name := ContextName,
    initiated_at := InitiatedAt
} = Event) ->
    %% Store denormalized - everything needed for queries in one row
    Sql = "INSERT OR REPLACE INTO divisions "
          "(division_id, venture_id, context_name, description, "
          " current_phase, status, initiated_at) "
          "VALUES (?1, ?2, ?3, ?4, ?5, ?6, ?7)",

    query_divisions_store:execute(Sql, [
        DivisionId,
        VentureId,
        ContextName,
        maps:get(description, Event, undefined),
        <<"discovery_n_analysis">>,  %% Initial phase
        1,                            %% INITIATED bit flag
        InitiatedAt
    ]);

project(_InvalidEvent) ->
    {error, invalid_event_format}.
```

---

## Read Model Design Principles

### 1. Denormalize for reads

```sql
-- Read model for divisions dashboard
CREATE TABLE divisions (
    division_id TEXT PRIMARY KEY,
    venture_id TEXT,
    context_name TEXT NOT NULL,
    description TEXT,
    current_phase TEXT DEFAULT 'discovery_n_analysis',
    status INTEGER DEFAULT 1,

    -- Denormalized counters (updated by projections)
    finding_count INTEGER DEFAULT 0,
    term_count INTEGER DEFAULT 0,
    dossier_count INTEGER DEFAULT 0,
    desk_count INTEGER DEFAULT 0,

    -- Denormalized progress flags
    plan_approved INTEGER DEFAULT 0,
    skeleton_created INTEGER DEFAULT 0,
    build_verified INTEGER DEFAULT 0,

    -- Timestamps for sorting/filtering
    initiated_at INTEGER,
    phase_started_at INTEGER,
    completed_at INTEGER
);

-- Query is now a simple SELECT, no joins
SELECT * FROM divisions
WHERE venture_id = ? AND current_phase = 'testing_n_implementation';
```

### 2. One projection per event type

```
agent_followed_v1       → agent_followed_v1_to_followers
agent_unfollowed_v1     → agent_unfollowed_v1_to_followers
finding_recorded_v1     → finding_recorded_v1_to_findings
```

### 3. Projections may update multiple tables

```erlang
%% finding_recorded_v1 updates two tables
project(Event) ->
    %% 1. Insert finding record
    ok = insert_finding(Event),

    %% 2. Update denormalized counter on parent
    ok = increment_finding_count(maps:get(division_id, Event)).
```

### 4. Handle event versioning

```erlang
%% Support multiple event versions
project(#{event_type := <<"finding_recorded_v1">>} = E) ->
    project_v1(E);
project(#{event_type := <<"finding_recorded_v2">>} = E) ->
    project_v2(E);
project(_) ->
    {error, unknown_event_version}.

%% Upgrade old events to new format
project_v1(#{finding_id := Id, title := Title} = E) ->
    %% V1 didn't have priority, default it
    project_v2(E#{priority => <<"should">>}).

project_v2(#{finding_id := Id, title := Title, priority := Priority} = E) ->
    %% V2 has all fields
    do_insert(E).
```

---

## Directory Structure

```
apps/query_divisions/src/
├── query_divisions_app.erl
├── query_divisions_sup.erl
├── query_divisions_store.erl            # SQLite connection
├── query_divisions_subscriber.erl       # Event subscription
│
│ # Projections (flat in src/, naming reveals intent)
├── division_initiated_v1_to_divisions.erl
├── phase_transitioned_v1_to_divisions.erl
├── finding_recorded_v1_to_findings.erl
├── term_defined_v1_to_terms.erl
├── desk_inventoried_v1_to_desk_inventory.erl
│
│ # Query handlers (in slice directories)
├── get_division_by_id/
│   └── get_division_by_id.erl
├── get_divisions_page/
│   └── get_divisions_page.erl
└── get_findings_page/
    └── get_findings_page.erl
```

---

## Checklist

Before writing a projection:

- [ ] Is it named `{event_type}_to_{read_model}`?
- [ ] Does it only read from the event data (single source of truth)?
- [ ] Does it pre-compute all derived data?
- [ ] Does it update denormalized counters/flags?
- [ ] Does it use parameterized queries (not string interpolation)?
- [ ] Does it handle missing optional fields with defaults?
- [ ] Does it return `ok | {error, term()}`?

---

## Key Takeaways

1. **Single source of truth** - Projections read ONLY from event data
2. **Pre-compute everything** - Do all calculations in the projection
3. **Denormalize for reads** - No joins at query time
4. **One event, one projection** - Clear mapping from event to read model
5. **Counters and flags** - Update parent records for dashboard queries
6. **Idempotent writes** - Use `INSERT OR REPLACE` for replay safety

---

## Training Note

This example teaches:
- Event-to-read-model transformation patterns
- Denormalization strategies for fast queries
- Subscriber and projection architecture
- Single source of truth principle
- Naming conventions for projection modules

*Date: 2026-02-08*
*Origin: Hecate daemon query services implementation*
