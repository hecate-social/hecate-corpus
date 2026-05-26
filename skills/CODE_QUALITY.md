---
title: Implementation Standards
layer: skill
audience: [agent, human]
stage: stable
---

# CODE_QUALITY.md — Implementation Standards

*Rules for clean, maintainable BEAM code (Erlang and Elixir). These are NON-NEGOTIABLE.*

**Sources:** Erlang/OTP Programming Rules (erlang.se), Inaka Erlang Guidelines, WhatsApp erlfmt, Jesper Louis Andersen, Fred Hebert, Learn You Some Erlang, Official Elixir docs, Credo.

---

## 1. LET IT CRASH — The BEAM Philosophy

**This is not optional. This is how BEAM systems work.**

### What "Let It Crash" Means

- Code the **happy path**. Crash on the unexpected.
- Supervision trees recover processes automatically.
- A crashed process restarts in a **known good state**.
- BEAM cleans up resources (file descriptors, sockets, ports) when a process dies.

### What "Let It Crash" Does NOT Mean

- It does NOT mean "never handle errors"
- It does NOT mean "show crashes to users"
- It does NOT mean "ignore failures"
- It does NOT mean "no error tuples"

### The Decision Matrix

| Error Type | Action | Example |
|-----------|--------|---------|
| **Expected** | Handle with `{ok, V}` / `{error, R}` | File not found, network timeout, invalid user input |
| **Unexpected** | Let it crash, supervisor restarts | Corrupt state, impossible match, programming bug |

### Erlang Examples

```erlang
%% CORRECT: Handle the EXPECTED, crash on the UNEXPECTED
case file:open(Filename, [raw, binary, read]) of
    {ok, Fd} -> read_contents(Fd);
    {error, enoent} -> {error, not_found}
    %% DO NOT catch eacces, eisdir, enospc here.
    %% If those happen, something is fundamentally wrong — CRASH.
end.

%% CORRECT: Assert success with pattern match
{ok, Fd} = file:open(Filename, [raw, binary, read]).
ok = file:write(Fd, Data).
ok = file:close(Fd).
%% If any fail, the process crashes with a clear match error.
```

### Elixir Examples

```elixir
# CORRECT: Pattern match asserts success
{:ok, content} = File.read(path)

# CORRECT: Handle expected failures
case File.read(path) do
  {:ok, content} -> process(content)
  {:error, :enoent} -> {:error, :not_found}
  # Let other errors crash — they indicate a bug or system problem
end
```

---

## 2. TRY/CATCH IS AN ANTI-PATTERN

**Severity:** ABSOLUTE RULE
**Applies to:** Erlang, Elixir

### The Rule

> **try/catch (Erlang) and try/rescue (Elixir) are anti-patterns in almost all application code. Silent error swallowing is FORBIDDEN.**

The Erlang Programming Rules state:
> "Use catch and throw with extreme care."

The Elixir official docs state:
> "Errors in Elixir are reserved for unexpected and/or exceptional situations, never for controlling the flow of our code."
> "Elixir developers rarely use the try/rescue construct."

### Anti-Patterns (FORBIDDEN)

```erlang
%% ANTI-PATTERN 1: Silent catch-all — HIDES BUGS
try risky_operation() of
    Result -> Result
catch
    _:_ -> {error, something_went_wrong}
end.

%% ANTI-PATTERN 2: Exceptions for control flow
try
    Value = maps:get(Key, Map),
    process(Value)
catch
    error:badarg -> use_default()     %% Use maps:find/2 instead!
end.

%% ANTI-PATTERN 3: Nested try/catch
try
    try inner_op() catch _:_ -> fallback() end
catch
    _:_ -> outer_fallback()
end.

%% ANTI-PATTERN 4: try block kills tail recursion
try
    recursive_loop(N)     %% NO tail call optimisation inside try
catch
    error:E -> handle(E)
end.
```

```elixir
# ANTI-PATTERN: rescue for control flow
try do
  File.read!(path)
rescue
  _ -> handle_missing_file()   # Use File.read/1 + case instead
end

# ANTI-PATTERN: silent swallowing
try do
  do_something(data)
rescue
  _ -> :ok     # SWALLOWED — you will NEVER know about the bug
end
```

### Why Silent Swallowing Is Catastrophic

1. **Masks bugs** — real programming errors become invisible
2. **Corrupts state** — the system continues with wrong assumptions
3. **Debugging nightmare** — the symptom appears far from the cause
4. **Violates fail-fast** — the exact opposite of "let it crash"
5. **Defeats supervision** — supervisors can only restart what crashes

### When try/catch IS Acceptable (Exactly 3 Cases)

**Case 1: Observability — log and re-raise**
```erlang
try critical_operation() of
    Result -> Result
catch
    Class:Reason:Stacktrace ->
        logger:error("Unexpected: ~p:~p~n~p", [Class, Reason, Stacktrace]),
        erlang:raise(Class, Reason, Stacktrace)     %% RE-RAISE, not swallow
end.
```

**Case 2: System boundary — untrusted external input**
```erlang
try json:decode(UntrustedBytes) of
    Decoded -> {ok, Decoded}
catch
    error:badarg -> {error, malformed_json}     %% SPECIFIC error, not catch-all
end.
```

**Case 3: Third-party libraries that only raise (no tuple API)**
```elixir
try do
  ThirdPartyLib.do_thing!(input)
rescue
  ThirdPartyLib.SpecificError -> {:error, :third_party_failed}
end
```

### The Checklist Before Writing try/catch

- [ ] Can I use `{ok, V}` / `{error, R}` tuples instead?
- [ ] Can I use pattern matching to handle expected cases?
- [ ] Am I catching a **specific** error, not `_:_`?
- [ ] Am I at a **system boundary** (external input, third-party lib)?
- [ ] If I catch, do I **log** or **re-raise**, not silently discard?

If you answer "no" to all of these, **do not use try/catch.**

---

## 3. NO DEFENSIVE PROGRAMMING

**Severity:** ABSOLUTE RULE
**Applies to:** Erlang, Elixir

The Erlang Programming Rules state:
> "Do not program defensively. One should not test input data to functions for correctness. Most code should be written assuming input data is correct. Only check data when it enters the system initially."

### The Rule

> **Validate at the boundary. Trust internal code. Crash on violations inside.**

### Where to Validate (System Boundaries)

- Public API handlers (HTTP requests)
- Mesh message listeners (external FACTs)
- CLI argument parsing
- File/config parsing
- OTP behaviour callbacks receiving external messages

### Where NOT to Validate (Internal Code)

- Internal helper functions
- Functions called only from tested code paths
- Data that has already been validated upstream
- Functions called within the same module

### Anti-Pattern: Defensive Type Conversion

```erlang
%% BAD: "Accepts anything" converter — diffuses responsibility
to_integer(I) when is_integer(I) -> I;
to_integer(F) when is_float(F) -> round(F);
to_integer(L) when is_list(L) -> list_to_integer(L);
to_integer(B) when is_binary(B) -> binary_to_integer(B).
%% The CALLER should send the right type. If it doesn't, CRASH.

%% GOOD: Validate at boundary, trust internally
register_user(Name, Age) when is_binary(Name), is_integer(Age), Age > 0 ->
    do_register(Name, Age).     %% Internal — no guards needed

do_register(Name, Age) ->
    #{name => Name, age => Age}.
    %% If bad data reaches here, it crashes — the bug is in the caller.
```

### Anti-Pattern: Null/Undefined Propagation

```erlang
%% BAD: Billion-dollar mistake in Erlang
do_x(undefined) -> undefined;
do_x(Value) -> compute(Value).
%% Now EVERY caller must check for undefined. Infection spreads.

%% GOOD: Force the caller to provide a value
do_x(Value) -> compute(Value).
%% If undefined reaches here, it crashes — the bug is in the caller.
```

```elixir
# BAD: Defensive nil propagation
def process(nil), do: nil
def process(data), do: transform(data)

# GOOD: Crash if nil — the caller is buggy
def process(data), do: transform(data)
```

### When Guards ARE Appropriate

| Context | Use Guards? | Why |
|---------|-------------|-----|
| Public module exports | YES | System boundary — validate input |
| OTP callbacks (`handle_call`, `handle_info`) | YES | Receives external messages |
| Aggregate `execute/2` | YES | Business rule enforcement (bit flags, status checks) |
| Internal helpers | NO | Already validated upstream |
| Private functions (`-spec` only) | NO | Trust the caller |

---

## 4. PATTERN MATCHING OVER CASE/IF

**Severity:** STRONG RULE
**Applies to:** Erlang, Elixir

### The Erlang Rule

> **Never use `if`.** Prefer multiple function clauses over `case`. Use `case` only when dispatching on a single computed value inside a function.

The Inaka guidelines state:
> "Don't use if. The if construct reduces flexibility and can confuse developers."
> "Use pattern-matching in clause functions rather than case statements."

```erlang
%% BAD: case expression for top-level dispatch
process(Msg) ->
    case Msg of
        {login, User} -> handle_login(User);
        {logout, User} -> handle_logout(User);
        {msg, From, Text} -> handle_message(From, Text)
    end.

%% GOOD: function heads for top-level dispatch
process({login, User}) -> handle_login(User);
process({logout, User}) -> handle_logout(User);
process({msg, From, Text}) -> handle_message(From, Text).
```

### When `case` IS Acceptable

Dispatching on a **computed result** inside a function:

```erlang
connect(Opts) ->
    Url = build_url(Opts),
    case httpc:request(Url) of
        {ok, {{_, 200, _}, _, Body}} -> {ok, Body};
        {ok, {{_, Code, _}, _, _}} -> {error, {http, Code}};
        {error, Reason} -> {error, Reason}
    end.
```

### The Elixir Rule

> **Avoid `if` and `cond`. Use multi-clause functions and `with`.**

```elixir
# BAD: Nested if
def initials(name) do
  if name == nil or name == "" do
    "?"
  else
    if String.contains?(name, " ") do
      # ...deeply nested logic
    end
  end
end

# GOOD: Multi-clause function heads
def initials(name) when name in [nil, ""], do: "?"
def initials(name), do: extract_initials(name)
```

### The Hierarchy (Best to Worst)

| Rank | Construct | When to Use |
|------|-----------|-------------|
| 1 | **Function head pattern match** | Top-level dispatch, multiple shapes |
| 2 | **Guards (`when`)** | Preconditions, type/range checks |
| 3 | **`with` (Elixir only)** | Sequential fallible operations |
| 4 | **`case`** | Single computed value dispatch |
| 5 | **`cond`** | Multiple boolean conditions (rare) |
| 6 | **`if`** | AVOID — almost never needed |

---

## 5. NO NESTED CONTROL STRUCTURES

**Severity:** ABSOLUTE RULE
**Applies to:** Erlang, Elixir

### The Rule

> **Maximum 1 level of nesting. Transform nested control into function calls.**

The Erlang Programming Rules state:
> "Try to limit most of your code to a maximum of two levels of indentation."

The Inaka guidelines state:
> "Try not to nest more than 3 levels deep."

**We are stricter: 1 level maximum.**

### The Anti-Pattern (Arrow Code)

```erlang
%% BAD: 3 nested case statements
handle(Req) ->
    case parse(Req) of
        {ok, Data} ->
            case validate(Data) of
                ok ->
                    case store(Data) of
                        {ok, Id} -> {ok, Id};
                        {error, R} -> {error, R}
                    end;
                {error, R} -> {error, R}
            end;
        {error, R} -> {error, R}
    end.
```

### The Fix: Extract to Functions

```erlang
%% GOOD: Flat with helper functions
handle(Req) ->
    case parse(Req) of
        {ok, Data} -> handle_parsed(Data);
        {error, R} -> {error, R}
    end.

handle_parsed(Data) ->
    case validate(Data) of
        ok -> store(Data);
        {error, R} -> {error, R}
    end.
```

### The Fix: Elixir `with`

```elixir
# GOOD: with expression flattens the pipeline
def handle(req) do
  with {:ok, data} <- parse(req),
       :ok <- validate(data),
       {:ok, id} <- store(data) do
    {:ok, id}
  end
end
```

### Techniques to Flatten Code

| Technique | Language | When to Use |
|-----------|----------|-------------|
| Pattern matching on function heads | Both | Multiple cases with different handling |
| Helper functions with meaningful names | Both | Complex steps that deserve names |
| `with` expression | Elixir | Chained fallible operations |
| Pipeline (`\|>`) | Elixir | Sequential transformations |
| List comprehension | Both | Filter + transform |

---

## 6. DECLARATIVE OVER IMPERATIVE

**Severity:** STRONG RULE
**Applies to:** Erlang, Elixir

### The Rule

> **Write declarative code. State WHAT should happen, not HOW to do it step by step.**

### Avoid Boolean Blindness

```erlang
%% BAD: Boolean return forces a second lookup
case api:resource_exists(ID) of
    true ->
        Resource = api:fetch_resource(ID),     %% duplicate lookup!
        process(Resource);
    false ->
        not_found
end.

%% GOOD: Return carries the evidence
case api:fetch_resource(ID) of
    {ok, Resource} -> process(Resource);
    not_found -> not_found
end.
```

### Prefer List Comprehensions Over Map+Filter

```erlang
%% IMPERATIVE
Results = lists:map(fun(X) -> transform(X) end,
             lists:filter(fun(X) -> is_valid(X) end, Items)).

%% DECLARATIVE
Results = [transform(X) || X <- Items, is_valid(X)].
```

### Declarative State Updates (Erlang)

```erlang
%% GOOD: Chain optional updates declaratively
apply_refined(E, State) ->
    S1 = maybe_update(brief, E, State),
    S2 = maybe_update(repos, E, S1),
    maybe_update(tags, E, S2).

maybe_update(Field, E, State) ->
    case get_value(Field, E) of
        undefined -> State;
        Value -> setelement(field_pos(Field), State, Value)
    end.
```

### Declarative Pipelines (Elixir)

```elixir
# IMPERATIVE: Step-by-step mutation tracking
def process(data) do
  validated = validate(data)
  enriched = enrich(validated)
  result = transform(enriched)
  save(result)
end

# DECLARATIVE: Pipeline
def process(data) do
  data
  |> validate()
  |> enrich()
  |> transform()
  |> save()
end
```

---

## 7. FUNCTION AND MODULE SIZE

**Severity:** STRONG RULE

### The Rules

| Element | Limit | Source |
|---------|-------|--------|
| Function body | **15-20 lines** maximum | Erlang Programming Rules |
| Module | **400 lines** maximum | Erlang Programming Rules |
| Function arity | **5 parameters** maximum | Credo, common sense |

> "Don't write functions with more than 15 to 20 lines of code. Split large functions into smaller ones."

> "A module should not contain more than 400 lines of source code."

If a module exceeds 400 lines, split it. If a function exceeds 20 lines, extract helpers.

---

## 8. CONSISTENT ERROR RETURNS

**Severity:** STRONG RULE

### The Rule

> **Always use `{ok, Value}` / `{error, Reason}` tuples. Never mix wrapped and unwrapped returns.**

```erlang
%% GOOD: Consistent tagged tuples
validate(X) ->
    case check(X) of
        true -> {ok, X};
        false -> {error, invalid}
    end.

%% BAD: Mixed — sometimes wrapped, sometimes not
validate(X) ->
    case check(X) of
        true -> X;                    %% Unwrapped!
        false -> {error, invalid}     %% Wrapped!
    end.
```

### Assertion Pattern (When Failure Is a Bug)

When you expect success and failure means a programming bug:

```erlang
%% Pattern match asserts success — crashes on failure (which is correct)
{ok, Value} = operation().
ok = file:write(Fd, Data).
```

---

## 9. NO MAGIC NUMBERS OR STRINGS

```erlang
%% BAD
handle_status(5) -> ...

%% GOOD
-define(STATUS_COMPLETED, 5).
handle_status(?STATUS_COMPLETED) -> ...
```

---

## 10. NO COMMENTED-OUT CODE

Delete it. Git remembers.

```erlang
%% BAD
%% Old implementation:
%% handle(X) -> do_old_thing(X).
handle(X) -> do_new_thing(X).

%% GOOD
handle(X) -> do_new_thing(X).
```

---

## 11. MEANINGFUL NAMES

```erlang
%% BAD
handle_data(D) ->
    R = process(D),
    save(R).

%% GOOD
handle_order(OrderData) ->
    ProcessedOrder = validate_and_enrich(OrderData),
    persist_order(ProcessedOrder).
```

---

## 12. ERLANG-SPECIFIC RULES

### Never Use `if` in Erlang

The `if` expression in Erlang is almost never the right choice. Use function heads or `case`.

```erlang
%% BAD: Erlang if
check_age(Age) ->
    if
        Age >= 18 -> adult;
        Age < 18 -> minor
    end.

%% GOOD: Function heads with guards
check_age(Age) when Age >= 18 -> adult;
check_age(_Age) -> minor.
```

### Use `andalso`/`orelse`, Not `and`/`or`

```erlang
%% BAD: and/or evaluate both sides
is_valid(X) when is_integer(X) and X > 0 -> true.

%% GOOD: andalso/orelse short-circuit
is_valid(X) when is_integer(X) andalso X > 0 -> true.

%% BEST in guards: Use comma (and) and semicolon (or)
is_valid(X) when is_integer(X), X > 0 -> true.
```

### No `begin...end` Blocks

```erlang
%% BAD: begin/end is a code smell — extract a function
Result = begin
    A = compute_x(),
    B = compute_y(A),
    combine(A, B)
end.

%% GOOD: Extract to a function
Result = compute_combined().

compute_combined() ->
    A = compute_x(),
    B = compute_y(A),
    combine(A, B).
```

### Predicate Functions: `is_` or `has_` Prefix

```erlang
%% GOOD
is_valid(X) -> ...
has_permission(User, Action) -> ...

%% BAD
valid(X) -> ...
check_permission(User, Action) -> ...     %% "check" is imperative
```

---

## 13. ELIXIR-SPECIFIC RULES

### Use `with` to Flatten Nested Case

```elixir
# BAD: Pyramid of doom
def process(params) do
  case fetch_user(params["id"]) do
    {:ok, user} ->
      case validate(user) do
        :ok ->
          case update(user) do
            {:ok, updated} -> {:ok, updated}
            {:error, r} -> {:error, r}
          end
        {:error, r} -> {:error, r}
      end
    {:error, r} -> {:error, r}
  end
end

# GOOD: with flattens the pipeline
def process(params) do
  with {:ok, user} <- fetch_user(params["id"]),
       :ok <- validate(user),
       {:ok, updated} <- update(user) do
    {:ok, updated}
  end
end
```

### Only `<-` Clauses Inside `with`

```elixir
# BAD: Non-matching clauses inside with
with ref = make_ref(),
     {:ok, user} <- create(ref) do
  user
end

# GOOD: Move non-matching code outside
ref = make_ref()
with {:ok, user} <- create(ref) do
  user
end
```

### Prefer Implicit Try

```elixir
# BAD: Explicit try wrapping entire body
def parse(input) do
  try do
    to_string(input)
  rescue
    _ -> :error
  end
end

# GOOD: Implicit try (when rescue IS needed)
def parse(input) do
  to_string(input)
rescue
  _ -> :error
end
```

### No `unless...else`

```elixir
# BAD
unless condition do
  a()
else
  b()
end

# GOOD
if condition do
  b()
else
  a()
end
```

---

## Quick-Reference Decision Table

| Situation | Erlang | Elixir |
|-----------|--------|--------|
| Expected failure | `{ok, V}` / `{error, R}` + `case` | `{:ok, v}` / `{:error, r}` + `case` or `with` |
| Unexpected failure | Let it crash | Let it crash |
| Multiple shapes | Function head matching | Multi-clause functions |
| Sequential fallible ops | Extract helper functions | `with` expression |
| External input | Guards on public exports | Guards on public functions |
| Internal helper | No guards — crash on bad input | No guards — crash on bad input |
| Boolean dispatch | Avoid — return evidence | Avoid — return evidence |
| Collection transform | List comprehension `[f(X) \|\| X <- L]` | `Enum.map/2` or comprehension |
| Nesting > 1 level | Extract to named function | Extract to named function or `with` |
| Error logging | `logger:error/2` then crash | `Logger.error/1` then crash |
| Control flow | Function heads + guards | Function heads + guards + `with` |
| Never | `if`, `try/catch` (catch-all), `begin...end` | `try/rescue` (catch-all), `unless...else` |

---

## Self-Audit Checklist

Before committing code, verify:

- [ ] **No try/catch** unless at system boundary with specific error + log/re-raise
- [ ] **No silent error swallowing** — every catch block either logs+re-raises or handles a specific, named error
- [ ] **No defensive guards** on internal functions — only at public API boundaries
- [ ] **No `if`** in Erlang — use function heads or `case`
- [ ] **No nesting > 1 level** — extract helper functions
- [ ] **No magic numbers** — use defines/macros
- [ ] **No commented-out code** — delete it
- [ ] **No boolean returns** where evidence-carrying tuples would serve better
- [ ] **No `undefined` propagation** — crash on unexpected nil/undefined
- [ ] **Functions under 20 lines** — split if longer
- [ ] **Modules under 400 lines** — split if longer
- [ ] **Consistent `{ok, V}` / `{error, R}`** — no mixed return patterns
- [ ] **Meaningful names** — no single-letter variables outside comprehensions

---

*Sources: [Erlang Programming Rules](https://cndoc.github.io/Erlang-ProgrammingRules-cn/doc_en.html), [Inaka Guidelines](https://github.com/inaka/erlang_guidelines), [WhatsApp erlfmt](https://github.com/WhatsApp/erlfmt/blob/main/StyleGuide.md), [Jesper Louis Andersen](https://jlouis.github.io/posts/erlang-and-code-style/), [Fred Hebert](https://ferd.ca/it-s-about-the-guarantees.html), [Learn You Some Erlang](https://learnyousomeerlang.com/errors-and-exceptions), [Elixir Official Docs](https://hexdocs.pm/elixir/try-catch-and-rescue.html), [Credo](https://github.com/rrrene/credo)*
