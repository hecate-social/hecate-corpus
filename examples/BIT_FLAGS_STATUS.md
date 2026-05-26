---
title: "Example: Bit Flag Status Fields"
layer: example
audience: [agent, human]
stage: stable
---

# Example: Bit Flag Status Fields

*Canonical example: Using integers as bit flags for aggregate status*

---

## The Pattern

Aggregate status fields are stored as a single integer where each bit represents a boolean state. This pattern is memory-efficient, CPU-fast, and query-friendly.

Instead of:
```erlang
-record(order, {
    is_validated :: boolean(),
    is_processing :: boolean(),
    is_completed :: boolean(),
    is_cancelled :: boolean()
}).
```

Use:
```erlang
-record(order, {
    status :: non_neg_integer()  %% BIT FLAGS
}).
```

---

## Why Bit Flags?

| Concern | Multiple Booleans | Bit Flags |
|---------|-------------------|-----------|
| Memory | N bytes (one per field) | 1-8 bytes (one integer) |
| CPU | N comparisons | Single bitwise op |
| Network | N fields serialized | One integer |
| Database | N columns | One column |
| Event size | Larger events | Compact events |
| Queries | Multiple ANDs | Bitwise AND |

**Real impact:** In event-sourced systems, events are stored forever. Compact representation saves storage and speeds up replay.

---

## Wrong Way: Multiple Boolean Fields

```erlang
%% ❌ WRONG: Separate boolean fields
-record(order, {
    id :: binary(),
    is_created :: boolean(),
    is_validated :: boolean(),
    is_processing :: boolean(),
    is_shipped :: boolean(),
    is_delivered :: boolean(),
    is_cancelled :: boolean()
}).

%% Awkward state checks
can_ship(#order{is_validated = true, is_processing = false,
                is_shipped = false, is_cancelled = false}) ->
    true;
can_ship(_) ->
    false.
```

**Problems:**
- 6 separate fields = 6 bytes minimum
- Pattern matching becomes unwieldy
- Adding new states requires schema migration
- Queries need multiple column checks

---

## Wrong Way: Atom Status Field

```erlang
%% ❌ WRONG: Single atom status
-record(order, {
    id :: binary(),
    status :: created | validated | processing | shipped | delivered | cancelled
}).

%% Can't represent multiple states simultaneously
%% What if order is both shipped AND has_issue?
```

**Problems:**
- Cannot combine states (shipped + has_issue)
- State explosion (shipped_with_issue, shipped_without_issue, ...)
- No partial state queries

---

## Correct Way: Bit Flag Status

### Step 1: Define Flags as Powers of 2

```erlang
%% order.hrl
-define(ORDER_CREATED,     1).   %% 2^0 = 1     = 0b00000001
-define(ORDER_VALIDATED,   2).   %% 2^1 = 2     = 0b00000010
-define(ORDER_PROCESSING,  4).   %% 2^2 = 4     = 0b00000100
-define(ORDER_SHIPPED,     8).   %% 2^3 = 8     = 0b00001000
-define(ORDER_DELIVERED,  16).   %% 2^4 = 16    = 0b00010000
-define(ORDER_CANCELLED,  32).   %% 2^5 = 32    = 0b00100000
-define(ORDER_HAS_ISSUE,  64).   %% 2^6 = 64    = 0b01000000
-define(ORDER_REFUNDED,  128).   %% 2^7 = 128   = 0b10000000
```

**Critical rule:** Each flag MUST be a power of 2 (1, 2, 4, 8, 16, 32, 64, 128, 256...). This ensures each flag occupies a unique bit position.

### Step 2: Define Aggregate with Integer Status

```erlang
-record(order, {
    id :: binary(),
    status :: non_neg_integer(),  %% BIT FLAGS - not atom!
    items :: [item()]
}).
```

### Step 3: Use evoq_bit_flags for Manipulation

```erlang
-module(order_aggregate).
-include("order.hrl").

%% Initialize with CREATED flag
init(OrderId) ->
    #order{
        id = OrderId,
        status = ?ORDER_CREATED,
        items = []
    }.

%% Apply events - set flags
apply_event(#order{status = S} = O, {order_validated, _}) ->
    O#order{status = evoq_bit_flags:set(S, ?ORDER_VALIDATED)};

apply_event(#order{status = S} = O, {order_shipped, _}) ->
    NewStatus = evoq_bit_flags:set(S, ?ORDER_SHIPPED),
    O#order{status = NewStatus};

apply_event(#order{status = S} = O, {order_cancelled, _}) ->
    NewStatus = evoq_bit_flags:set(S, ?ORDER_CANCELLED),
    O#order{status = NewStatus};

apply_event(#order{status = S} = O, {issue_reported, _}) ->
    %% Can set HAS_ISSUE alongside other states
    NewStatus = evoq_bit_flags:set(S, ?ORDER_HAS_ISSUE),
    O#order{status = NewStatus};

apply_event(#order{status = S} = O, {issue_resolved, _}) ->
    %% Clear the issue flag
    NewStatus = evoq_bit_flags:clear(S, ?ORDER_HAS_ISSUE),
    O#order{status = NewStatus}.
```

---

## evoq_bit_flags Operations

The `evoq_bit_flags` module provides all needed operations:

### has_flag/2 - Check if flag is set

```erlang
%% Is order validated?
evoq_bit_flags:has_flag(Status, ?ORDER_VALIDATED).
%% => true | false
```

### set/2 - Set a flag

```erlang
%% Mark as shipped
NewStatus = evoq_bit_flags:set(Status, ?ORDER_SHIPPED).
```

### clear/2 - Clear a flag

```erlang
%% Remove issue flag
NewStatus = evoq_bit_flags:clear(Status, ?ORDER_HAS_ISSUE).
```

### has_any/2 - Check if ANY of multiple flags is set

```erlang
%% Is order shipped OR delivered?
evoq_bit_flags:has_any(Status, [?ORDER_SHIPPED, ?ORDER_DELIVERED]).
```

### has_all/2 - Check if ALL flags are set

```erlang
%% Is order both validated AND shipped?
evoq_bit_flags:has_all(Status, [?ORDER_VALIDATED, ?ORDER_SHIPPED]).
```

### toggle/2 - Flip a flag

```erlang
%% Toggle issue flag
NewStatus = evoq_bit_flags:toggle(Status, ?ORDER_HAS_ISSUE).
```

---

## Business Rules with Bit Flags

```erlang
-module(order_rules).
-include("order.hrl").

%% Can we ship this order?
can_ship(#order{status = S}) ->
    evoq_bit_flags:has_flag(S, ?ORDER_VALIDATED) andalso
    not evoq_bit_flags:has_any(S, [?ORDER_SHIPPED, ?ORDER_CANCELLED]).

%% Can we cancel this order?
can_cancel(#order{status = S}) ->
    not evoq_bit_flags:has_any(S, [?ORDER_SHIPPED, ?ORDER_DELIVERED]).

%% Is order terminal (no more transitions)?
is_terminal(#order{status = S}) ->
    evoq_bit_flags:has_any(S, [?ORDER_DELIVERED, ?ORDER_REFUNDED]).

%% Order has problems?
has_problems(#order{status = S}) ->
    evoq_bit_flags:has_any(S, [?ORDER_HAS_ISSUE, ?ORDER_CANCELLED]).
```

---

## Command Handler Pattern

```erlang
-module(maybe_ship_order).
-include("order.hrl").

handle(Cmd, #order{status = S} = Aggregate) ->
    case can_ship(S) of
        true ->
            Event = order_shipped_v1:new(Cmd),
            {ok, [Event]};
        false ->
            {error, cannot_ship_order}
    end.

can_ship(S) ->
    evoq_bit_flags:has_flag(S, ?ORDER_VALIDATED) andalso
    not evoq_bit_flags:has_any(S, [?ORDER_SHIPPED, ?ORDER_CANCELLED]).
```

---

## Querying with Bit Flags

Bit flags work naturally with SQL bitwise operators:

### SQLite / PostgreSQL

```sql
-- Find all shipped orders
SELECT * FROM orders WHERE (status & 8) = 8;

-- Find orders that are shipped OR delivered
SELECT * FROM orders WHERE (status & 24) != 0;  -- 8 + 16 = 24

-- Find validated orders that are NOT cancelled
SELECT * FROM orders
WHERE (status & 2) = 2      -- has VALIDATED
  AND (status & 32) = 0;    -- not CANCELLED

-- Find orders with any issues
SELECT * FROM orders WHERE (status & 64) != 0;
```

### In Erlang (for in-memory queries)

```erlang
%% Filter list of aggregates
ShippedOrders = [O || O <- Orders,
                      evoq_bit_flags:has_flag(O#order.status, ?ORDER_SHIPPED)].

%% Complex filter
ProblematicShipped = [O || O <- Orders,
                           evoq_bit_flags:has_flag(O#order.status, ?ORDER_SHIPPED),
                           evoq_bit_flags:has_flag(O#order.status, ?ORDER_HAS_ISSUE)].
```

---

## Projection Example

```erlang
-module(order_shipped_v1_to_orders).

project(Event, Db) ->
    OrderId = order_shipped_v1:get_order_id(Event),

    %% Read current status, set SHIPPED flag, write back
    case sqlite:query(Db, "SELECT status FROM orders WHERE id = ?", [OrderId]) of
        [{Status}] ->
            NewStatus = evoq_bit_flags:set(Status, ?ORDER_SHIPPED),
            sqlite:execute(Db,
                "UPDATE orders SET status = ?, shipped_at = ? WHERE id = ?",
                [NewStatus, erlang:system_time(millisecond), OrderId]);
        [] ->
            {error, order_not_found}
    end.
```

---

## Common Flag Patterns

### Lifecycle Flags

```erlang
-define(CREATED,     1).
-define(VALIDATED,   2).
-define(ACTIVE,      4).
-define(SUSPENDED,   8).
-define(TERMINATED, 16).
-define(ARCHIVED,   32).
```

### Permission Flags

```erlang
-define(CAN_READ,    1).
-define(CAN_WRITE,   2).
-define(CAN_DELETE,  4).
-define(CAN_ADMIN,   8).
```

### Feature Flags

```erlang
-define(FEATURE_BETA,     1).
-define(FEATURE_PREMIUM,  2).
-define(FEATURE_API,      4).
-define(FEATURE_EXPORT,   8).
```

---

## Flag Definition Rules

1. **Always powers of 2** - 1, 2, 4, 8, 16, 32, 64, 128, 256, 512, 1024...
2. **Use defines, not magic numbers** - `?ORDER_SHIPPED` not `8`
3. **Put in header file** - `order.hrl` for reuse across modules
4. **Document meaning** - Comment each flag's purpose
5. **Reserve space** - Don't use all 32/64 bits immediately
6. **Group logically** - Lifecycle flags, permission flags, feature flags

### Flag Calculation Helper

| Bit Position | Value | Calculation |
|--------------|-------|-------------|
| 0 | 1 | 2^0 |
| 1 | 2 | 2^1 |
| 2 | 4 | 2^2 |
| 3 | 8 | 2^3 |
| 4 | 16 | 2^4 |
| 5 | 32 | 2^5 |
| 6 | 64 | 2^6 |
| 7 | 128 | 2^7 |
| 8 | 256 | 2^8 |
| 9 | 512 | 2^9 |
| 10 | 1024 | 2^10 |

---

## Debugging Bit Flags

```erlang
%% Pretty-print status for debugging
format_status(Status) ->
    Flags = [
        {?ORDER_CREATED, "CREATED"},
        {?ORDER_VALIDATED, "VALIDATED"},
        {?ORDER_PROCESSING, "PROCESSING"},
        {?ORDER_SHIPPED, "SHIPPED"},
        {?ORDER_DELIVERED, "DELIVERED"},
        {?ORDER_CANCELLED, "CANCELLED"},
        {?ORDER_HAS_ISSUE, "HAS_ISSUE"},
        {?ORDER_REFUNDED, "REFUNDED"}
    ],
    ActiveFlags = [Name || {Flag, Name} <- Flags,
                           evoq_bit_flags:has_flag(Status, Flag)],
    io_lib:format("~p (~s)", [Status, string:join(ActiveFlags, "|")]).

%% Usage:
%% format_status(11) => "11 (CREATED|VALIDATED|SHIPPED)"
```

---

## Antipatterns to Avoid

### Using Non-Powers of 2

```erlang
%% ❌ WRONG: These overlap!
-define(FLAG_A, 1).
-define(FLAG_B, 2).
-define(FLAG_C, 3).   %% 3 = 1 + 2, overlaps with A and B!
```

### Mixing Atoms and Integers

```erlang
%% ❌ WRONG: Type confusion
-record(order, {
    status :: created | validated | non_neg_integer()  %% Pick one!
}).
```

### Hardcoding Flag Values

```erlang
%% ❌ WRONG: Magic numbers
apply_event(O, {validated, _}) ->
    O#order{status = O#order.status bor 2}.  %% What is 2?

%% ✅ CORRECT: Named constant
apply_event(O, {validated, _}) ->
    O#order{status = evoq_bit_flags:set(O#order.status, ?ORDER_VALIDATED)}.
```

---

## Key Takeaways

1. **Status fields are integers** - not atoms, not multiple booleans
2. **Flags are powers of 2** - 1, 2, 4, 8, 16, 32...
3. **Use evoq_bit_flags** - set, clear, has_flag, has_any, has_all
4. **Define in header files** - reuse across modules
5. **Query with bitwise AND** - `(status & FLAG) = FLAG`
6. **Combine states freely** - shipped + has_issue is natural
7. **Compact for events** - saves storage, speeds replay

---

## Training Note

This example teaches:
- Bit flag pattern for aggregate status
- Memory and performance benefits
- evoq_bit_flags API usage
- Database querying with bitwise operators
- Common antipatterns to avoid
- Flag definition rules (powers of 2)

*Date: 2026-02-08*
*Origin: Hecate daemon aggregate patterns*
