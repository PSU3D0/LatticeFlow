Status: Draft
Purpose: epic
Owner: Runtime
Last reviewed: 2025-12-12

# Phase 3 Preview — Bridge/Host Refactor Plan

> Status (2024-XX-XX): baseline split completed — `host-inproc` crate exists, `host-web-axum` now consumes it, and `EnvironmentPlugin` hooks are available for platform-specific capability injection. Remaining notes capture follow-on work (bridge catalog, remote workers) to revisit during Phase 3.

*Draft – to be iterated alongside Phase 3 kickoff*

## 1. Motivation
- Today each “host” crate (e.g. `host-web-axum`) handles *both* ingress adaptation and workflow execution.  
- Phase 3 introduces additional transports (Redis queue, scheduled jobs, control-plane triggered runs). Re-embedding the executor and capability plumbing in every crate will be redundant and makes it harder to introduce a shared control plane later.  
- Several upcoming deployments (Workers, Lambda) mainly need an invocation adapter; execution semantics remain identical.

## 2. Current State Snapshot
- `FlowExecutor` already encapsulates scheduling/backpressure/metrics.  
- Hosts materialise `ValidatedIR` and call `run_once`, but no common abstraction exists above that.  
- Axum host exports router + make-service, but mixes HTTP bridge code with execution logic, metrics, and capability setup.  
- Phase 2 tests (`run_local`, SSE streaming) rely on the combined host.

## 3. Proposed Architecture
### 3.1 Core Concepts
- **Invocation Bridge** — component that converts an upstream event into `Invocation` (canonical payload + metadata). Responsible for ingress policy (auth, retries, dedupe).  
- **Compute Host Runtime (`host-inproc`)** — reusable crate that accepts an `Invocation`, loads/instantiates the flow, injects runtime capabilities, and drives execution. Emits telemetry and surfaces results/errors.  
- Bridges compose with the runtime; bridges may run in the same process (Axum) or forward to remote workers.

### 3.2 Key Traits & Types (updated sketch)

Note: In 0.1.x, **flow selection is out-of-band** (deployment/host config). Flow identity is carried for observability via metadata (e.g. `lf.flow_id`), not as an invocation field.

```rust
pub struct InvocationParts {
    pub trigger_alias: String,
    pub capture_alias: String,
    pub payload: serde_json::Value,
    pub deadline: Option<std::time::SystemTime>,
    pub metadata: InvocationMetadata,
}

pub trait InvocationBridge {
    type Error;
    async fn next_invocation(&mut self) -> Result<InvocationParts, BridgeOutcome<Self::Error>>;
}

pub struct HostRuntime {
    executor: FlowExecutor,
    registry: Arc<NodeRegistry>,
}

impl HostRuntime {
    pub async fn execute(&self, invocation: InvocationParts) -> RunResult;
}
```

### 3.3 Crate Layout Changes
- Crate `crates/host-inproc` (already implemented) exports the shared runtime utilities (execution/metrics pieces extracted from Axum host).  
- Existing host crates become thin bridges:
  - `host-web-axum` provides HTTP router → `Invocation` mapping, then calls `HostRuntime::execute`.  
  - `bridge-queue-redis` (Phase 3) reads queue messages, resolves acknowledgements, and feeds the runtime.  
  - Future bridges (Workers, Lambda) implement the same interface.
- Shared tests for the runtime (backpressure, cancellations) move into `host-inproc`, while bridge crates focus on ingress semantics.

### 3.4 Capability Injection Contract
- Each bridge assembles a `ResourceBag` that declares the concrete capability providers (HTTP clients, KV stores, blob backends, dedupe stores, etc.).
- `HostRuntime` owns that bag. When constructed via `with_resource_bag`/`with_resource_access`, it reconfigures the embedded `FlowExecutor` so every node run executes with the same capability handle.
- `FlowExecutor` exposes the resources task-locally via `capabilities::context::with_resources`. Node implementations can retrieve the bag inside async handlers using `capabilities::context::with_current_async(|resources| async { … })` without requiring additional plumbing in macros.
- Streaming captures are wrapped so each poll re-enters the same capability scope; SSE/websocket bridges therefore see consistent capability availability across incremental results.
- OpenDAL-backed capability crates rely on the shared `cap-opendal-core` helpers for operator construction and error mapping before registering the resulting provider inside the bag (e.g., `cap-blob-opendal` provides the default blob store, and `cap-kv-opendal` supplies the key-value capability with metadata about TTL/consistency guarantees).

## 4. Migration Plan
1. **Introduce host-inproc crate** *(completed)*
   - Extract execution helpers, metrics guards, and configuration from `host-web-axum`.  
   - Provide constructor taking a `ValidatedIR`, `FlowExecutor`, and expose a builder (`with_resource_bag`) that wires capability bags through to the executor + task-local context.  
   - Ensure existing Axum integration continues to pass all tests.
2. **Define Invocation structures**
   - Add canonical `Invocation` + metadata types in a shared module (`types-common` or within `host-inproc`).  
   - Adjust CLI `load_example` helpers to emit the same structure.
3. **Refactor Axum host into bridge** *(completed)*
   - Axum routes deserialize HTTP payloads, build `Invocation`, and delegate to `HostRuntime`.  
   - Streaming/SSE logic continues to live in Axum bridge but uses runtime-provided stream handle wrappers.  
   - Update integration tests to ensure behaviour is unchanged.
4. **Author Redis queue bridge (Phase 3 scope)**
   - Implement message polling, visibility timers, dedupe integration.  
   - Reuse host runtime for execution.  
   - Provide integration tests with in-memory Redis (or mock) verifying ack/retry semantics.
5. **Document bridge/host split**
   - Update RFC §6 and `impl-plan.md` with new terminology.  
   - Highlight that future control plane may forward invocations to remote workers that just embed `HostRuntime`.

## 5. Example Compositions & Capability Story
### 5.1 Host-inproc + Redis Bridge + Lambda-specific Capabilities
- **Redis bridge**: polls a stream/list, enforces dedupe/visibility, normalises payload into `Invocation { flow_id, trigger_alias, capture_alias, payload, metadata }`. Metadata includes Redis message IDs, retry counts, and any upstream sharding hints.
- **Host-inproc**: runs in a container/VM or Lambda runtime. The bridge either calls the host inline (same process) or forwards the invocation over RPC/Invoke API. `host-inproc` loads `ValidatedIR`, attaches capability providers (HTTP, KV, Blob) and executes via `FlowExecutor`.
- **Lambda plugin**: when executed inside Lambda, a lightweight capability plugin exposes Lambda context (request ID, deadlines, X-Ray trace IDs), SQS/DynamoDB clients, and policy hooks (e.g., `lambda::sleep`, `lambda::next_invocation_deadline`). The plugin registers capability traits with `host-inproc`, allowing nodes to opt-in without turning Lambda itself into a monolithic host crate.
- Result: Redis remains the ingress surface, Lambda provides compute (and extra capabilities), and both share the same canonical invocation contract. No duplicate executor logic is needed.

### 5.2 Additional Bridge/Host Combinations
- **Webhook bridge (Axum) → host-inproc on Kubernetes**: multiple HTTP bridges feed a shared pool of `host-inproc` pods. Pods expose additional capability providers (PostgreSQL, internal gRPC) via dependency injection.
- **Scheduler bridge → host-inproc worker pool**: cron service emits invocations into a control-plane queue; remote workers (each embedding `host-inproc`) pull jobs and execute flows with offline/batch oriented capabilities (e.g., Hadoop connectors).
- **Workers (Cloudflare) bridge → host-inproc (WASM)**: Workers runtime normalises requests, then hands off to a WASM-flavoured `host-inproc` build that exposes Workers KV, Durable Objects, and DO cron triggers via plugin traits.
- **Redis bridge → host-inproc (self-managed) + external plugin**: for on-prem deployments, the bridge forwards to a fleet of in-house worker binaries. A custom plugin adds observability hooks (Prometheus, OpenTelemetry) and internal service discovery.
- These compositions validate that the bridge/host split scales across deployment targets without recreating the executor surface each time.

## 6. Future-Proofing Notes
- When remote workers arrive, `HostRuntime` can be embedded in a standalone binary (gRPC / HTTP service) without rewriting execution code.  
- Invocation metadata should include fields for tenant/project IDs, trace IDs, and capability hints so bridges can enrich runs as needed.  
- Bridges may run in constrained environments (Workers, Lambda); ensure `HostRuntime` remains `Send + Sync` friendly and avoids blocking calls outside Tokio.  
- Consider feature flags for optional capability providers (e.g., bring-your-own cache) so bridges can opt-in without bloating binary size.

## 7. Open Questions (status)
- **Invocation type placement** — Resolved. `Invocation` and `InvocationMetadata` live in `host-inproc`; the abandoned `types-common` stub can be removed if it stays unused.
- **Idempotency metadata propagation** — Active but scoped. Bridges encode dedupe keys/TTLs into `InvocationMetadata` (labels: `lf.dedupe.key`, `lf.dedupe.ttl_ms`, extension `lf.dedupe`) and queue bridges translate that into capability calls. Additional policy validation belongs in Phase 3 queue work.
- **Bridge/runtime error contract** — Resolved for in-process hosts. `HostRuntime::execute` exposes `HostExecutionResult`/`HostExecutionError`; bridges surface failures as `BridgeError::ExecutionFailure` and decide whether to retry or surface terminal errors upstream.
- **Streaming response adaptation** — Resolved. `FlowExecutor` now emits a `StreamHandle` that remains capability-scoped; bridges map it to protocol-specific transports (SSE today, WebSocket/gRPC later).

---

Next steps: circulate this draft during Phase 3 planning, align on trait/type definitions, then schedule the extraction work ahead of `bridge-queue-redis`. Once agreed, update the RFC and implementation plan with the final architecture.
