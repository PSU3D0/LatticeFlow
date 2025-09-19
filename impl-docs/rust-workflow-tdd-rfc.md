# Rust Workflow Platform — Technical Design Document & RFC (v1.0)

## Document Metadata
- **Title:** Rust Workflow Platform — Technical Design Document & RFC (v1.0)
- **Authors:** Core Runtime Team, Agent Platform Team
- **Reviewers:** Connector Guild, Studio Backend, DevOps/SRE, Security & Compliance, Product Strategy
- **Status:** Draft for Cross-Team Review
- **Last Updated:** 2025-09-18
- **Intended Audience:** Kernel engineers, connector authors, agent platform developers, DevOps/SRE, security/compliance, product/PM, technical program management
- **Related ADRs:**
  - ADR-001 — Kernel Strategy & Host Adapters
  - ADR-002 — Effects & Determinism Enforcement
  - ADR-003 — Orchestrator Abstraction & Temporal Lowering
  - ADR-004 (pending) — Registry Certification Requirements
- **References:** n8n product docs, community threads, Temporal documentation, async-openai/official SDK docs (citations inline where used)

---

## 1. Background & Motivation

### 1.1 Market context
- n8n is positioned (per 2025 coverage) as an “AI workflow automation platform” pursuing an agentic automation narrative. Investors view it as a possible backbone for agent-first businesses.
- n8n’s valuation thesis hinges on a visual interface that non-developers can use. Agents, however, require compile-time guarantees that low-code JSON/JavaScript graphs do not provide.
- There is no credible open-source Rust alternative with full n8n parity. Adjacent Rust-based engines (Windmill, Vector, Arroyo, Temporal SDK) validate the language/runtime choice for durable, typed workflows.

### 1.2 Why code-first Rust
- **Safety:** Typed ports, explicit effects, and determinism metadata enable static verification. Agents can reason about flows, auto-refactor, and rely on compiler diagnostics.
- **Portability:** Rust + WASM + gRPC plugin support allows running the same flow in local dev, queue workers, Temporal, edge devices, or browsers.
- **Policy compliance:** Capability-based resource access and egress allowlists give enterprises control absent from typical low-code tools.
- **Developer & agent velocity:** Compiler feedback (type errors, effect warnings) guides both humans and agents; inline nodes allow quick business logic.
- **Import opportunity:** We can import a large corpus of existing n8n workflows (e.g., Zie619/n8n-workflows) to drive connector coverage and prove compatibility.

### 1.3 Document purpose
This RFC describes the end-to-end technical design for the Rust workflow platform (“workflow.core”), including:
- Macro authoring surface
- Flow IR and validation
- Kernel runtime & host adapters
- Plugin & connector architecture
- Effects/determinism enforcement
- Registry and certification
- Importer strategy
- Agent Studio backend & CLI
- Observability, security, rollout

It is intended for cross-team review before implementation begins.

---

## 2. Goals & Non-Goals

### 2.1 Goals
1. **Typed macro DSL** for nodes, triggers, workflows, subflows, inline nodes, checkpointing, and policy metadata.
2. **Single kernel runtime** with capability/facet injection, supporting multiple hosts (Tokio, Redis queue, Axum/web, Temporal, WASM, Python).
3. **Registry** that enforces effects, determinism, idempotency, egress, scopes, and testing requirements for connectors/subflows.
4. **Importer pipeline** to map n8n JSON workflows into typed Rust flows with adapters and lossy notes.
5. **Plugin architecture** for Rust, WASM (WIT), and out-of-process (gRPC) nodes.
6. **Agent-first tooling**: Studio backend, CLI, agent loops, manifests powering generated UI.
7. **Policy, security, observability** baked into the platform.
8. **Harvesting plan** for the Zie619/n8n-workflows corpus to drive coverage and tests.
9. **Portable caching/replay**: memoize `Strict`/`Stable` subgraphs via content-addressed snapshots with policy-driven cache eviction and invalidation.

### 2.2 Non-goals (initial release)
- Front-end builder UI beyond generated forms.
- Marketplace/monetization design.
- Hard real-time guarantees or GPU graph fusion.
- Full expression language parity (we provide adapters, not a JS interpreter).
- Implementing or maintaining the Temporal Rust SDK (we integrate via Go/TS codegen first).
- Global mutable context or ambient singletons inside flows or nodes.

---

## 3. System Overview

```
Authors/Agents -> Macro DSL -> Flow IR -> Validator -> ExecPlan -> Host Adapter -> Kernel Runtime -> Activities/Plugins
                                                             |              |
                                                             v              v
                                                        Registry         Observability
```

### 3.1 Workflow authoring lifecycle
1. Author defines nodes/triggers/workflows with macros.
2. Macros emit Rust code + Flow IR + manifests.
3. Flow IR is validated (types, cycles, effects, determinism, policies).
4. Lowering stage creates an execution plan (`ExecPlan`) per target host.
5. Host adapter executes plan using kernel runtime + plugin hosts.
6. Observability and policy engines record metadata and enforce rules.
7. Registry stores certified connectors/subflows.

### 3.2 Primary components
- **Macro crates** (`dag-macros`): derive macros and DSL.
- **Core runtime** (`dag-core`, `kernel-exec`): Flow IR structs, scheduler, capability logic.
- **Host adapters**: Tokio, Redis queue, Axum web, Temporal, WASM, Python gRPC.
- **Plugin hosts**: WASM component runtime, Python worker, native node registry.
- **Registry service**: metadata store + certification harness.
- **Studio backend**: API + agent orchestrator.
- **CLI**: developer tooling (`flows` command).
- **Importer**: n8n JSON -> Flow IR -> Rust.
- **Harvester**: metadata mining from existing repo (Zie619) + analytics.

### 3.3 Control-plane vs data-plane
- **Control-plane (multi-tenant)**: registry, policy engine, Studio API, importer pipeline, certification runners, build/publish services, SBOM/signing infrastructure.
- **Data-plane (per-tenant isolation tier)**: kernel runtime, activity workers, adapter hosts (Tokio/Redis/Temporal), capability gateways, cost/budget enforcers.
- **Cross-cutting contract**: control-plane emits signed manifests + policies; data-plane enforces them at execution time and streams telemetry back for policy feedback.

---

## 4. Macro Surface Specification

### 4.1 `#[node]`
- Attributes: `name`, `summary`, `category`, `in`, `out`, `params`, `resources`, `batch`, `concurrency`, `idempotency`, `retry_owner`, `delivery`, `cache`, `determinism`, `effects`, `docs`.
- Generates:
  - Rust struct implementing `Node` trait (`async fn run` with typed stream input/output).
  - Flow IR entry (NodeIR).
  - NodeSpec manifest (JSON) for registry/UI.
- Validations (compile-time & build-time):
  - DAG001 PortTypeMissing
  - DAG002 ParamsNotSchema
  - DAG003 ResourceUnknown
  - DAG004 IdempotencyInvalid
  - DAG005 ConcurrencyOutOfRange
  - DAG006 BatchConflict
  - EFFECT201 Effect mismatch between resources and declared effects
  - DET301 Determinism claim incompatible with capabilities
  - RETRY010 BothSidesEnabled
  - CACHE001 StrictWithoutCachePolicy (warn)
  - CACHE002 StableWithoutPin
  - SAGA201 IncompatibleCompensatorPort

### 4.2 `#[trigger]`
- Additional fields: `respond`, `path`, `method`, scheduling metadata.
- Output: `Trigger` trait impl, NodeSpec, Flow IR entry.
- Validations: DAG101 TriggerHasNoOut, DAG102 RespondModeMismatch, DAG103 HttpPathConflict.

### 4.3 `workflow!`
- Declarative graph: `use`, `vars!`, node instantiation, `connect!`, `hitl!`, policies, exports.
- `connect!` supports edge metadata (`delivery = ExactlyOnce`, `buffer = {...}`) with validators ensuring DedupeStore bindings and partition coverage.
- Validations: DAG200 CycleDetected, DAG201 PortTypeMismatch, DAG202 VarUnbound, DAG203 CredsUnbound, DAG204 NoTerminalRespond, DAG205 DuplicateNodeName, DAG206 ExporterUnavailable, DAG300 HitlOnTrigger, DAG301 ViewNotSerializable, DAG320 NonPositiveQPS, DAG330 ProviderMissing, DAG331 ScopeDenied, DAG340 VarShadow, DAG341 VarTypeMismatch.
- Supports `expose_as_node: FooNode` to reuse flows as nodes.
- Web routes must declare latency budgets (default 300 ms p95) and leverage `timeout!` to enforce request deadlines.

### 4.4 `subflow!`
- Function-like composite: single INPUT/OUTPUT, optional `effects:` (`Pure`, `ReadOnly`, `Effectful`) and `determinism:`.
- Internally expands to `workflow!` but ensures no triggers or top-level `Respond`.
- Useful for reusable composites and domain packs.

### 4.5 `inline_node!`
- Quick inline transforms; default `effects: Pure`, `determinism: Strict`.
- Accepts `resources(...)` list to opt into capabilities; compile-time gating ensures correct effect level.
- Determinism guard: if an inline node references `Clock`, `Rng`, or any unpinned external resource, the macro falls back to `BestEffort` and emits a diagnostic requiring author acknowledgement or removal of the dependency.

### 4.6 Other macros
- `hitl!(label, on = node|edge, view = ...)`
- `on_error!(scope, strategy = Retry/DeadLetter/Halt/Escalate)`
- `rate_limit!(scope, qps, burst)`
- `creds!(scope, Provider { ... })` declares least-privilege scopes; CLI verifies granted scopes ⊆ requested via provider introspection; validator `SECR201 OverPrivilegedSecret` fires otherwise.
- `vars! { NAME: Type = default }`
- `export!(format = "serde_json" | "dot" | "wit" | custom)`
- `probe!(...)` for metrics instrumentation
- `partition_by!(target, key = expr)` to declare partition/shard keys for parallel execution, child workflow routing, and idempotency scope isolation.
- `timeout!(scope, dur)` to apply consistent wall-clock limits to nodes or subgraphs; binds both execution and retry budgets. Temporal maps to `ScheduleToClose`/`StartToClose` depending on scope; runtime emits `TIME015 OverBudget` on violation.

### 4.7 Diagnostics & help
- Errors reference exact macro spans with fixes (e.g., “implement `From<Action> for UploadInput` or insert `Map<Action, UploadInput>`”).
- Effects/determinism lints provide direct guidance (e.g., “remove `Clock` or downgrade determinism to BestEffort”).

### 4.8 Standard combinators
- Std library ships canonical `Map`, `Join<K, A, B>`, and `Zip` nodes to model multi-input synchronization without tuple gymnastics. `Join` enforces keyed fan-in with partition-aware buffering; `Zip` pairs ordered streams with backpressure awareness.
- Temporal lowering and backpressure semantics assume these combinators; custom multi-input nodes must implement equivalent contracts or provide proofs in certification.
- Event-time aware joins use watermarks described in §4.8.1 to align streams and honor lateness policies.

### 4.8.1 Windowing & event-time semantics
- `window!` macro and `Window` node support `tumbling`, `sliding`, and `session` windows. Authors declare `time_source: EventTime|IngestTime|ProcessingTime`, `allowed_lateness`, and `watermark` strategy.
- Watermarks propagate alongside data; late events beyond `allowed_lateness` route to side outputs or trigger policy-defined reprocessing. Defaults: EventTime with watermark = max(event_ts) - lateness.
- Determinism: EventTime windows with bounded lateness qualify for `Stable`; ProcessingTime windows are `BestEffort`.
- Temporal lowering schedules timers for window close and watermark advancement; late arrivals dispatch child activities annotated with compensation metadata.

---

## 5. Flow IR Specification

### 5.1 Core structures
```rust
struct FlowIR {
    id: FlowId,
    name: String,
    version: SemVer,
    profile: Profile,
    nodes: Vec<NodeIR>,
    edges: Vec<EdgeIR>,
    checkpoints: Vec<CheckpointIR>,
    policies: FlowPolicies,
    metadata: FlowMetadata,
}

struct EdgeIR {
    from: NodeId,
    to: NodeId,
    ordering: Ordering,
    delivery: Delivery,
    buffer: BufferPolicy,
    partition_key: Option<KeyExpr>,
    timeout: Option<Duration>,
}

struct NodeIR {
    id: NodeId,
    name: String,
    kind: NodeKind,
    in_schema: SchemaRef,
    out_schema: SchemaRef,
    effects: Effects,
    determinism: Determinism,
    schedule: ScheduleHints,
    retry_owner: RetryOwner,
    idem: IdempotencySpec,
    idem_key: IdempotencyKeySpec,
    state_model: StateModel,
    cost: CostEstimate,
    cache: Option<CacheSpec>,
    compensate: Option<CompensationSpec>,
    params: JsonValue,
    capabilities: Vec<CapabilityRef>,
    docs: NodeDocs,
}

struct CacheSpec {
    key: KeyExpr,
    tier: CacheTier,
    ttl: Option<Duration>,
    invalidate_on: Vec<CacheInvalidate>,
}

struct IdempotencyKeySpec {
    encoding: KeyEncoding,
    hash: HashAlgorithm,
    fields: Vec<KeyField>,
}
```

### 5.2 Type glossary
- `Ordering`: FIFO per `(edge, partition_key)` when `Ordered` (default). `Unordered` unlocks parallel merges; downstream nodes must document emitted order.
- `Delivery`: `AtLeastOnce` (platform default), `AtMostOnce` (only for idempotent cache lookups), `ExactlyOnce` (requires bound `DedupeStore`, idempotent sink contract, and certification proof).
- `BufferPolicy`: explicit queue budget (`max_items`, `spill_threshold_bytes`, `spill: BlobTier`, `on_drop`). Profiles provide maxima; flows can request tighter bounds.
- `ScheduleHints`: `batch: BatchPolicy`, `concurrency: NonZeroU16`, `timeout: Duration` surfaced from macros (`timeout!`) or profile defaults.
- `RetryOwner`: `Connector`, `Orchestrator`, or `None`; determines which layer owns retry semantics.
- `IdempotencySpec`: `key: KeyExpr`, `scope: IdemScope` (Node | Edge | Partition). Keys must reference deterministic fields only; lints flag unstable expressions.
- `IdempotencyKeySpec`: canonical description of key encoding/hashing and deterministic field selection; see §5.2.1.
- `StateModel`: `None`, `Ephemeral`, or `Durable { consistency: Local | Global, ttl: Option<Duration>, version: SchemaVersion }`. Durable nodes require migration hooks (§5.5).
- `CompensationSpec`: maps to a compensating node/subflow plus timeout budget for SAGA unwinds.
- `CostEstimate`: static or learned estimator `Cost { cpu_ms, mem_mb, storage_bytes, tokens, currency }` with confidence interval.
- `CacheSpec`: optional cache description for Strict/Stable nodes including deterministic key, tier, TTL, and invalidation triggers.
- `CacheTier`: enumerates storage tiers (`Memory`, `Disk`, `Remote(region)`); policy may restrict tiers per profile.
- `CacheInvalidate`: `SchemaBump`, `EgressChange`, `CapabilityUpgrade`, `PolicyRule(Name)`.
- Stable caches require pinned resource identifiers (`PinnedUrl`, `VersionId`, `HashRef`). Missing pins downgrade determinism and trigger `CACHE002`.
- Cache entries record provenance (schema version, capability version, egress hash); mismatches invalidate entries automatically to prevent poisoning.
- Cold-start behavior defaults to singleflight recompute to avoid thundering herd; policy can dictate prewarm fixtures.
- `IdempotencyKeySpec`: references encoding/hash algorithms (`KeyEncoding::CborCanonical`, `HashAlgorithm::Blake3`) and deterministic field list.
- `KeyEncoding`: `CborCanonical` (default) with future extension hooks; `HashAlgorithm`: `Blake3` (default) with upgrade policy requiring registry approval.

### 5.2.1 Canonical idempotency keys
- **Encoding**: keys use canonical CBOR encoding of a map `{ schema: SchemaId, node: NodeId, partition: Option<Bytes>, fields: BTreeMap<String, Value> }`. Map keys are lexicographically sorted; integers encoded as CBOR varints; strings normalized to NFC.
- **Hashing**: the CBOR bytes are hashed with BLAKE3 (256-bit). Hash collisions emit telemetry `IDEM099 CollisionSuspected` and abort execution.
- **Field selection**: `fields` may only reference deterministic inputs (Strict/Stable). Validator `IDEM025 UnstableKeyField` fires if BestEffort data participates.
- **Scope composition**: for `IdemScope::Partition`, the partition hash is appended prior to hashing. Edge scope introduces upstream edge id in the CBOR map.
- **Test vectors**: repository ships three fixtures (JSON + expected BLAKE3 digest) to keep implementations consistent across languages.
- **Collision policy**: hashes are treated as opaque identifiers; collision detection triggers fail-close behavior with operator alerting.

### 5.3 Derived metadata
- Aggregated effects/determinism (for entire flow).
- List of required credentials/secrets.
- Egress domains & scopes.
- Idempotency coverage (which nodes have keys).
- Deterministic segments (for caching/replay) and cache eligibility.
- Partition topology and shard fan-out.
- State surfaces with version & migration status.
- Budget footprint per node (cost + memory).
- Retry ownership manifest (ensures no double retries).
- Cache registry (nodes with CacheSpec + tier usage).
- Dedupe store bindings per delivery edge.
- Capability version requirements per node/capability binding.

### 5.4 Semantics & validation pipeline
1. Schema checks (JsonSchema existence + version compatibility).
2. Graph checks (acyclic, reachable nodes, terminal sinks).
3. Capability resolution (declared vs available).
4. Effects/determinism propagation (subflow becomes lattice extrema of children).
5. Idempotency validation: ensure `idem.key` uses deterministic fields; fail effectful nodes with missing keys; enforce `IDEM020 MissingPartitionOrIdemKey` for effectful sinks without partition key or explicit dedupe.
6. Retry ownership enforcement: `retry_owner` must be singular; emit `RETRY010 BothSidesEnabled` if connector and orchestrator retries collide.
7. Exactly-once guardrails: `Delivery::ExactlyOnce` requires DedupeStore binding plus node shape compatibility; otherwise `EXACT001 NoDedupeStoreBinding`/`EXACT005 NonIdempotentSink` fire.
8. Cache validation: Strict nodes without CacheSpec trigger `CACHE001 StrictWithoutCachePolicy` (warn); Stable nodes without pinned inputs or CacheSpec fail (`CACHE002 StableWithoutPin`).
9. Compensation readiness: ensure effectful nodes with `CompensationSpec` have schema-compatible compensators; raise `SAGA201 IncompatibleCompensatorPort` otherwise.
10. State validation: durable state requires registered store capability, declared schema version, migration hooks, and read/write isolation policy per §5.6.
11. Buffer/backpressure policy: enforce `BufferPolicy` budgets, spill settings, and drop strategies permitted by tenant policy.
12. Cost & budget policy: compare `CostEstimate` against workflow/org budgets; fail or warn according to policy engine.
13. Data classification: schemas for ports/params must include field-level tags; regulated profiles treat missing tags as `DATA101 MissingClassification` (error).
14. Policy rules (rate limits, residency, data classification, retry owners, egress intersection `EGRESS101`).
15. Export steps (fail if exporter missing).

### 5.5 State, ordering, and partition semantics
- **State**: scope is `(node, partition_key, run_id)` for `Local`; `Global` lifts to tenant-level consistent state with compare-and-swap. Every durable node declares forward/backward migrations and hitless rollout strategy.
- **State isolation**: `Durable { consistency: Local }` provides per-partition read/write isolation—partitions cannot observe each other's writes within the same run. `Global` state opts into shared visibility with optimistic locking and retry.
- **Ordering**: edges marked `Ordered` preserve submission order per partition. `Unordered` edges allow reordering; merge nodes must expose resulting order semantics in manifests so downstream linting understands stability.
- **Partitioning/sharding**: `partition_by!` populates `EdgeIR.partition_key`. Validation ensures downstream concurrency ≥ shard count or schedules child workflows. Temporal lowering allocates child workflow IDs keyed by partition value with automatic Continue-As-New thresholds.
- **Idempotency keys**: stored alongside execution receipts; registry cert ensures keys are stable under replays. `Partition` scope appends partition hash; `Edge` scope includes upstream edge ID for dedupe across fan-ins. Effectful sinks with `Delivery::AtLeastOnce` must declare either a `partition_key` or explicit dedupe key; missing both triggers `IDEM020 MissingPartitionOrIdemKey`.
- **Edge semantics**: `Delivery` × `BufferPolicy` determines retry + drop behavior. Spills write to configured blob tier with manifest pointer; OOM escalations escalate to policy engine before killing workers.

#### Dedupe store
- **DedupeStore capability.** A durable, idempotent key store with operations `put_if_absent(key, ttl) -> bool` and `forget(key)`. Keys are scoped by `IdemScope` and derive solely from deterministic fields. The capability declares durability, retention, and namespace isolation per tenant.
- `Delivery = ExactlyOnce` is forbidden unless a DedupeStore is bound at compile/deploy time and the node exposes an idempotent sink signature; otherwise validator `EXACT001 NoDedupeStoreBinding` fails.
- Dedupe keys inherit the partition hash when scope ≥ `Partition`; TTL defaults to the longer of edge buffer retention or policy minimum to prevent replays after spill.
- **Sink contract**: exactly-once sinks must expose `upsert(key, payload)` or `ensure_once(key, action)` semantics where reapplication with the same key causes the same observable state. Validators flag `EXACT005 NonIdempotentSink` when connectors lack a documented contract.
- **Cross-domain boundaries**: exactly-once is scoped to the sink’s consistency domain. Side-effects spanning multiple systems (DB + email) require independent idempotency keys per domain combined with SAGA compensation.
- **Outbox pattern**: recommended architecture persists events to an outbox table (idempotent insert keyed by hash) and projects to external systems via dedicated workers. The RFC ships SQL + API examples in Appendix G.
- **Certification harness**: registry duplicates deliveries in bursts, with delayed replays and time-skewed duplicates, asserting a single committed side effect per key. Harness logs proof artifacts for audit.

#### Cache semantics
- `CacheSpec` describes deterministic caches for Strict/Stable nodes: key derivation, storage tier, TTL, and invalidation triggers (schema bump, egress change, capability version upgrade, policy toggle).
- Registry policies may require caches for Stable connectors (`require_cache_for_stable` flag). Strict nodes without caches raise `CACHE001` (warn); Stable nodes missing pinned resources or caches fail `CACHE002`.
- Cache poisoning rules: caches are invalidated automatically when Flow IR schema versions change, when egress allowlists update, or when capability versions change; policy can inject custom invalidators.

### 5.6 Schema versioning & migrations
- `SchemaRef` includes `{ id, version }`; params/ports advance via semver. Minor bumps require compatibility proof; major bumps require Studio guardrail approval (§13.1) and migration hooks.
- Durable state registers `migrate(from: SchemaVersion, to: SchemaVersion)` + `validate(snapshot)` functions. Certification replays fixtures through migrations; missing hooks block publish.
- Flow manifests declare compatibility (`consumes: { upstream.flow: ">=1.2 <2.0" }`). Importer emits migration TODOs when incoming flows mismatch declared ranges.

### 5.7 Schema data classification
- JsonSchema definitions for ports and params include `x-data-class` annotations per field (`public`, `pii.email`, `financial.card`, etc.). Schemas lacking annotations fail validation in regulated profiles (`DATA101`).
- Classification matrix feeds policy checks, logging redaction, and egress enforcement; Studio provides lint autofix suggestions when fields inherit from known types.

---

## 6. Kernel Architecture

### 6.1 Kernel responsibilities
- Manage scheduling, backpressure, retries, checkpoints, and metrics for a Flow IR instance.
- Provide `ResourceAccess` object to nodes with capability gating.
- Bridge host adapters to runtime (Tokio, queue, Temporal).
- Interface with plugin runtimes (WASM, Python, gRPC).
- Enforce determinism and effect policies at runtime (fail fast if violation detected).

### 6.2 Traits
```rust
#[async_trait]
pub trait Node {
    type In: Port;
    type Out: Port;
    type Params: DeserializeOwned + JsonSchema + Clone + Send + Sync;
    type Resources: ResourceAccess + Send + Sync;
    async fn run(&self, input: NodeStream<Self::In>, ctx: &mut Ctx<Self::Params, Self::Resources>) -> NodeResult<NodeStream<Self::Out>>;
}

#[async_trait]
pub trait Trigger {
    type Out: Port;
    type Params: DeserializeOwned + JsonSchema + Clone + Send + Sync;
    type Resources: ResourceAccess + Send + Sync;
    async fn start(&self, ctx: &mut Ctx<Self::Params, Self::Resources>) -> NodeResult<NodeStream<Self::Out>>;
}
```

### 6.3 ResourceAccess, capabilities, facets
```rust
pub trait Capability: Send + Sync + 'static {
    const NAME: &'static str;
}

pub trait ResourceAccess {
    fn http_read(&self) -> Option<&dyn HttpRead>;
    fn http_write(&self) -> Option<&dyn HttpWrite>;
    fn kv_read(&self) -> Option<&dyn KvRead>;
    fn kv_write(&self) -> Option<&dyn KvWrite>;
    fn blob_read(&self) -> Option<&dyn BlobRead>;
    fn blob_write(&self) -> Option<&dyn BlobWrite>;
    fn session_store(&self) -> Option<&dyn SessionStore>;
    fn db(&self) -> Option<&dyn DbPool>;
    fn secrets(&self) -> Option<&dyn SecretsProvider>;
    fn rate_limiter(&self) -> Option<&dyn RateLimiter>;
    fn dedupe_store(&self) -> Option<&dyn DedupeStore>;
    fn cache(&self) -> Option<&dyn Cache>;
    fn facet<F: Facet>(&self) -> Option<&F>;
}

pub trait Facet: Send + Sync + 'static {
    const NAME: &'static str;
}

pub struct RequestScope { pub session_id: Option<String>, pub ip: Option<IpAddr>, pub user_agent: Option<String> }

pub trait DedupeStore: Capability {
    fn put_if_absent(&self, key: &[u8], ttl: Duration) -> Result<bool, DedupeError>;
    fn forget(&self, key: &[u8]) -> Result<(), DedupeError>;
}

pub trait Cache: Capability {
    fn get(&self, key: &[u8]) -> Result<Option<Bytes>, CacheError>;
    fn set(&self, key: &[u8], val: &[u8], ttl: Option<Duration>) -> Result<(), CacheError>;
}
```

- `DedupeStore` implementers publish durability guarantees (multi-AZ, journaling) and purge schedules; execution runtimes reject `Delivery::ExactlyOnce` if the capability is absent.
- `Cache` implementers expose tier and eviction policy metadata so policy can enforce where Stable/Strict caches live.
- Facets are immutable snapshots of request context; mutations must flow through explicit capabilities to avoid hidden side-effects.

#### 6.3.1 Capability versioning & compatibility
- Capabilities advertise `(name, semver)`; Flow IR records required ranges on each node binding. Hosts must provide compatible versions at deploy time; otherwise validator `CAPV101 IncompatibleCapabilityVersion` fires.
- Registry stores capability compatibility matrices; upgrades triggering major version bumps require certification reruns for dependent nodes.
- Policy gates can block automatic upgrades or enforce staged rollouts with per-tenant overrides.

### 6.4 Host adapters
- **Tokio engine:** In-process scheduler, fluent for dev and unit tests.
- **Queue/Redis:** HTTP/webhook front-end enqueues items; workers use `ActivityRuntime` to run nodes, ensure idempotency, ack results.
- **Axum adapter:** Binds triggers to HTTP routes; injects `RequestScope`; handles `Respond` bridging with oneshot channels; allows session/middleware outside kernel.
- **WASM host:** Executes WIT-defined nodes; capabilities exposed via WIT handles; strict allowlist (no network unless declared).
- **Python/gRPC host:** `plugin-python` crate starts gRPC subprocess; nodes call remote functions (with schema handshake).
- **Temporal adapter:** Converts Flow IR pipelines to Temporal workflows (Go/TS) plus activities; we run a Rust activity worker using our Node runtime.

#### 6.4.1 Web streaming & cancellation
- Web profile supports streaming responses via Server-Sent Events (SSE), chunked HTTP/1.1 transfer, and WebSocket bridging. `Respond(stream = true)` wires edge backpressure to HTTP body writes with 64 KiB default chunks.
- Client disconnects propagate a cancellation token; nodes observe `ctx.is_cancelled()` and should cease work promptly. Cancel events emit `WEB201 ClientDisconnected` telemetry.
- Multipart uploads/downloads stream directly to BlobStore (chunk size ≤ 8 MiB, configurable). Determinism downgrades to `BestEffort` unless payloads are hashed and referenced via `HashRef`.

### 6.5 Scheduling & backpressure
- Bounded MPSC channels per edge obey `EdgeIR.buffer` (`max_items`, `spill_threshold_bytes`, policy for head/tail drops). Defaults come from profile policies; flows may only tighten.
- Token-bucket rate limiters per node/workflow ingest `rate_limit!` + policy overlays; saturation raises `RUN030 BackpressureStall` with recommended scale actions.
- Concurrency per node derives from `ScheduleHints.concurrency`; runtime enforces via cooperative semaphores rather than hidden defaults.
- `timeout!` macros translate to watchdog timers; expirations trigger policy-configured actions (retry vs dead-letter) and surface metrics.
- Timeout overruns emit `TIME015 OverBudget` events, propagated to Temporal (Activity `TimeoutType`) and queue engine for policy handling.
- Spill behavior: when `spill_threshold_bytes` reached, items persist to blob tier defined by policy; runtime records `RUN040 BufferSpill` events with size metadata.
- Buffer overflow actions (`Backpressure`, `DropOldest`, `DeadLetter`) require explicit policy approval; otherwise validation fails.
- Exactly-once edges consult `ResourceAccess::dedupe_store()` prior to dispatch; absence switches delivery back to AtLeastOnce with warning unless validation already failed.

### 6.6 Memory budgets & OOM handling
- Per-node soft/hard memory budgets derive from `CostEstimate.mem_mb` + tenant policy. Soft limit induces scheduling stall; hard limit causes deterministic fail with `RUN050 OOM` and triggers compensation if configured.
- Global worker memory budget enforces fair-share across flows; runtime samples heap usage and spills cold partitions first.
- Spill tiers (`EdgeIR.buffer.spill`) map to configured blob stores (local disk, S3). Policies declare allowable latency/retention.
- Default spill threshold: 64 MiB per edge with backpressure before spill; spill bandwidth capped to prevent disk thrash. Spills use CBOR chunk format with CRC32 per chunk and background scrubbers respecting TTL.
- Resume path validated by tests ensuring checksum integrity and deterministic replay after spill recovery.

### 6.7 Cancellation & shutdown semantics
- Graceful shutdown sends cooperative cancel to active nodes; nodes receive `Ctx::is_cancelled` flag and have bounded grace period equal to `ScheduleHints.timeout`.
- On SIGINT/SIGTERM the kernel drains in-flight edges, checkpoints durable state, and records resume tokens. Temporal adapter propagates cancellation into workflow execution, aligning with signal semantics.
- Restart or resume resumes from last committed checkpoint/state; effectful nodes without compensation are retried per idempotency guarantees.

### 6.8 Fairness & partition scheduling
- Scheduler uses weighted-fair queues keyed by `(partition_key, priority)` to avoid starvation. Hot partitions are throttled once they exceed configured `max_parallel_shards`.
- Cooperative yields after configurable batch size ensure long-running partitions do not monopolize executor threads.
- Runtime surfaces fairness metrics (`fairness.skew_ratio`) for observability.
- Default shard cap: 64 concurrent partitions per worker; additional shards queue until capacity frees. Credit-based flow control coordinates across workers via Control-plane signals to prevent overload.

### 6.9 Cost estimation hooks
- Each capability implements `estimate_cost(inputs) -> CostEstimate` returning CPU, memory, storage, and token projections along with unit basis (`PerItem`, `PerBatch(size)`, `PerTokens(1000)`) and confidence interval.
- Nodes can override via procedural macros; estimates feed into admission control, per-run budgets, and Studio’s planner (§13.1). Policies may require explicit budgets before deployment.
- Runtime records actual cost vs estimated, computes rolling MAPE, and emits `COST101 UnderestimatedByX` warnings when drift exceeds policy threshold. Logs persist estimated vs actual values for calibration.

### 6.10 HITL checkpoints
- `hitl!` inserts `CheckpointIR` with label, view metadata, TTL, escalation policy.
- Runtime pauses at checkpoint, persists state; host exposes `resume(checkpoint_id, payload)` API.
- Temporal: translated to Signal wait; queue engine: block until resume message.
- Audit logs record actor actions.

### 6.11 Kernel SLOs
- Scheduler latency p99 < 50 ms under design load; alerts fire (`SLO101 SchedulerLag`) when breached for 3 consecutive windows.
- Spill latency p95 < 500 ms; longer spills escalate to autoscaling hints and policy review.
- Checkpoint resume latency p95 < 5 s (queue) / < 30 s (Temporal); HITL dashboards surface real-time drift.
---

## 7. Execution vs Orchestration

### 7.1 Role separation
- **Execution:** Node runtime, plugin ABI, capability enforcement, streaming, backpressure, idempotency.
- **Orchestration:** Sequencing, timers, fan-out/fan-in, human signals, durable state.

### 7.2 Normative placement rules

| Node Effects  | Determinism       | Placement          | Temporal Mapping                                  |
| ------------- | ----------------- | ------------------ | ------------------------------------------------- |
| Pure          | Strict            | Inline in workflow | Workflow code (no Activities); cached for replay  |
| Pure/ReadOnly | Stable/BestEffort | Activity           | Activity with retry/backoff & dedupe              |
| Effectful     | Any               | Activity           | Activity; requires idempotency + compensation opt |

- `Wait`/`Delay` nodes become Temporal timers; inline runtime uses kernel timers driven by the same `timeout!` metadata.
- Checkpoints are Temporal signals with explicit payload schemas and TTL; queue host exposes equivalent resume endpoints.
- Fan-out/fan-in: edges with `partition_key` spawn child workflows named by `(flow_id, partition)`; without partitioning we stay in the parent workflow but enforce `max_concurrency` on Activities.
- Streams lower to batches per `ScheduleHints.batch`; history growth triggers `ContinueAsNew` once thresholds in placement policy are crossed (default < 2k events/3h per profile).
- Retry ownership is singular: either Activity retries via Temporal options **or** connector capability performs retries (declared in manifest). Validation at compile time blocks double retries.
- When `retry_owner = Connector`, Temporal Activities disable SDK retries (`ScheduleToClose` single attempt) and rely on connector-level retry + DedupeStore; queue engine mirrors by acking only after connector signals terminal state.
- Implementation in Go/TS persists; Rust worker handles Activities using kernel runtime; pinned for determinism equivalence.
- Guardrail `WFLOW010 NonDeterministicInline`: lowering fails if any inline unit is not `(Pure, Strict)`; authors must wrap I/O or nondeterminism in Activities.
- History budgets per profile (events/time) are computed from `ScheduleHints`; manifest exports the expected history footprint so planners can flag risks pre-deploy. Codegen injects proactive Continue-As-New once projections breach the budget.
- Windowed nodes translate watermarks to Temporal timers; `allowed_lateness` spawns child workflows handling late data with compensation when necessary.
- Default budgets: 2,000 history events or 3 hours wall-clock, whichever first; profiles may tighten. Workflow manifests expose `history_budget` metadata for planners.
- Child workflow IDs follow `wf::<flow_id>::p::<partition>::v::<semver>::r::<run_epoch>` to guarantee uniqueness and traceability across reruns.
- Timers add 5–10% jitter to avoid stampedes on resume; jitter policy configurable per profile.

#### 7.2.1 Retry ownership matrix

| Capability family          | Default owner   | Notes |
| -------------------------- | ----------------| ----- |
| Generic HTTP (429/5xx)     | Orchestrator    | Uses Temporal/queue retries with exponential backoff. Connectors disable internal retries (`RETRY011` if overridden without waiver). |
| Payment gateways / Stripe  | Connector       | Requires upstream idempotency keys; orchestrator performs single attempt (`ScheduleToClose`). |
| Databases / KV             | Connector       | Connector applies limited retries on deadlocks/timeouts; orchestrator only retries on classified `Transient`. |
| Blob/object storage        | Orchestrator    | Handles network/transient errors; connectors must be idempotent. |
| LLM/AI APIs                | Orchestrator    | Retries gated by budget policy; connectors record cost even on failure. |

- Validators: `RETRY011 DisallowedOwner` triggers if declared owner differs from default without policy waiver; connectors owning retries must supply idempotency keys.
- Matrix is versioned in registry metadata so policy can evolve per capability release.
### 7.3 Local & queue modes
- Local dev (Tokio) uses same ExecPlan but runs inline with async tasks.
- Queue mode uses Redis for durable event queue; ensures at-least-once semantics with idempotency checks.
- Profiles adjust channel sizes, concurrency, and default retries.
- Queue engine enforces visibility timeout aligned with `ScheduleHints.timeout`; redelivery window equals `timeout * retry_multiplier`. Workers ack after post-idempotency persistence; duplicates trigger DedupeStore lookups before re-execution.
- Idempotency key TTL in the queue store defaults to visibility timeout × max retries, ensuring duplicates arriving after spill remain suppressed.
- Web profile enforces default request deadlines (e.g., 25s) and maps `RespondPolicy` to HTTP status: timed-out responses return 504, while explicit `on_last` responses can emit 202 + callback payload hooks.
- SSE/chunked/WebSocket streaming adopt the same deadlines but allow partial responses; cancellation triggers graceful shutdown of upstream nodes.

---

## 8. Effects & Determinism Enforcement

### 8.1 Effects lattice
```
Pure < ReadOnly < Effectful
```
- Derived from capabilities: presence of any write capability implies Effectful.
- Pure nodes cannot access ResourceAccess at all (ctx.resources.* returns none). Lints for hidden effects (e.g., `spawn_blocking` with I/O).
- ReadOnly nodes get only read capabilities; compile fails if they call write methods.

### 8.2 Determinism lattice
```
Strict < Stable < BestEffort < Nondeterministic
```
- Strict: deterministic, no time, no randomness, no unpinned external reads. Typically inline pure functions or content-addressed reads.
- Stable: reads pinned by version/etag/hash (`PinnedUrl`, `HashRef`, `VersionId` newtypes). Outputs remain identical if upstream resource unchanged.
- BestEffort: may vary (HTTP GET without pin, asynchronous data, network). Default.
- Nondeterministic: explicit randomness or time-critical data (rare).
- Randomness: `Stable` nodes require a seeded RNG passed via input or environment; otherwise determinism drops to `BestEffort` and lint fires.
- Clock: any `Clock` usage forces downgrade to `BestEffort` unless the timestamp is provided as explicit input (e.g., event timestamp) and marked `Pinned`.

### 8.3 Enforcement mechanisms
- Compile-time: macros derive defaults; authors can tighten if compliant. Lints detect contradictions (Clock/Rng without seed, unpinned HTTP) and emit DET30x errors. Missing idempotency on effectful nodes fails DAG004.
- Build-time: capability typestates reject misuse (e.g., calling `HttpRead::get` inside a `Strict` node). The compiler prevents linking ReadOnly nodes against write traits.
- Registry-time: certification executes record/replay—`Strict` must be byte-identical, `Stable` must emit identical logical results with pinned references (hash recorded). Failures block publish.
- Runtime: policy engine enforces tenant rules (e.g., forbid `BestEffort` in financial orgs) and can require HITL or caching for downgraded segments.
- Exactly-once claims rely on DedupeStore guarantees; runtime refuses to start nodes marked `Delivery::ExactlyOnce` without the capability binding.

### 8.4 Resource API typestates
- `BlobRead::get_by_hash` (Strict), `BlobRead::get_by_key` (BestEffort).
- `HttpReadExt::get_pinned(PinnedUrl)` (Stable) vs `get(Url)` (BestEffort).
- Capability types enforce compile-time semantics; using a non-compliant method fails build.
- `VersionedStore::load(VersionId)` vs `load_latest()` ensures durable state reads maintain declared determinism.
- `Rng::seeded(seed)` required for Stable randomness; `Rng::default()` banned in Strict/Stable contexts.

### 8.5 Compensation & SAGA semantics
- Effectful nodes MAY declare `CompensationSpec` referencing a compensating node/subflow. When downstream failure occurs after an effect, compensation executes in reverse topological order with the original `partition_key` and idempotency key.
- Compensation nodes inherit `effects: Effectful`, `determinism: BestEffort` (unless proven otherwise) and must expose idempotency keys to avoid double-undo bugs.
- Temporal lowering emits a compensation chain (child workflow or activity sequence) and records completion in history; queue engine persists compensation log to durable store.
- Policies can require compensation for specific capabilities (e.g., payments) via registry tags.
- `SAGA201` ensures compensators accept the effect node's emitted schema (or a declared projection); incompatible ports block publish.
- Compensators must define an `undo_key` using the canonical idempotency encoding to guarantee idempotent rollbacks. Validator `SAGA205 NonIdempotentCompensator` fires if absent.
- Partial unwind policy: on compensator failure, runtime retries per `retry_owner`; if irrecoverable, policy chooses between halt, manual intervention (HITL), or forward-only waiver. Registry records this choice in certification evidence.

### 8.6 Determinism proofs & registry metadata
- Nodes claiming `Strict`/`Stable` publish replay evidence (hash of inputs/outputs, pinned references) in registry metadata. Certification compares evidence on upgrade.
- Registry stores provenance of pinned resources (etag/hash). If evidence missing or stale, publish is rejected with actionable diagnostics.
- Agents can query registry for determinism proofs before reusing connectors in new flows.

---

## 9. Plugin Architecture

### 9.1 Native Rust
- Highest performance; nodes compiled into binary.
- Node registry holds constructors; macros register nodes globally.

### 9.2 WASM component model
- WIT interface `flow:node` with streaming I/O.
- Host exposes capabilities via WIT imports; only allow declared capabilities.
- Workload isolation for third-party nodes; run under Wasmtime with resource limits.
- ABI versioning negotiated via WIT world version; registry tracks compatibility matrix (M:N) and rejects plugins targeting unsupported ABI.
- Binary streaming: max chunk size 256 KiB per frame, backpressure propagated via async streams; large payloads spill to configured blob tier using presigned URLs with policy-controlled expiry.
- Offline cache writeback: WASM nodes accumulating Strict/Stable caches store entries locally with vector clocks; reconciliation merges entries via canonical idempotency keys and capability version checks to maintain determinism.

### 9.3 Python/gRPC
- gRPC service interface `Run(stream<Item>) -> stream<Item>`.
- Schema handshake ensures JSON Schema compatibility.
- Useful for DS workloads; run out-of-proc with network sandbox (iptables/egress filter).
- Sandbox policies: seccomp/AppArmor profile, CEL-configured egress policy, per-invocation CPU/memory caps, and wall-clock timeouts derived from `timeout!` metadata.

### 9.4 Packaging & discovery
- Registry entries record plugin type and runtime requirements.
- Wasm nodes published as `.wasm` + manifest; Python nodes packaged with metadata and container image references.
- SBOM (SPDX) and signature (Sigstore/cosign) artifacts accompany every publish (see §10).

### 9.5 Hot reload
- Dev profile supports hot reload for Rust & WASM nodes via versioned dynamic registries; production requires rolling worker restart unless node flagged `hot_reload_safe` through certification.
- Kernel validates compatibility before swapping implementations; stateful nodes with `Durable` state cannot reload without migration step.
- `hot_reload_safe` criteria: no durable state schema changes, ABI unchanged, effects ≤ ReadOnly, deterministic claims preserved. Certification emits waiver artifacts for review.

---

## 10. Registry & Certification

### 10.1 Artifacts stored
- `NodeSpec`, `SubflowSpec`, `WorkflowSpec` json files.
- Schema definitions (JsonSchema files for ports/params).
- Metadata: effects, determinism, capabilities, egress, scopes, rate limits, idempotency strategy, docs.
- Tests: contract fixtures, determinism replay results, coverage metrics.
- Signatures/hashes for integrity (Sigstore/cosign) plus SBOMs (SPDX) per artifact.
- Cost estimator manifests per node (declared cost model + variance bounds).
- Data classification tags for every schema field (`public`, `pii.email`, `financial.card`, etc.).
- Capability compatibility tables (supported semver ranges, upgrade notes).

### 10.2 Certification checks
1. Metadata completeness (no default placeholders).
2. Effects vs capabilities (static finish).
3. Determinism: run record/replay; fail on mismatch. Strict requires binary equality; Stable allows limited drift with pinned references.
4. Contract tests: happy path + error cases (401, 429, 5xx) for network connectors.
5. Security: check egress domains, scopes, no inline secrets.
6. Idempotency presence for effectful connectors.
7. Rate-limit hints validated (if missing, warn/fail based on policy).
8. Vulnerability scanning on dependency tree; critical CVEs block publish unless explicitly waived with expiration.
9. Data classification enforcement: flows sending PII to connectors lacking approved residency/egress policies are rejected.
10. Budget gate: cost estimator compared against declared policy budget; high-variance nodes require HITL approval tag.
11. Validator codes enforced: `RETRY010`, `EXACT001`, `CACHE001/2`, `DATA101`, `EGRESS101`, `SECR201`, `SAGA201` surface as hard failures with remediation guidance.
12. Capability compatibility: ensure hosts satisfy capability version ranges; `CAPV101` fails otherwise.
13. PII-in-logs scan ensures telemetry/exporters redact non-public data-class fields; failures emit actionable diagnostics before publish.

### 10.3 Publishing pipeline
- `flows publish connector path/to/spec.yaml` triggers codegen + tests + metadata.
- On success, manifest uploaded to registry bucket; CLI fetches signed versions.
- Deprecation process: mark versions as deprecated with replacement instructions; enforced via policy (lint in flows using deprecated nodes).
- Multi-region residency tags propagate from manifest; deployment blocks if runtime region not allowlisted.
- Production deploys require cosign signatures and SBOMs with no critical CVEs (or approved waivers with expiry); CI gates fail otherwise.

---

## 11. Connector Library Strategy

### 11.1 Repo organization
```
crates/
  connectors-std/
  connectors/<provider>/
  packs/pack-<domain>/
  types-common/
  adapters/
```

### 11.2 ConnectorSpec -> codegen
- YAML defines params, ports, effects, determinism, capabilities, egress, rate limits, tests.
- Schemas declare `x-data-class` annotations per field; generator refuses to emit code otherwise.
- Retry ownership (`retry_owner: connector|orchestrator|none`) declared explicitly; generator ensures connectors disable internal retries when orchestrator owns them.
- Generator outputs: node struct, params struct, manifest, tests scaffolding.
- Agents produce YAML; humans review; CI runs generator & tests.

### 11.3 Capability adapters
- Wrap third-party SDKs behind capability traits (`OpenAIClient`, `StripeClient`, etc.) enforcing budgets, policies, and mapping errors.
- Example: `OpenAIRead` (completions, embeddings) vs `OpenAIWrite` (fine-tunes, file uploads). Provide deterministic options (seed, pinned model). Budget tracking built into capability (token counts, cost). 
- LLM adapters must support preflight token estimation, `dry_run` mode, and explicit budget ceilings; sampling downgrades determinism to `BestEffort` with policy guardrails.

### 11.4 Testing & maintenance
- VCR-style HTTP recordings for connectors (no live data in tests). Budgeted tests for error cases.
- Determinism tests with pinned seeds/versions.
- Registry rejects connectors lacking tests or metadata.
- Monitoring: nightly canary tests hitting provider sandboxes to detect API drift.
- Standardized `NodeError` taxonomy (`Auth`, `RateLimited`, `Transient`, `Permanent`, `SchemaMismatch`, `PolicyDenied`) powers consistent retries and Studio surfacing.
- Retry ownership flag (`retry_owner: Connector|Orchestrator`) ensures only one layer performs retries; validators block double retry loops.

---

## 12. Importer & Corpus Harvesting

### 12.1 Corpus (Zie619/n8n-workflows)
- ~2,053 workflow JSON files with scripts for indexing. Repo history includes DMCA-related rewrite; treat as read-only metadata.
- Build harvester crate: parse JSON to capture metadata (node types, connectors, triggers, expressions, complexity). Store derived data only (DuckDB/SQLite). Track file hash/path/provenance.
- Output coverage reports, top connectors, risk tags (code nodes, paired items, binary usage, expressions). No redistribution of raw JSON.

### 12.2 Importer pipeline
1. Parse n8n JSON to intermediate structure (nodes, connections, parameters, expressions).
2. Map each node to canonical connectors/subflows (lookup table). Where missing, emit adapter stub.
3. Convert expressions ($json, $items()) to typed selectors; fallback to dynamic adapter with lint messages.
4. Map paired items to `Item.link` structures; flag differences.
5. Convert binary handles to BlobStore references.
6. Generate Rust flow using macros; run `cargo check`; agent iterates on errors until compile passes.
7. Run stubbed replays (no external network) to ensure structural outputs; note lossy semantics.

### 12.3 Metrics & outputs
- Import success rate (compiling flows).
- Expression fallback count.
- Connectors lacking coverage.
- Determinism/effect classification (imported flows default to typed metadata). 
- Gap backlog for connectors/adapters/expressions.
- Publish gate: imported flows must have ≤10% expression fallbacks (profile configurable), zero effectful sinks without idempotency keys, and full data-class annotations before registry admission.

### 12.4 Legal posture
- Store derived metadata only; respect DMCA; maintain blocklist for removed files.
- Generated Rust flows derived from metadata are safe to distribute; do not publish raw JSON.
- Machine-readable provenance log retains source commit, path, hash, and reason for exclusion (if any).

### 12.5 Expression coverage matrix
- Maintain matrix enumerating expression families (`$json`, `$items()`, `$node`, substring/regex, arithmetic, date/time, ternaries). Each entry tracks typed translation support, fallback to dynamic evaluation, and open gaps.
- Importer lints when falling back to dynamic evaluation; Studio surfaces the gap with remediation suggestions.
- Typed selectors produce schema traces to feed determinism and idempotency analysis.
- Fallback to the dynamic engine downgrades affected edges to `BestEffort` determinism and, in regulated profiles, enforces either HITL checkpoints or cache guards before publish.
- Top fallback patterns surface auto-suggested typed selectors; agents must resolve suggestions or acknowledge policy waivers before publish.

### 12.6 Lossy semantics catalog
- Importer writes per-workflow machine-readable report (`lossy.json`) categorizing losses: `ordering_change`, `expression_fallback`, `binary_semantics`, `trigger_timing`, `determinism_downgrade`, `hitl_required`.
- Policies can reject flows containing prohibited loss categories (e.g., regulated profiles banning `binary_semantics`).
- Registry stores catalog for traceability; Studio surfaces diff before publishing imported flows and flags impacted policies.

---

## 13. Agent Design Studio

### 13.1 Capabilities
- Intent capture (natural language to preliminary flow).
- Q&A (“Product manager” agent asks clarifying questions).
- Skeleton generator (produces Rust macros, connectors selection).
- Compiler loop (run `cargo check`, parse diagnostics, fix code).
- Fixture management (capture inputs/outputs, golden tests).
- Importer runner (convert n8n JSON).
- HITL backend (render checkpoints, accept approvals, resume flows).
- Packaging exporter (WASM, manifests, docs).
- Guarded diffs: when agents change effects/determinism (tighten or loosen) Studio renders a diff, requires human approval for downgrades (e.g., Stable→BestEffort) and policy-defined upgrades.
- Budget planner: aggregates `CostEstimate` data to show predicted per-run and per-month spend, compare to policy budgets, and suggest throttling strategies.
- Machine-readable diagnostics: compile/runtime errors stream as JSON lines with codes + fix hints so builder agents can auto-remediate.
- Connector templates: agents can request `flows template connector --provider foo` to scaffold spec YAML, node skeleton, and determinism claims.

### 13.2 API sketch
- `POST /intents`: submit textual intent.
- `POST /answers`: respond to follow-up questions.
- `POST /compile`: run compile, return diagnostics.
- `POST /run`: execute flow (local/queue), return run ID.
- `GET /runs/{id}`: fetch run events and metrics.
- `POST /import/n8n`: submit workflow JSON + traces; return Rust flow + notes.
- `POST /hitl/{checkpoint}/resume`: resume checkpoint.
- `POST /publish`: push flow/connector to registry (after certification).

### 13.3 CLI (flows)
```
flows init
flows add node connectors/http
flows add trigger http
flows graph new notion_drive
flows graph check
flows run local
flows queue up
flows export --format wit
flows import n8n path/to/workflow.json
flows hitl approve <checkpoint>
flows publish connector path/to/spec.yaml
flows test
flows tail
flows template connector --provider openai
```

### 13.4 Agent loop examples
- PM agent collects `NOTION_DB`, `GDRIVE_FOLDER`, approval policy.
- Builder agent generates flow, resolves DAG201 errors by inserting adapter nodes or implementing `From`.
- Policy agent verifies no effectful nodes without idempotency; ensures egress allowlisted.
- Test agent records golden outputs and property tests.
- Deploy agent packages flow for queue and Temporal profiles.

---

## 14. Observability & Operations

### 14.1 Events & logs
- Start/end, retries, errors, backpressure, rate-limit, checkpoint wait/resume.
- Structured logs include node ID, run ID, flow version, capability usage.
- Log entries carry data-class tags (PII, financial) to enable log redaction downstream.
- PII tags trigger automatic redaction; non-public fields are masked unless explicit audit policy allows exposure.

### 14.2 Metrics
- Node-level latency/throughput, queue depth, retry counts.
- Workflow-level metrics: active runs, success/failure, approval latency.
- Exposed via OpenTelemetry; can integrate with Prometheus/DataDog.
- Cardinality controls: metric label allowlist prohibits raw IDs; exemplars carry sample run IDs instead.
- Profiles set hard caps (≤100 distinct partition labels / 5 min window by default); overages drop labels and surface WARN `METRIC201 CardinalityExceeded` while emitting exemplars.

### 14.3 Tracing
- Span per node execution; context propagation across edges and connectors.
- Edge-level attributes (buffer size, wait time).
- Adopt W3C Trace Context; include partition key hash, idempotency key, checkpoint labels as span attributes (hashed where necessary).
- Partition-level sampling surfaces exemplar spans for top-N hot shards; exporters must not emit unbounded label sets.
- Error spans annotate standardized taxonomy (`Auth|RateLimited|Transient|Permanent|PolicyDenied|SchemaMismatch`).

### 14.4 Ops tooling
- CLI: `flows tail` for run logs; `flows run --inspect` for local debugging.
- Admin UI: view checkpoints, resume, cancel runs, adjust rate limits, view registry metadata.

### 14.5 Run bookkeeping
- Runtime records per-run summary: total cost (actual vs estimated), API call counts, retries, HITL dwell time, spill events.
- Bookkeeping entries feed into policy audits and Studio dashboards; retention aligns with data residency policy.
- Lineage metadata traces data-class tags through nodes so telemetry can redact non-public fields while preserving derivation evidence for auditors.

---

## 15. Policy & Security

### 15.0.1 Policy evaluation phases
- **Compile-time**: enforce effects/determinism lattice, idempotency presence, capability declarations, egress domains, secret scopes, and schema classifications.
- **Deploy-time**: validate capability version compatibility, region residency, budget declarations, and policy waivers. Deployment produces a policy evidence report stored alongside registry artifacts.
- **Run-time**: monitor per-tenant budgets, live egress decisions, hit dynamic kill-switches, and enforce data residency in adapters. Runtime violations emit policy alerts and can pause offending flows.

### 15.1 Capability allowlists
- Each deployment profile defines allowed capabilities and egress domains.
- Compile-time check: Flow requires capability -> must be provided by host; otherwise fail.
- Runtime enforcement: ResourceAccess denies access if capability not allowed.
- Egress allowlist resolution order: node manifest domains ∩ capability provider allowlist ∩ org policy allowlist. Empty intersections raise `EGRESS101 NoAllowedEgress`.

### 15.2 Secrets management
- Secrets retrieved via providers (Vault/AWS SM/SOPS). `creds!` declares provider + scope.
- Manifests reference secret handles, not values.
- Admission checks call provider introspection APIs to ensure granted scopes ⊆ requested scopes; over-privileged bindings fail with `SECR201` unless an approved waiver exists.
- Runtime scrubs secrets from logs/spans by default and zeroizes buffers on panic to avoid leakage.

### 15.3 HITL audit
- Checkpoint actions logged (actor, timestamp, payload summary).
- Workflows can require human approval before effectful action.

### 15.4 Plugin sandboxing
- WASM nodes run with restricted capabilities (no network unless declared).
- Python/gRPC nodes run in containers with network policies and TLS.

### 15.5 Compliance
- Determinism metadata enables flows to meet audit requirements (only Strict nodes allowed in regulated paths).
- Idempotency enforced; flows lacking keys fail certification.
- Policy engine uses CEL for policy expressions; macros allow attaching snippets at workflow/org scope (e.g., `policy!(workflow, org_policies::restrict_pii)`).
- Data residency tags on capabilities and connectors enforce region allowlists during deployment; violations cause compile-time errors with suggested remediations.
- Data classification tags from registry drive egress checks and redaction rules across logging/metrics.

---

## 16. Testing Strategy

### 16.1 Unit tests
- Macro expansion & diagnostics.
- Capability gating (Ensure Pure nodes can’t call resources).
- Flow IR validation functions.

### 16.2 Property tests
- Merge/Switch invariants (no item loss/dup).
- Idempotency key determinism (hash uniqueness).
- Lineage propagation through subflows.
- Reordering tolerance: edges marked `AtLeastOnce` validated to produce idempotent outputs under shuffled/deduped inputs.

### 16.3 Golden tests
- Notion↔Drive flow (fixtures for Notion pages, Drive responses).
- Marketing site (HTTP GET/POST handling; snapshot HTML).
- Lead pipeline (scoring results).
- DS pipeline (CSV outputs).

### 16.4 Contract tests
- Connectors: VCR recordings for success + error cases.
- Rate limit/backoff behavior.

### 16.5 Importer tests
- Sample n8n workflows -> generated Rust -> compile & replay stubs.
- Expression conversions covering $json, $items(), $node.

### 16.6 Temporal integration tests
- Local Temporal dev cluster; run sample flow with signals, timers, ContinueAsNew, child workflow.
- Validate idempotency & effect enforcement.

### 16.7 Fault injection & chaos
- Network failure, 429, 5xx replayed across retries and compensation paths.
- Worker crashes before/after checkpoint persistence and mid-activity to validate crash/replay guarantee set.
- Temporal history replays with reordered signals and late timers to assert determinism metadata holds.
- HITL timeout and rejection paths to confirm policy escalations.

### 16.8 Reordering & duplication harness
- Harness injects reordered, duplicated, and delayed deliveries on edges flagged `AtLeastOnce`; asserts sinks use idempotency keys correctly.
- Includes Temporal history replays plus queue-engine simulations to ensure consistent semantics across hosts.

### 16.9 Backpressure & memory tests
- Saturate narrow bottleneck nodes to ensure upstream applies stall/backoff instead of silent drops.
- Force `BufferPolicy` spill thresholds and validate blob spill + resume without data loss.
- Simulate OOM conditions to ensure soft limit stalls and hard limit fails with compensation triggers.

### 16.10 CI gating
- `cargo fmt`, `cargo clippy`, `cargo test`, `cargo check` for all flows.
- `flows graph check` on changed workflows.
- `flows publish` runs certification harness.
- Coverage thresholds (connectors, importer tests).

### 16.11 Acceptance criteria
- Compile-time: Pure nodes resolve with zero `ResourceAccess` usage; ReadOnly nodes cannot link write-capability traits (lint enforced).
- Runtime: Effectful activities must emit idempotency key; registry rejects otherwise.
- Determinism: Connectors claiming `Strict`/`Stable` pass replay certification in CI using recorded fixtures.
- Policy: Flows emitting PII can egress only to allowlisted domains; CI blocks violating manifests.
- Idempotency: 100% effectful sinks use canonical idempotency keys; duplicate-delivery harness asserts single side-effect.
- Cache: Strict nodes define CacheSpec; Stable nodes demonstrate pinned reads and ≥95% cache hit rate on replay fixtures.
- Temporal: 99% of runs stay below history budget; child workflow failure isolation verified in integration tests.
- Web: SSE/chunked paths maintain <30 ms p95 backpressure bridge and propagate cancellation (`ctx.is_cancelled()` verified).

### 16.12 Additional certification harnesses
- Exactly-once harness injects duplicate deliveries and asserts DedupeStore prevents double side effects.
- Cache replay suite runs Strict/Stable nodes through hit/miss cycles to confirm byte-identical outputs and TTL enforcement.
- Policy enforcement scenarios cover missing data-class tags, over-privileged secrets, and non-allowlisted egress; CI fails with actionable diagnostics.
- Idempotency key fixtures ensure all language bindings produce identical CBOR+BLAKE3 digests.
- Capability upgrade harness replays connectors against new capability versions to verify compatibility before bumping ranges.

---

## 17. Deployment & Rollout

### 17.1 Phase roadmap
- **Month 0–1:** Macros + kernel core + CLI skeleton + Notion↔Drive example (local). 
- **Month 2:** connectors-std initial set (15–20), harvester data, importer alpha, registry prototype (manual approvals), CLI expansions.
- **Month 3:** Redis queue profile, plugin hosts (WASM, Python), policy engine, CLI `flows queue up` and `flows publish`.
- **Month 4:** Temporal adapter (Go/TS codegen + Rust activity worker), registry automation, Studio backend MVP, importer success ≥70% sample.
- **Month 5:** Connector expansion (50+), importer success ≥85%, determinism enforcement in registry, policy-based gating.
- **Month 6:** GA candidate: docs, internal dogfood, security review, compatibility benchmark vs n8n queue mode.
- **Month 7:** External beta, marketing site built on platform, optional open-source components.
- **Success metrics:** importer compile success ≥70% @ M3, ≥85% @ M5; ≥90% connectors published with explicit effects/determinism (non-default); ≥95% effectful nodes exposing idempotency keys; Temporal history growth maintained below profile threshold; backpressure stalls >N seconds kept under X% of runs.

### 17.2 Deployment profiles
- Dev (local): minimal capabilities, mock connectors.
- Web: Axum adapter, small buffers, explicit `Respond` bridging.
- Queue: Redis queue, rate limits, worker auto scale.
- Temporal: durable orchestration for long-running flows.
- WASM: edge deployment with strictly limited capabilities.

---

## 18. Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Connector coverage lag | Importer success limited | Use corpus data to prioritize; CI gating on new connectors; automation via ConnectorSpec |
| Determinism mislabeling | Cache & replay errors | Certification replay tests; static analyzer for clock/rng usage |
| Temporal complexity | Slower adoption | Keep queue engine as fallback; start with small flows; generate code; monitor history size |
| Policy bypass | Security incidents | Compile-time capability gating, runtime enforcement, registry certification |
| Developer friction | Adoption issues | CLI, Studio, agent assistance; default Pure/ReadOnly; clear diagnostics |
| Legal issues from corpus | Compliance risk | Derived metadata only; track provenance; blocklist DMCA items |
| Performance in Temporal (batch vs stream) | Throughput limits | Document expectations; use local streaming engine where needed |
| Agent misuse | Production incidents | Default Strict/ReadOnly; require policy agent + human approval for Effectful flows |
| Double-retry loops | Cost blowups, Temporal history bloat | Retry ownership flag + lint; certification ensures connectors disable internal retries when orchestrator owns them |
| PII leakage | Regulatory incidents | Mandatory data-class tags, CEL policy enforcement, egress allowlists |
| Exactly-once illusion | Duplicate side effects | Enforce DedupeStore binding + `Delivery::ExactlyOnce` validation; run duplicate-delivery tests |

### 18.1 Red-team notes (watch points)
- **Temporal history growth:** enforce `partition_by!` on high-QPS triggers and Continue-As-New thresholds; otherwise workflows crawl.
- **Hidden nondeterminism:** static analyzer scans inline nodes for clock/UUID usage; registry rejects Strict claims with risky dependencies.
- **Ambiguous idempotency:** lint reveals which input fields feed idempotency keys and their determinism class to avoid unstable keys.
- **LLM cost blowups:** require preflight token estimates and default `dry_run=true` in Dev; policy engine gates deployments lacking budgets before Prod.
- **High-cardinality telemetry:** metric label rules forbid raw IDs; exporters fall back to exemplars to protect TSDB.

---

## 19. Open Questions
1. Should registry reject BestEffort nodes unless paired with HITL or caching policy? (Security & Product) 
2. How aggressively to auto-convert complex n8n expressions? (Importer team) 
3. When to adopt Temporal Rust SDK vs continue Go/TS codegen? (Runtime team) 
4. Governance model for third-party connector submissions? (Product/Security) 
5. Do we surface flow determinism/effects in end-user UI? (UX/Product) 
6. Policy DSL: which CEL extensions and governance process do we need for capability/egress rules? (Policy team) 
7. Packaging story for offline/edge (WASM) regarding binary streaming? (Runtime) 
8. Should certain org policies (e.g., financial) mandate compensation hooks on specific effectful capabilities? (Policy/Security)
9. Do we require per-node cost estimates (manual or learned) before admitting flows into production tiers? (Finance/Product)
10. Org stance on BestEffort determinism in effectful paths: mandate HITL/cache guard (Option A) vs waiver-based allowance (Option B)? (Policy/Product)
11. Should “critical” capability tags (payments/file mutation/ticketing) require `CompensationSpec` or explicit waiver? (Policy/Security)

---

## 20. Next Actions
1. **Kernel team:** finalize Flow IR structs, capability traits, macros enforcement. 
2. **Connector team:** draft ConnectorSpec schema & generator; seed top 10 connectors. 
3. **Importer team:** implement JSON parser → Flow IR; compile loop; sample 200 workflows. 
4. **Studio team:** scaffold backend API; implement compile loop; integrate CLI. 
5. **Registry team:** build metadata store, certification harness (effects/determinism tests, contract tests). 
6. **Policy/Security:** define capability allowlists, egress rules, registry gating policies. 
7. **DevOps:** set up CI (cargo + flows commands), Temporal dev cluster, registry storage. 
8. **Program mgmt:** schedule ADR reviews, align cross-team milestones, track risks. 

---

## 21. ADR Update Recommendations
- **ADR-002 (Effects & Determinism):** incorporate SAGA/compensation optionality, pinned typestates (`PinnedUrl`, `HashRef`, `VersionId`), and explicit RNG/Clock rules.
- **ADR-003 (Orchestrator Abstraction):** codify normative placement table, retry ownership policy, and Continue-As-New thresholds tied to partition keys.
- **ADR-004 (Registry):** extend scope to signatures/SBOM, data-class tagging, cost estimator requirements, and certification gates outlined in §10.
- **ADR-005 (Capability Versioning):** document capability semver policy, compatibility matrices, and deployment gates outlined in §6.3.1.

---

## Appendix A — Example Flows (selected)

### Notion → Drive sync
See main document (Section 4.7) for code sample.

### Lead qualifying subflow
```
subflow! {
  name: qualify_lead,
  in: Lead,
  out: QualifiedLead,
  effects: ReadOnly,
  determinism: Stable,
  resources(http(HttpRead), kv(KvRead)),
  graph {
    let norm = NormalizeEmail;
    let enr  = ClearbitEnrich;
    let score= ScoreLead; // inline Pure node
    connect!(INPUT -> norm);
    connect!(norm -> enr);
    connect!(enr  -> score);
    connect!(score -> OUTPUT);
  }
}
```

### Marketing site graph
See Section 4.7 for full example.

### Incident dedup pipeline
```
inline_node! {
  name:"WindowDedup",
  in: Event,
  out: Dedup,
  effects: ReadOnly,
  determinism: BestEffort,
  resources(kv(KvRead), clock(Clock)),
  |ctx, e| {
    let now = ctx.resources.clock().now()...
    /* logic */
  }
}
```

### DS pipeline with Python node
```
workflow! {
  name: ds_pipeline, profile: Dev;
  let http  = HttpTrigger::post("/ds").respond(on_last);
  let norm  = NormalizeInput; // inline pure
  let py    = PyPandasMap;    // Python plugin
  let csv   = CsvToHttpResponse;
  connect!(http -> norm);
  connect!(norm -> py);
  connect!(py -> csv);
}
```

---

## Appendix B — Flow IR JSON Schema
(See the file `schemas/flow_ir.schema.json` in repo for canonical version.)

---

## Appendix C — Capability vs Effect Mapping
| Capability | Allowed Effects | Determinism Hints |
|------------|-----------------|-------------------|
| HttpRead::get | ReadOnly | BestEffort |
| HttpRead::get_pinned | ReadOnly | Stable |
| HttpWrite | Effectful | BestEffort |
| KvRead | ReadOnly | Stable if key content-addressed |
| KvWrite | Effectful | BestEffort |
| BlobRead::get_by_hash | ReadOnly | Strict |
| BlobWrite | Effectful | BestEffort |
| Clock | — | Prohibits Strict/Stable |
| Rng | — | Prohibits Strict/Stable |

---

## Appendix D — ConnectorSpec YAML Example
```
id: openai.chat.complete
provider: openai
effects: ReadOnly
determinism: BestEffort
capability: OpenAIRead
method: chat_complete
params:
  model: { type: string, required: true, default: "gpt-4.1-mini" }
  temperature: { type: number, default: 0.0 }
  max_tokens: { type: integer, default: 512 }
  seed?: { type: integer }
ports:
  in: { $ref: "ChatReq.schema.json" }
  out: { $ref: "ChatResp.schema.json" }
rate_limits: { qps: 50, burst: 150 }
egress: ["api.openai.com"]
idempotency: "by_input_hash"
tests:
  fixtures:
    - name: happy
      request: fixtures/openai/chat/happy.json
    - name: rate_limit
      request: fixtures/openai/chat/429.json
documentation: "Completes a chat conversation using OpenAI APIs."
```

---

## Appendix E — Registry Manifest Template
```
{
  "id": "connectors.openai.chat_complete",
  "version": "1.0.0",
  "effects": "ReadOnly",
  "determinism": "BestEffort",
  "capabilities": ["OpenAIRead"],
  "egress": ["api.openai.com"],
  "scopes": ["chat:write"],
  "idempotency": "by_input_hash",
  "rate_limits": { "qps": 50, "burst": 150 },
  "ports": {
    "in": "schemas/ChatReq.schema.json",
    "out": "schemas/ChatResp.schema.json"
  },
  "tests": {
    "fixtures": [
      "fixtures/openai/chat/happy.json",
      "fixtures/openai/chat/429.json",
      "fixtures/openai/chat/5xx.json"
    ],
    "determinism": {
      "replay_hash": "sha256:..."
    }
  },
  "docs": {
    "summary": "...",
    "params": "...",
    "examples": "..."
  }
}
```

---

## Appendix F — Importer Algorithm (Pseudo-code)
```
for workflow_json in corpus:
    wf = parse_json(workflow_json)
    ir = FlowIR::new()
    for node in wf.nodes:
        mapped = map_n8n_node(node)
        ir.add_node(mapped)
    for edge in wf.connections:
        ir.add_edge(map_edge(edge))
    rewrite_expressions(ir)
    annotate_effects(ir)
    annotate_determinism(ir)
    rust_code = emit_rust(ir)
    while !cargo_check(rust_code):
        diagnostics = parse_diagnostics()
        apply_fixes(diagnostics, rust_code)
    run_stubbed_replay(ir)
    record_lossy_notes(ir)
```

---

## Appendix G — Temporal Lowering (Pseudo-code)
```
fn lower_to_temporal(plan: ExecPlan) -> TemporalWorkflow {
    for unit in plan.units {
        match unit {
            RunUnit::InlinePure(id) => emit_inline(id),
            RunUnit::Activity(id, spec) => emit_activity(id, spec),
            RunUnit::Subflow(subflow_plan) => emit_child_workflow(subflow_plan),
        }
    }
    for cp in plan.checkpoints {
        emit_signal_wait(cp)
    }
    for rl in plan.rate_limits {
        emit_token_bucket(rl)
    }
    maybe_emit_continue_as_new(plan.thresholds)
    workflow_code
}
```

---

## Appendix H — Harvester Schema
- `workflows(workflow_id, file_path, hash, name, node_count, trigger_kind, complexity, has_code_node, has_binary)`
- `nodes(workflow_id, node_index, node_type, service, has_expression, has_code)`
- `edges(workflow_id, from_node, to_node)`
- `expressions(workflow_id, node_index, pattern, complexity_score)`
- `risk_tags(workflow_id, tags ARRAY)`

---

## Appendix I — Agent Prompt Templates
- **Harvester agent:** “Parse workflow JSON, emit metadata rows, compute connector frequencies.”
- **Rewriter agent:** “Map n8n nodes to canonical nodes, generate Rust macros, run cargo check, fix errors iteratively, document lossy semantics.”
- **Connector agent:** “Given ConnectorSpec YAML, generate node implementation, tests, manifest.”
- **Policy agent:** “Validate flow connectors vs policy; block effectful nodes missing idempotency.”
- **Test agent:** “Execute golden tests, contract fixtures, determinism replay; record results.”

---

## Appendix J — ADR Summaries
- **ADR-001:** One kernel with host adapters (Tokio, Web, Queue, Temporal, WASM). Multiple kernels rejected to avoid fragmentation.
- **ADR-002:** Effects/determinism gating (Pure/ReadOnly/Effectful, Strict/Stable/BestEffort/Nondeterministic) enforced at compile-time and registry.
- **ADR-003:** Orchestrator abstraction & Temporal lowering (Flow IR → ExecPlan → Temporal Workflows/Activities). Execution and orchestration decoupled.

---

## Appendix K — Testing Matrix
| Test Type | Focus |
|-----------|-------|
| Unit | Macros, capability gating, Flow IR validation |
| Property | Merge invariants, idempotency uniqueness, lineage |
| Golden | Key flows with fixtures |
| Contract | Connector HTTP fixtures |
| Importer | n8n samples -> Rust -> replay |
| Integration | Queue, Temporal, WASM |
| Fault Injection | Errors, rate limits, worker crashes |
| Determinism | Record/replay Strict/Stable |
| Security | Capability enforcement, egress |

---

## Appendix L — Security & Compliance Checklist
- [ ] Capability allowlists configured per profile
- [ ] Secrets via providers only
- [ ] Idempotency keys on effectful nodes
- [ ] Egress domains declared
- [ ] HITL checkpoints logged and auditable
- [ ] WASM plugins sandboxed
- [ ] Python/gRPC nodes network constrained
- [ ] Registry certification passed (effects/determinism/tests)
- [ ] Policy engine enforces organization rules
- [ ] Audit log stored for run events and approvals
