# Task Brief — SSE Streaming (Phase 2 Gap)

**Status: Completed.** Executor streams now surface as `ExecutionResult::Stream`, `host-web-axum` promotes captures to SSE, `flows run local --stream`/`flows run serve` cover S2, and docs/schema snapshots have been updated.

## Goal
Deliver the S2 “site status” scenario with full Server-Sent Events (SSE) support across executor, Axum host, and CLI, closing the streaming portion of the Phase 2 DoD.

## Deliverables
1. **Flow Example (`examples/s2_site`)**
   - Macro-defined workflow with streaming capture node (per RFC §4.9.3).
   - Provides both initial snapshot and incremental updates; ensures Flow IR marks the capture as streaming (`NodeSpec::kind`, `out_schema`).

2. **Executor Enhancements (`crates/kernel-exec`)**
   - Support for streaming outputs (`NodeOutput::Stream`) with bounded channel backpressure.
   - Proper cancellation propagation when a client disconnects.
   - Tests covering partial consumption, cancellation, and error propagation (Tokio-based).

3. **Axum Host (`crates/host-web-axum`)**
   - Route handler that detects streaming captures and upgrades the response to SSE (`Content-Type: text/event-stream`).
   - Heartbeat/keepalive support and graceful close when executor finishes.
   - Deadline + cancellation logic shared with HTTP JSON path.

4. **CLI Integration (`crates/cli`)**
   - `flows run local --example s2_site --stream` prints events in order.
   - `flows run serve --example s2_site` exposes `/site/stream` SSE endpoint.
   - Update CLI tests to cover the streaming code path (use `tokio::time::timeout` to guard hangs).

5. **Docs & Schema**
   - Refresh `impl-docs/rust-workflow-tdd-rfc.md` (SSE control surface, host behaviour).
   - Update `impl-docs/impl-plan.md` Phase 2 DoD checklist.
   - Include Flow IR snapshot for S2 under `schemas/examples`.

## Testing Checklist
- `cargo test -p kernel-exec` (streaming unit tests).
- `cargo test -p host-web-axum` (response writer tests).
- CLI integration tests for `run local` streaming and `serve` SSE (loopback with `reqwest::Client::get().send().text_stream()`).
- Manual smoke: `flows run serve --example s2_site` + `curl -N`.

## Notes
- Reuse capability hints: streaming connectors should emit determinism hints (e.g., `resource::sse`) once defined.
- Ensure fallbacks (non-stream capture) keep working—share routing code, avoid regressions.
