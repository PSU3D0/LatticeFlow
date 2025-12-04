# Cloudflare Workers Integration — Implementation Plan

## Phase 0 — Discovery
- Audit `ref-crates/workers-rs` (`worker`, `worker-macros`, `worker-kv`, `worker-sys`) to confirm supported APIs, streaming patterns (`Response::from_stream`), and Durable Object interfaces.
- Inventory existing Flow IR metadata to ensure HTTP trigger information (path, method, streaming flag) is sufficient for Workers routing; document any schema additions.

## Phase 1 — Core Host Adapter
1. **Runtime Glue**
   - Implement `host-workers/src/lib.rs` with `#[event(fetch)] async fn fetch(req: HttpRequest, env: Env, ctx: Context)` entrypoint.
   - Wrap Flow executor in a Workers-friendly runtime (`worker::env::run::<_, Result<Response>>()`), providing bridges for request/response conversion.
   - Provide helper `HostHandle::invoke(&self, body: Vec<u8>, headers: &Headers)` returning `ExecutionResult`.
   - Tests: unit tests using `worker::Router` + `worker::Env::default()` (via `worker::match_req!` macro) to exercise 200/streaming responses.
2. **Streaming & Cancellation**
   - Map `ExecutionResult::Stream` to `Response::from_stream` with proper SSE formatting; propagate client disconnect via Workers’ `abort` signal to kernel cancellation token.
   - Tests: integration-level test verifying multiple chunks emitted and cancellation triggered.
3. **Error Handling**
   - Standardize JSON error envelope (align with Axum host) and map Workers exceptions to Flow diagnostics.

## Phase 2 — Capability Crates
1. `cap-cloudflare-kv`
   - Wrap `KvStore::get`, `put`, `delete`; register effect/determinism hints (`kv_read`, `kv_write`).
   - Add integration tests using Workers’ test harness with `MockKv`.
2. `cap-cloudflare-r2`
   - Provide streaming upload/download, multipart support; hints (`blob_read`, `blob_write`).
   - Tests: mock R2 binding verifying metadata translation and determinism downgrade when writes occur.
3. `cap-cloudflare-queue`
   - Expose enqueue API + optional trigger metadata (`queue_name`, `visibility_timeout`).
   - Tests: ensure validator catches missing capability binding when triggers declare queue usage.
4. `cap-cloudflare-do`
   - Manage Durable Object stub interactions for stateful nodes; hints degrade determinism to `BestEffort` on writes.
   - Tests: simulate DO stub with recorded responses.

## Phase 3 — Deployment Tooling
- Extend CLI with `flows package workers` producing:
  - `wrangler.toml` (bindings, routes), `.cargo/config` targeting `wasm32-unknown-unknown`, and entrypoint scaffolding.
  - Optional `wrangler deploy` integration (behind flag).
- Tests: golden fixture verifying generated project structure; CLI smoke test building S1 flow into Workers bundle.

## Phase 4 — Examples & Docs
- Create `examples/s3_workers_echo` demonstrating HTTP + streaming triggers on Workers.
- Document environment bindings (KV, R2, DO) and required Wrangler configuration in README + `impl-docs/cloudflare`.
- Update `impl-docs/work-log.md` and RFC host section with Workers references.

## Phase 5 — Advanced Features (Future)
- Queue-triggered flows using Workers’ queue consumers (alpha).
- Scheduled flows via Workers Cron Triggers (map Flow metadata to wrangler schedule arrays).
- Multi-region metrics + logging integration (Workers Analytics Engine).

## Risks & Mitigations
- **Runtime limits**: enforce Flow timeouts shorter than Workers’ max to avoid abrupt termination.
- **Binary size**: ensure host + kernel compile within Workers’ bundle limits; strip unused features.
- **Streaming support**: SSE currently supported; WebSocket support deferred until Workers provides stable API.
