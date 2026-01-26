Status: Draft
Owner: Runtime
Epic: 02 (Workers host minimal)
Phase: 02.1

# Ticket: Kernel-exec wasm readiness (retain native performance)

## Goal
Make `kernel-exec` build and run under `wasm32-unknown-unknown` without regressing native performance or semantics. Preserve the existing scheduler/backpressure model while removing OS-only dependencies and enabling a constrained Workers runtime profile.

## Scope
- Introduce a wasm-compatible runtime profile for `kernel-exec`.
- Remove or feature-gate OS-only dependencies (notably filesystem spill).
- Keep the native Tokio path fast and unchanged by default.

## Non-goals
- Do not implement Workers host adapter here (separate ticket).
- Do not add Cloudflare capability providers (Epic 03).
- Do not change Flow IR semantics or contract surfaces.

## Requirements
1) **Runtime compatibility**
   - `kernel-exec` must compile on `wasm32-unknown-unknown`.
   - wasm profile must avoid `tokio::fs`, `tempfile`, `tokio::net`, `tokio::signal`.
   - wasm profile uses a single-threaded async runtime model.

2) **Spill strategy (ADR-0003)**
   - Spill-to-filesystem is disabled under wasm.
   - If `spill_tier` is configured without a wasm-supported tier, preflight must reject deterministically (e.g., `CTRL901` or a new spill-specific code).
   - Keep the path open for future blob-backed spill tiers.

3) **Native parity**
   - Native build should retain existing behavior and performance.
   - No changes required for existing host bridges (Axum, queue, etc.) unless compiled with wasm features.

## Design Notes
- Prefer a minimal runtime abstraction to avoid scattering `cfg(wasm32)` across the scheduler.
- If a runtime trait is introduced, define it in `kernel-exec` (or a small `kernel-rt` module) with:
  - `spawn`, `sleep`, `timeout`, `channel`/`semaphore` primitives,
  - and an async time interface.
- If a runtime trait is not introduced, use feature flags (`kernel-exec/wasm`) with conditional imports and a reduced Tokio feature set for wasm.

## Implementation Plan
1) **Define build profiles**
   - Add `kernel-exec` feature flag: `wasm`.
   - Move Tokio features to per-crate scope; wasm uses only `rt`, `time`, `sync`, `macros`.

2) **Spill gating**
   - Wrap `tempfile`/`tokio::fs` usage behind `#[cfg(not(target_arch = "wasm32"))]` or `feature = "spill-fs"`.
   - Update spill initialization to return a deterministic error when spill is configured but unsupported under wasm.

3) **Runtime abstraction (decision)**
   - Option A: runtime trait (preferred if feasible within ticket scope).
   - Option B: explicit cfg/feature gating with minimal code changes.

4) **Tests**
   - Add a `cargo check --target wasm32-unknown-unknown` CI step for the minimal kernel set.
   - Ensure existing unit/property tests continue to pass under native builds.

## Acceptance Gates (Definition of Done)
- `cargo check -p kernel-exec --target wasm32-unknown-unknown --no-default-features -F wasm` succeeds.
- Native `cargo test -p kernel-exec` remains green.
- Spill settings are rejected deterministically under wasm (documented error).
- ADR-0003 constraints are reflected in code/docs (as needed).

## Files to Touch (expected)
- `crates/kernel-exec/src/lib.rs`
- `crates/kernel-exec/Cargo.toml`
- `Cargo.toml` (workspace tokio features if needed)
- Optional: `impl-docs/adrs/adr-0003-workers-native-constraints-and-spill.md` (update notes)

## Notes
- Keep host/bridge crates unchanged; wasm builds should exclude them via feature flags or target-specific build sets.
