Status: Draft
Purpose: spec
Owner: Core
Last reviewed: 2025-12-12

# LatticeFlow RFC (Umbrella)

This file is the umbrella RFC for LatticeFlow.

It is intentionally short. Canonical semantics live in `impl-docs/spec/` and irreversible decisions live in `impl-docs/adrs/`.

Historical longform RFC (pre-consolidation):
- `impl-docs/archive/rust-workflow-tdd-rfc-longform-2025-09-18.md`

## What Is LatticeFlow

LatticeFlow is a Rust-first workflow platform where:
- workflows are represented as a typed-ish Flow IR DAG
- nodes declare effects/determinism/idempotency
- capabilities are injected at the host boundary (provider-agnostic node code)
- validation emits stable diagnostic codes

The intended differentiator is agent-native automation: machines can import/repair/optimize flows using stable IR + diagnostics + evidence.

## Canonical 0.1 Contract (Specs)

These docs define the 0.1.x contract. They are the target for tooling stability.

- Flow IR semantics + compatibility rules: `impl-docs/spec/flow-ir.md`
- Invocation ABI (host boundary): `impl-docs/spec/invocation-abi.md`
- Control surfaces (routing/orchestration metadata): `impl-docs/spec/control-surfaces.md`
- Capabilities + binding model (opaque ResourceBag in 0.1): `impl-docs/spec/capabilities-and-binding.md`

Schema source of truth:
- `schemas/flow_ir.schema.json`

## ADRs (Forever Decisions)

- Invocation ABI and metadata keys: `impl-docs/adrs/adr-0001-invocation-abi-and-metadata.md`
- Control surfaces 0.1 scope (runtime vs lint): `impl-docs/adrs/adr-0002-control-surfaces-0-1-scope.md`
- Workers-native constraints + spill strategy: `impl-docs/adrs/adr-0003-workers-native-constraints-and-spill.md`
- Retry ownership model: `impl-docs/adrs/adr-0004-retry-ownership.md`
- Determinism claims policy: `impl-docs/adrs/adr-0005-determinism-claims-policy.md`

## Roadmap (Epics + Phases)

- Epic index: `impl-docs/roadmap/epics.md`

Near-term target: Workers-native deployment + one real vertical slice (Resend sequencing).

## Scenarios (Acceptance Targets)

- Primary scenario pack: `impl-docs/user-stories.md`
- Cloudflare-specific scenarios: `impl-docs/cloudflare/USER_STORIES.md`
- Archived MCP notes (not on current roadmap): `impl-docs/archive/mcp/USER_STORIES.md`

## Current Implementation Snapshot (Reality Check)

Implemented (non-trivial):
- Flow IR core types and diagnostics: `crates/dag-core`
- Macro authoring surface (nodes + workflow skeleton): `crates/dag-macros`
- Validator (effects/determinism/idempotency + delivery rules): `crates/kernel-plan`
- In-proc executor with streaming and spill hooks: `crates/kernel-exec`
- In-proc host runtime boundary: `crates/host-inproc`
- Axum host (HTTP + SSE): `crates/host-web-axum`
- Redis queue bridge (partial Phase 3): `crates/bridge-queue-redis`
- Capability hint registry + core traits: `crates/capabilities`
- OpenDAL-backed KV/blob providers: `crates/cap-opendal-core`, `crates/cap-kv-opendal`, `crates/cap-blob-opendal`
- Examples: `examples/s1_echo`, `examples/s2_site`

Planned / stubbed:
- n8n importer: `crates/importer-n8n`
- connector spec + std connectors: `crates/connector-spec`, `crates/connectors-std`
- registry certification + policy engine: `crates/registry-cert`, `crates/policy-engine`
- Workers host + Workers-native caps: planned under `impl-docs/cloudflare/`

## Metrics + Diagnostics

- Metrics catalog: `impl-docs/metrics.md`
- Error taxonomy: `impl-docs/error-taxonomy.md`
- Error code registry: `impl-docs/error-codes.md`
