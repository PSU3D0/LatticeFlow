Status: Archived
Purpose: notes
Owner: Core
Last reviewed: 2025-12-12

# Task Brief — Runtime Metrics & Observability

## Goal
Implement the “richer metrics” slice promised in Phase 2: expose executor, host, and CLI telemetry that meets RFC §14 requirements and primes Phase 3+ for policy enforcement and operational dashboards.

## Deliverables
1. **Metric Catalog**
   - Document metrics in-code (e.g., `metrics.md` under `impl-docs/` or module-level docs):  
     - Executor: active nodes, queue depth per edge, capture backpressure, cancellations, run duration histogram.  
     - Host (Axum): request latency histogram, in-flight requests, SSE clients, deadlines exceeded.  
     - CLI (local runner): per-node timing + success/failure counters (emitted via tracing).
   - Include labels (flow name, profile, node alias) and units.

2. **Instrumentation (`kernel-exec`, `host-web-axum`)**
   - Integrate with `tracing` + `metrics` (add workspace dependency if needed).  
   - Emit structured events and counters whenever:  
     - Nodes start/finish/error  
     - Channels accept/reject messages  
     - Deadlines/cancellations trigger  
     - SSE clients connect/disconnect.
   - Provide a pluggable recorder (feature-flag default no-op, allow `metrics-exporter-prometheus` in dev).

3. **CLI Surfaces**
   - `flows run local` prints a summary table (latency, success/failure counts, captured results count).  
   - Optional `--json` output bundling metrics for scripting.

4. **Documentation & Examples**
   - Update `impl-docs/rust-workflow-tdd-rfc.md` §14 with concrete metric names + sample payload.  
   - Note how these metrics will map to future Cloudflare Workers/Neon deployments (e.g., use `tracing` spans to integrate with workers-rs logs).

## Testing Checklist
- Unit tests using a test recorder to assert counters/histograms are incremented (wrap `metrics::Recorder`).  
- CLI integration test comparing summary output against known values for S1 run.  
- Ensure metrics path doesn’t panic when no recorder is installed (default no-op).

## Notes
- Keep the API surface minimal: return internal handles or guard objects for timing (e.g., RAII span for node execution).  
- Align naming conventions with OpenTelemetry best practices (`lattice.*` namespace).  
- Coordinate with future cache/dedupe metrics (Phase 3) to avoid naming conflicts.
