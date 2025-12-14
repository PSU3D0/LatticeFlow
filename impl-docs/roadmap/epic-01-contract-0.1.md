Status: Draft
Purpose: epic
Owner: Core
Last reviewed: 2025-12-12

# Epic 01: 0.1 Stable Surface (Contracts)

Goal
- Define the 0.1.x "stable surface" so hosts, importers, agents, and UI can rely on:
  - Flow IR semantics and compatibility
  - Invocation ABI and result/error envelopes
  - Diagnostics contract (stable codes + stable host error envelope + `lf.*` metadata conventions)
  - A minimal but semantics-bearing macro/control-surface set
  - Trigger/entrypoint wiring rules

Value
- Makes later epics parallelizable: capabilities/connectors/importers can iterate without destabilizing the execution boundary.

Scope
- Contracts and semantics.
- Minimal implementation changes required to make the contracts real.

Non-goals
- No Cloudflare-specific providers (Epic 03).
- No Resend vertical slice (Epic 04).

Dependencies
- Specs:
  - `impl-docs/spec/flow-ir.md`
  - `impl-docs/spec/invocation-abi.md`
  - `impl-docs/spec/control-surfaces.md`
  - `impl-docs/spec/capabilities-and-binding.md`
- ADRs:
  - `impl-docs/adrs/adr-0001-invocation-abi-and-metadata.md`
  - `impl-docs/adrs/adr-0002-control-surfaces-0-1-scope.md`

Phases (ticket-sized)

01.1 Invocation ABI: freeze the host boundary
- Freeze semantics of `host-inproc::Invocation` and `InvocationMetadata`.
- Standardize reserved metadata keys (`lf.*`) and minimum host population.
- Tighten result/error envelopes:
  - `ExecutionResult::Value` and `ExecutionResult::Stream`
  - deterministic JSON error envelope shape for hosts

Acceptance gates
- Unit tests verifying invocation part roundtrip (bridge safety).
- Host conformance (reference: Axum): stable JSON error envelopes for all `ExecutionError` variants.
- Host conformance: reserved surfaces rejected deterministically with `CTRL901` (no silent ignore).

01.2 Capability preflight (domain-level, 0.1)
- Implement a host-level preflight that derives required capability domains from Flow IR hints.
- Fail fast before serving traffic (or before executing, depending on host).

Acceptance gates
- A flow that requires a domain (e.g., dedupe/kv/http_write) fails preflight when `ResourceBag` lacks it.
- Host surfaces stable error code `CAP101` for missing capability bindings.

Learnings / notes (01.2)
- Derive required capability domains from Flow IR `effectHints[]` only (avoid `determinismHints[]` pulling clock/RNG-like requirements).
- Treat unknown `resource::*` effect hints as missing (fail-fast) so new domains cannot silently ship without bindings.
- Runtime-level preflight inside `HostRuntime::execute` is simplest for correctness; hosts MAY also preflight at startup to fail before serving traffic.

01.3 Macro primitives v0 (semantics-bearing subset)
- Add macro surface for edge-native controls:
  - `delivery!` (writes `EdgeIR.delivery`)
  - `buffer!` / `spill!` (writes `EdgeIR.buffer`)
  - `timeout!` (writes `EdgeIR.timeout_ms`)
- Add macro surface for routing controls:
  - `if!` / `switch!` emit `FlowIR.control_surfaces`
  - `kernel-exec` respects routing (only schedule selected branch)

Acceptance gates
- trybuild tests for macro syntax (`timeout!` emits `EdgeIR.timeout_ms`).
- `kernel-plan` validates surface structure (e.g., `CTRL101` for invalid timeout budgets).
- `kernel-exec` either executes `if/switch` correctly or fails deterministically (no silent ignore).

Learnings / notes (01.3)
- `timeout!` is edge-scoped and must reference an existing `connect!` edge; do not allow implicit edge creation.
- Validation should reject nonsensical values (`timeout_ms = 0`) even if runtimes don’t enforce timeouts yet.

01.4 Reserved surfaces (spec now, runtime later)
- Define stable config shapes for:
  - `on_error!`, `rate_limit!`, `partition_by!`, `for_each!`, `window!`, `await!`
- In 0.1:
  - allow structural validation
  - runtime may reject deterministically if unimplemented (see `CTRL901`)

Acceptance gates
- Spec is complete and referenced by macro docs.
- Runtime rejection for unsupported reserved surfaces is deterministic (`CTRL901`).

01.5 Triggers and entrypoints (wiring rules)
- Keep trigger nodes in Flow IR (`NodeKind::Trigger`).
- Define deployment-side entrypoint config rules:
  - trigger alias, capture alias, deadline, route/event binding
- Optionally package an entrypoints manifest as a `FlowIR.artifacts[]` sidecar (no IR schema changes required).

Acceptance gates
- Host refuses to serve if trigger/capture alias is missing.
- CLI can validate "flow + entrypoints config" bundle.

Risks
- Control surfaces are easy to emit but easy to ignore; runtime MUST not silently ignore them.
- Over-scoping "facets" too early can freeze the wrong abstraction; keep 0.1 requirements domain-level.

References
- Flow IR types: `crates/dag-core/src/ir.rs`
- Invocation boundary: `crates/host-inproc/src/lib.rs`
- Axum mapping example: `crates/host-web-axum/src/lib.rs`
