# Task Brief — Broader Capability Coverage

## Goal
Extend the capability registry and foundational adapters so Phase 2 flows (and forthcoming Phase 3–5 work) have the metadata, hint propagation, and stubs required for common infrastructure domains (HTTP, DB, cache, blob, queue, clock, RNG, etc.). This sets the stage for post-Phase 5 targets such as Cloudflare Workers (KV, R2, Queues) and Neon Postgres plugins.

## Deliverables
1. **Capability Registry Expansion (`crates/capabilities`)**
   - Add dedicated modules + hint sets for:
     - `blob` (e.g., R2 / S3 write vs. read)
     - `queue` (enqueue/dequeue semantics)
     - `kv` (Workers KV read/write)
     - `db` (Postgres read/write, transaction hints)
     - `rng` / `crypto` (determinism-only hints)
   - Register effect/determinism constraints with detailed guidance (align with `impl-docs/error-taxonomy.md`).
   - Unit tests ensuring `hints::infer` returns canonical IDs for these domains.

2. **Macro Integration (`dag-macros`)**
   - Verify resource alias patterns produce the new hints automatically.
   - Add trybuild cases for blob/queue mismatches similar to existing HTTP/DB fixtures.

3. **Example Capability Providers**
   - Lightweight in-memory stubs for KV, queue, blob (under `crates/capabilities/tests` or `examples/`).
   - Ensure they implement the hint registration functions (enables runtime wiring later).

4. **Documentation**
   - Update `impl-docs/rust-workflow-tdd-rfc.md` capability tables with new hint IDs, default severities, and example usage.
   - Note integration expectations for Cloudflare Workers / Neon Postgres in `impl-docs/impl-plan.md` (Phase 4+ outlook).

## Testing Checklist
- `cargo test -p capabilities` (new inference tests).
- `cargo test -p dag-macros --test trybuild` (new compile-fail fixtures).
- Optional doctests illustrating capability usage (especially for future Workers KV/R2 wrappers).

## Notes
- Keep hint names stable (`resource::<domain>::<operation>`)—these will map into Flow IR schema (`effectHints`, `determinismHints`).
- Prepare for future plugin surface: record which hints correspond to “service connectors” vs. “third-party agnostic extensions” so validation can enforce correct policy later.

## Status (2025-10-14)
- Canonical hints for HTTP, DB (read/write), KV, blob, queue, clock, and RNG are implemented in `crates/capabilities`, and the compile-time registries populate automatically via `hints::infer`.
- `dag-macros` emits the new hints and `trybuild` now covers queue publish and blob determinism conflicts.
- In-memory stubs (`MemoryKv`, `MemoryBlobStore`, `MemoryQueue`) ship alongside `MemoryCache`/`SystemClock` for local tests; future platform adapters should wrap these traits to inherit diagnostics.
