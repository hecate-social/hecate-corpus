---
title: Status Handling in Projections
layer: codegen
audience: [codegen]
stage: stable
---

# BIT_FLAGS_STATUS_PROJECTION.md — Status Handling in Projections

_How to properly handle aggregate status bit flags in Division Architecture._

**Target:** Erlang/OTP with `evoq` + `evoq_bit_flags`

---

## Overview

Aggregate status is stored as integer bit flags. The human-readable label (`status_label`)
MUST be computed at projection write time and stored in SQLite. Query modules NEVER compute labels.

---

## Step 1: Define Flag Macros in `.hrl` Header

**Location:** `apps/{cmd_domain}/include/{aggregate}_status.hrl`

```erlang
%%% Flag macros for {aggregate} status.
%%% Include via: -include_lib("{cmd_domain}/include/{aggregate}_status.hrl").

-define({AGG}_INITIATED,     1).   %% 2^0
-define({AGG}_PHASE_ACTIVE,  2).   %% 2^1
-define({AGG}_PHASE_DONE,    4).   %% 2^2
-define({AGG}_COMPLETED,     8).   %% 2^3
-define({AGG}_ARCHIVED,     16).   %% 2^4

-define({AGG}_FLAG_MAP, #{
    0                    => <<"New">>,
    ?{AGG}_INITIATED     => <<"Initiated">>,
    ?{AGG}_PHASE_ACTIVE  => <<"Active">>,
    ?{AGG}_PHASE_DONE    => <<"Done">>,
    ?{AGG}_COMPLETED     => <<"Completed">>,
    ?{AGG}_ARCHIVED      => <<"Archived">>
}).
```

**Rules:**
- Flags MUST be powers of 2
- Prefix macros with aggregate abbreviation to avoid collisions
- The flag map macro is `?{AGG}_FLAG_MAP`
- Include from both CMD aggregate and PRJ projections via `-include_lib`

---

## Step 2: Aggregate Uses `.hrl`

```erlang
-module({aggregate}_aggregate).
-behaviour(evoq_aggregate).
-include("{aggregate}_status.hrl").

-export([flag_map/0]).

flag_map() -> ?{AGG}_FLAG_MAP.
```

---

## Step 3: Binary Key Helper in Projections

Events from evoq/ReckonDB arrive with binary keys (`<<"venture_id">>` not `venture_id`).
Each projection includes this inline helper:

```erlang
%% @doc Get value from event map, handling both atom and binary keys.
-spec get(atom(), map()) -> term().
get(Key, Map) when is_atom(Key) ->
    case maps:find(Key, Map) of
        {ok, V} -> V;
        error -> maps:get(atom_to_binary(Key, utf8), Map, undefined)
    end.
```

---

## Step 4: INSERT Projection (Birth Event)

For `{aggregate}_initiated_v1` events:

```erlang
-module({event}_to_sqlite_{table}).
-include_lib("{cmd_domain}/include/{aggregate}_status.hrl").
-export([project/1]).

project(Event) ->
    Id = get({aggregate}_id, Event),
    Status = evoq_bit_flags:set_all(0, [?{AGG}_INITIATED, ?{AGG}_PHASE_ACTIVE]),
    Label = evoq_bit_flags:to_string(Status, ?{AGG}_FLAG_MAP),
    Sql = "INSERT OR REPLACE INTO {table} "
          "({aggregate}_id, name, status, status_label, initiated_at) "
          "VALUES (?1, ?2, ?3, ?4, ?5)",
    query_{table}_store:execute(Sql, [
        Id,
        get(name, Event),
        Status,
        Label,
        get(initiated_at, Event)
    ]).

get(Key, Map) when is_atom(Key) ->
    case maps:find(Key, Map) of
        {ok, V} -> V;
        error -> maps:get(atom_to_binary(Key, utf8), Map, undefined)
    end.
```

---

## Step 5: UPDATE Projection (Phase Transition)

For events that SET a new flag on existing status:

```erlang
-module({event}_to_sqlite_{table}).
-include_lib("{cmd_domain}/include/{aggregate}_status.hrl").
-export([project/1]).

project(Event) ->
    Id = get({aggregate}_id, Event),
    %% 1. Set the flag (SQL bitwise OR is needed for atomic read-modify-write)
    ok = query_{table}_store:execute(
        "UPDATE {table} SET status = status | ?1 WHERE {aggregate}_id = ?2",
        [?{AGG}_PHASE_DONE, Id]),
    %% 2. Read back new status, recompute label
    case query_{table}_store:query(
        "SELECT status FROM {table} WHERE {aggregate}_id = ?1", [Id]) of
        {ok, [[NewStatus]]} ->
            Label = evoq_bit_flags:to_string(NewStatus, ?{AGG}_FLAG_MAP),
            query_{table}_store:execute(
                "UPDATE {table} SET status_label = ?1 WHERE {aggregate}_id = ?2",
                [Label, Id]);
        _ -> ok
    end.

get(Key, Map) when is_atom(Key) ->
    case maps:find(Key, Map) of
        {ok, V} -> V;
        error -> maps:get(atom_to_binary(Key, utf8), Map, undefined)
    end.
```

**Note:** SQL `status | ?1` is correct here because the bitwise OR must happen
atomically within SQLite (read current value + set flag). You cannot do this
in Erlang without a race condition between read and write.

---

## Step 6: Query Module (No Enrichment)

**CRITICAL naming:** Use `get_{aggregate}_by_id` for single lookups, `get_{aggregates}_page`
for lists. NEVER `list_{aggregates}` or `get_all_{aggregates}`. See `CODEGEN_ERLANG_NAMING.md`.

```erlang
-module(get_{aggregates}_page).
-include_lib("{cmd_domain}/include/{aggregate}_status.hrl").
-export([execute/1]).

-define(DEFAULT_PAGE_SIZE, 50).
-define(MAX_PAGE_SIZE, 200).

execute(Opts) ->
    Page = maps:get(page, Opts, 1),
    PageSize = min(maps:get(page_size, Opts, ?DEFAULT_PAGE_SIZE), ?MAX_PAGE_SIZE),
    Offset = (Page - 1) * PageSize,
    Sql = "SELECT {aggregate}_id, name, status, status_label "
          "FROM {table} "
          "WHERE (status & ?1) = 0 "
          "ORDER BY initiated_at DESC "
          "LIMIT ?2 OFFSET ?3",
    case query_{table}_store:query(Sql, [?{AGG}_ARCHIVED, PageSize, Offset]) of
        {ok, Rows} ->
            {ok, #{items => [row_to_map(R) || R <- Rows],
                   page => Page, page_size => PageSize}};
        Error -> Error
    end.

row_to_map([Id, Name, Status, StatusLabel]) ->
    #{{aggregate}_id => Id, name => Name,
      status => Status, status_label => StatusLabel}.
%% NO enrich_status function. Label comes from SQLite.
```

---

## Step 7: SQLite Schema

```sql
CREATE TABLE IF NOT EXISTS {table} (
    {aggregate}_id TEXT PRIMARY KEY,
    name TEXT,
    status INTEGER DEFAULT 0,
    status_label TEXT DEFAULT 'New',
    initiated_at INTEGER
);
```

---

## Anti-Patterns (What NOT To Do)

| Anti-Pattern | Correct |
|--------------|---------|
| `enrich_status(Row)` in query module | `status_label` column in SQLite |
| `Status = 3` (magic number) | `evoq_bit_flags:set_all(0, [?VNT_INITIATED, ?VNT_DNA_ACTIVE])` |
| `-define(ARCHIVED, 32)` in projection | `-include_lib("guide_venture_lifecycle/include/venture_status.hrl")` |
| `#{venture_id := Id} = Event` | `Id = get(venture_id, Event)` (handles binary keys) |
| `normalize_keys(Event)` | `get/2` helper (no full map conversion) |
| `venture_aggregate:flag_map()` in query | `?VNT_FLAG_MAP` macro via `.hrl` include |
| `?VNT_INITIATED bor ?VNT_DNA_ACTIVE` | `evoq_bit_flags:set_all(0, [...])` |
| `Status band ?FLAG =:= 0` (outside guards) | `evoq_bit_flags:has_not(Status, ?FLAG)` |
| `get_venture` (ambiguous lookup) | `get_venture_by_id` (screaming intent) |
| `list_ventures` (unbounded) | `get_ventures_page` (paged, bounded) |

**Note on guards:** Erlang guards only allow BIFs. `band` in guards is fine
because `evoq_bit_flags:has/2` cannot be used in guard expressions. Outside
guards, always use the `evoq_bit_flags` API.

---

## Fine-Tuning Examples

### Example 1: Wrong (Read-Time Enrichment)

**Input:** "Project venture_initiated_v1 to SQLite domains table"

**Wrong Output:**
```erlang
project(#{venture_id := VentureId} = Event) ->
    store:execute("INSERT INTO domains (venture_id, status) VALUES (?1, ?2)",
        [VentureId, 1]).

%% In list_ventures.erl:
enrich_status(#{status := S} = Row) ->
    Row#{status_label => evoq_bit_flags:to_string(S, venture_aggregate:flag_map())}.
```

**Why wrong:**
- Pattern matches atom keys (events have binary keys)
- Hardcodes `1` instead of using macro
- No `status_label` stored — defers to query-time enrichment
- Query module couples to CMD aggregate internals

### Example 2: Correct (Write-Time Label Storage)

**Input:** "Project venture_initiated_v1 to SQLite domains table"

**Correct Output:**
```erlang
-include_lib("guide_venture_lifecycle/include/venture_status.hrl").

project(Event) ->
    VentureId = get(venture_id, Event),
    Status = evoq_bit_flags:set_all(0, [?VNT_INITIATED, ?VNT_DNA_ACTIVE]),
    Label = evoq_bit_flags:to_string(Status, ?VNT_FLAG_MAP),
    query_venture_lifecycle_store:execute(
        "INSERT OR REPLACE INTO domains (venture_id, status, status_label) VALUES (?1, ?2, ?3)",
        [VentureId, Status, Label]).

get(Key, Map) when is_atom(Key) ->
    case maps:find(Key, Map) of
        {ok, V} -> V;
        error -> maps:get(atom_to_binary(Key, utf8), Map, undefined)
    end.
```

### Example 3: Wrong (Magic Number Phase Transition)

**Input:** "Project discovery_completed_v1 to update division status"

**Wrong Output:**
```erlang
project(#{division_id := Id}) ->
    store:execute("UPDATE divisions SET status = status | 4 WHERE division_id = ?1", [Id]).
```

**Why wrong:** Magic number `4`, atom key match, no status_label update.

### Example 4: Correct (Phase Transition with Label)

**Input:** "Project discovery_completed_v1 to update division status"

**Correct Output:**
```erlang
-include_lib("guide_division_alc/include/division_alc_status.hrl").

project(Event) ->
    Id = get(division_id, Event),
    ok = query_division_alc_store:execute(
        "UPDATE divisions SET status = status | ?1 WHERE division_id = ?2",
        [?DIV_DISCOVERY_COMPLETE, Id]),
    case query_division_alc_store:query(
        "SELECT status FROM divisions WHERE division_id = ?1", [Id]) of
        {ok, [[NewStatus]]} ->
            Label = evoq_bit_flags:to_string(NewStatus, ?DIV_FLAG_MAP),
            query_division_alc_store:execute(
                "UPDATE divisions SET status_label = ?1 WHERE division_id = ?2",
                [Label, Id]);
        _ -> ok
    end.

get(Key, Map) when is_atom(Key) ->
    case maps:find(Key, Map) of
        {ok, V} -> V;
        error -> maps:get(atom_to_binary(Key, utf8), Map, undefined)
    end.
```
