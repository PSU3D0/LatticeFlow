Status: Canonical
Purpose: spec
Owner: Core
Last reviewed: 2025-12-12

# Runtime Metrics Catalog

Use this catalog whenever we add instrumentation to executors, hosts, or tooling. All metric names live under the `lattice.*` namespace and follow OpenTelemetry semantic conventions where possible.

## Executor (`lattice.executor.*`)
- `active_nodes` (gauge, labels: `flow`, `profile`): current number of in-flight node tasks.
- `queue_depth` (gauge, labels: `flow`, `edge`): buffered messages per edge channel.
- `node_latency_ms` (histogram, labels: `flow`, `node`, `profile`): wall-clock duration for a node invocation.
- `node_errors_total` (counter, labels: `flow`, `node`, `error_kind`): terminal errors surfaced by the node.
- `cancellations_total` (counter, labels: `flow`, `node`, `reason`): cancellation or deadline triggers.
- `capture_backpressure_ms` (histogram, labels: `flow`, `capture`): time producers spent blocked waiting for downstream consumers.
- `stream_clients_total` (counter, labels: `flow`, `node`): number of streaming clients spawned by node captures.

## Host (Axum) (`lattice.host.*`)
- `http_requests_total` (counter, labels: `host`, `flow`, `route`, `status_class`).
- `http_request_latency_ms` (histogram, labels: `host`, `flow`, `route`).
- `http_inflight_requests` (gauge, labels: `host`, `flow`, `route`).
- `sse_clients` (gauge, labels: `host`, `flow`, `route`): currently connected SSE clients.
- `deadline_exceeded_total` (counter, labels: `host`, `flow`, `route`).

## CLI (`lattice.cli.*`)
- `run_duration_ms` (histogram, labels: `flow`, `example`).
- `nodes_succeeded_total` / `nodes_failed_total` (counter, labels: `flow`, `node`).
- `captures_emitted_total` (counter, labels: `flow`, `node`, `capture`).
- JSON summary output mirrors these aggregates for scripting.

## Queue Bridge (`lattice.queue.*`)
- `enqueued_total` / `dequeued_total` (counter, labels: `bridge`, `queue`).
- `success_total` / `failure_total` (counter, labels: `bridge`, `queue`).
- `requeued_total` / `duplicates_total` (counter, labels: `bridge`, `queue`).
- `lease_extensions_total` / `lease_expired_total` (counter, labels: `bridge`, `queue`).
- `processing_inflight` (gauge, labels: `bridge`, `queue`): number of messages currently executing.

## Label Guidelines
- `flow`: stable flow identifier (snake_case), never raw UUIDs; hosts attach this alongside `host` and `route`.
- `node`: alias emitted in Flow IR; avoid dynamic suffixes.
- `profile`: runtime profile (e.g., `dev`, `prod`, `sandbox`).
- `route`: HTTP route template (`/flows/:id/run`).
- `error_kind` / `reason`: values from error taxonomy (`Transient`, `Permanent`, `PolicyDenied`, …).
- Captured exemplars may include a hashed `run_id` for debugging; retain ≤1 exemplar per 5 s window.

## Recorders
- Default build installs a no-op recorder.
- Tests and dev tooling can install a recorder such as `metrics-exporter-prometheus` when needed.
- Hosts and executors take optional recorder handles; when unset, instrumentation remains cheap (atomic increments only).
