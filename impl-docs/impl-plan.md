# Implementation Plan — LatticeFlow v1 Foundations

> Objective: deliver the smallest test-driven slice that gets us from the macro DSL to end-to-end workflow execution across Web/Queue/WASM profiles, unlock automated connector farming from n8n, and ship the scaffolding for policy/compliance while deferring higher-level Studio automation. This plan tracks directly to the RFC sections noted inline.

---

## Phase 0 — Workspace Scaffold & CI Spine

**Scope**
- Initialize cargo workspace (`impl-docs/surface-and-buildout.md:5`).
- Establish lint/test tooling (`rustfmt`, `clippy`, coverage, deny `unsafe` in core/macros) (`surface-and-buildout.md:154`).
- Create error-code registry (`DAG*`, `EFFECT*`, `DET*`) and baseline docs (`surface-and-buildout.md:157`).

**Deliverables**
- `Cargo.toml` workspace + empty crate stubs.
- `ci/` pipelines running fmt/clippy/test/doc.
- Error-code documentation + check enforcing code → doc sync.

**Tests**
- `cargo fmt -- --check`, `cargo clippy -- -D warnings` enforced in CI.
- Skeleton integration test ensuring workspace builds cleanly (`cargo check`).

**Exit criteria**
- CI is green on Linux/macOS, MSRV pinned at 1.77.
- Error-code doc generated and validated.

---

## Phase 1 — Core Types, Macros, IR Serialization

**References**: RFC §4 (DSL), §5 (Flow IR), JSON Schema (`schemas/flow_ir.schema.json`).

**Scope**
- Implement `dag-core`: Effects/Determinism enums, `Node`, `Trigger`, Flow IR structs, serde + schemars (`rust-workflow-tdd-rfc.md:215`).
- Implement `dag-macros`: `#[node]`, `#[trigger]`, `workflow!`, `inline_node!`, `connect!`, attribute helpers (`#[flow::switch]`, `#[flow::for_each]`) per §4.9.
- Emit Flow IR JSON + DOT exporters (`surface-and-buildout.md:164`).
- Implement validator skeleton (`kernel-plan::validate`) covering DAG topology, schema presence, effects/determinism lattice, basic policy hooks (`rust-workflow-tdd-rfc.md:322`).
- Land JSON Schema + reference artifact (already drafted) and wire snapshot tests.

**Tests (aggressive TDD)**
- `dag-core`: unit tests for Effects/Determinism conversions, serde round-trips.
- `dag-macros`: `trybuild` suite for success/failure cases (effects gating, determinism downgrades, control-surface metadata).
- Validator property tests: cycle detection, port type mismatch, control surface emission.
- Snapshot tests: Flow IR JSON matches schema for S1 & ETL example.

**Exit criteria**
- Example `examples/s1_echo` (Web ping) compiles (`cargo test -p examples -- s1_echo_check`).
- `flows graph check` CLI subcommand (stub) validates IR.

**CLI milestone**: We can run `flows graph check` locally to validate graphs.

---

## Phase 2 — In-Process Executor & Web Host

**References**: RFC §6.1–§6.5 (kernel), §4.7–§4.9 (control surfaces), user story S1/S2.

**Scope**
- Implement `kernel-exec` (Tokio-based scheduler, bounded channels, cancellation, backpressure) (`rust-workflow-tdd-rfc.md:333`).
- Implement backpressure metrics + cancellation propagation (`rust-workflow-tdd-rfc.md:423`–`441`).
- Build the HTTP bridge (`host-web-axum`) that translates requests into canonical invocations (including metadata forwarding) and delegates execution to the shared `host-inproc` runtime which now includes environment plugin hooks.
- Add capability registry + minimal capability implementations (`capabilities` crate) for HttpRead/Write, Clock, etc.
- Expand capability hint taxonomy (`resource::kv`, `resource::blob`, `resource::queue`, `resource::db::read`) with in-memory stubs (`MemoryKv`, `MemoryBlobStore`, `MemoryQueue`) so validators fire without waiting on platform adapters.
- Implement inline cache stub (memory) to satisfy Strict/Stable nodes w/out hitting policy errors (no remote caches yet).

**Tests**
- Unit tests: channel backpressure stalls, cancellation propagation, streaming captures (`streaming_capture_emits_events`, `dropping_stream_cancels_run`), and executor metrics assertions built with `DebuggingRecorder`.
- Integration: CLI `flows run local --example s1_echo` via `assert_cmd`; SSE coverage via `flows run local --example s2_site --stream` and `flows run serve --example s2_site` hitting `/site/stream` with `reqwest` + `timeout`; CLI JSON mode exercised by `run_local_emits_json_summary`.
- Regression harness: binary-level CLI tests live in `crates/cli/tests/` covering `run local` (value + stream + json summary) and Axum host wiring/metrics (`records_host_metrics_for_success`).
- Property-based coverage: executor backpressure invariants (`flow_instance_respects_capture_capacity`), SSE ordering (`sse_stream_preserves_event_sequence`), validator hint combinations (`fuzz_registry_hint_enforcement`), and CLI payload normalisation (`run_local_handles_property_inputs`). Metadata forwarding and environment plugin wiring are covered by host-inproc/host-web-axum tests.

**Exit criteria**
- `flows run local` executes S1 within latency budget, and `flows run serve` hosts S2 with SSE smoke tests.
- Executor/host metrics (`latticeflow.executor.*`, `latticeflow.host.*`) and CLI summaries (`--json` + stderr text) land per RFC §14.

**CLI milestone**: first `flows run local` success.

---

## Phase 3 — Queue Profile, Dedupe, Idempotency Harness

**References**: RFC §6.4 (queue), §5.2 (Idempotency), §5.4 (validation pipeline), user story S3, acceptance criteria (Idempotency/Windowing).

**Scope**
- Leverage the shared `host-inproc` runtime and implement `bridge-queue-redis` as a bridge handling ingestion, worker loop, visibility timeouts, and dedupe integration.
- Build `cap-dedupe-redis`, `cap-cache-redis`, `cap-blob-fs` for spill (`surface-and-buildout.md:185`).
- Extend validator: enforce `Delivery::ExactlyOnce` prerequisites, idempotency TTLs, cache requirements.
- Implement spill-to-blob + resume logic in kernel.
- Build idempotency certification harness (`testing/harness-idem`) injecting duplicates and reorders (`surface-and-buildout.md:188`).

**Tests**
- Unit tests for dedupe store TTL, idempotency key CBOR fixtures, ExactlyOnce guardrails.
- Queue integration test: run `examples/s3_etl` with duplicates (ensures single upsert) and lateness harness.
- Spill tests: saturate buffer -> spill -> resume with checksum validation.

**Exit criteria**
- `flows queue run` executes S3 example, idempotency harness green.
- Validator fails flows lacking idempotency keys / dedupe.

**CLI milestone**: `flows queue up` (local) + `flows certify idem` produce reports.

---

## Phase 4 — Registry Skeleton, Certification, ConnectorSpec tooling

**References**: RFC §10–§11, user story acceptance criteria, addendum `surface-and-buildout.md:194`.

**Scope**
- Implement `registry-client` (local file-store version) with signature stubs.
- Implement `registry-cert` harness: determinism replay for Strict/Stable, contract fixture runner, policy evidence bundler (`rust-workflow-tdd-rfc.md:618`).
- Define `connector-spec` schema + code generator targeting `connectors/<provider>` crates.
- Seed first connectors (Stripe charge/refund, SendGrid send, Slack alert) with full manifests, tests, and policy metadata.
- Extend CLI with `flows publish connector`, `flows certify connector`.
- Add policies for effects/determinism/idempotency gating.

**Tests**
- Connector generator snapshot tests (YAML → Rust + manifest).
- Certification harness integration tests (happy path + 401/429/5xx fixtures).
- Determinism replay test for Strict connectors with pinned resources.

**Exit criteria**
- At least 3 connectors certified and published locally.
- Registry evidence bundle generated per publish (JSON artifacts + DOT).

**CLI milestone**: `flows publish connector path/to/spec.yaml` end-to-end.

---

## Phase 5 — Plugin Hosts (WASM & Python), Capability Extensibility

**References**: RFC §9 (plugins), §6.4 (WASM host), user stories S7/S8.

**Scope**
- Implement `plugin-wasi`: WIT world, capability shims, sandbox enforcement (`rust-workflow-tdd-rfc.md:410`).
- Implement `plugin-python`: gRPC runner with sandbox policies and capability gates.
- Extend kernel to route plugin nodes with backpressure, deterministic IO (CBOR streaming).
- Provide base plugin SDKs (Rust for WASM, Python client) with examples.
- Ensure capability registry distinguishes between (a) third-party service connectors (Stripe, Slack) vs (b) third-party-agnostic extensions (e.g., CSV parser) vs (c) tenant-defined custom plugins. Document semantics difference.

**Tests**
- WASM integration tests (Pure/Strict) verifying determinism replay and sandbox.
- Python plugin tests covering network deny, timeouts, error propagation.
- Mixed flow (Rust inline + WASM + connector) verifying streaming interplay.

**Exit criteria**
- `examples/wasm/offline_cache` runs in WASM profile; DS Python pipeline runs with sandbox enforcement.
- Capability registry supports plugin-specific compatibility matrices.

**CLI milestone**: `flows run wasm` (local wasmtime) with example plugin.

---

## Phase 6 — Edge Deploy (WASM → worker), n8n Importer Harvest

**References**: RFC §12 (Importer), §9.2 (WASM), §7.2 (orchestration placement for edge), user story S8.

**Scope**
- Implement deployment packaging for WASM target (component + manifest + capability declarations).
- Build CLI `flows export wasm` to produce edge bundle.
- Implement importer MVP: parse n8n JSON, map connectors, emit Rust flows with `switch!/for_each!` hints, run compile loop (`rust-workflow-tdd-rfc.md:672`).
- Create harvesting pipeline to ingest metadata and prioritize connectors (support `flows import n8n --report`).
- Add automation harness for subagents (not fully autonomous yet): scriptable CLI flows to compile/import connectors.

**Tests**
- Integration: run harvested sample flows (w/out external side effects) through compile + validate pipeline.
- WASM deployment test: run generated bundle on simulated edge runtime (wasmtime + limited capabilities).
- Regression: ensure importer outputs Flow IR conforming to schema and control surfaces.

**Exit criteria**
- `flows import n8n path/to/workflow.json` outputs compilable flow + lossy report.
- WASM bundle deployable to edge target with policy-compliant manifest.

**CLI milestone**: first edge deployment + importer report delivered.

---

## Phase 7 — Connector Farming Automation & Subagent Harness (Stretch for MVP)

**Scope**
- Build orchestration scripts to assign connectors to builder/test/policy agents leveraging CLI (`flows template connector`, `flows certify connector`).
- Provide templated prompts/guidelines, not full Studio automation yet.
- Establish nightly job to pull top n8n connectors, generate specs, run certifications.

**Tests**
- Dry-run harness ensures CLI commands remain idempotent and machine-readable.
- Sample connectors auto-generated and certified end-to-end.

**Exit criteria**
- Automated pipeline can take a connector YAML skeleton, generate code, run tests, publish to local registry without manual intervention.

---

## Capability focus & carve-outs

### Third-party service connectors vs third-party-agnostic extensions
- **Service connectors** (Stripe, Slack) live under `connectors/<provider>`, rely on provider-specific capabilities, and are certified via registry harnesses.
- **Capability extensions** (e.g., data transforms, CSV parsers, rate limiters) live under `capabilities` or `packs/`, are provider-agnostic, and extend the platform without external dependencies.
- **Tenant-defined plugins** (WASM/Python) extend functionality per-tenant; they must declare capabilities + policies and are vetted via plugin harness, not connector registry.

We ensure the build plan separates these: Phase 4 targets service connectors; Phase 5 targets capability extensions and custom plugins; Phase 6 introduces tenant-specific WASM deployments.

---

## Out-of-scope (initial build)
- Studio backend/API (`rust-workflow-tdd-rfc.md:715`) and autonomous agent loops beyond scripted CLI usage.
- Temporal adapter (Phase 6+ per addendum) — queued for later milestone.
- Budget planner, policy diff UI, advanced cost telemetry (`rust-workflow-tdd-rfc.md:452`, §13).
- Full registry security hardening (cosign signatures, SBOM verification); initial registry is local/insecure.
- Multi-region deployment controls and tenant governance beyond lint + local policy engine.
- Advanced caching tiers (remote caches), global durable state migrations; initial plan ships only memory/disk caches.
- Connector marketplace / monetization features.

---

## Summary timeline (indicative)
1. **Phase 0–1 (Weeks 0–3)**: DSL + IR + validator skeleton.
2. **Phase 2 (Weeks 3–5)**: Kernel + Web host + local CLI runs.
3. **Phase 3 (Weeks 5–7)**: Queue profile + idempotency harness.
4. **Phase 4 (Weeks 7–10)**: Registry + connector pipeline.
5. **Phase 5 (Weeks 10–13)**: Plugin hosts + capability extensions.
6. **Phase 6 (Weeks 13–16)**: WASM edge deploy + importer MVP.
7. **Phase 7 (optional, Weeks 16+)**: Automated connector farming.

Tests and CLI milestones gate every phase to keep the system verifiable and agent-ready.
