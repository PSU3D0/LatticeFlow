# Cloudflare Workers Integration — User Stories

## 1. Edge Workflow Publisher
- **As** a platform engineer deploying global workflows  
- **I want** to compile my Flow IR into a Workers deployment  
- **So that** I can run low-latency HTTP/SSE flows close to users without managing infrastructure.  
- **Acceptance**  
  - `flows package workers --example s1_echo` generates a runnable project (wrangler.toml, wasm bundle).  
  - Deploying with `wrangler deploy` yields working `/echo` routes and streaming endpoints.  
  - Capability bindings (KV, R2) are validated before deploy.

## 2. Capability Author (Cloudflare Services)
- **As** a developer integrating Workers KV/R2/Queues  
- **I want** first-class capability crates that expose these services with correct effect/determinism hints  
- **So that** validators enforce policies and Flow IR stays portable across environments.  
- **Acceptance**  
  - Adding `cap-cloudflare-kv` lets nodes call `kv.get/put`; mismatched policies trigger `EFFECT201`/`DET302`.  
  - Queue trigger metadata maps to Workers queue consumers, with CLI scaffolding generating binding declarations.

## 3. Durable Session Owner
- **As** an engineer building conversational flows  
- **I want** Durable Objects surfaced as a capability/facet  
- **So that** I can persist session state per user and resume work after halts or HITL steps.  
- **Acceptance**  
  - Workflows access a `DurableObjectContext` facet with session id; writes downgrade determinism automatically.  
  - Halt/HITL nodes can persist state in DO and resume after operator approval.

## 4. Observability & Operations Lead
- **As** an SRE monitoring Workers flows  
- **I want** metrics and logs emitted via Workers bindings  
- **So that** I can trace latency, failures, and retries across regions.  
- **Acceptance**  
  - Host emits structured logs to Workers’ log stream; metrics integrate with Workers Analytics or push via HTTP to the control plane.  
  - Failures reference Flow IDs, capability hints, and request ids for debugging.

## 5. Hybrid Deployment Architect
- **As** an architect mixing Workers with Neon-backed components  
- **I want** the ability to invoke Neon (via HTTP or direct TCP tunnel) from Workers flows  
- **So that** data lives in Neon while computation runs at the edge.  
- **Acceptance**  
  - `cap-postgres-neon` works from Workers via HTTP-compatible client, registering `db_read`/`db_write` hints.  
  - Validator prevents deployment when required secrets (connection tokens) are absent.  
  - Flow IR remains identical whether run on Workers or Axum; only host profile changes.
