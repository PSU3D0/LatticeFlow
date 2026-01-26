Status: Draft
Owner: Runtime
Epic: 02 (Workers host minimal)
Phase: 02.2

# Ticket: host-workers adapter (Cloudflare Workers)

## Goal
Implement a Workers-native host adapter that maps HTTP requests to Flow IR execution using the in-proc runtime, producing responses compatible with the Axum host error envelope and streaming semantics.

## Scope
- Add `host-workers` crate with `#[event(fetch)]` entrypoint.
- Map Workers `Request` → `Invocation` (trigger alias + payload + metadata).
- Execute via `HostRuntime` + `kernel-exec` and return `Response`.
- Support streaming output and cancellation.

## Non-goals
- No Cloudflare capability providers (Epic 03).
- No queue or cron triggers yet.

## Requirements
1) **Wasm build**
   - `host-workers` must compile under `wasm32-unknown-unknown`.
   - Depends on the wasm-ready kernel-exec profile (Epic 02.1).

2) **Invocation + metadata**
   - Use the standard invocation ABI (payload, trigger alias, capture alias, deadline).
   - Populate `lf.*` metadata and HTTP metadata as available.

3) **Streaming + cancellation**
   - Map `ExecutionResult::Stream` to Workers streaming response.
   - Cancellation on client disconnect must propagate to the runtime.

4) **Error envelope parity**
   - Errors must match the Axum host envelope shape (code + details when present).

## Implementation Plan
1) **Create `host-workers` crate**
   - Fetch handler uses `worker` crate API.
   - Deserialize body to JSON and map to `Invocation`.

2) **Runtime integration**
   - Create `HostRuntime` from `FlowBundle` (or direct IR/registry if FlowBundle not yet merged).
   - Execute and map results.

3) **Streaming**
   - For streaming results, use `Response::from_stream` with SSE-compatible framing.

4) **Error mapping**
   - Mirror Axum host’s JSON error shape and HTTP status mapping.

## Acceptance Gates (Definition of Done)
- `cargo check -p host-workers --target wasm32-unknown-unknown` succeeds.
- S1 echo flow runs in Workers dev harness.
- Streaming example emits multiple SSE chunks and cancels cleanly.
- Error responses match Axum envelope shape in golden tests.

## Files to Touch (expected)
- `crates/host-workers/*` (new)
- `Cargo.toml` workspace members
- `impl-docs/cloudflare` notes (optional update)

## Notes
- Align with `impl-docs/roadmap/epic-02-workers-host-minimal.md` phases.
- Prefer minimal dependencies to keep wasm size small.
