Status: Archived
Purpose: notes
Owner: Core
Last reviewed: 2025-12-12

# Task Brief — Property & Behavioural Tests

## Goal
Add the property-based and behavioural tests called out in Phase 2 to ensure backpressure, cancellation, and validation logic remain robust as the runtime evolves.

## Targets
1. **Executor Backpressure (`crates/kernel-exec`)**
   - Property test (e.g., `proptest`) for token-bucket / semaphore limits: vary burst size, cancel timing, and ensure invariants (no more than N in-flight messages, permits released after drain).
   - Deterministic scheduler harness for cancellation propagation (fail fast node ⇒ downstream nodes never start).

2. **Capture Streams**
   - Randomized stream production (vary event counts, inter-arrival) to confirm SSE capture respects ordering and completes gracefully on disconnect.

3. **Validator**
   - Fuzz combinations of hints (effect/determinism) to ensure diagnostics are produced exactly when constraints are violated and absent when satisfied.

4. **CLI / Host Round Trips**
   - Utilize property-driven payload generation (e.g., random strings with whitespace, Unicode) to ensure normalization + echo semantics remain stable.

## Implementation Notes
- Introduce `proptest` or `quickcheck` as optional dev-dependency; gate large runs behind `--ignored` or `cfg(test)` helpers.
- Reuse existing helper APIs (`FlowExecutor::instantiate`, `HostHandle::router`) to keep tests short and deterministic.
- Capture flaky cases: ensure timeouts via `tokio::time::timeout` and deterministic RNG seeds.

## Documentation
- Update `impl-docs/impl-plan.md` Phase 2 DoD (property tests checkbox).  
- Record invariants + counterexamples in `impl-docs/work-log.md` once implemented.

## Test Commands
- `CARGO_TARGET_DIR=.target cargo test -p kernel-exec -- --ignored` (long-running property tests).  
- `cargo test -p host-web-axum` (stream property tests).  
- `cargo test -p kernel-plan` (validator fuzz suite).
