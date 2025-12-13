Status: Draft
Purpose: epic
Owner: Core
Last reviewed: 2025-12-12

# Epics (DRAFT)

This is the working roadmap. Each epic is a deliverable that can ship value.
Each epic is decomposed into phases; phases should be ticket-sized and test-gated.

Assumption: Workers-native execution is acceptable as the near-term target.

## Roadmap Principles

- One contract at a time: stabilize IR/ABI surfaces before expanding features.
- Prefer opaque binding first: hosts build `ResourceBag` from their environment; `caps.lock` later.
- Requirements are domain-level in 0.1.x (via IR hints); add capability facets/properties later.
- Every phase has an acceptance gate (tests/fixtures/CLI commands) and rollback story.

## Epic 0 — Documentation + Baseline Hygiene

Doc: `epic-00-docs-and-hygiene.md`

Goal: consolidate docs into spec/ADR/roadmap/scenario layers; ensure repo status is clear.

Why:
- Planning docs are currently too "mega"; we need epics -> phases -> tickets.
- Make it obvious what is canonical vs aspirational vs superseded.

Phases (ticket-sized):
- Add `impl-docs/README.md` index + folder taxonomy (non-destructive).
- Tag each legacy doc with: `Status: Canonical | Draft | Superseded | Archived`.
- Migrate `impl-docs/impl-plan.md` into epics/phases; move detailed steps into epic phase tickets.
- Archive old phase notes under `impl-docs/archive/` with pointer stubs.

Acceptance gates:
- `impl-docs/README.md` points to canonical sources (one roadmap, one spec set).
- `cargo fmt --check` and `cargo test --workspace` are green.

## Epic 1 — 0.1 Stable Surface: Flow IR + Invocation ABI + Macro/Trigger Surface v0

Doc: `epic-01-contract-0.1.md`

Goal: define a stable-ish surface area that all hosts (Axum, Workers, queue bridge) can share.

Why:
- Everything else (Workers, connectors, importer, marketplace) becomes easier if the execution ABI and authoring surface are stable.

Scope notes:
- This epic is where new macro primitives land (at least as spec + validation). Some also become runtime-semantic.

Phases (ticket-sized):

1) Invocation ABI (host-agnostic execution boundary)
- Freeze `host_inproc::Invocation` semantics (payload, aliases, metadata, deadline).
- Freeze `ExecutionResult` semantics (Value/Stream) and error envelope expectations.
- Standardize core metadata keys (run_id, event_id, idem_key, tenant, route).

Acceptance gates:
- Bridge-safe serialize/deserialize of invocation parts; deterministic error formatting.

2) Capability requirements + runtime preflight (domain-level)
- Derive required capability domains from Flow IR (primarily via node resource hints).
- Host refuses to start/serve if required capabilities are missing in `ResourceBag`.

Acceptance gates:
- Deterministic preflight failures for missing bindings.

3) Macro/IR control surface v0 (authoring surface)
- Ship semantics-bearing primitives that already exist in IR:
  - `delivery!` -> sets `EdgeIR.delivery` (planner already enforces ExactlyOnce rules).
  - `buffer!`/`spill!` -> sets `EdgeIR.buffer` (executor already uses buffer/spill config).
  - `timeout!` -> sets `EdgeIR.timeout_ms` (host/executor enforcement can be incremental).

- Ship switch/if as semantics-bearing (not just UI metadata):
  - `if!` / `switch!` emit `FlowIR.control_surfaces` with stable config shapes.
  - `kernel-exec` respects routing surfaces (only schedule selected branch).

- Define but do not necessarily implement (yet):
  - `on_error!`, `rate_limit!`, `partition_by!`, `for_each!`, `window!`
  - In 0.1, these can be: spec + validation + deterministic runtime "unsupported" error until implemented.

Acceptance gates:
- trybuild tests for macro syntax.
- `kernel-plan` validates surface structure.
- `kernel-exec` implements if/switch routing OR hard-fails deterministically.

4) Triggers/entrypoints surface v0
- Keep trigger nodes in Flow IR (`NodeKind::Trigger`) as today.
- Keep deployment trigger wiring in host config (e.g., Axum `RouteConfig`, Workers equivalent).
- Optionally introduce an entrypoints manifest as a Flow IR artifact (referenced via `FlowIR.artifacts`) without changing core IR.

Acceptance gates:
- Host refuses to serve if trigger/capture aliases don't exist.
- CLI can validate flow + entrypoint config bundle.

## Epic 2 — Workers-Native Host Adapter (Minimal)

Doc: `epic-02-workers-host-minimal.md`

Goal: run Flow IR in Cloudflare Workers (wasm32-unknown-unknown) for HTTP triggers.

Why:
- Validates portability and deployment substrate.
- Unblocks the first "real" vertical slice on Workers.

Phases (ticket-sized):
- Build target: make kernel + host compile for wasm.
  - Feature-gate OS-only pieces (tempfile spill, filesystem assumptions).
  - Tokio setup tuned for wasm execution profile.
- Implement `host-workers` fetch handler: request -> invocation -> executor -> response.
- Error envelope aligned with Axum host.

Acceptance gates:
- wasm build is green for the minimal profile.
- Echo flow runs in Workers dev environment.

## Epic 3 — Cloudflare Capability Providers Needed for One Vertical Slice

Doc: `epic-03-cloudflare-caps-minimal.md`

Goal: provide the minimum Cloudflare-native capabilities to support stateful event workflows.

Why:
- The Resend flow needs (a) outbound HTTP, (b) durable per-key state for sequencing/dedupe.

Phases (ticket-sized):
- Workers HTTP client capability provider (fetch-backed).
- Durable Object state/sequencer capability provider.
- Secrets/env binding contract (names only; no secrets in artifacts).

Acceptance gates:
- Preflight detects missing bindings/secrets.
- Concurrency test: single-writer semantics per DO key.

## Epic 4 — Resend Email Sequencing Vertical Slice (0.1 Candidate)

Doc: `epic-04-resend-vertical-slice.md`

Goal: "on inbound email webhook" -> stateful sequencing -> resend send mechanics, deployed to Workers.

Why:
- Validates: triggers, effectful connectors, idempotency, state, deploy packaging, observability.

Phases (ticket-sized):
- Resend webhook trigger adapter (signature verify) + normalization.
- Sequencing logic backed by DO state (per-thread ordering).
- Resend send connector (HTTP write + idempotency key strategy).
- Scenario test: duplicate/out-of-order deliveries do not double-send; ordering maintained per thread.

Acceptance gates:
- Running in Workers: duplicate injection yields a single visible side-effect.
- Observability: logs/metrics include run_id + event_id + idem_key.

## Epic 5 — Connector Certification Harness (Pre-marketplace)

Doc: `epic-05-certification-harness.md`

Goal: evidence-driven confidence for effectful connectors.

Why:
- Marketplace only works if we can certify effectful behavior (idempotency, error mapping).

Phases (ticket-sized):
- Minimal connector contract tests (200/401/429/5xx) + standard error taxonomy mapping.
- Idempotency injection harness: duplicate deliveries, reorder, retry storms.
- Emit machine-readable evidence artifacts per run.

Acceptance gates:
- Resend connector can be certified locally with fixtures; evidence artifact produced.

## Epic 6 — n8n Farmability (Importer + Coverage + HITL)

Doc: `epic-06-n8n-farmability.md`

Goal: import n8n JSON -> Flow IR for the common subset; report/triage the rest.

Why:
- This is the "agents can maintain workflows" unlock: deterministic conversion + deterministic diffs.

Phases (ticket-sized):
- Implement core node mapping: webhook/cron/http/set/if/switch.
- Expression lowering for `={{...}}`:
  - typed adapters for common patterns (jsonpath, templating)
  - fallback sandboxed compute node (explicit BestEffort/Nondeterministic)
- Build a standard library of transform nodes (pure/readonly) needed for importer output.
- Corpus runner: convert templates, compute coverage, generate HITL diffs.

Acceptance gates:
- Large subset of templates convert; unsupported patterns flagged deterministically.

## Epic 7 — Marketplace/Registry (Publish + Policy Gates)

Doc: `epic-07-marketplace-registry.md`

Goal: publish connectors/subflows with certification evidence and policy gates.

Why:
- Enables sharing + reuse + safe upgrades.

Phases (ticket-sized):
- Registry metadata model + versioning.
- Publish/consume CLI.
- Policy gates based on evidence artifacts.

Acceptance gates:
- Publish blocks without required proofs; consumers can verify artifacts.

## Later Epics (not scheduled yet)

- Wait/Resume ("Await external event") + checkpoints (approval, HITL, long-lived workflows)
- SQL capability + migrations + schema guarantees
- Temporal lowering + compensation/SAGA
- Windowing/watermarks
- Plugin WASI/Python sandbox hardening
- Expanded capability catalog + provider selection (`caps.lock`, Terraform integration)
