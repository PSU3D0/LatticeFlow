Status: Draft
Owner: Runtime
Epic: 03 (Capability providers)
Phase: 03.2

# Ticket: Durable Objects capability provider

## Goal
Deliver a Durable Objects-backed capability provider for sequencing/dedupe that maps Flow capability calls to the Workers Durable Objects API with clear configuration, limits, and test coverage.

## Scope
- Add a `cap-do-workers` adapter crate compiled for `wasm32-unknown-unknown`.
- Implement `capabilities::dedupe::DedupeStore` (single-writer/idempotency ledger primitives) using Durable Objects storage.
- Bind capability calls to `worker::Env::durable_object` and `worker::durable::ObjectNamespace` APIs.
- Support namespace resolution, object ID creation, and stub invocation via fetch for DO-backed state helpers.
- Expose storage, alarms, and SQLite-backed storage surfaces where the sequencer needs them.
- Document Durable Objects limits, migrations, and plan constraints (SQLite vs key-value backends).

## Non-goals
- No alternate DO providers or portability layer across runtimes.
- No management APIs for DO namespace lifecycle outside `wrangler` migration workflows.
- No auto-scaling or sharding strategies beyond Workers default placement.

## Requirements
1) **Capability API surface**
   - Implement `capabilities::dedupe::DedupeStore` (align error semantics with `DedupeError`).
   - Namespace binding: `binding` string resolves to a DO `ObjectNamespace`.
   - Object IDs: support `id_from_name`, `id_from_string`, `unique_id`, and `unique_id_with_jurisdiction`.
   - Stub access: `get_by_name`/`get_stub` helpers to perform `fetch` calls.
   - Optional helpers for `alarm`, `websocket`, and `wait_until` usage where supported.

2) **Workers API mapping**
   - Use `Env::durable_object("<BINDING>")` to acquire `ObjectNamespace`.
   - Implement ID creation via `ObjectNamespace::id_from_name`, `id_from_string`, `unique_id`, `unique_id_with_jurisdiction`.
   - Resolve object stubs via `ObjectNamespace::get_by_name` or `ObjectId::get_stub`, then call `Stub::fetch_*`.
   - DO implementations must be `#[durable_object]` structs implementing `DurableObject`.

3) **State, storage, and alarms**
   - Use `State::storage()` for get/put/list/delete and `transaction` support.
   - Support alarm operations (`get_alarm`, `set_alarm`, `delete_alarm`) for TTL expiry.
   - Support SQLite storage via `State::storage().sql()` with `exec` and cursor iteration.

4) **Limits & constraints (from CF docs)**
   - SQLite-backed DOs: storage per object 10 GB; key + value <= 2 MB.
   - SQLite storage per account: 5 GB (Free) / unlimited (Paid); max DO classes 100 (Free) / 500 (Paid).
   - SQL limits: 100 columns/table, row size 2 MB, statement length 100 KB, bound params <= 100, SQL function args <= 32, LIKE/GLOB pattern <= 50 bytes.
   - CPU per request defaults to 30s and is configurable to 5 min via `limits.cpu_ms`.
   - WebSocket message size limit: 32 MiB (received messages).
   - Soft throughput target ~1k req/sec per object; overload errors are possible under bursty load.
   - Key-value backed DOs (paid only): key <= 2 KiB, value <= 128 KiB; storage per account 50 GB; storage per object unlimited.
   - Workers Free plan supports SQLite-backed DOs only; key-value backend requires Workers Paid.

## Wrangler Config
Include example bindings, migrations, and CPU limits in docs/tests:

```toml
[durable_objects]
bindings = [
  { name = "FLOW_DO", class_name = "FlowDurableObject" }
]

[[migrations]]
tag = "v1"
new_classes = ["FlowDurableObject"]

[[migrations]]
tag = "v2"
new_sqlite_classes = ["FlowSqlDurableObject"]

[limits]
cpu_ms = 300_000
```

Notes:
- Migrations are required for create/rename/delete/transfer operations.
- You cannot switch an existing class to SQLite later; must define a new class.
- Migration validation is deployment-driven (use `wrangler deploy`).

## Implementation Plan
1) **Capability surface**
   - Implement `DedupeStore` semantics with a deterministic mapping from dedupe key -> object id.
   - Map Workers errors to `DedupeError::Other`, include binding-missing/invalid-id/limit violations.

2) **Workers adapter**
   - Acquire `ObjectNamespace` from `Env` based on binding name.
   - Implement stub fetch forwarding with request/response mapping for internal DO helpers.
   - Expose storage, alarms, and SQLite entry points needed by the sequencer logic.

3) **Docs + examples**
   - Document binding setup, migrations, limits, and plan constraints.
   - Provide a minimal Flow example invoking dedupe operations against the DO-backed provider.

## Acceptance Gates (Definition of Done)
- `cargo check -p cap-do-workers --target wasm32-unknown-unknown` succeeds.
- Dedupe operations (`put_if_absent`, `forget`) work in a Workers dev harness with `wrangler dev`.
- Concurrency test demonstrates per-key serialization (no double-write for same key).
- Storage operations (get/put/list/delete), alarms, and SQLite exec/cursor work in the harness.
- Docs include DO limits, plan constraints, and `wrangler.toml` example with migrations.

## Dependencies
- Epic 02.2 host-workers adapter (Workers execution environment).
- Capability binding surfaces in `impl-docs/spec/capabilities-and-binding.md`.
- `capabilities::dedupe` interface in `crates/capabilities`.
- workers-rs `worker` crate Durable Objects APIs (`worker::durable`).

## Tests
- Integration test in Workers dev harness covering: dedupe `put_if_absent` + `forget`, ID creation, stub fetch, storage CRUD.
- Alarm schedule/trigger test in harness (manual or scripted).
- SQLite storage smoke test: exec + cursor iteration.
- Migration validation via `wrangler deploy` in a dedicated test environment.

## Risks / Constraints
- Migration mistakes are only caught at deploy time; requires safe rollout discipline.
- SQLite-backed DOs cannot be converted to key-value backed DOs (must create a new class).
- DO per-request CPU and throughput limits can throttle heavy flows.
- WebSocket message size and DO storage limits may constrain payload-heavy workloads.

## Files to Touch (expected)
- `crates/cap-do-workers/*` (new)
- `Cargo.toml` workspace members
- `impl-docs/spec/capabilities-and-binding.md` (if API changes)
