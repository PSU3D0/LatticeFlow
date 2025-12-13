Status: Draft
Purpose: epic
Owner: Runtime
Last reviewed: 2025-12-12

# Epic 03: Cloudflare Capabilities (Minimal Set)

Goal
- Provide the minimum Workers-native capability providers needed for the first vertical slice.

Value
- Lets workflows remain provider-agnostic while being deployable on Workers.

Scope
- HTTP capability provider backed by Workers fetch.
- Durable Object-backed state provider for sequencing/dedupe.
- Secrets/env binding conventions.

Non-goals
- Full KV/R2/Queue capability catalog (future expansion).

Dependencies
- Epic 01 contracts.
- Epic 02 Workers host.

Phases (ticket-sized)

03.1 Workers HTTP provider
- Implement a Workers-backed provider that satisfies:
  - `capabilities::http::HttpRead`
  - `capabilities::http::HttpWrite`

Acceptance gates
- A node can call HTTP capability in Workers without using `reqwest`.

03.2 Durable Object state / sequencer provider
- Implement a capability that provides:
  - per-key single-writer state machine operations
  - idempotency ledger primitives needed for sequencing

Acceptance gates
- Concurrency test: parallel events for same key are serialized.

03.3 Secrets + env binding contract
- Define how secrets are referenced:
  - names only in configs
  - never embed secrets in Flow IR or artifacts

Acceptance gates
- Preflight detects missing bindings and fails before executing effectful work.

Risks
- "Sequencing" is correctness-critical. Prefer a single-writer primitive (Durable Objects) over eventual KV.

References
- Capability hint registry: `crates/capabilities/src/lib.rs`, `crates/capabilities/src/hints.rs`
- Cloudflare plans: `impl-docs/cloudflare/IMPL_PLAN.md`
