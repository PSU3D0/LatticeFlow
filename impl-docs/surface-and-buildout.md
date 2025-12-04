## A. Workspace layout (authoritative)

```
/Cargo.toml                # [workspace]
/crates
  /dag-core                # types: FlowIR, ExecPlan, traits, errors, schemas
  /dag-macros              # #[node], #[trigger], workflow!, subflow!, inline_node!
  /kernel-plan             # IR validator + lowering -> ExecPlan
  /kernel-exec             # in-proc executor (Tokio), scheduler, backpressure
  /host-inproc             # Shared runtime harness reused by bridges
  /host-web-axum           # Axum bridge (HttpTrigger, Respond bridge, SSE)
  /bridge-queue-redis      # Queue bridge (Redis), ack/dedupe integration
  /host-temporal           # Temporal codegen (Go/TS) + Rust activity worker
  /plugin-wasi             # Wasmtime host, WIT world, capability shims
  /plugin-python           # gRPC plugin host + sandbox policies
  /capabilities            # http_{read,write}, kv_{read,write}, blob_*, rng, clock, ...
  /cap-http-reqwest        # concrete Http impl (read/write); rate-limit/backoff
  /cap-kv-sqlite           # local kv read/write; used for tests/dev
  /cap-blob-fs             # fs blob store; “by_hash” Strict reads
  /contrib/redis/cap-redis        # Shared Redis plumbing
  /contrib/redis/cap-dedupe-redis # DedupeStore impl
  /contrib/redis/cap-cache-redis  # Cache impl
  /exporters               # serde_json, dot, wit
  /policy-engine           # CEL-based policy, compilation stages, evidence
  /registry-client         # read/publish artifacts, signatures, SBOM ingest
  /registry-cert           # certification harnesses (determinism, idem, contract)
  /connector-spec          # YAML schema + codegen (templates)
  /connectors-std          # umbrella crate re-exporting concrete connectors
  /connectors/<provider>   # per-provider crates (openai, slack, stripe, ...)
  /packs/pack-<domain>     # subflow libraries (marketing, commerce, observability)
  /types-common            # EmailAddress, Money, UrlRef, FileRef, …
  /adapters                # From<T> for U mappings (type adapters)
  /importer-n8n            # JSON -> FlowIR -> Rust generator
  /studio-backend          # API for agents (compile, run, certify, publish)
  /cli                     # flows (clap): init, graph check, run, certify, publish
  /testing/harness-*       # idem, determinism, contract, load, temporal, web
/examples                  # runnable examples (S1..S10)
```

**Layering rule (enforced in CI)**

* `dag-macros` → depends only on `dag-core`
* `kernel-plan` → depends on `dag-core`
* `kernel-exec` → depends on (`dag-core`, `kernel-plan`, `capabilities`)
* hosts/plugins can depend on `kernel-exec` but **never** the other way
* connectors depend on `capabilities`, optional `adapters`, never on hosts

---

## B. Public API surfaces (stable-to-agents)

> Agents interact primarily via these crates; keep their APIs stable and well‑documented.

### B1. `dag-core` (foundation)

```rust
// Effects/Determinism
pub enum Effects { Pure, ReadOnly, Effectful }
pub enum Determinism { Strict, Stable, BestEffort, Nondeterministic }

// Node & Trigger
#[async_trait]
pub trait Node {
  type In: Port; type Out: Port; type Params: Param; type Resources: ResourceAccess;
  async fn run(&self, input: NodeStream<Self::In>, ctx: &mut Ctx<Self::Params, Self::Resources>)
      -> NodeResult<NodeStream<Self::Out>>;
}
#[async_trait]
pub trait Trigger { /* start(...) -> NodeStream<Out> */ }

// Flow IR & Plan
pub struct FlowIR { /* sections §5 in RFC */ }
pub struct ExecPlan { /* run units, schedule hints, io bridges */ }

// Orchestrator abstraction
pub trait Orchestrator { /* schedule_activity, await_signal, set_timer, spawn_child, continue_as_new */ }
pub trait ActivityRuntime { fn run_activity(&self, node: &NodeIR, input: Bytes) -> ActivityResult; }

// Exporters
pub trait Exporter { fn formats(&self)->&[&'static str]; fn export(&self, wf:&WorkflowSpec)->Vec<Artifact>; }

// Policy evidence
pub struct PolicyEvidence { /* what was checked, results */ }
```

### B2. `dag-macros` (authoring)

* `#[node]`, `#[trigger]`, `workflow!`, `subflow!`, `inline_node!`, `connect!`, `hitl!`, `partition_by!`, `delivery!`, `timeout!`, `on_error!`, `rate_limit!`, `creds!`, `export!`, `vars!`, `probe!`.

**Diagnostics contract:** all macro errors use stable codes (`DAG201`, `EFFECT201`, `DET301`, …) and structured help messages (agents parse these).

### B3. `kernel-plan` (validation + lowering)

```rust
pub fn validate(ir: &FlowIR, policy: &Policy) -> Result<ValidatedIR, Vec<Diagnostic>>;
pub fn lower(ir: &ValidatedIR, target: HostTarget) -> Result<ExecPlan, Diagnostic>;
```

### B4. `kernel-exec` (in-proc engine)

```rust
pub struct Kernel;
impl Kernel {
  pub fn new(profile: Profile, caps: CapRegistry, exporters: Vec<Box<dyn Exporter>>) -> Self;
  pub async fn run(&self, plan: ExecPlan) -> RunHandle;
  pub async fn resume(&self, checkpoint: CheckpointId, payload: Bytes) -> Result<RunId>;
}
```

### B5. Host adapters

* `host-web-axum::AxumBridge::mount(kernel, routes...)`
* `bridge-queue-redis::QueueBridge::enqueue/spawn_workers(...)`
* `host-temporal::{generate_workflow_code, ActivityWorker::run(...)}`

### B6. Capabilities (typestates)

```rust
pub trait HttpRead: Capability { fn get(&self, url:&Url) -> Result<Bytes>; fn get_pinned(&self, p:&PinnedUrl)->Result<Bytes>; }
pub trait HttpWrite: Capability { fn post(&self, ...); /* ... */ }
pub trait KvRead: Capability { fn get<T: DeserializeOwned>(&self, key:&str)->Result<Option<T>>; }
pub trait KvWrite: Capability { fn set<T: Serialize>(&self, key:&str, val:&T)->Result<()>; }
pub trait BlobRead: Capability { fn get_by_hash(&self, h:&HashRef)->Result<Bytes>; /* Strict */ }
pub trait DedupeStore: Capability { fn put_if_absent(&self,key:&[u8],ttl:Duration)->Result<bool>; }
```

---

## C. Features & MSRV

* **MSRV:** 1.77 (document it; bump via RFC).
* **Cargo features** (top‑level):

  * `web` (enables `host-web-axum`, SSE, multipart)
  * `queue-redis`
  * `temporal`
  * `wasm`
  * `python`
  * `importer`
  * `registry`
  * `std-connectors` (pulls `connectors-std`)
* **No `unsafe`** in core/macros; `plugin-wasi` may use `unsafe` only in sandbox shims (audited).

---

## D. Buildout phases with **Definitions of Done** (DoD)

Each phase includes: scope, artifacts, tests, and the **checklist that CI must pass** before merge.

### Phase 0 — **Scaffold & CI**

**Scope:** Workspace; linting; error code schema.
**DoD:**

* `cargo fmt`, `clippy -D warnings`, `rustdoc` builds.
* Error code registry doc: table of `DAG*`, `EFFECT*`, `DET*`, …
* GitHub Actions matrix (linux, macOS) with cache.
* MSRV gate + deny `unsafe_code` in core/macros.

### Phase 1 — **Core types + macros + IR + validation**

**Scope:** `dag-core`, `dag-macros`, `kernel-plan::validate`, exporters(json,dot).
**Artifacts:** Port/Effects/Determinism, FlowIR, macro diagnostics; `flows graph check`.
**Tests:**

* Unit: all macro errors; IR round‑trip.
* Property: `Join/Merge` invariants (no loss/dup).
* Golden: S1‑Echo flow compiles & exports DOT.
  **DoD:** `flows graph check examples/s1_echo` passes; coverage ≥ 80% for `dag-macros`; JSON Schema + example emitted at `schemas/flow_ir.schema.json`.

### Phase 2 — **In‑proc executor + Web host**

**Scope:** `kernel-exec`, `host-web-axum`, SSE streaming & cancellation; Inline nodes.
**Artifacts:** S1 (Echo) & S2 (Website) runnable.
**Tests:**

* Latency: p95 < 50ms (S1), SSE gap < 30ms (S2).
* Cancellation propagation test.
  **DoD:** `examples/s1_echo`, `s2_site` pass perf tests in CI.

### Phase 3 — **Queue/Redis + Dedupe + Idempotency**

**Scope:** `bridge-queue-redis`, `contrib/redis/cap-dedupe-redis`, `contrib/redis/cap-cache-redis`, spill to `cap-blob-fs`.
**Artifacts:** S3 ETL (without Temporal) including windowing.
**Tests:**

* Idem harness injects duplicates → single side‑effect.
* Spill/resume; backpressure metrics; OOM soft‑fail.
  **DoD:** `flows certify idem` green; spill harness passes.

### Phase 4 — **Registry & Certification + ConnectorSpec + Std connectors**

**Scope:** `registry-client`, `registry-cert`, `connector-spec`, `connectors-std` (15–20 providers).
**Artifacts:** publish path; SBOM + cosign; contract tests harness.
**Tests:** 200/401/429/5xx fixtures for 5 core connectors; determinism replay on Strict/Stable.
**DoD:** Registry gate rejects missing effects/determinism/idempotency; 10 connectors certified.

### Phase 5 — **Plugins: WASM + Python**

**Scope:** `plugin-wasi` (WIT), `plugin-python` (gRPC sandbox), capability allowlists.
**Artifacts:** S7 DS pipeline runs, WASM node demo.
**Tests:** Canonical CBOR IO; sandbox denies network; timeout enforcement.
**DoD:** determinism replay for Pure/Strict WASM node; python sandbox tests pass.

### Phase 6 — **Temporal adapter**

**Scope:** `host-temporal` codegen & Rust activity worker; signals/timers/child workflows/continue‑as‑new; retry ownership rules.
**Artifacts:** S4 Payments & S6 Onboarding run on dev cluster.
**Tests:** History budget, SAGA compensation, HITL signal latency.
**DoD:** Temporal flows pass; history < thresholds; retry ownership lint enforced.

### Phase 7 — **Importer + Corpus harvesting**

**Scope:** `importer-n8n`, `harvester`; lossy catalog & SLO gates.
**Artifacts:** 200 workflows imported; ≥70% compile at M3, ≥85% at M5.
**Tests:** Expression fallback cap; determinism downgrade lints.
**DoD:** Importer CI job produces report; failing SLO gates block publish.

### Phase 8 — **Studio backend + CLI polish**

**Scope:** `studio-backend` (compile/run/certify APIs), `cli` UX, policy evidence.
**Artifacts:** Agent loops automated; flows publish end‑to‑end.
**Tests:** CLI contract snapshots; policy evidence JSON schema validation.
**DoD:** “intent → green build → certify → publish” in one CI job on examples S1–S5.

---

## E. “Definition of Done” checklists (per crate)

**`dag-core` (DoD)**

* Public API docs for all traits/enums.
* No `unsafe`; `Send + Sync` bounds documented.
* JsonSchema for FlowIR exported and validated against sample IRs.
* Benchmarks: idempotency key hash < 1µs P50 (64‑byte payload).

**`dag-macros`**

* Error codes stable; snapshot tests for messages.
* Macro expansion tests (trybuild) for success/failure cases.
* Determinism/effects derivation unit tests.

**`kernel-plan`**

* Validates all 15 rule classes (ids enumerated in RFC §5.4).
* Lowering rejects non‑Pure inline units for Temporal.
* DOT exporter verified against small graphs.

**`kernel-exec`**

* Backpressure: bounded channels; stall rather than drop by default.
* Spill: deterministic format; CRC; resume test.
* Cancellation: propagates to node `Ctx`.

**`host-web-axum`**

* SSE; explicit deadlines; request facet; respond bridge.
* Multipart upload and large body streaming tests.

**`bridge-queue-redis`**

* Visibility timeout semantics; at‑least-once + dedupe.
* Rate limiter per node; fairness scheduling.

**`host-temporal`**

* Codegen reproducible; activity worker handles serde/CBOR rigorously.
* Signals, timers, child workflows, continue‑as‑new covered by tests.

**`plugin-wasi`**

* ABI versioned; deny‑all capabilities by default; memory limits enforced.

**`plugin-python`**

* Container profile (seccomp/AppArmor), network deny by default; canonical JSON/CBOR.

**`registry-*`**

* Cosign signing; CycloneDX SBOM; vulnerability scan gate (critical CVEs block).
* Evidence bundle: determinism replay, idem results, contract logs.

**`connector-spec`**

* YAML schema (`schemars`) published; codegen templates snapshot‑tested.
* Generator refuses missing data‑class tags.

**`cli`**

* All subcommands return **machine‑readable JSON** with `--json` flag.
* Non‑zero exit codes for any failed gate.

---

## F. CI pipeline (prescriptive)

* **Stages:** lint → build → unit → macro trybuild → examples → harnesses (idem/det/contract) → integration (web/queue/temporal) → publish (dry‑run).
* **Caches:** sccache for Rust; registry of connector fixtures (VCR) keyed by hash.
* **Artifacts:** DOT graphs, Policy evidence JSON, SBOMs, certification logs, coverage HTML.

---

## G. Coding agent affordances

* All tools return **structured JSON** (errors, diagnostics, hints) behind `--json`.
* Error codes mapped to **remediation playbooks** (short text plus “fix‑it” diffs).
* Template generators (`flows template`) for nodes/connectors/subflows/flows.
* Example repositories for each canonical story (S1–S10) with golden tests, so agents can copy/modify and stay green.

---

## H. Risks & mitigations (for buildout)

* **Cross‑crate coupling drift** → enforce layering via `cargo deny` + custom check that scans `Cargo.lock` edges.
* **API churn** → mark `dag-core` and `dag-macros` as **stability tier 1**; changes require ADR + minor bump.
* **Temporal adapter complexity** → start with one reference flow (payments), lock patterns, add more later.
* **Fixture flakiness** → determinism harness records seeds/etags; CI never calls live endpoints outside canaries.

---

## I. Concrete “first tickets” (ready to file)

1. **dag-core#1:** Define `Effects`, `Determinism`, `Item<J,B,L>`, `Node`, `Trigger` + docs.
2. **dag-macros#1:** `#[node]` MVP: in/out/params/resources, EFFECT/DET derivation + 10 trybuild tests.
3. **kernel-plan#1:** FlowIR structs + validator for DAG200/201 + exporter(json,dot).
4. **host-web-axum#1:** HttpTrigger + Respond (non-stream) + deadline; S1 example green.
5. **kernel-exec#1:** Bounded channels + cancellation; unit tests.
6. **exporters#1:** DOT exporter snapshot tests.
7. **testing/harness-idem#1:** Duplicate injection harness + minimal sink.
8. **cli#1:** `flows graph check` & `flows run local` (S1).
9. **cap-http-reqwest#1:** HttpRead/Write adapters with backoff; unit tests.
10. **policy-engine#1:** CEL evaluator MVP; allowlist of capabilities/egress.

---

## J. “Definition of Green Build” (end-to-end)

A PR is “green” when:

* All **phase‑gated** crates in scope meet their DoD checklists.
* `examples/s1_echo` and `examples/s2_site` run & pass perf tests (Phase 2).
* harness‑idem and determinism harnesses pass on included sample nodes (Phase 3+).
* If connectors change: registry cert harness passes for changed connectors.
* All CLI commands used in CI emit JSON and **no unstructured warnings**.

---

### Bottom line

Your RFC nails *what* to build. This addendum nails **how** to build it in crates, in what order, with **APIs agents can depend on**, and **crisp DoD gates**. If you want, I can generate the actual workspace skeleton with `Cargo.toml`s, a minimal `dag-core`/`dag-macros` pair, and the S1/S2 examples so your team can push the first green CI run immediately.
