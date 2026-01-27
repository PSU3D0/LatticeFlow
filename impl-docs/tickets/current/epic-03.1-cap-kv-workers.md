Status: Draft
Owner: Runtime
Epic: 03 (Capability providers)
Phase: 03.1

# Ticket: Workers KV capability provider

## Goal
Deliver a Workers KV capability provider that maps Flow capability calls to the Workers KV API with clear constraints, configuration, and test coverage.

## Scope
- Add a `cap-kv-workers` adapter crate (or module under `cap-kv`) compiled for `wasm32-unknown-unknown`.
- Bind `capabilities::kv::KeyValue` calls to `worker::kv::KvStore` via `Env::kv`.
- Support KV operations: `get` (text/json/bytes), `get_with_metadata`, `put` (text/bytes/stream), `list`, and `delete`.
- Expose per-write TTL/expiration and metadata fields in the capability surface.
- Populate `KvCapabilityInfo` with Workers KV consistency/durability/TTL descriptors.
- Document Workers KV limits, consistency, cache semantics, and key constraints.

## Non-goals
- No strong consistency or transactions (use Durable Objects for that).
- No alternate KV providers or cross-provider portability layer.
- No bulk import/export, analytics, or namespace management.

## Requirements
1) **Capability API surface**
   - Implement `capabilities::kv::KeyValue` + `KvCapabilityInfo` (align with `cap-kv-opendal`).
   - `get` returns text/json/bytes with optional metadata payload; surface `cache_ttl` on reads.
   - `put` accepts `expiration` (unix timestamp seconds), `expiration_ttl` (seconds), and `metadata` (JSON-serializable).
   - `list` supports `prefix`, `cursor`, and `limit`, returning `{ keys, list_complete, cursor }` with optional `expiration`/`metadata` per key (limit max 1000).
   - `delete` is async and returns unit; missing keys are not errors.

2) **Workers API mapping**
   - Use `Env::kv("<BINDING>")` to acquire a `KvStore`.
   - Implement `get` with `text/json/bytes` and `get_with_metadata` equivalents (single key only).
   - Implement `put` using `put`/`put_bytes`/`put_stream` and call `.execute()`.
   - Implement `list` using `limit/cursor/prefix` options.

3) **Limits & consistency**
   - Enforce/validate KV limits: key <= 512 bytes, metadata <= 1 KiB, value <= 25 MiB.
   - Reject empty keys and `.`/`..` (Workers KV constraint).
   - `expiration`/`expiration_ttl` and `cache_ttl` must be >= 60 seconds.
   - Keys are listed in lexicographic order (UTF-8 bytes).
   - Acknowledge eventual consistency and negative lookup cache behavior.
   - Respect rate limits: 1 write/sec per key and 1k external ops per invocation.
   - Populate `KvCapabilityInfo`: consistency=eventual, durability=durable, ttl=per_write (with 60s minimum).

## Wrangler Config
Include example bindings and environment overrides in docs/tests:

```toml
kv_namespaces = [
  { binding = "FLOW_KV", id = "<id>", preview_id = "<preview_id>" }
]

[env.staging]
kv_namespaces = [
  { binding = "FLOW_KV", id = "<staging_id>", preview_id = "<staging_preview_id>" }
]
```

## Implementation Plan
1) **Capability surface**
   - Define request/response types for KV operations, including metadata and TTL.
   - Populate `KvCapabilityInfo` and map errors to capability error codes (include limit violations).

2) **Workers adapter**
   - Acquire `KvStore` from `Env` based on binding name.
   - Implement CRUD methods using workers-rs APIs.

3) **Docs + examples**
   - Document binding setup and behavior constraints.
   - Provide a minimal Flow example invoking KV read/write.

## Acceptance Gates (Definition of Done)
- `cargo check -p cap-kv-workers --target wasm32-unknown-unknown` succeeds.
- KV read/write/list/delete works in a Workers dev harness with `wrangler dev`.
- TTL/expiration and metadata are round-tripped in the harness scenario.
- Docs include Workers KV limits, consistency notes, and `wrangler.toml` example.

## Dependencies
- Epic 02.2 host-workers adapter (Workers execution environment).
- Capability binding surfaces in `impl-docs/spec/capabilities-and-binding.md`.
- workers-rs `worker` crate KV APIs (`worker::kv`).

## Tests
- Integration test in Workers dev harness covering: put/get/get_with_metadata/list/delete.
- Limit validation tests for key/metadata/value sizes and `cache_ttl` minimum.
- Negative cache behavior described in docs (manual verification acceptable).

## Risks / Constraints
- KV is eventually consistent; reads after writes may lag.
- Negative lookup cache can mask newly written keys briefly.
- Per-key write rate (1/sec) and per-invocation external op limit (1k) can throttle flows.
- Large values (25 MiB max) can exceed memory budgets in Workers.

## Files to Touch (expected)
- `crates/cap-kv-workers/*` (new)
- `Cargo.toml` workspace members
- `impl-docs/spec/capabilities-and-binding.md` (if API changes)
