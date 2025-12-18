Status: Draft
Purpose: agent handoff
Owner: Core
Last reviewed: 2025-12-18

# Recontextualization: Real Bindings Resolver (Catalog -> Lock)

This doc is a handoff for a fresh agent/human to resume work on the **real implementation** of bindings resolution.
It describes current state, the target architecture, and the open decisions.

## Current State (Implemented)

### What exists in code

- CLI consumes a machine-generated lockfile:
  - `flows run local --bindings-lock <path>`
  - `flows run serve --bindings-lock <path>`
  - Mutually exclusive with `--bind`.
  - File: `crates/cli/src/main.rs`.

- Lock validation invariants (enforced):
  - `version == 1`
  - `generated_at` required
  - `content_hash` required and verified
    - sha256 of canonical JSON excluding `content_hash` (sorted object keys, preserved array order)
  - `connect` and `config` must be objects
  - `isolation` wrappers are rejected for now (not implemented in binder)

- Minimal provider allowlist in the lock consumer:
  - `kv.memory` -> `capabilities::kv::MemoryKv`
  - `http.reqwest` -> `cap_http_reqwest::ReqwestHttpClient`

- CLI tooling to generate a valid lockfile for built-in examples:
  - `flows bindings lock generate --example <name> --out <path> [--bind ...]`
  - Generates `instances` + `flows[flow_id].use` and computes `content_hash`.

- Tests:
  - Unit tests around hashing/validation in `crates/cli/src/main.rs`.
  - CLI integration tests in `crates/cli/tests/bindings_lock.rs`.

### Example lockfile in-repo

- `examples/s4_preflight/bindings.lock.json`
- Regeneration script: `examples/s4_preflight/regen_bindings_lock.sh`

## Why this is not a “real resolver” yet

- There is no `resources.catalog.json` ingestion.
- There is no selection logic across instances (no candidate filtering / tie-breaking).
- There is no provider/wrapper registry with schemas.
- `connect/config` are structurally validated as objects, but not consumed.
- Isolation wrappers are not supported.

## Target Architecture (Recommended)

### Boundary definitions

- Resolver boundary:
  - Inputs: Flow requirements (from Flow IR `effectHints[]`), `resources.catalog.json`, policy/env selectors.
  - Output: `bindings.lock.json`.
  - Properties: deterministic, offline, no secret fetching.

- Binder boundary (host/CLI runtime):
  - Inputs: `bindings.lock.json`, environment (for secret resolution).
  - Output: constructed runtime providers in a `ResourceBag`.

Terraform/pulumi/etc integration belongs outside the resolver, as a **catalog producer**:
- Terraform exports module outputs/state to generate `resources.catalog.json`.
- Resolver remains deterministic and independent of Terraform internals.

### Isolation and multi-tenant semantics

Isolation semantics (namespacing, encryption, tenancy constraints) can be modeled in two ways:

1) Wrapper composition (portable)
- Provider kind is “plain” (e.g. `kv.redis`).
- A set of ordered wrappers are applied by the binder (e.g. prefix keys, encrypt values).

2) Composite provider kinds (enforced)
- Provider kind itself is “zero trust” (e.g. `kv.zero_trust_upstash`).
- Provider requires config like namespace strategy and encryption key refs and enforces them.

Both are valid. Wrappers maximize composability across backends; composite providers can enforce invariants by construction.

### Shared resources across multiple flows

One physical resource instance (e.g. one Postgres cluster, one Upstash account) can serve many flows.
`bindings.lock.json` supports this naturally: many flows can point at the same `instances.<name>`.

If a deployment needs per-flow scoping for a shared physical service, prefer (for 0.1) having the resolver synthesize per-flow derived instances in the lockfile:
- Example: `kv_shared` plus `kv_flow_<flow_id>` instances, where derived instances reference the same upstream but differ in scoping config.
- This avoids changing the lock schema.

Future evolution: allow per-use isolation config (e.g. `flows[flow_id].use[hint] = { instance, isolation }`).

## Open Decisions (Must resolve before implementing resolver)

1) Isolation representation in 0.1
- Option A (recommended for 0.1): derived instances in the lockfile, keep `flows.<flow_id>.use[hint] = "instance"`.
- Option B: change lock schema to allow per-use isolation/wrapper config.

2) Registry shape
- Do we want a single “provider registry” that also defines wrappers?
- Or separate registries: provider kinds and wrapper kinds?

3) Secret reference format
- Decide on a stable URI-like format for secret refs in `connect` (e.g. `env://NAME`, `aws-secretsmanager://...`, `gcp-sm://...`).
- Resolver treats these as opaque strings; binders/hosts interpret.

## Implementation Roadmap (Concrete Next Steps)

1) Introduce a dedicated bindings crate
- Move lock/catalog types + canonical hash into a new crate (suggested: `crates/bindings-spec`).
- Keep it dependency-light (serde/serde_json/sha2).

2) Add `resources.catalog.json` parsing + validation
- Validate `provides` entries start with `resource::`.
- Validate provider_kind is known (registry) and that instance `provides` is compatible.
- (Later) validate `connect/config/isolation` against declared schemas.

3) Implement the resolver command
- CLI command proposal:
  - `flows bindings resolve --flow <path> --catalog <path> --out <path> [--env <name>]`
- Deterministic selection:
  - stable tie-break rules (sorted names) and explicit precedence.
- Output a lockfile with computed `content_hash`.

4) Extend binders to consume more provider kinds
- Start with "real-ish" local providers (e.g. sqlite/redis/opendal) if already present.
- Keep secrets out; accept refs.

5) Add isolation wrapper support
- Implement a minimal wrapper or two (KV prefix + encrypt-values stub) as proof of architecture.
- Decide whether wrappers are applied in the binder or inside composite providers.

## Key References

- Specs:
  - `impl-docs/spec/capabilities-and-binding.md`
  - `impl-docs/spec/resource-catalog.md`

- Roadmap learnings:
  - `impl-docs/roadmap/epic-01-contract-0.1.md` (01.2)

- FlowId stability:
  - `crates/dag-core/src/ir.rs` (`FlowId::new` is deterministic UUIDv5 over name+version)

- Current lock consumer + generator:
  - `crates/cli/src/main.rs`

- Current integration tests:
  - `crates/cli/tests/bindings_lock.rs`
