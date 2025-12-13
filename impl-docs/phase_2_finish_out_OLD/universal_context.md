Status: Archived
Purpose: notes
Owner: Core
Last reviewed: 2025-12-12

# Current Task — Universal Context

Use this brief to get oriented before tackling any follow-on work.

## Baseline State (2024-XX-XX)
- **Workspace:** Fully scaffolded Rust workspace (Rust 1.90) with core crates (`dag-core`, `dag-macros`, `kernel-plan`, `kernel-exec`), Axum host, CLI, and streaming example `examples/s2_site` in addition to S1.
- **Runtime:** Phase 2 executor slice is live (bounded channels, cancellation, capture backpressure) and now emits `ExecutionResult::Stream` handles for streaming captures. Axum host promotes capture streams to SSE responses.
- **CLI:** `flows run local --example s1_echo`, `flows run local --example s2_site --stream`, and `flows run serve --example s2_site` all succeed; integration tests cover both JSON round-trips and SSE streaming.
- **Diagnostics:** Macro-generated `NodeSpec` carries effect/determinism hints via the shared capability registry; validator tests assert mismatches for HTTP/DB/clock hints.
- **Docs:** `impl-plan.md` and `work-log.md` updated with SSE completion; remaining Phase 2 gaps are capability registry expansion, runtime metrics, and property tests.

## Key References
- Technical design: `impl-docs/rust-workflow-tdd-rfc.md` (§4 macros, §5 Flow IR, §6 runtime, §14 observability).
- Implementation plan & DoD: `impl-docs/impl-plan.md` (Phase 2 table).
- Error catalog: `impl-docs/error-taxonomy.md`, `impl-docs/error-codes.md`.
- Scenario requirements: `impl-docs/user-stories.md` (S1 echo, S2 streaming site, etc.).
- Workspace surface: `impl-docs/surface-and-buildout.md`.

## Tooling & Commands
- Format / lint: `cargo fmt`, `cargo clippy --all-targets --all-features`.
- Runtime tests: `CARGO_TARGET_DIR=.target cargo test -p kernel-exec`, `-p host-web-axum`, `-p flows-cli`.
- UI diagnostics: `cargo test -p dag-macros --test trybuild`.

## Current Focus
The remaining Phase 2 items are tracked in companion task briefs within `impl-docs/current_task/`:
1. SSE streaming path (`task_sse_streaming.md`) — completed, kept for context.
2. Capability registry expansion (`task_capability_expansion.md`)
3. Metrics instrumentation (`task_metrics_instrumentation.md`)
4. Property-based / behavioural tests (`task_property_tests.md`)

Review this context *and* the specific task brief before editing code. Each brief lists deliverables, tests, and doc touchpoints to keep the overall plan coherent.
