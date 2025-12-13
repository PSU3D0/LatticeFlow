Status: Draft
Purpose: epic
Owner: Runtime
Last reviewed: 2025-12-12

# Epic 02: Workers-Native Host (Minimal)

Goal
- Execute validated Flow IR inside Cloudflare Workers (wasm32-unknown-unknown) for HTTP triggers.

Value
- Validates the portability thesis early.
- Becomes the substrate for the first real vertical slice (Epic 04).

Scope
- `host-workers` adapter.
- Minimal build profile changes needed for wasm.

Non-goals
- No Cloudflare capability providers beyond what is needed for the host itself (Epic 03).
- No queue consumers / cron triggers yet.

Dependencies
- Epic 01 contracts.
- ADR: Workers constraints + spill strategy: `impl-docs/adrs/adr-0003-workers-native-constraints-and-spill.md`
- Existing Cloudflare notes (to be decomposed into phase tickets):
  - `impl-docs/cloudflare/TECHNICAL_DESIGN_DOCUMENT.md`
  - `impl-docs/cloudflare/IMPL_PLAN.md`

Phases (ticket-sized)

02.1 Build profile: wasm-compatible kernel/host
- Make `kernel-exec` build on `wasm32-unknown-unknown` for a constrained profile.
- Feature-gate OS-only behaviors (notably filesystem-backed spill via `tempfile`).

Acceptance gates
- `cargo build --workspace --target wasm32-unknown-unknown` succeeds for the minimal Workers set.

02.2 Implement `host-workers`
- Implement a fetch entrypoint that:
  - maps HTTP request body -> JSON payload
  - constructs `Invocation`
  - executes via `HostRuntime`
  - maps `ExecutionResult` to Workers `Response`

Acceptance gates
- Echo flow runs under Workers dev harness.

02.3 Streaming + cancellation (SSE)
- Map `ExecutionResult::Stream` to Workers streaming response.
- Cancel execution when the client disconnects / request aborts.

Acceptance gates
- Streaming example emits multiple chunks.
- Cancellation is observable (execution stops, permits released).

02.4 Error envelope consistency
- Ensure Workers host error responses align with Axum host envelope rules.

Acceptance gates
- Golden tests for error bodies (shape and key names).

Risks
- wasm builds tend to fail due to transitive dependencies (fs, sockets, thread runtime).
- Keep the first pass constrained; add capabilities in Epic 03.

References
- Workers integration plan: `impl-docs/cloudflare/IMPL_PLAN.md`
- Workers design notes: `impl-docs/cloudflare/TECHNICAL_DESIGN_DOCUMENT.md`
