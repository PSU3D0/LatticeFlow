Status: Draft
Owner: Runtime
Epic: 02 (Workers host minimal)
Phase: 02.3

# Ticket: host-workers Miniflare/workerd harness

## Goal
Provide a repeatable Miniflare/workerd-based E2E harness for `host-workers` that validates the Workers execution path (fetch → invocation → execution → response) and serves as the baseline for future Workers capability tests.

## Context (existing harness patterns)
- `crates/cap-http-workers/workerd-tests/` runs Vitest + Miniflare with a standalone wasm build (`worker-build`) and a `wrangler.toml` pointing at `build/worker/shim.mjs`.
- JS harness uses Miniflare modules rules to load `.wasm`, and dispatches `fetch` via `mf.dispatchFetch`.
- A standalone Rust test worker crate is used to exercise the capability and return responses.

## Scope
- Add a `workerd-tests` harness under `crates/host-workers/` modeled on the `cap-http-workers` harness.
- Provide a minimal Worker test entrypoint that wires `host-workers` and serves:
  - `/health` for sanity
  - `/echo` for baseline invocation
  - `/stream` for streaming (SSE) validation once ready
  - `/cancel` for cancellation/abort validation
- Add a JS/TS test runner (Vitest + Miniflare) that builds the worker and issues requests through Miniflare.
- Document local run steps and CI wiring.

## Non-goals
- No production deployment or `wrangler deploy` automation (dev harness only).
- No capability provider testing beyond what `host-workers` already routes.
- No full end-to-end CLI packaging (handled by separate flow packaging tickets).

## Requirements
1) **Harness parity with cap-http-workers**
   - Mirror the build + Miniflare structure to reduce cognitive load.
   - Use `worker-build --release` with `wrangler.toml` pointing at the generated shim.

2) **Minimal worker entrypoint**
   - `#[event(fetch)]` entrypoint that calls `host-workers` with a small embedded Flow bundle fixture.
   - Provide deterministic outputs for tests (e.g., `echo` returns body; `stream` emits 3 chunks).

3) **Streaming + cancellation hooks**
   - Add a known stream endpoint for SSE behavior and an abortable endpoint to validate cancellation.
   - Align with Epic 02 streaming/cancellation requirements.

## Harness reuse decision
**Recommendation: Extract shared JS harness utilities, keep per-crate `workerd-tests` crates.**

### Pros (shared utilities)
- Consolidates Miniflare bootstrap (modules rules, wasm loading) and helper `dispatchFetch` utilities.
- Avoids duplicating `vitest.config.ts`, `mf.ts`, and Miniflare setup for each Workers-targeted crate.
- Easier CI maintenance (one place to adjust Miniflare version or config).

### Cons (shared utilities)
- Adds a cross-crate dependency for tests that may be too early in the pipeline.
- Potential version drift if a crate needs a different compatibility date or flags.

### Proposed approach
- Create a lightweight `tools/workers-harness/` (JS-only) with:
  - `createMiniflare({ scriptPath, compatibilityDate, fetchMock })`
  - Shared `vitest` config defaults.
- Each crate keeps its own `workerd-tests/` Rust worker crate and `package.json`, importing the shared harness module.
- If this adds friction, keep the harness local for `host-workers` first, and extract once there are ≥2 consumers (cap-http-workers + host-workers).

## Implementation Plan
1) **Workerd tests scaffold**
   - Add `crates/host-workers/workerd-tests/` with `Cargo.toml`, `package.json`, `vitest.config.ts`, `wrangler.toml`.
   - Mirror `cap-http-workers` structure and versions for consistency.

2) **Test worker crate**
   - Add a minimal worker entrypoint that calls `host-workers` runtime with a known Flow bundle.
   - Expose `/health`, `/echo`, `/stream`, `/cancel` routes to cover core behaviors.

3) **JS test suite**
   - Use Miniflare to dispatch requests and assert response bodies/stream chunks.
   - Add explicit timeout + cancellation tests (abort controller).

4) **Docs + CI**
   - Document `npm test` flow for the harness and `wrangler dev --local` for manual runs.
   - Add optional CI job to run harness in the Workers lane.

## Acceptance Gates (Definition of Done)
- `cargo check -p host-workers --target wasm32-unknown-unknown` succeeds.
- `npm test` in `crates/host-workers/workerd-tests/` passes on CI (Miniflare run).
- `/echo` and `/health` pass in Miniflare harness.
- `/stream` emits multiple chunks and completes cleanly.
- `/cancel` shows cancellation propagation (response aborted or explicit cancellation signal).

## Dependencies
- Epic 02.2 `host-workers` adapter.
- Epic 02 streaming/cancellation changes (if implemented separately).
- `worker` crate and `worker-build` for wasm packaging.

## Test Plan
- `npm install` and `npm test` under `crates/host-workers/workerd-tests/`.
- Verify harness logs show Miniflare boot and compiled wasm load.
- Add CI runner step using Node LTS + `worker-build` install.

## CI
- Add a dedicated Workers harness job:
  - Install Node + `worker-build`.
  - `npm test` in `crates/host-workers/workerd-tests/`.
  - Cache `target/` and `node_modules/` to reduce runtime.

## Notes
- Roadmap currently lists Phase 02.3 as streaming/cancellation; confirm phase numbering for this follow-up ticket.
