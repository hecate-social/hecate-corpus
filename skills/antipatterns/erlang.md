---
title: "ANTIPATTERNS: Erlang/OTP Gotchas"
layer: skill
audience: [agent, human]
stage: stable
---

# ANTIPATTERNS: Erlang/OTP Gotchas

*Demons about Erlang-specific pitfalls. esqlite3, maps, gen_server, and OTP.*

[Back to Index](INDEX.md)

---

## 🔥 Demon 19: esqlite3 Returns Lists, Not Tuples

**Date exorcised:** 2026-02-12
**Where it appeared:** 12+ `row_to_map/1` functions across all `query_*` apps
**Cost:** Silent pattern match failure — queries returned `{ok, []}` instead of data

### The Lie

"SQLite rows come back as tuples like `{Col1, Col2, Col3}`."

### What Happened

All `row_to_map/1` functions used tuple patterns:
```erlang
%% WRONG — tuples
row_to_map({Id, Name, Status, Label}) ->
    #{id => Id, name => Name, status => Status, label => Label}.
```

But `esqlite3:fetchall/1` returns rows as **lists of lists**, not lists of tuples:
```erlang
{ok, [[<<"id1">>, <<"name1">>, 1, <<"Active">>],
      [<<"id2">>, <<"name2">>, 2, <<"Done">>]]}
```

The tuple pattern `{Id, Name, Status, Label}` never matches `[Id, Name, Status, Label]`, so the list comprehension `[row_to_map(R) || R <- Rows]` silently produces an empty list.

### The Correct Pattern

```erlang
%% CORRECT — lists
row_to_map([Id, Name, Status, Label]) ->
    #{id => Id, name => Name, status => Status, label => Label}.
```

### Why This Happens

1. Many Erlang database libraries (mnesia, ets) return tuples
2. The mental model "row = tuple" is deeply ingrained
3. Without tests that exercise the full query path, the bug only surfaces at runtime
4. The empty-list result looks like "no data" rather than "pattern match failed"

### Prevention

```erlang
%% Test that row_to_map works with list input
row_to_map_test() ->
    Row = [<<"id">>, <<"name">>, 1, <<"Active">>],
    Result = row_to_map(Row),
    ?assertEqual(<<"id">>, maps:get(id, Result)).
```

### The Lesson

> **esqlite3 returns lists. Always. Test your row_to_map with actual list inputs.**
> **This bug was in the BIT_FLAGS_STATUS_PROJECTION.md template — the "correct" example was wrong.**

---

## 🔥 Demon 20: Eager Default in `maps:get/3`

**Date exorcised:** 2026-02-12
**Where it appeared:** 10 command modules (`*_v1.erl`) in `mentor_llms`
**Cost:** `noproc` crash when gen_server not running (e.g., in tests)

### The Lie

"Use `maps:get(key, Map, default_value())` to provide a fallback."

### What Happened

Command modules used function calls as `maps:get/3` defaults:
```erlang
%% WRONG — hecate_identity:agent_id() is ALWAYS called
SubmitterId = maps:get(<<"submitter_id">>, Map, hecate_identity:agent_id()),
```

In Erlang, `maps:get(Key, Map, Default)` evaluates `Default` **before** checking if `Key` exists. When `Default` is a function call to a gen_server (`hecate_identity`), it crashes with `noproc` if that server isn't running — even when the key IS present in the map.

### The Correct Pattern

```erlang
%% CORRECT — lazy evaluation via case
SubmitterId = case maps:find(<<"submitter_id">>, Map) of
    {ok, V} -> V;
    error -> hecate_identity:agent_id()
end,
```

### Why This Happens

1. `maps:get/3` looks like it should be lazy (only use default when key is missing)
2. In many other languages, default values ARE lazy (Python's `dict.get(k, v)` doesn't evaluate `v` eagerly)
3. Erlang evaluates all function arguments before calling the function — there are no lazy arguments
4. The bug only manifests when the gen_server isn't running, which may not happen in production but always happens in unit tests

### When This Matters

| Default Expression | Safe? | Why |
|-------------------|-------|-----|
| `maps:get(k, M, <<>>)` | Yes | Literal — no side effects |
| `maps:get(k, M, undefined)` | Yes | Atom literal |
| `maps:get(k, M, 0)` | Yes | Integer literal |
| `maps:get(k, M, gen_server:call(...))` | **NO** | Evaluated even when key exists |
| `maps:get(k, M, hecate_identity:agent_id())` | **NO** | gen_server call, crashes if down |
| `maps:get(k, M, os:timestamp())` | **NO** | Side effect always runs |

### The Lesson

> **Never use function calls as `maps:get/3` defaults. Use `maps:find/2` + `case` for lazy evaluation.**
> **If the default is a literal, `maps:get/3` is fine. If it's a function call, it's a bug waiting to happen.**

---

## 🔥 Demon 21: esqlite3 Argument Order (Db First)

**Date exorcised:** 2026-02-12
**Where it appeared:** 8 `query_*_store.erl` files
**Cost:** `function_clause` crash on store initialization

### The Lie

"Just call `esqlite3:exec(SQL, Db)` — SQL first, then connection."

### What Happened

Store `init/1` functions had the arguments reversed:
```erlang
%% WRONG — SQL first
esqlite3:exec("PRAGMA journal_mode=WAL", Db),
esqlite3:exec("CREATE TABLE IF NOT EXISTS ...", Db),
```

The esqlite3 API is `esqlite3:exec(Db, SQL)` — connection first:
```erlang
%% CORRECT — Db first
esqlite3:exec(Db, "PRAGMA journal_mode=WAL"),
esqlite3:exec(Db, "CREATE TABLE IF NOT EXISTS ..."),
```

### The API

| Function | Signature | Note |
|----------|-----------|------|
| `esqlite3:open/1` | `open(Path)` | Returns `{ok, Db}` |
| `esqlite3:exec/2` | `exec(Db, SQL)` | **Db first** |
| `esqlite3:q/2` | `q(Db, SQL)` | **Db first** |
| `esqlite3:q/3` | `q(Db, SQL, Params)` | **Db first** |

### Why This Happens

1. Many Erlang database APIs use `Module:exec(SQL, Connection)` — SQL first
2. The "subject.verb(object)" pattern (`SQL.exec_on(Db)`) feels natural
3. esqlite3 follows the C SQLite API convention where the handle comes first
4. Without type specs or dialyzer, the crash only appears at runtime

### The Lesson

> **esqlite3: connection (Db) ALWAYS comes first. `exec(Db, SQL)`, `q(Db, SQL, Params)`.**
> **When in doubt, check the esqlite3 source — don't assume argument order from other libraries.**

---

## 🔥 gen_server Self-Call Deadlock

**Date:** 2026-03-05
**Origin:** reckon_db emitter pool fix deadlocked on `is_active/1`

### The Antipattern

Calling a gen_server from within its own process via a function that does `gen_server:call`:

```erlang
%% In reckon_db_leader handle_cast({activate, StoreId}, State):
%%   → save_default_subscriptions(StoreId)
%%     → subscribe/5
%%       → setup_event_notification
%%         → reckon_db_leader:is_active(StoreId)  %% gen_server:call back to self!
%%           → DEADLOCK
```

### Why It's Wrong

`gen_server:call` sends a message and waits for a reply. If the target is the calling process itself, the process is already handling a message and can't process the call. Erlang raises `{calling_self, {gen_server, call, [...]}}`.

### The Correct Pattern

Use a non-blocking check that doesn't require the gen_server to respond:

```erlang
%% CORRECT — check if the supervisor process exists via whereis/1
SupName = reckon_db_naming:emitter_sup_name(StoreId),
maybe_start_emitter_pool(StoreId, Key, Sub, whereis(SupName)).

maybe_start_emitter_pool(_StoreId, _Key, _Sub, undefined) -> ok;
maybe_start_emitter_pool(StoreId, Key, Sub, _SupPid) ->
    case reckon_db_emitter_pool:start_emitter(StoreId, Sub) of
        {ok, _Pid} ->
            logger:info("Started emitter pool for ~s (store: ~p)", [Key, StoreId]);
        {error, {already_started, _}} -> ok;
        {error, _} -> ok
    end.
```

**Rule:** If a function might be called from inside a gen_server, never use `gen_server:call` to query that same server. Use `whereis/1`, ETS lookups, or process dictionary reads instead.

---

## 🔥 Demon 38: Emoji Literals in esqlite3 SQL Strings

**Date exorcised:** 2026-03-06
**Where it appeared:** `project_launcher_store:create_tables/1` and 3 projection modules
**Cost:** `badarg` crash on startup — daemon crash-loops, never boots

### The Lie

"Just put the emoji in the SQL DEFAULT value: `DEFAULT '🔌'`."

### What Happened

The SQLite CREATE TABLE statement had emoji literals in DEFAULT clauses:
```erlang
%% WRONG — emoji literal in SQL string
esqlite3:exec(Db,
    "CREATE TABLE IF NOT EXISTS launcher_entries ("
    "  icon TEXT NOT NULL DEFAULT '🔌'"
    ");").
```

`esqlite3_nif:exec/2` received a charlist containing codepoints > 127 (the plug emoji U+1F50C is 4 bytes in UTF-8: `\xF0\x9F\x94\x8C`). The NIF expects iodata but chokes on the high codepoints, crashing with `badarg`. The error is buried in a kernel pid termination message — no clear "emoji bad" error.

### The Correct Pattern

Use UTF-8 byte escapes in Erlang binaries, passed as bind parameters:
```erlang
%% CORRECT — byte escapes in binary, passed as bind parameter
project_launcher_store:execute(
    "INSERT INTO launcher_groups (name, icon) VALUES (?1, ?2)",
    [GroupName, <<"\xF0\x9F\x93\x81">>]).

%% CORRECT — empty default in DDL, real icon comes from event data
"icon TEXT NOT NULL DEFAULT ''"
```

For SQL DDL (CREATE TABLE), use empty string defaults — the actual emoji values come from event data via bind parameters, which handle UTF-8 correctly.

### Common Emoji Byte Sequences

| Emoji | UTF-8 Bytes | Erlang Binary |
|-------|-------------|---------------|
| 📁 (folder) | F0 9F 93 81 | `<<"\xF0\x9F\x93\x81">>` |
| 🔌 (plug) | F0 9F 94 8C | `<<"\xF0\x9F\x94\x8C">>` |
| ⚙️ (gear) | E2 9A 99 EF B8 8F | `<<"\xE2\x9A\x99\xEF\xB8\x8F">>` |

### Why This Happens

1. Erlang strings are charlists — `"🔌"` is `[128268]`, a single codepoint > 127
2. esqlite3's NIF expects iodata (binaries or byte-range charlists)
3. High codepoints in charlists are NOT valid iodata
4. The crash message is opaque: `{badarg, [{esqlite3_nif, exec, [#Ref<...>, [67,82,69,65,...]]}]}` — just raw codepoints
5. Binaries with `\xNN` escapes ARE valid UTF-8 iodata and pass through cleanly

### The Lesson

> **Never put emoji (or any non-ASCII) directly in esqlite3 SQL strings.**
> **Use `<<"\xF0\x9F\x...">>` binaries as bind parameters. Use empty defaults in DDL.**
> **This applies to all esqlite3 functions: `exec/2`, `prepare/2`, etc.**

---

*We burned these demons so you don't have to. Keep the fire going.*
