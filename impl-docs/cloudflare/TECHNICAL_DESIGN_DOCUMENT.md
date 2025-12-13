Status: Draft
Purpose: notes
Owner: Runtime
Last reviewed: 2025-12-12

NOTE: Non-canonical design notes. Canonical epic docs:
- `impl-docs/roadmap/epic-02-workers-host-minimal.md`
- `impl-docs/roadmap/epic-03-cloudflare-caps-minimal.md`

# Cloudflare Workers Integration — Technical Design

## Purpose & Scope
- Deliver a first-class `host-workers` adapter that runs Flow IR workloads inside Cloudflare Workers using the `worker` crate from `ref-crates/workers-rs`.
- Provide capability crates for Workers-native services (KV, R2, Durable Objects, Queues) with effect/determinism hint registration so validators enforce policy (§6.3–6.4 of the RFC).
- Enable deployment packaging that targets Workers’ `wasm32-unknown-unknown` runtime while reusing the existing kernel executor semantics (Phase 2 foundation).

## Architecture Overview
```
Cloudflare Workers runtime (wasm32) ──> host-workers fetch handler
                                     │       │
                                     │       └─▶ Kernel executor (async runtime tuned for Workers)
                                     │               └─▶ Capability bag (KV, R2, DO, Neon via HTTP)
                                     └─▶ Platform bindings (Env, Durable Object stubs, Secrets)
```
- `host-workers` compiles to a single wasm module. It exposes `#[event(fetch)]` (or Router) entrypoints that deserialize HTTP requests into Flow trigger payloads (reuse Flow IR metadata).  
- Executor scheduling uses Workers’ async runtime (no Tokio). We rely on `worker::Env` for timers, async I/O, and secrets. Backpressure primitives reuse the kernel’s bounded queues but must honor Workers’ per-request compute limits.
- Streaming: SSE-style streaming is implemented via Workers’ `Response::from_stream`, providing chunked output for flows returning `ExecutionResult::Stream`.
- Capabilities live in dedicated crates:  
  - `cap-cloudflare-kv`: wraps `KvStore` binding, registers `kv_read` / `kv_write` hints.  
  - `cap-cloudflare-r2`: exposes blob operations with `blob_read` / `blob_write` hints.  
  - `cap-cloudflare-queue`: publish/consume queue messages; can act as trigger metadata.  
  - `cap-cloudflare-do`: Durable Object client handles stateful interactions with determinism downgrade to `BestEffort` when writes occur.
- Each provider-specific crate reuses the shared `cap-opendal-core` helpers to construct operators, attach optional layers, and translate OpenDAL errors into capability-friendly variants.
- Deployment packaging: CLI command emits Workers project scaffold (wrangler.toml, build script) with Flow IR artifacts embedded or fetched at runtime.

## Platform Considerations
- **Runtime constraints**: Workers limit CPU time per request and memory footprint. The host enforces watchdog timeouts derived from Flow IR `timeout!` metadata and gracefully fails before Workers terminates execution.
- **Persistence**: Durable Objects or KV provide state for HITL halts, dedupe, and sessions; capabilities expose consistent APIs so validators know when determinism drops.
- **Secrets & env**: Capabilities read bindings via `Env::secret`; validator ensures required secrets are listed in deployment manifest.
- **Cold starts**: Module initialization must be synchronous; we lazily load Flow IR manifests and capability registries on first invocation and cache in global statics guarded by Workers’ `Lazy`.

## Compatibility
- Aligns with `impl-docs/rust-workflow-tdd-rfc.md` §6.4 (Host adapters) and §6.5 (Scheduling/backpressure).  
- Works with the planned invocation gateway: the gateway can route requests to Workers deployments via durable service routes.  
- Coexists with Axum host—flows remain portable; only hosting surface changes.

## Future Extensions
- WebSocket support once Workers stabilizes duplex streaming.  
- Cloudflare AI/Vector capabilities as additional plugins.  
- Deployment automation via `flows publish workers` hooking into Wrangler API.  
- Multi-region routing using Workers’ bindings + automatic failover.
