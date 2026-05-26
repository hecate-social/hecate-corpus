---
title: Testing Erlang Checklists
layer: skill
audience: [agent, human]
stage: stable
---

# Testing Erlang Checklists

Per-app test coverage checklist and template variable reference for all domain lifecycle apps.

---

## Per CMD App Test Coverage Checklist

For each CMD app, verify ALL of the following exist and pass:

### Layer 1: Dossier

- [ ] `initial_state_test` — zeroed record, status=0
- [ ] `init_callback_test` — `init/1` returns `{ok, initial_state}`
- [ ] One `apply_{event}_test` per event type — status bit set, fields populated
- [ ] `apply_refine_partial_test` (if applicable) — partial updates preserve other fields
- [ ] `full_lifecycle_state_test` — chain all events, final status = sum of all bits
- [ ] `apply_unknown_event_test` — unknown event leaves state unchanged
- [ ] `apply_with_atom_keys_test` — atom-keyed events work identically
- [ ] `flag_map_test` — `flag_map/0` returns expected labels
- [ ] `apply_callback_order_test` — `apply(State, Event)` state-first order

### Layer 2: Commands

Per command module:
- [ ] `new_valid_test` — valid params return `{ok, Cmd}`
- [ ] `new_with_all_fields_test` — all optional fields populated
- [ ] `new_missing_fields_test` — missing required fields return `{error, missing_required_fields}`
- [ ] `from_map_binary_keys_test` — binary-keyed map works
- [ ] `from_map_atom_keys_test` — atom-keyed map works (if applicable)
- [ ] `to_map_round_trip_test` — `to_map -> from_map` preserves all fields
- [ ] `validate_valid_test` — valid command passes validation
- [ ] `accessor_functions_test` — each `get_*/1` returns correct field

### Layer 2: Handlers

Per handler module:
- [ ] `handle_valid_test` — valid command returns `{ok, [Event]}`
- [ ] `handle_event_fields_match_test` — event fields match command fields

### Layer 2: Aggregate execute/2

- [ ] One success test per command type on valid state
- [ ] One error test per guard clause (see error map below)
- [ ] `execute_unknown_command_test`
- [ ] `execute_missing_command_type_test`
- [ ] `execute_evoq_callback_order_test` — verify `execute(State, Payload)` order
- [ ] `execute_repeat_operation_test` (if applicable) — e.g., refine after refine

### Layer 2: Events

Per event module:
- [ ] `new_test` — `new/1` creates event with auto-timestamp
- [ ] `to_map_includes_event_type_test` — `<<"event_type">>` key present
- [ ] `round_trip_test` — `to_map -> from_map` preserves all fields
- [ ] `from_map_invalid_test` — empty map returns `{error, invalid_event}`
- [ ] `timestamp_auto_test` — auto-generated timestamp is in valid range

### Layer 3: Integration (in QRY app)

- [ ] `setup_projects_to_sqlite` — projection inserts row
- [ ] `query_by_id_found` — returns projected data with all fields
- [ ] `query_by_id_not_found` — returns `{error, not_found}`
- [ ] `archive_updates_status` — sets ARCHIVED bit
- [ ] `page_returns_list` — returns multiple domains
- [ ] `page_excludes_archived` — default filter hides archived
- [ ] `page_includes_archived` — filter override shows archived
- [ ] `page_limit_offset` — pagination works
- [ ] `projection_idempotency` — duplicate projection = no duplicate row
- [ ] `full_cqrs_flow` — command -> handler -> event -> projection -> query

### Layer 4: Side Effects

Per emitter:
- [ ] `emit_reaches_members` — pg broadcast delivers message
- [ ] `emit_message_format` — message is `{group_atom, EventMap}` tuple
- [ ] `emit_with_no_members` — no crash when group is empty
- [ ] `emit_reaches_multiple_members` — all members receive

Per projection:
- [ ] `projection_creates_correct_row` — all columns have expected values
- [ ] `projection_status_label_correct` — label matches `evoq_bit_flags:to_string`
- [ ] `projection_json_fields_encode` — JSON fields decode correctly
- [ ] `projection_undefined_fields_handled` — NULL for optional fields
- [ ] `archive_preserves_existing_bits` — bitwise OR preserves previous flags

### Layer 4c: ReckonDB Dispatch

Per CMD app (except guide_venture):
- [ ] `dispatch_persists_event` — event exists in ReckonDB after dispatch
- [ ] `dispatch_returns_version_and_events` — return type is `{ok, int(), [map()]}`
- [ ] `dispatch_second_command_appends_to_stream` — two commands = two events in stream
- [ ] `read_back_event_data_matches` — event data in store matches dispatched command

---

## Template Variable Tables

### CMD Apps

| # | App | Aggregate | Status HRL | Init Event | Archive Event | Flags |
|---|-----|-----------|-----------|------------|---------------|-------|
| 1 | `setup_venture` | `setup_aggregate` | `venture_status.hrl` | `venture_setup_v1` | `venture_archived_v1` | SETUP=1, REFINED=2, SUBMITTED=4, ARCHIVED=8 |
| 2 | `discover_divisions` | `discovery_aggregate` | `discovery_status.hrl` | `discovery_started_v1` | `discovery_archived_v1` | INITIATED=1, ACTIVE=2, PAUSED=4, COMPLETED=8, ARCHIVED=16 |
| 3 | `design_division` | `design_aggregate` | `design_status.hrl` | `design_started_v1` | `design_archived_v1` | (same lifecycle pattern) |
| 4 | `plan_division` | `plan_aggregate` | `plan_status.hrl` | `plan_started_v1` | `plan_archived_v1` | (same lifecycle pattern) |
| 5 | `generate_division` | `generation_aggregate` | `generation_status.hrl` | `generation_started_v1` | `generation_archived_v1` | (same lifecycle pattern) |
| 6 | `test_division` | `testing_aggregate` | `testing_status.hrl` | `testing_started_v1` | `testing_archived_v1` | (same lifecycle pattern) |
| 7 | `deploy_division` | `deployment_aggregate` | `deployment_status.hrl` | `deployment_started_v1` | `deployment_archived_v1` | (same lifecycle pattern) |
| 8 | `monitor_division` | `monitoring_aggregate` | `monitoring_status.hrl` | `monitoring_started_v1` | `monitoring_archived_v1` | (same lifecycle pattern) |
| 9 | `rescue_division` | `rescue_aggregate` | `rescue_status.hrl` | `rescue_started_v1` | `rescue_archived_v1` | (same lifecycle pattern) |
| 10 | `guide_venture` | `guide_aggregate` | `guide_status.hrl` | `guide_started_v1` | `guide_archived_v1` | (passive orchestrator) |

### QRY Apps

| # | App | Store | Table | Source CMD |
|---|-----|-------|-------|-----------|
| 1 | `query_ventures` | `query_ventures_store` | `domains` | `setup_venture` |
| 2 | `query_discoveries` | `query_discoveries_store` | `discoveries` | `discover_divisions` |
| 3 | `query_designs` | `query_designs_store` | `designs` | `design_division` |
| 4 | `query_plans` | `query_plans_store` | `plans` | `plan_division` |
| 5 | `query_generations` | `query_generations_store` | `generations` | `generate_division` |
| 6 | `query_tests` | `query_tests_store` | `tests` | `test_division` |
| 7 | `query_deployments` | `query_deployments_store` | `deployments` | `deploy_division` |
| 8 | `query_monitoring` | `query_monitoring_store` | `monitoring` | `monitor_division` |
| 9 | `query_rescues` | `query_rescues_store` | `rescues` | `rescue_division` |

---

## Error Map Reference (setup_venture)

These are the aggregate guard clause errors for `setup_venture`. Other lifecycle apps follow the same pattern with domain-specific error atoms.

| Command | State Condition | Error |
|---------|----------------|-------|
| `setup_venture` | Already setup (status != 0) | `venture_already_setup` |
| `refine_vision` | Not found (venture_id = undefined) | `venture_not_found` |
| `refine_vision` | Archived | `venture_archived` |
| `refine_vision` | Not setup | `venture_not_setup` |
| `refine_vision` | Already submitted | `vision_already_submitted` |
| `submit_vision` | Not found | `venture_not_found` |
| `submit_vision` | Archived | `venture_archived` |
| `submit_vision` | Not setup | `venture_not_setup` |
| `submit_vision` | Already submitted | `vision_already_submitted` |
| `archive_venture` | Not found | `venture_not_found` |
| `archive_venture` | Already archived | `venture_already_archived` |
| `archive_venture` | Not setup | `venture_not_setup` |
| Any | Unknown command_type | `unknown_command` |

---

## Pre-Release Test Verification

Before tagging a release:

```bash
# CMD app tests (L1+L2) — use --app= (module names match app modules)
rebar3 eunit --app=setup_venture
rebar3 eunit --app=discover_divisions
rebar3 eunit --app=design_division
rebar3 eunit --app=plan_division
rebar3 eunit --app=generate_division
rebar3 eunit --app=test_division
rebar3 eunit --app=deploy_division
rebar3 eunit --app=monitor_division
rebar3 eunit --app=rescue_division
rebar3 eunit --app=guide_venture

# QRY app tests (L3+L4) — MUST use --module= (test module names don't match app modules)
rebar3 eunit --module=venture_cqrs_integration_tests,venture_projection_tests
rebar3 eunit --module=discovery_cqrs_integration_tests,discovery_projection_tests
rebar3 eunit --module=design_cqrs_integration_tests,design_projection_tests
rebar3 eunit --module=plan_cqrs_integration_tests,plan_projection_tests
rebar3 eunit --module=generation_cqrs_integration_tests,generation_projection_tests
rebar3 eunit --module=testing_cqrs_integration_tests,testing_projection_tests
rebar3 eunit --module=deployment_cqrs_integration_tests,deployment_projection_tests
rebar3 eunit --module=monitoring_cqrs_integration_tests,monitoring_projection_tests
rebar3 eunit --module=rescue_cqrs_integration_tests,rescue_projection_tests

# L4c: Dispatch tests (requires ReckonDB + evoq apps running)
rebar3 eunit --module=setup_venture_dispatch_tests
rebar3 eunit --module=discover_divisions_dispatch_tests
rebar3 eunit --module=design_division_dispatch_tests
rebar3 eunit --module=plan_division_dispatch_tests
rebar3 eunit --module=generate_division_dispatch_tests
rebar3 eunit --module=test_division_dispatch_tests
rebar3 eunit --module=deploy_division_dispatch_tests
rebar3 eunit --module=monitor_division_dispatch_tests
rebar3 eunit --module=rescue_division_dispatch_tests

# Dialyzer
rebar3 dialyzer

# Compile warnings (warnings_as_errors applies — one unused function blocks everything)
rebar3 compile
```

**Why `--app=` vs `--module=`?** `rebar3 eunit --app=X` uses EUnit's application discovery, which finds `{module}_tests` modules matching application module names. CMD test modules like `setup_aggregate_tests` match `setup_aggregate`. But QRY test modules like `venture_cqrs_integration_tests` don't match any app module, so `--module=` is required.

---

## Rollout Status

**Completed (2026-02-11):**

All 10 CMD apps and 9 QRY apps have full test coverage:

| Layer | Apps | Tests | Status |
|-------|------|-------|--------|
| L1+L2 (Dossier + Domain) | 10 CMD apps | 137 | Done |
| L3+L4a+L4b (Integration + Side Effects) | 9 QRY apps | 128 | Done |
| L4c (Dispatch) | 9 CMD apps | 36 | Done |
| **Total** | **19 apps** | **301** | **All passing** |

### Files Created Per QRY App (Pattern)

Each QRY app gets 3 test files:

1. `{domain}_test_store_proxy.erl` — Gen_server proxy registering as `{store_module}`, creates tables, provides `ensure_tuples/1`
2. `{domain}_cqrs_integration_tests.erl` — 10+ tests covering start→query→lifecycle→child entities→idempotency→full flow
3. `{domain}_projection_tests.erl` — 3 tests: correct_flags, label_correct, archive_bits

### CRITICAL: Unique proxy module names

Each QRY app's proxy MUST have a unique module name because all test modules compile into one namespace:

| QRY App | Proxy Module Name |
|---------|-------------------|
| `query_ventures` | `ventures_test_store_proxy` |
| `query_discoveries` | `discoveries_test_store_proxy` |
| `query_designs` | `designs_test_store_proxy` |
| `query_plans` | `plans_test_store_proxy` |
| `query_generations` | `generations_test_store_proxy` |
| `query_tests` | `tests_test_store_proxy` |
| `query_deployments` | `deployments_test_store_proxy` |
| `query_monitoring` | `monitoring_test_store_proxy` |
| `query_rescues` | `rescues_test_store_proxy` |
