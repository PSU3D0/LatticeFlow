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
- Keep resolver and binder separate: resolver produces `bindings.lock.json` deterministically from Flow IR + catalog; binder constructs runtime providers and resolves secrets.
- Multi-tenant isolation can be modeled either as isolation wrappers or as composite provider kinds ("zero trust" providers). If per-flow config is needed, prefer derived instances in the lockfile in 0.1.

01.3 Macro primitives v0 (semantics-bearing subset)
- Add macro surface for edge-native controls:
  - `delivery!` (writes `EdgeIR.delivery`)
  - `buffer!` / `spill!` (writes `EdgeIR.buffer`)
  - `timeout!` (writes `EdgeIR.timeout_ms`)
- Add macro surface for routing controls:
  - `if_!` / `switch!` emit `FlowIR.control_surfaces`
  - `kernel-exec` respects routing (only schedule selected branch)

Acceptance gates
- trybuild tests for macro syntax (`timeout!`/`delivery!`/`buffer!`/`spill!` emit `EdgeIR` controls; `switch!` emits `FlowIR.control_surfaces`).
- `kernel-plan` validates surface structure (e.g., `CTRL101`/`CTRL102` for invalid edge budgets; `CTRL110`/`CTRL111`/`CTRL112` for switch surface shape/edges; `CTRL120`/`CTRL121`/`CTRL122` for if surface shape/edges).
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

Learnings / notes (01.4)
- `Partition` is metadata-only in 0.1 (canonical is `EdgeIR.partition_key`); runtime must not reject or change behavior.
- `ForEach` remains reserved; structural validation should require an explicit `source -> body_entry` edge so delivery/buffer/timeout semantics stay attached to edges.
- `await` is checkpoint-based in 0.1; do not add a new `ControlSurfaceKind` until runtime semantics are ready.

Acceptance gates
- Spec is complete and referenced by macro docs.
- Runtime rejection for unsupported reserved surfaces is deterministic (`CTRL901`).

01.5 Triggers and entrypoints (wiring rules)
- Keep trigger nodes in Flow IR (`NodeKind::Trigger`).
- Define deployment-side entrypoint config rules:
  - trigger alias, capture alias, deadline, route/event binding
- Enforce agent-safety default: single trigger node per flow unless `policies.lint.allow_multiple_triggers=true`.
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
