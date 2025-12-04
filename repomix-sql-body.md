# Files

## File: impl-docs/opendal-capability-plan.md
````markdown
# OpenDAL Capability Integration — Phased Plan

> Roadmap for introducing OpenDAL-backed capability crates while preserving the capability contracts defined in `impl-docs/impl-plan.md` Phase 3 and the platform-specific affordances described in `impl-docs/cloudflare`. Each phase is deliverable on its own (no interdependencies beyond prior phases) and results in user-visible documentation and tests.

---

## Phase 1 — `cap-opendal-core` Scaffold

- **Status**: ✅ Implemented (`cap-opendal-core` crate landed).
- **How / Where / Why**
  - Create `crates/cap-opendal-core` housing shared builders and error adapters.
  - Mirror OpenDAL service/layer features in `Cargo.toml`; default to no features, expose helper macros for registering scheme-specific toggles.
  - Provide `OperatorFactory` trait with async `build(&self) -> Operator`, plus `layer_if` combinators so downstream crates compose retries/logging without rewriting boilerplate.
  - Add uniform error mappers (`opendal::ErrorKind::NotFound` → `CapabilityError::NotFound`, etc.) reused by blob/KV wrappers.
- **Documentation Targets**
  - `impl-docs/phase3-host-runtime-refactor.md`: add note on shared OpenDAL core feeding capability injection via `ResourceBag`.
  - New section in `impl-docs/cloudflare/TECHNICAL_DESIGN_DOCUMENT.md` documenting how Cloudflare capability crates will consume the shared factory.
  - README for `cap-opendal-core` explaining feature toggles and extension trait pattern.
- **Critical Tests**
  - Unit tests covering feature gating (e.g., enabling `services-s3` compiles builder).
  - Error translation snapshot tests verifying mapping of representative `ErrorKind`s.
  - Builder smoke test: construct memory backend and ensure `build().await` succeeds.
- **Definition of Done**
  - Workspace builds with new crate and adds no new warnings.
  - Docs updated and cross-linked; clippy/rustfmt stay clean.
  - CI job runs new unit tests under `cargo test -p cap-opendal-core`.

---

## Phase 2 — `cap-blob-opendal` Capability

- **Status**: ✅ Implemented (`cap-blob-opendal` providing `OpendalBlobStore`).
- **How / Where / Why**
  - Implement `crates/cap-blob-opendal` exposing `OpendalBlobStore` that wraps an `Operator` and implements `capabilities::blob::BlobStore`.
  - Accept a `BlobStoreBuilder` that references `cap-opendal-core::OperatorFactory`; support per-backend extensions via trait-based builder modifiers (e.g., `S3BlobExt`, `FsBlobExt`).
  - Translate OpenDAL’s streaming readers into `AsyncRead` / `Stream<Item = Result<Bytes>>` helpers for future executor spill refactors.
  - Ensure `ResourceBag::with_blob(Arc<OpendalBlobStore>)` works without additional adapters.
- **Documentation Targets**
  - Update `impl-docs/work-log.md` with Phase 2 entry for the new capability.
  - Extend `impl-docs/impl-plan.md` Phase 3 tasks to reference OpenDAL-backed blob integration.
  - Document builder usage and extension traits in `crates/cap-blob-opendal/README.md`.
- **Critical Tests**
  - Async put/get/delete round-trip using OpenDAL memory backend.
  - Streaming read test verifying partial reads propagate without buffering entire object.
  - Error propagation test covering missing keys and injected IO failures.
- **Definition of Done**
  - Blob capability trait satisfied and imported in at least one example (e.g., executor spill smoke test flagged TODO for later replacement).
  - Documentation updated, tests passing, `cargo check` workspace-wide remains green.

---

## Phase 3 — `cap-kv-opendal` Capability & Metadata

- **Status**: ✅ Implemented (`cap-kv-opendal` providing `OpendalKvStore`).
- **How / Where / Why**
  - Add `crates/cap-kv-opendal` implementing `capabilities::kv::KeyValue` on top of the shared core.
  - Provide `KvStoreBuilder` supporting TTL semantics (`PerWrite`, `NamespaceDefault`, `None`) and consistency advertised by backend.
  - Publish `KvCapabilityInfo` struct (TTL + consistency enums) retrievable from the store for runtime inspection.
  - Reuse builder extension traits for backend-specific tuning (e.g., `CloudflareKvExt::with_namespace_default_ttl`).
- **Documentation Targets**
  - Update `impl-docs/error-taxonomy.md` / capability sections to describe how TTL/consistency metadata complements determinism/effect hints.
  - Add trait documentation explaining metadata retrieval to `capabilities/src/lib.rs` comments.
  - README describing supported OpenDAL schemes and capability guarantees.
- **Critical Tests**
  - Unit tests validating TTL behaviour for in-memory backend and namespace-default override for Cloudflare KV mock.
  - Consistency metadata tests asserting correct enum for known backends (Memory = Strong, Cloudflare KV = Eventual).
  - Regression test verifying `ResourceBag::with_kv` exposes usable capability in host runtime (`host-inproc` smoke).
- **Definition of Done**
  - Capability metadata accessible at runtime; validator or harness code updated to consume it (even if only logged initially).
  - Docs synchronized; tests run under `cargo test -p cap-kv-opendal`.

---

## Phase 4 — Provider Extension Crates

- **How / Where / Why**
  - Layer provider-specific crates (e.g., `cap-cloudflare-kv`, `cap-cloudflare-r2`, future AWS/Azure variants) atop the OpenDAL capabilities.
  - Each crate re-exports the base builder plus extension traits, injects default layers (retry, logging), and registers effect/determinism hints.
  - Update host bridges (Axum, Workers) to wire these presets into their `ResourceBag`s conditionally by profile.
- **Documentation Targets**
  - Expand `impl-docs/cloudflare/TECHNICAL_DESIGN_DOCUMENT.md` Phase 2 capability section to reference OpenDAL-backed implementations and list extra extension APIs (bulk delete, multipart tuning).
  - Add provider crate READMEs showing configuration mapping (env vars → builder options).
  - Document bridge configuration in `host-web-axum` / `host-workers` READMEs.
- **Critical Tests**
  - Integration tests in each provider crate (feature-gated) using mocked HTTP clients to assert namespace/account routing and TTL enforcement.
  - Host runtime test that assembles a `ResourceBag` with provider preset and runs a simple flow.
  - Feature-flag compile checks ensuring disabling provider feature removes dependency footprint.
- **Definition of Done**
  - Provider crates publish the same capability traits while exposing provider-only extensions.
  - Bridge wiring (at least one host) demonstrated via test or example; docs linked.

---

## Phase 5 — Capability Descriptor Integration & Spill Adoption

- **How / Where / Why**
  - Propagate `KvCapabilityInfo` / `BlobCapabilityInfo` through `ResourceAccess` so executors and validators adapt behaviour (e.g., warn on eventual stores for strict flows, adjust spill thresholds).
  - Rework executor spill path (`crates/kernel-exec/src/lib.rs:400-617`) to use `BlobStore` streaming APIs; replace tempdir implementation with injected capability via `ResourceBag`.
  - Update idempotency harness to log TTL/consistency info for certification.
- **Documentation Targets**
  - Amend `impl-docs/phase3-host-runtime-refactor.md` with final capability injection narrative (Blob/KV metadata usage).
  - Add new diagnostics or guidance to `impl-docs/error-taxonomy.md` around inconsistent capabilities (e.g., `KV_EVT001` warning).
  - Update executor README / docs with spill capability expectations.
- **Critical Tests**
  - Executor spill integration test using `cap-blob-opendal` mock backend ensuring spill/load/delete round-trip.
  - Validator/unit test verifying warnings fire when strict determinism meets eventual store.
  - Harness test capturing TTL metadata in report JSON.
- **Definition of Done**
  - Tempdir spill implementation fully replaced by capability-backed storage.
  - Capability metadata influences runtime/validator behaviour.
  - Documentation references updated, all affected test suites green.

---

## Future Considerations

- Extend shared builders to support secret management adapters (aligning with Phase 4/Phase 5 CLI packaging).
- Consider generalising capability metadata into a registry so third-party crates can declare additional invariants without modifying core traits.
- Monitor OpenDAL releases for new service/layer flags; update `cap-opendal-core` feature matrix accordingly and append release notes to `impl-docs/work-log.md`.
````

## File: crates/cap-kv-sqlite/src/lib.rs
````rust
// TODO
````

## File: crates/cap-kv-sqlite/Cargo.toml
````toml
[package]
name = "cap-kv-sqlite"
version = "0.1.0"
authors.workspace = true
edition.workspace = true
rust-version.workspace = true
license.workspace = true
repository.workspace = true
homepage.workspace = true
description = "SQLite-backed key/value capability for dev and testing profiles."

[dependencies]
capabilities = { path = "../capabilities" }
anyhow.workspace = true
rusqlite.workspace = true
serde.workspace = true
thiserror.workspace = true
tracing.workspace = true
````

## File: crates/cap-kv-sqlite/README.md
````markdown
# cap-kv-sqlite

`cap-kv-sqlite` provides a lightweight SQLite-backed KV capability for development and test profiles. It allows Strict/Stable reads via content-addressing and offers simple local persistence.

## Surface
- `KvRead`/`KvWrite` implementations with optional TTL and transactional guards.
- Schema migration helpers for evolving local state.
- Utilities for test harnesses to seed fixtures.

## Next steps
- Implement CRUD operations with deterministic read options and JSON value storage.
- Cover optimistic-locking and conflict scenarios in unit tests.
- Integrate with spill/cache policies once `capabilities` exposes them.

## Depends on
- Trait contracts from `capabilities`.
- Optionally consumes shared type definitions from `types-common` when available.
````

## File: crates/dag-core/src/determinism.rs
````rust
use std::sync::RwLock;

use once_cell::sync::Lazy;

use crate::Determinism;

/// Registry entry describing a determinism conflict for a given resource hint.
#[derive(Debug, Clone)]
pub struct DeterminismConstraint {
    /// Resource hint identifier (hierarchical; use `::` separators).
    pub hint: &'static str,
    /// Minimum determinism level required when the resource is present.
    pub minimum: Determinism,
    /// Human-friendly guidance describing the mitigation.
    pub guidance: &'static str,
}

impl DeterminismConstraint {
    /// Construct a new constraint definition.
    pub const fn new(hint: &'static str, minimum: Determinism, guidance: &'static str) -> Self {
        Self {
            hint,
            minimum,
            guidance,
        }
    }

    /// Determine whether the supplied resource hint matches this constraint.
    pub fn matches(&self, resource_hint: &str) -> bool {
        resource_hint == self.hint || resource_hint.starts_with(self.hint)
    }
}

static CONSTRAINTS: Lazy<RwLock<Vec<DeterminismConstraint>>> = Lazy::new(|| {
    RwLock::new(vec![
        DeterminismConstraint::new(
            "resource::clock",
            Determinism::BestEffort,
            "Clock access introduces wall-clock time; downgrade determinism to BestEffort or surface the timestamp as an input.",
        ),
        DeterminismConstraint::new(
            "resource::http",
            Determinism::BestEffort,
            "HTTP calls depend on external systems; downgrade determinism or provide cached/pinned responses before claiming Stable.",
        ),
        DeterminismConstraint::new(
            "resource::rng",
            Determinism::BestEffort,
            "Random number generation requires a deterministic seed; downgrade determinism or use a seeded RNG provided via input.",
        ),
    ])
});

/// Register an additional determinism constraint. Existing hints are replaced.
pub fn register_determinism_constraint(constraint: DeterminismConstraint) {
    let mut guard = CONSTRAINTS
        .write()
        .expect("determinism constraint registry poisoned");
    if let Some(existing) = guard.iter_mut().find(|item| item.hint == constraint.hint) {
        *existing = constraint;
    } else {
        guard.push(constraint);
    }
}

/// Find the first constraint matching the provided resource hint.
pub fn constraint_for_hint(hint: &str) -> Option<DeterminismConstraint> {
    let guard = CONSTRAINTS
        .read()
        .expect("determinism constraint registry poisoned");
    guard.iter().find(|c| c.matches(hint)).cloned()
}

/// Snapshot all registered determinism constraints.
pub fn all_constraints() -> Vec<DeterminismConstraint> {
    let guard = CONSTRAINTS
        .read()
        .expect("determinism constraint registry poisoned");
    guard.clone()
}
````

## File: crates/dag-core/src/effects_registry.rs
````rust
use std::sync::RwLock;

use once_cell::sync::Lazy;

use crate::Effects;

/// Registry entry describing the minimum effects level required for a resource hint.
#[derive(Debug, Clone)]
pub struct EffectConstraint {
    /// Resource hint identifier (namespaced using `::`).
    pub hint: &'static str,
    /// Minimum effects level required when this hint is present.
    pub minimum: Effects,
    /// Guidance surfaced alongside diagnostics.
    pub guidance: &'static str,
}

impl EffectConstraint {
    /// Construct a new constraint description.
    pub const fn new(hint: &'static str, minimum: Effects, guidance: &'static str) -> Self {
        Self {
            hint,
            minimum,
            guidance,
        }
    }

    /// Returns true when the provided hint matches this constraint.
    pub fn matches(&self, resource_hint: &str) -> bool {
        resource_hint == self.hint || resource_hint.starts_with(self.hint)
    }
}

static CONSTRAINTS: Lazy<RwLock<Vec<EffectConstraint>>> = Lazy::new(|| {
    RwLock::new(vec![
        EffectConstraint::new(
            "resource::http::read",
            Effects::ReadOnly,
            "HTTP reads interact with external services; declare effects = ReadOnly or wrap in effectful activity.",
        ),
        EffectConstraint::new(
            "resource::http::write",
            Effects::Effectful,
            "HTTP writes are effectful; declare effects = Effectful or refactor to read-only operations.",
        ),
        EffectConstraint::new(
            "resource::db::write",
            Effects::Effectful,
            "Database writes require Effectful; downgrade only for read-only transactions.",
        ),
    ])
});

/// Register an additional effect constraint. Existing hints are replaced.
pub fn register_effect_constraint(constraint: EffectConstraint) {
    let mut guard = CONSTRAINTS
        .write()
        .expect("effect constraint registry poisoned");
    if let Some(existing) = guard.iter_mut().find(|item| item.hint == constraint.hint) {
        *existing = constraint;
    } else {
        guard.push(constraint);
    }
}

/// Look up the constraint that matches the provided hint, if any.
pub fn constraint_for_hint(hint: &str) -> Option<EffectConstraint> {
    let guard = CONSTRAINTS
        .read()
        .expect("effect constraint registry poisoned");
    guard.iter().find(|c| c.matches(hint)).cloned()
}

/// Snapshot all registered effect constraints.
pub fn all_constraints() -> Vec<EffectConstraint> {
    let guard = CONSTRAINTS
        .read()
        .expect("effect constraint registry poisoned");
    guard.clone()
}
````

## File: crates/dag-core/README.md
````markdown
# dag-core

`dag-core` defines the foundational types, traits, and Flow IR structures that every other LatticeFlow crate consumes. It covers effect/determinism enums, node/trigger contracts, capability references, and the serialisable IR that powers validation, planning, and registry export.

## Surface
- Core trait definitions for `Node`, `Trigger`, `Capability`, and execution contexts.
- Flow IR data structures plus serde/schemars integration for export tooling.
- Shared error and result types used across hosts, planners, and connectors.

## Next steps
- Flesh out the type system according to RFC §4–§5, including canonical idempotency/caching specs.
- Add serde/schemars implementations and unit tests for IR round-tripping.
- Wire in lint helpers that downstream crates can reuse for effect/determinism validation.

## Depends on
- No prior crates are required, but this crate should land before `dag-macros`, `kernel-plan`, and `capabilities` implementations begin.
````

## File: Cargo.toml
````toml
[workspace]
resolver = "3"
members = [
  "crates/adapters",
  "crates/cap-blob-fs",
  "crates/cap-blob-opendal",
  "crates/cap-kv-opendal",
  "crates/cap-opendal-core",
  "crates/contrib/redis/cap-cache-redis",
  "crates/contrib/redis/cap-dedupe-redis",
  "crates/contrib/redis/cap-redis",
  "crates/cap-http-reqwest",
  "crates/cap-kv-sqlite",
  "crates/capabilities",
  "crates/cli",
  "crates/connector-spec",
  "crates/connectors-std",
  "crates/dag-core",
  "crates/dag-macros",
  "crates/exporters",
  "crates/bridge-queue-redis",
  "crates/host-temporal",
  "crates/host-inproc",
  "crates/host-web-axum",
  "crates/importer-n8n",
  "crates/kernel-exec",
  "crates/kernel-plan",
  "crates/plugin-python",
  "crates/plugin-wasi",
  "crates/policy-engine",
  "crates/registry-cert",
  "crates/registry-client",
  "crates/studio-backend",
  "crates/testing-harness-idem",
  "crates/types-common",
  "examples/s1_echo",
  "examples/s2_site"
]

[workspace.package]
authors = ["LatticeFlow Authors <opensource@latticeflow.dev>"]
edition = "2024"
homepage = "https://github.com/psu3d0/LatticeFlow"
license = "MIT OR Apache-2.0"
repository = "https://github.com/psu3d0/LatticeFlow"
rust-version = "1.90"
description = "Rust workflow platform workspace"

[workspace.metadata]
msrv = "1.90"

[workspace.dependencies]
anyhow = "1"
async-trait = "0.1"
axum = { version = "0.7", features = ["json", "tokio", "http1"] }
bytes = "1"
clap = { version = "4.5", features = ["derive"] }
config = "0.14"
redis = { version = "0.32.7", features = ["tokio-comp"] }
reqwest = { version = "0.12", default-features = false, features = ["json", "rustls-tls", "native-tls-vendored"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
serde_yaml = "0.9"
schemars = "0.8"
thiserror = "1"
tonic = { version = "0.11", features = ["transport"] }
tokio = { version = "1", default-features = false, features = ["macros", "net", "rt-multi-thread", "signal", "sync", "time", "fs"] }
tokio-stream = "0.1"
tokio-util = "0.7"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "fmt"] }
futures = "0.3"
hyper = { version = "1", features = ["full"] }
proc-macro2 = "1"
quote = "1"
syn = { version = "2", features = ["full"] }
serde_with = "3"
uuid = { version = "1", features = ["v4", "v5", "serde"] }
wasmtime = { version = "16", features = ["async"] }
wit-bindgen = "0.24"
rustls = { version = "0.23", features = ["ring"] }
rustls-pemfile = "2"
rusqlite = { version = "0.30", features = ["bundled"] }
flate2 = "1"
hex = "0.4"
walkdir = "2"
tempfile = "3"
once_cell = "1.19"
semver = { version = "1.0", features = ["serde"] }
trybuild = "1.0"
tower = "0.4"
async-stream = "0.3"
metrics = "0.24.2"
metrics-util = "0.20"
proptest = "1.5"
````

## File: crates/capabilities/src/lib.rs
````rust
use async_trait::async_trait;
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::sync::{Arc, Mutex, OnceLock};
use std::time::{Duration, Instant};

pub mod hints;

/// Marker trait implemented by all capability providers.
pub trait Capability: Send + Sync + 'static {
    /// Human-friendly capability identifier.
    fn name(&self) -> &'static str;
}

/// Unified view of capabilities exposed to node execution contexts.
pub trait ResourceAccess: Send + Sync + 'static {
    fn http_read(&self) -> Option<&dyn http::HttpRead> {
        None
    }

    fn http_write(&self) -> Option<&dyn http::HttpWrite> {
        None
    }

    fn clock(&self) -> Option<&dyn clock::Clock> {
        None
    }

    fn cache(&self) -> Option<&dyn cache::Cache> {
        None
    }

    fn kv(&self) -> Option<&dyn kv::KeyValue> {
        None
    }

    fn blob(&self) -> Option<&dyn blob::BlobStore> {
        None
    }

    fn queue(&self) -> Option<&dyn queue::Queue> {
        None
    }

    fn dedupe_store(&self) -> Option<&dyn dedupe::DedupeStore> {
        None
    }
}

/// Mutable collection of capability providers surfaced to the executor.
#[derive(Default, Clone)]
pub struct ResourceBag {
    http_read: Option<Arc<dyn http::HttpRead>>,
    http_write: Option<Arc<dyn http::HttpWrite>>,
    clock: Option<Arc<dyn clock::Clock>>,
    cache: Option<Arc<dyn cache::Cache>>,
    kv: Option<Arc<dyn kv::KeyValue>>,
    blob: Option<Arc<dyn blob::BlobStore>>,
    queue: Option<Arc<dyn queue::Queue>>,
    dedupe: Option<Arc<dyn dedupe::DedupeStore>>,
}

impl ResourceBag {
    pub fn new() -> Self {
        Self::default()
    }

    pub fn with_http_read<T>(mut self, capability: Arc<T>) -> Self
    where
        T: http::HttpRead + 'static,
    {
        let capability: Arc<dyn http::HttpRead> = capability;
        self.http_read = Some(capability);
        self
    }

    pub fn with_http_write<T>(mut self, capability: Arc<T>) -> Self
    where
        T: http::HttpWrite + 'static,
    {
        let capability: Arc<dyn http::HttpWrite> = capability;
        self.http_write = Some(capability);
        self
    }

    pub fn with_clock<T>(mut self, capability: Arc<T>) -> Self
    where
        T: clock::Clock + 'static,
    {
        let capability: Arc<dyn clock::Clock> = capability;
        self.clock = Some(capability);
        self
    }

    pub fn with_cache<T>(mut self, capability: Arc<T>) -> Self
    where
        T: cache::Cache + 'static,
    {
        let capability: Arc<dyn cache::Cache> = capability;
        self.cache = Some(capability);
        self
    }

    pub fn with_kv<T>(mut self, capability: Arc<T>) -> Self
    where
        T: kv::KeyValue + 'static,
    {
        let capability: Arc<dyn kv::KeyValue> = capability;
        self.kv = Some(capability);
        self
    }

    pub fn with_blob<T>(mut self, capability: Arc<T>) -> Self
    where
        T: blob::BlobStore + 'static,
    {
        let capability: Arc<dyn blob::BlobStore> = capability;
        self.blob = Some(capability);
        self
    }

    pub fn with_queue<T>(mut self, capability: Arc<T>) -> Self
    where
        T: queue::Queue + 'static,
    {
        let capability: Arc<dyn queue::Queue> = capability;
        self.queue = Some(capability);
        self
    }

    pub fn with_dedupe<T>(mut self, capability: Arc<T>) -> Self
    where
        T: dedupe::DedupeStore + 'static,
    {
        let capability: Arc<dyn dedupe::DedupeStore> = capability;
        self.dedupe = Some(capability);
        self
    }
}

/// Utilities for scoping capability access to the current execution.
pub mod context {
    use super::ResourceAccess;
    use std::future::Future;
    use std::sync::Arc;

    use tokio::task_local;

    task_local! {
        static CURRENT_RESOURCES: Arc<dyn ResourceAccess>;
    }

    /// Execute `future` with the provided resource access scoped to the current task.
    pub async fn with_resources<Fut, R>(resources: Arc<dyn ResourceAccess>, future: Fut) -> R
    where
        Fut: Future<Output = R>,
    {
        CURRENT_RESOURCES.scope(resources, future).await
    }

    /// Invoke the callback with the currently scoped resource access, if present.
    pub fn with_current<F, R>(callback: F) -> Option<R>
    where
        F: FnOnce(&dyn ResourceAccess) -> R,
    {
        CURRENT_RESOURCES
            .try_with(|resources| callback(resources.as_ref()))
            .ok()
    }

    /// Invoke the async callback with the currently scoped resource access, if present.
    pub async fn with_current_async<F, Fut, R>(callback: F) -> Option<R>
    where
        F: FnOnce(Arc<dyn ResourceAccess>) -> Fut,
        Fut: Future<Output = R>,
    {
        let resources = CURRENT_RESOURCES
            .try_with(|resources| Arc::clone(resources))
            .ok()?;
        Some(callback(resources).await)
    }

    /// Clone the currently scoped resource access handle, if any.
    pub fn current_handle() -> Option<Arc<dyn ResourceAccess>> {
        CURRENT_RESOURCES
            .try_with(|resources| Arc::clone(resources))
            .ok()
    }
}

impl ResourceAccess for ResourceBag {
    fn http_read(&self) -> Option<&dyn http::HttpRead> {
        self.http_read
            .as_ref()
            .map(|cap| cap.as_ref() as &dyn http::HttpRead)
    }

    fn http_write(&self) -> Option<&dyn http::HttpWrite> {
        self.http_write
            .as_ref()
            .map(|cap| cap.as_ref() as &dyn http::HttpWrite)
    }

    fn clock(&self) -> Option<&dyn clock::Clock> {
        self.clock
            .as_ref()
            .map(|cap| cap.as_ref() as &dyn clock::Clock)
    }

    fn cache(&self) -> Option<&dyn cache::Cache> {
        self.cache
            .as_ref()
            .map(|cap| cap.as_ref() as &dyn cache::Cache)
    }

    fn kv(&self) -> Option<&dyn kv::KeyValue> {
        self.kv
            .as_ref()
            .map(|cap| cap.as_ref() as &dyn kv::KeyValue)
    }

    fn blob(&self) -> Option<&dyn blob::BlobStore> {
        self.blob
            .as_ref()
            .map(|cap| cap.as_ref() as &dyn blob::BlobStore)
    }

    fn queue(&self) -> Option<&dyn queue::Queue> {
        self.queue
            .as_ref()
            .map(|cap| cap.as_ref() as &dyn queue::Queue)
    }

    fn dedupe_store(&self) -> Option<&dyn dedupe::DedupeStore> {
        self.dedupe
            .as_ref()
            .map(|cap| cap.as_ref() as &dyn dedupe::DedupeStore)
    }
}

pub mod http {
    use super::*;

    pub const HINT_HTTP: &str = "resource::http";
    pub const HINT_HTTP_READ: &str = "resource::http::read";
    pub const HINT_HTTP_WRITE: &str = "resource::http::write";

    static REGISTRATION: OnceLock<()> = OnceLock::new();

    /// Ensure HTTP capability hints are registered with the shared effect/determinism registries.
    pub fn ensure_registered() {
        REGISTRATION.get_or_init(|| {
            dag_core::effects_registry::register_effect_constraint(
                dag_core::effects_registry::EffectConstraint::new(
                    HINT_HTTP_READ,
                    dag_core::Effects::ReadOnly,
                    "HTTP reads reach external systems; declare effects = ReadOnly or Effectful.",
                ),
            );
            dag_core::effects_registry::register_effect_constraint(
                dag_core::effects_registry::EffectConstraint::new(
                    HINT_HTTP_WRITE,
                    dag_core::Effects::Effectful,
                    "HTTP writes are effectful; declare effects = Effectful and provide idempotency keys.",
                ),
            );
            dag_core::determinism::register_determinism_constraint(
                dag_core::determinism::DeterminismConstraint::new(
                    HINT_HTTP,
                    dag_core::Determinism::BestEffort,
                    "HTTP calls vary across retries; downgrade determinism or pin responses via caching.",
                ),
            );
        });
    }

    /// HTTP method supported by canonical client implementations.
    #[derive(Debug, Clone, Copy, Serialize, Deserialize, PartialEq, Eq)]
    #[serde(rename_all = "UPPERCASE")]
    pub enum HttpMethod {
        Get,
        Head,
        Post,
        Put,
        Patch,
        Delete,
    }

    impl HttpMethod {
        pub fn as_str(self) -> &'static str {
            match self {
                HttpMethod::Get => "GET",
                HttpMethod::Head => "HEAD",
                HttpMethod::Post => "POST",
                HttpMethod::Put => "PUT",
                HttpMethod::Patch => "PATCH",
                HttpMethod::Delete => "DELETE",
            }
        }
    }

    #[derive(Debug, Clone, Serialize, Deserialize, Default)]
    pub struct HttpHeaders(pub HashMap<String, String>);

    impl HttpHeaders {
        pub fn insert(&mut self, key: impl Into<String>, value: impl Into<String>) {
            self.0.insert(key.into(), value.into());
        }

        pub fn get(&self, key: &str) -> Option<&String> {
            self.0.get(key)
        }

        pub fn iter(&self) -> impl Iterator<Item = (&String, &String)> {
            self.0.iter()
        }
    }

    #[derive(Debug, Clone, Serialize, Deserialize)]
    pub struct HttpRequest {
        pub method: HttpMethod,
        pub url: String,
        #[serde(default)]
        pub headers: HttpHeaders,
        pub body: Option<Vec<u8>>,
        pub timeout_ms: Option<u64>,
    }

    impl HttpRequest {
        pub fn new(method: HttpMethod, url: impl Into<String>) -> Self {
            Self {
                method,
                url: url.into(),
                headers: HttpHeaders::default(),
                body: None,
                timeout_ms: None,
            }
        }

        pub fn with_body(mut self, body: impl Into<Vec<u8>>) -> Self {
            self.body = Some(body.into());
            self
        }

        pub fn with_header(mut self, key: impl Into<String>, value: impl Into<String>) -> Self {
            self.headers.insert(key, value);
            self
        }
    }

    #[derive(Debug, Clone, Serialize, Deserialize)]
    pub struct HttpResponse {
        pub status: u16,
        #[serde(default)]
        pub headers: HttpHeaders,
        pub body: Vec<u8>,
    }

    impl HttpResponse {
        pub fn is_success(&self) -> bool {
            (200..300).contains(&self.status)
        }
    }

    /// Canonical error type surfaced by HTTP capabilities.
    #[derive(Debug, thiserror::Error)]
    pub enum HttpError {
        #[error("transport error: {0}")]
        Transport(#[from] anyhow::Error),
        #[error("request timed out after {0}ms")]
        Timeout(u64),
        #[error("invalid response: {0}")]
        InvalidResponse(String),
    }

    pub type HttpResult<T> = Result<T, HttpError>;

    #[async_trait]
    pub trait HttpRead: Send + Sync {
        async fn send(&self, request: HttpRequest) -> HttpResult<HttpResponse>;
    }

    #[async_trait]
    pub trait HttpWrite: Send + Sync {
        async fn send(&self, request: HttpRequest) -> HttpResult<HttpResponse>;
    }

    #[cfg(test)]
    mod tests {
        use super::*;

        #[test]
        fn ensure_http_registration_is_idempotent() {
            ensure_registered();
            ensure_registered();

            let effect_read = dag_core::effects_registry::constraint_for_hint(HINT_HTTP_READ)
                .expect("http read constraint registered");
            assert_eq!(effect_read.minimum, dag_core::Effects::ReadOnly);

            let effect_write = dag_core::effects_registry::constraint_for_hint(HINT_HTTP_WRITE)
                .expect("http write constraint registered");
            assert_eq!(effect_write.minimum, dag_core::Effects::Effectful);

            let det = dag_core::determinism::constraint_for_hint(HINT_HTTP)
                .expect("http determinism constraint registered");
            assert_eq!(det.minimum, dag_core::Determinism::BestEffort);
        }
    }
}

pub mod clock {
    use super::*;
    use std::time::SystemTime;

    pub const HINT_CLOCK: &str = "resource::clock";
    static REGISTRATION: OnceLock<()> = OnceLock::new();

    /// Ensure the shared determinism hint for clocks is registered.
    pub fn ensure_registered() {
        REGISTRATION.get_or_init(|| {
            dag_core::determinism::register_determinism_constraint(
                dag_core::determinism::DeterminismConstraint::new(
                    HINT_CLOCK,
                    dag_core::Determinism::BestEffort,
                    "Clock access is nondeterministic; declare determinism = BestEffort or lower.",
                ),
            );
        });
    }

    /// Abstract clock capability used by runtimes.
    pub trait Clock: Capability {
        fn now(&self) -> SystemTime;
    }

    /// Wall-clock implementation backed by `SystemTime::now`.
    #[derive(Default)]
    pub struct SystemClock;

    impl Capability for SystemClock {
        fn name(&self) -> &'static str {
            "clock.system"
        }
    }

    impl Clock for SystemClock {
        fn now(&self) -> SystemTime {
            SystemTime::now()
        }
    }

    #[cfg(test)]
    mod tests {
        use super::*;

        #[test]
        fn register_clock_hint_once() {
            ensure_registered();
            ensure_registered();
            let constraint =
                dag_core::determinism::constraint_for_hint(HINT_CLOCK).expect("clock hint");
            assert_eq!(constraint.minimum, dag_core::Determinism::BestEffort);
        }

        #[test]
        fn system_clock_produces_time() {
            ensure_registered();
            let clock = SystemClock::default();
            let now = clock.now();
            assert!(now.elapsed().is_ok());
        }
    }
}

pub mod dedupe {
    use super::*;
    use std::time::Duration;

    pub const HINT_DEDUPE: &str = "resource::dedupe";
    pub const HINT_DEDUPE_WRITE: &str = "resource::dedupe::write";

    static REGISTRATION: OnceLock<()> = OnceLock::new();

    /// Register effect/determinism constraints for dedupe stores.
    pub fn ensure_registered() {
        REGISTRATION.get_or_init(|| {
            dag_core::effects_registry::register_effect_constraint(
                dag_core::effects_registry::EffectConstraint::new(
                    HINT_DEDUPE_WRITE,
                    dag_core::Effects::Effectful,
                    "Dedupe stores persist state; declare effects = Effectful when binding.",
                ),
            );
            dag_core::determinism::register_determinism_constraint(
                dag_core::determinism::DeterminismConstraint::new(
                    HINT_DEDUPE,
                    dag_core::Determinism::BestEffort,
                    "Dedupe lookups depend on external state; downgrade determinism or provide proofs.",
                ),
            );
        });
    }

    /// Errors surfaced by dedupe store capabilities.
    #[derive(Debug, thiserror::Error)]
    pub enum DedupeError {
        #[error("async runtime not available to execute redis operations")]
        RuntimeUnavailable,
        #[error("operation failed: {0}")]
        Other(String),
    }

    #[async_trait]
    pub trait DedupeStore: Capability {
        async fn put_if_absent(&self, key: &[u8], ttl: Duration) -> Result<bool, DedupeError>;
        async fn forget(&self, key: &[u8]) -> Result<(), DedupeError>;
    }

    #[cfg(test)]
    mod tests {
        use super::*;

        #[test]
        fn registers_constraints_once() {
            ensure_registered();
            ensure_registered();
            let effect = dag_core::effects_registry::constraint_for_hint(HINT_DEDUPE_WRITE)
                .expect("dedupe effect constraint");
            assert_eq!(effect.minimum, dag_core::Effects::Effectful);
            let det = dag_core::determinism::constraint_for_hint(HINT_DEDUPE)
                .expect("dedupe determinism constraint");
            assert_eq!(det.minimum, dag_core::Determinism::BestEffort);
        }
    }
}

pub mod cache {
    use super::*;

    /// Shared cache capability error type.
    #[derive(Debug, thiserror::Error)]
    pub enum CacheError {
        #[error("value not found")]
        NotFound,
        #[error("operation failed: {0}")]
        Other(String),
    }

    /// Cache capability interface surfaced to nodes.
    pub trait Cache: Capability {
        fn get(&self, key: &[u8]) -> Result<Option<Vec<u8>>, CacheError>;
        fn set(&self, key: &[u8], value: &[u8], ttl: Option<Duration>) -> Result<(), CacheError>;
        fn remove(&self, key: &[u8]) -> Result<(), CacheError>;
    }

    struct Entry {
        value: Vec<u8>,
        expires_at: Option<Instant>,
    }

    /// Simple process-local cache backed by a mutexed HashMap.
    pub struct MemoryCache {
        entries: Mutex<HashMap<Vec<u8>, Entry>>,
    }

    impl MemoryCache {
        pub fn new() -> Self {
            Self {
                entries: Mutex::new(HashMap::new()),
            }
        }

        fn is_expired(expires_at: Option<Instant>) -> bool {
            match expires_at {
                Some(deadline) => Instant::now() > deadline,
                None => false,
            }
        }
    }

    impl Default for MemoryCache {
        fn default() -> Self {
            Self::new()
        }
    }

    impl Capability for MemoryCache {
        fn name(&self) -> &'static str {
            "cache.memory"
        }
    }

    impl Cache for MemoryCache {
        fn get(&self, key: &[u8]) -> Result<Option<Vec<u8>>, CacheError> {
            let mut entries = self.entries.lock().expect("cache mutex poisoned");
            if let Some(entry) = entries.get(key) {
                if Self::is_expired(entry.expires_at) {
                    entries.remove(key);
                    return Ok(None);
                }
                return Ok(Some(entry.value.clone()));
            }
            Ok(None)
        }

        fn set(&self, key: &[u8], value: &[u8], ttl: Option<Duration>) -> Result<(), CacheError> {
            let mut entries = self.entries.lock().expect("cache mutex poisoned");
            let expires_at = ttl.map(|duration| Instant::now() + duration);
            entries.insert(
                key.to_vec(),
                Entry {
                    value: value.to_vec(),
                    expires_at,
                },
            );
            Ok(())
        }

        fn remove(&self, key: &[u8]) -> Result<(), CacheError> {
            let mut entries = self.entries.lock().expect("cache mutex poisoned");
            entries.remove(key);
            Ok(())
        }
    }

    #[cfg(test)]
    mod tests {
        use super::*;
        use std::time::Duration;

        #[test]
        fn remembers_and_expires_values() {
            let cache = MemoryCache::default();
            cache
                .set(b"alpha", b"one", Some(Duration::from_millis(10)))
                .expect("set alpha");
            assert_eq!(
                cache.get(b"alpha").expect("get alpha"),
                Some(b"one".to_vec())
            );
            std::thread::sleep(Duration::from_millis(15));
            assert_eq!(cache.get(b"alpha").expect("get alpha"), None);
        }
    }
}

pub mod db {
    use super::*;

    pub const HINT_DB: &str = "resource::db";
    pub const HINT_DB_READ: &str = "resource::db::read";
    pub const HINT_DB_WRITE: &str = "resource::db::write";

    static REGISTRATION: OnceLock<()> = OnceLock::new();

    /// Register effect/determinism constraints for relational database access.
    pub fn ensure_registered() {
        REGISTRATION.get_or_init(|| {
            dag_core::effects_registry::register_effect_constraint(
                dag_core::effects_registry::EffectConstraint::new(
                    HINT_DB_READ,
                    dag_core::Effects::ReadOnly,
                    "Database reads reach external state; declare effects = ReadOnly or Effectful.",
                ),
            );
            dag_core::effects_registry::register_effect_constraint(
                dag_core::effects_registry::EffectConstraint::new(
                    HINT_DB_WRITE,
                    dag_core::Effects::Effectful,
                    "Database writes mutate external systems; declare effects = Effectful and supply idempotency.",
                ),
            );
            dag_core::determinism::register_determinism_constraint(
                dag_core::determinism::DeterminismConstraint::new(
                    HINT_DB,
                    dag_core::Determinism::BestEffort,
                    "Database results can vary across retries; downgrade determinism or pin revisions.",
                ),
            );
        });
    }

    #[cfg(test)]
    mod tests {
        use super::*;

        #[test]
        fn registers_constraints_once() {
            ensure_registered();
            ensure_registered();
            let read = dag_core::effects_registry::constraint_for_hint(HINT_DB_READ)
                .expect("db read constraint");
            assert_eq!(read.minimum, dag_core::Effects::ReadOnly);
            let write = dag_core::effects_registry::constraint_for_hint(HINT_DB_WRITE)
                .expect("db write constraint");
            assert_eq!(write.minimum, dag_core::Effects::Effectful);
            let det = dag_core::determinism::constraint_for_hint(HINT_DB).expect("db determinism");
            assert_eq!(det.minimum, dag_core::Determinism::BestEffort);
        }
    }
}

pub mod kv {
    use super::*;
    use std::collections::HashMap;

    pub const HINT_KV: &str = "resource::kv";
    pub const HINT_KV_READ: &str = "resource::kv::read";
    pub const HINT_KV_WRITE: &str = "resource::kv::write";

    static REGISTRATION: OnceLock<()> = OnceLock::new();

    pub fn ensure_registered() {
        REGISTRATION.get_or_init(|| {
            dag_core::effects_registry::register_effect_constraint(
                dag_core::effects_registry::EffectConstraint::new(
                    HINT_KV_READ,
                    dag_core::Effects::ReadOnly,
                    "KV reads access external state; declare effects = ReadOnly or stronger.",
                ),
            );
            dag_core::effects_registry::register_effect_constraint(
                dag_core::effects_registry::EffectConstraint::new(
                    HINT_KV_WRITE,
                    dag_core::Effects::Effectful,
                    "KV writes are effectful; declare effects = Effectful and ensure dedupe/idempotency.",
                ),
            );
            dag_core::determinism::register_determinism_constraint(
                dag_core::determinism::DeterminismConstraint::new(
                    HINT_KV,
                    dag_core::Determinism::BestEffort,
                    "KV values may change between executions; downgrade determinism or pin versions.",
                ),
            );
        });
    }

    /// Errors surfaced by key-value capabilities.
    #[derive(Debug, thiserror::Error)]
    pub enum KvError {
        #[error("value not found")]
        NotFound,
        #[error("operation failed: {0}")]
        Other(String),
    }

    /// Generic key-value interface exposed to nodes.
    #[async_trait]
    pub trait KeyValue: Capability {
        async fn get(&self, key: &str) -> Result<Option<Vec<u8>>, KvError>;
        async fn put(&self, key: &str, value: &[u8], ttl: Option<Duration>) -> Result<(), KvError>;
        async fn delete(&self, key: &str) -> Result<(), KvError>;
    }

    struct KvEntry {
        value: Vec<u8>,
        expires_at: Option<Instant>,
    }

    /// Simple in-memory KV store for tests and local dev.
    pub struct MemoryKv {
        entries: Mutex<HashMap<String, KvEntry>>,
    }

    impl MemoryKv {
        pub fn new() -> Self {
            Self {
                entries: Mutex::new(HashMap::new()),
            }
        }

        fn is_expired(expires_at: Option<Instant>) -> bool {
            match expires_at {
                Some(deadline) => Instant::now() > deadline,
                None => false,
            }
        }
    }

    impl Default for MemoryKv {
        fn default() -> Self {
            Self::new()
        }
    }

    impl Capability for MemoryKv {
        fn name(&self) -> &'static str {
            "kv.memory"
        }
    }

    #[async_trait]
    impl KeyValue for MemoryKv {
        async fn get(&self, key: &str) -> Result<Option<Vec<u8>>, KvError> {
            let mut entries = self.entries.lock().expect("kv mutex poisoned");
            if let Some(entry) = entries.get(key) {
                if Self::is_expired(entry.expires_at) {
                    entries.remove(key);
                    return Ok(None);
                }
                return Ok(Some(entry.value.clone()));
            }
            Ok(None)
        }

        async fn put(&self, key: &str, value: &[u8], ttl: Option<Duration>) -> Result<(), KvError> {
            let mut entries = self.entries.lock().expect("kv mutex poisoned");
            let expires_at = ttl.map(|duration| Instant::now() + duration);
            entries.insert(
                key.to_owned(),
                KvEntry {
                    value: value.to_vec(),
                    expires_at,
                },
            );
            Ok(())
        }

        async fn delete(&self, key: &str) -> Result<(), KvError> {
            let mut entries = self.entries.lock().expect("kv mutex poisoned");
            entries.remove(key).map(|_| ()).ok_or(KvError::NotFound)
        }
    }

    #[cfg(test)]
    mod tests {
        use super::*;

        #[test]
        fn registers_constraints_once() {
            ensure_registered();
            ensure_registered();
            let read = dag_core::effects_registry::constraint_for_hint(HINT_KV_READ)
                .expect("kv read constraint");
            assert_eq!(read.minimum, dag_core::Effects::ReadOnly);
            let write = dag_core::effects_registry::constraint_for_hint(HINT_KV_WRITE)
                .expect("kv write constraint");
            assert_eq!(write.minimum, dag_core::Effects::Effectful);
            let det = dag_core::determinism::constraint_for_hint(HINT_KV).expect("kv determinism");
            assert_eq!(det.minimum, dag_core::Determinism::BestEffort);
        }

        #[tokio::test]
        async fn memory_kv_honours_ttl() {
            let kv = MemoryKv::new();
            kv.put("k", b"v", Some(Duration::from_millis(5)))
                .await
                .expect("put");
            assert_eq!(kv.get("k").await.expect("get"), Some(b"v".to_vec()));
            tokio::time::sleep(Duration::from_millis(10)).await;
            assert_eq!(kv.get("k").await.expect("expired"), None);
        }

        #[tokio::test]
        async fn memory_kv_delete_returns_not_found() {
            let kv = MemoryKv::new();
            assert!(matches!(kv.delete("missing").await, Err(KvError::NotFound)));
        }
    }
}

pub mod blob {
    use super::*;
    use std::collections::HashMap;

    pub const HINT_BLOB: &str = "resource::blob";
    pub const HINT_BLOB_READ: &str = "resource::blob::read";
    pub const HINT_BLOB_WRITE: &str = "resource::blob::write";

    static REGISTRATION: OnceLock<()> = OnceLock::new();

    pub fn ensure_registered() {
        REGISTRATION.get_or_init(|| {
            dag_core::effects_registry::register_effect_constraint(
                dag_core::effects_registry::EffectConstraint::new(
                    HINT_BLOB_READ,
                    dag_core::Effects::ReadOnly,
                    "Blob reads access external storage; declare effects = ReadOnly or stronger.",
                ),
            );
            dag_core::effects_registry::register_effect_constraint(
                dag_core::effects_registry::EffectConstraint::new(
                    HINT_BLOB_WRITE,
                    dag_core::Effects::Effectful,
                    "Blob writes mutate external storage; declare effects = Effectful and supply idempotency.",
                ),
            );
            dag_core::determinism::register_determinism_constraint(
                dag_core::determinism::DeterminismConstraint::new(
                    HINT_BLOB,
                    dag_core::Determinism::BestEffort,
                    "Blob storage responses can change over time; downgrade determinism or pin versions.",
                ),
            );
        });
    }

    /// Errors exposed by blob storage capabilities.
    #[derive(Debug, thiserror::Error)]
    pub enum BlobError {
        #[error("object not found")]
        NotFound,
        #[error("operation failed: {0}")]
        Other(String),
    }

    /// Blob storage interface consumed by nodes.
    #[async_trait]
    pub trait BlobStore: Capability {
        async fn get(&self, key: &str) -> Result<Option<Vec<u8>>, BlobError>;
        async fn put(&self, key: &str, contents: &[u8]) -> Result<(), BlobError>;
        async fn delete(&self, key: &str) -> Result<(), BlobError>;
    }

    /// In-memory blob store for tests and local workflows.
    pub struct MemoryBlobStore {
        objects: Mutex<HashMap<String, Vec<u8>>>,
    }

    impl MemoryBlobStore {
        pub fn new() -> Self {
            Self {
                objects: Mutex::new(HashMap::new()),
            }
        }
    }

    impl Default for MemoryBlobStore {
        fn default() -> Self {
            Self::new()
        }
    }

    impl Capability for MemoryBlobStore {
        fn name(&self) -> &'static str {
            "blob.memory"
        }
    }

    #[async_trait]
    impl BlobStore for MemoryBlobStore {
        async fn get(&self, key: &str) -> Result<Option<Vec<u8>>, BlobError> {
            let objects = self.objects.lock().expect("blob mutex poisoned");
            Ok(objects.get(key).cloned())
        }

        async fn put(&self, key: &str, contents: &[u8]) -> Result<(), BlobError> {
            let mut objects = self.objects.lock().expect("blob mutex poisoned");
            objects.insert(key.to_owned(), contents.to_vec());
            Ok(())
        }

        async fn delete(&self, key: &str) -> Result<(), BlobError> {
            let mut objects = self.objects.lock().expect("blob mutex poisoned");
            objects.remove(key).map(|_| ()).ok_or(BlobError::NotFound)
        }
    }

    #[cfg(test)]
    mod tests {
        use super::*;

        #[test]
        fn registers_constraints_once() {
            ensure_registered();
            ensure_registered();
            let read = dag_core::effects_registry::constraint_for_hint(HINT_BLOB_READ)
                .expect("blob read constraint");
            assert_eq!(read.minimum, dag_core::Effects::ReadOnly);
            let write = dag_core::effects_registry::constraint_for_hint(HINT_BLOB_WRITE)
                .expect("blob write constraint");
            assert_eq!(write.minimum, dag_core::Effects::Effectful);
            let det =
                dag_core::determinism::constraint_for_hint(HINT_BLOB).expect("blob determinism");
            assert_eq!(det.minimum, dag_core::Determinism::BestEffort);
        }

        #[tokio::test]
        async fn memory_blob_round_trip() {
            let store = MemoryBlobStore::new();
            assert_eq!(store.get("missing").await.unwrap(), None);
            store.put("key", b"bytes").await.unwrap();
            assert_eq!(store.get("key").await.unwrap(), Some(b"bytes".to_vec()));
            store.delete("key").await.unwrap();
            assert_eq!(store.get("key").await.unwrap(), None);
        }

        #[tokio::test]
        async fn memory_blob_delete_missing() {
            let store = MemoryBlobStore::new();
            assert!(matches!(store.delete("missing").await, Err(BlobError::NotFound)));
        }
    }
}

pub mod queue {
    use super::*;
    use std::collections::VecDeque;

    pub const HINT_QUEUE: &str = "resource::queue";
    pub const HINT_QUEUE_PUBLISH: &str = "resource::queue::publish";
    pub const HINT_QUEUE_CONSUME: &str = "resource::queue::consume";

    static REGISTRATION: OnceLock<()> = OnceLock::new();

    pub fn ensure_registered() {
        REGISTRATION.get_or_init(|| {
            dag_core::effects_registry::register_effect_constraint(
                dag_core::effects_registry::EffectConstraint::new(
                    HINT_QUEUE_PUBLISH,
                    dag_core::Effects::Effectful,
                    "Queue publishes are effectful; ensure effects = Effectful with dedupe keys.",
                ),
            );
            dag_core::effects_registry::register_effect_constraint(
                dag_core::effects_registry::EffectConstraint::new(
                    HINT_QUEUE_CONSUME,
                    dag_core::Effects::ReadOnly,
                    "Queue consumption acknowledges messages; treat as at least ReadOnly.",
                ),
            );
            dag_core::determinism::register_determinism_constraint(
                dag_core::determinism::DeterminismConstraint::new(
                    HINT_QUEUE,
                    dag_core::Determinism::BestEffort,
                    "Queue ordering and visibility vary; downgrade determinism or add sequence checks.",
                ),
            );
        });
    }

    /// Errors raised by queue capabilities.
    #[derive(Debug, thiserror::Error)]
    pub enum QueueError {
        #[error("operation failed: {0}")]
        Other(String),
    }

    /// Queue capability interface covering enqueue/dequeue.
    pub trait Queue: Capability {
        fn enqueue(&self, payload: Vec<u8>) -> Result<(), QueueError>;
        fn dequeue(&self) -> Result<Option<Vec<u8>>, QueueError>;
    }

    /// In-memory queue backed by VecDeque for tests.
    pub struct MemoryQueue {
        entries: Mutex<VecDeque<Vec<u8>>>,
    }

    impl MemoryQueue {
        pub fn new() -> Self {
            Self {
                entries: Mutex::new(VecDeque::new()),
            }
        }
    }

    impl Default for MemoryQueue {
        fn default() -> Self {
            Self::new()
        }
    }

    impl Capability for MemoryQueue {
        fn name(&self) -> &'static str {
            "queue.memory"
        }
    }

    impl Queue for MemoryQueue {
        fn enqueue(&self, payload: Vec<u8>) -> Result<(), QueueError> {
            let mut entries = self.entries.lock().expect("queue mutex poisoned");
            entries.push_back(payload);
            Ok(())
        }

        fn dequeue(&self) -> Result<Option<Vec<u8>>, QueueError> {
            let mut entries = self.entries.lock().expect("queue mutex poisoned");
            Ok(entries.pop_front())
        }
    }

    #[cfg(test)]
    mod tests {
        use super::*;

        #[test]
        fn registers_constraints_once() {
            ensure_registered();
            ensure_registered();
            let publish = dag_core::effects_registry::constraint_for_hint(HINT_QUEUE_PUBLISH)
                .expect("queue publish constraint");
            assert_eq!(publish.minimum, dag_core::Effects::Effectful);
            let consume = dag_core::effects_registry::constraint_for_hint(HINT_QUEUE_CONSUME)
                .expect("queue consume constraint");
            assert_eq!(consume.minimum, dag_core::Effects::ReadOnly);
            let det =
                dag_core::determinism::constraint_for_hint(HINT_QUEUE).expect("queue determinism");
            assert_eq!(det.minimum, dag_core::Determinism::BestEffort);
        }

        #[test]
        fn memory_queue_round_trip() {
            let queue = MemoryQueue::new();
            assert_eq!(queue.dequeue().unwrap(), None);
            queue.enqueue(b"a".to_vec()).unwrap();
            queue.enqueue(b"b".to_vec()).unwrap();
            assert_eq!(queue.dequeue().unwrap(), Some(b"a".to_vec()));
            assert_eq!(queue.dequeue().unwrap(), Some(b"b".to_vec()));
            assert_eq!(queue.dequeue().unwrap(), None);
        }
    }
}

pub mod rng {
    use super::*;

    pub const HINT_RNG: &str = "resource::rng";

    static REGISTRATION: OnceLock<()> = OnceLock::new();

    pub fn ensure_registered() {
        REGISTRATION.get_or_init(|| {
            dag_core::determinism::register_determinism_constraint(
                dag_core::determinism::DeterminismConstraint::new(
                    HINT_RNG,
                    dag_core::Determinism::BestEffort,
                    "Randomness is nondeterministic; downgrade determinism or inject fixed seeds.",
                ),
            );
        });
    }

    #[cfg(test)]
    mod tests {
        use super::*;

        #[test]
        fn registers_constraints_once() {
            ensure_registered();
            ensure_registered();
            let det =
                dag_core::determinism::constraint_for_hint(HINT_RNG).expect("rng determinism");
            assert_eq!(det.minimum, dag_core::Determinism::BestEffort);
        }
    }
}

#[cfg(test)]
mod bag_tests {
    use super::*;
    use std::sync::Arc;

    struct NullHttp;

    #[async_trait]
    impl http::HttpRead for NullHttp {
        async fn send(&self, _request: http::HttpRequest) -> http::HttpResult<http::HttpResponse> {
            Err(http::HttpError::Timeout(0))
        }
    }

    #[async_trait]
    impl http::HttpWrite for NullHttp {
        async fn send(&self, _request: http::HttpRequest) -> http::HttpResult<http::HttpResponse> {
            Err(http::HttpError::Timeout(0))
        }
    }

    #[test]
    fn resource_bag_exposes_capabilities() {
        clock::ensure_registered();
        let bag = ResourceBag::new()
            .with_http_read(Arc::new(NullHttp))
            .with_http_write(Arc::new(NullHttp))
            .with_clock(Arc::new(clock::SystemClock::default()))
            .with_cache(Arc::new(cache::MemoryCache::default()))
            .with_kv(Arc::new(kv::MemoryKv::new()))
            .with_blob(Arc::new(blob::MemoryBlobStore::new()))
            .with_queue(Arc::new(queue::MemoryQueue::new()));

        assert!(bag.http_read().is_some());
        assert!(bag.http_write().is_some());
        assert!(bag.clock().is_some());
        assert!(bag.cache().is_some());
        assert!(bag.kv().is_some());
        assert!(bag.blob().is_some());
        assert!(bag.queue().is_some());
    }

    #[test]
    fn context_scopes_resources() {
        use crate::kv::KeyValue;
        let kv_store = Arc::new(kv::MemoryKv::new());
        let bag = ResourceBag::new().with_kv(kv_store.clone());
        let rt = tokio::runtime::Runtime::new().expect("tokio runtime");
        rt.block_on(async {
            let bag_arc = Arc::new(bag);
            context::with_resources(bag_arc.clone(), async {
                context::with_current_async(|resources| async move {
                    let kv = resources.kv().expect("kv capability available");
                    kv.put("ctx", b"value", None).await.expect("kv put");
                })
                .await
                .expect("resource scope available");
            })
            .await;
        });

        assert_eq!(
            rt.block_on(async { kv_store.get("ctx").await.expect("kv get after scope") }),
            Some(b"value".to_vec())
        );
        assert!(context::with_current::<_, ()>(|_| ()).is_none());
    }
}
````

## File: crates/dag-core/src/builder.rs
````rust
use std::collections::BTreeMap;

use semver::Version;

use crate::ir::{
    BufferPolicy, Delivery, EdgeIR, FlowIR, FlowId, NodeIR, NodeId, NodeSpec, Profile,
};

/// Errors produced by the flow builder.
#[derive(Debug, thiserror::Error)]
pub enum FlowBuilderError {
    /// Attempted to add a node with a duplicate alias.
    #[error("node alias `{0}` already exists in workflow")]
    DuplicateNode(String),
}

/// Handle referencing a node added to the builder.
#[derive(Debug, Clone)]
pub struct NodeHandle {
    alias: String,
}

impl NodeHandle {
    /// Access node alias.
    pub fn alias(&self) -> &str {
        &self.alias
    }
}

/// Handle referencing an edge created by the builder.
#[derive(Debug, Clone)]
pub struct EdgeHandle {
    pub(crate) from: String,
    pub(crate) to: String,
}

/// Builder used by macros to construct Flow IR instances.
pub struct FlowBuilder {
    flow: FlowIR,
    alias_map: BTreeMap<String, NodeId>,
}

impl FlowBuilder {
    /// Create a new builder for the given workflow.
    pub fn new(name: impl Into<String>, version: Version, profile: Profile) -> Self {
        let name = name.into();
        let id = FlowId::new(&name, &version);
        let flow = FlowIR {
            id,
            name,
            version,
            profile,
            summary: None,
            nodes: Vec::new(),
            edges: Vec::new(),
            control_surfaces: Vec::new(),
            checkpoints: Vec::new(),
            policies: Default::default(),
            metadata: Default::default(),
            artifacts: Vec::new(),
        };
        Self {
            flow,
            alias_map: BTreeMap::new(),
        }
    }

    /// Attach a summary to the workflow.
    pub fn summary(&mut self, summary: Option<impl Into<String>>) {
        self.flow.summary = summary.map(Into::into);
    }

    /// Add a node to the workflow.
    pub fn add_node(
        &mut self,
        alias: impl Into<String>,
        spec: &NodeSpec,
    ) -> Result<NodeHandle, FlowBuilderError> {
        let alias = alias.into();
        if self.alias_map.contains_key(&alias) {
            return Err(FlowBuilderError::DuplicateNode(alias));
        }

        let node_id = NodeId::new(format!("{}::{}", self.flow.id.as_str(), alias));
        let node_ir = NodeIR {
            id: node_id.clone(),
            alias: alias.clone(),
            identifier: spec.identifier.to_string(),
            name: spec.name.to_string(),
            kind: spec.kind,
            summary: spec.summary.map(|s| s.to_string()),
            in_schema: spec.in_schema.into_ref(),
            out_schema: spec.out_schema.into_ref(),
            effects: spec.effects,
            determinism: spec.determinism,
            idempotency: Default::default(),
            determinism_hints: spec
                .determinism_hints
                .iter()
                .map(|hint| hint.to_string())
                .collect(),
            effect_hints: spec
                .effect_hints
                .iter()
                .map(|hint| hint.to_string())
                .collect(),
        };
        self.flow.nodes.push(node_ir);
        self.alias_map.insert(alias.clone(), node_id);
        Ok(NodeHandle { alias })
    }

    /// Create an edge between two node handles.
    pub fn connect(&mut self, from: &NodeHandle, to: &NodeHandle) -> EdgeHandle {
        self.flow.edges.push(EdgeIR {
            from: from.alias.clone(),
            to: to.alias.clone(),
            delivery: Delivery::AtLeastOnce,
            ordering: Default::default(),
            partition_key: None,
            timeout_ms: None,
            buffer: BufferPolicy::default(),
        });
        EdgeHandle {
            from: from.alias.clone(),
            to: to.alias.clone(),
        }
    }

    /// Finalise and return the constructed Flow IR.
    pub fn build(self) -> FlowIR {
        self.flow
    }
}
````

## File: crates/dag-core/src/effects.rs
````rust
use schemars::JsonSchema;
use serde::{Deserialize, Serialize};
use thiserror::Error;

/// Effect lattice describing resource access.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize, JsonSchema)]
#[serde(rename_all = "snake_case")]
pub enum Effects {
    /// No observable side-effects.
    Pure,
    /// Read-only side-effects (e.g. cache lookups).
    ReadOnly,
    /// External side-effects (e.g. network, writes).
    Effectful,
}

impl Default for Effects {
    fn default() -> Self {
        Effects::Effectful
    }
}

impl Effects {
    /// Convert effects level into a comparable rank (higher is more permissive).
    pub const fn rank(self) -> u8 {
        match self {
            Effects::Pure => 0,
            Effects::ReadOnly => 1,
            Effects::Effectful => 2,
        }
    }

    /// Returns true if `self` is at least as permissive as `minimum`.
    pub const fn is_at_least(self, minimum: Effects) -> bool {
        self.rank() >= minimum.rank()
    }

    /// Human-readable representation of the effect level.
    pub const fn as_str(self) -> &'static str {
        match self {
            Effects::Pure => "Pure",
            Effects::ReadOnly => "ReadOnly",
            Effects::Effectful => "Effectful",
        }
    }
}

/// Determinism lattice describing replay guarantees.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize, JsonSchema)]
#[serde(rename_all = "snake_case")]
pub enum Determinism {
    /// Fully deterministic, no time or randomness.
    Strict,
    /// Stable under pinned resources.
    Stable,
    /// Best-effort determinism, may vary on retries.
    BestEffort,
    /// Explicitly non-deterministic.
    Nondeterministic,
}

impl Default for Determinism {
    fn default() -> Self {
        Determinism::BestEffort
    }
}

impl Determinism {
    /// Convert determinism level into a comparable rank (higher is less strict).
    pub const fn rank(self) -> u8 {
        match self {
            Determinism::Strict => 0,
            Determinism::Stable => 1,
            Determinism::BestEffort => 2,
            Determinism::Nondeterministic => 3,
        }
    }

    /// Returns true if `self` is at least as permissive as `minimum`.
    pub const fn is_at_least(self, minimum: Determinism) -> bool {
        self.rank() >= minimum.rank()
    }

    /// Human-readable representation used in diagnostics.
    pub const fn as_str(self) -> &'static str {
        match self {
            Determinism::Strict => "Strict",
            Determinism::Stable => "Stable",
            Determinism::BestEffort => "BestEffort",
            Determinism::Nondeterministic => "Nondeterministic",
        }
    }
}

/// Canonical node error type surfaced during execution.
#[derive(Debug, Error)]
pub enum NodeError {
    /// Generic error message.
    #[error("{0}")]
    Message(String),
}

impl NodeError {
    /// Construct a node error from displayable content.
    pub fn new(message: impl Into<String>) -> Self {
        NodeError::Message(message.into())
    }
}

impl From<anyhow::Error> for NodeError {
    fn from(err: anyhow::Error) -> Self {
        NodeError::Message(err.to_string())
    }
}

/// Convenient result alias for node execution.
pub type NodeResult<T> = Result<T, NodeError>;
````

## File: crates/dag-core/src/ir.rs
````rust
use schemars::JsonSchema;
use semver::Version;
use serde::{Deserialize, Serialize};
use uuid::Uuid;

use crate::effects::{Determinism, Effects};

mod version_serde {
    use semver::Version;
    use serde::de::Error as DeError;
    use serde::{Deserialize, Deserializer, Serializer};

    pub fn serialize<S>(version: &Version, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        serializer.serialize_str(&version.to_string())
    }

    pub fn deserialize<'de, D>(deserializer: D) -> Result<Version, D::Error>
    where
        D: Deserializer<'de>,
    {
        let s = String::deserialize(deserializer)?;
        Version::parse(&s).map_err(DeError::custom)
    }
}

/// Unique identifier for a workflow.
#[derive(
    Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize, JsonSchema, PartialOrd, Ord,
)]
#[serde(transparent)]
pub struct FlowId(pub String);

impl FlowId {
    /// Deterministically derive a flow id from the workflow name and semantic version.
    pub fn new(name: &str, version: &Version) -> Self {
        let namespace = Uuid::new_v5(&Uuid::NAMESPACE_DNS, b"latticeflow.flow");
        let key = format!("{name}:{version}");
        let uuid = Uuid::new_v5(&namespace, key.as_bytes());
        Self(uuid.to_string())
    }

    /// Access the underlying UUID.
    pub fn as_str(&self) -> &str {
        &self.0
    }
}

/// Unique identifier for a node inside a workflow.
#[derive(
    Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize, JsonSchema, PartialOrd, Ord,
)]
#[serde(transparent)]
pub struct NodeId(pub String);

impl NodeId {
    /// Construct a node id.
    pub fn new(value: impl Into<String>) -> Self {
        Self(value.into())
    }
}

/// Workflow execution profile.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize, JsonSchema)]
#[serde(rename_all = "snake_case")]
pub enum Profile {
    /// HTTP/Axum host.
    Web,
    /// Queue/Redis-backed workers.
    Queue,
    /// Temporal orchestration.
    Temporal,
    /// WASM/Edge runtime.
    Wasm,
    /// Local developer profile.
    Dev,
}

impl Default for Profile {
    fn default() -> Self {
        Profile::Dev
    }
}

/// High-level node categories.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize, JsonSchema)]
#[serde(rename_all = "snake_case")]
pub enum NodeKind {
    /// Trigger originating the workflow.
    Trigger,
    /// Inline rust node.
    Inline,
    /// Activity/connector node.
    Activity,
    /// Subflow invocation.
    Subflow,
}

/// Schema reference used within Flow IR.
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize, JsonSchema)]
#[serde(tag = "kind", rename_all = "snake_case")]
pub enum SchemaRef {
    /// Schema is opaque/unknown.
    Opaque,
    /// Strongly named schema reference.
    Named { name: String },
}

impl SchemaRef {
    /// Construct a named schema.
    pub fn named(name: impl Into<String>) -> Self {
        SchemaRef::Named { name: name.into() }
    }
}

/// Compile-time schema reference emitted by macros.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum SchemaSpec {
    /// Opaque schema.
    Opaque,
    /// Named schema reference.
    Named(&'static str),
}

impl SchemaSpec {
    /// Convert into owned Flow IR schema representation.
    pub fn into_ref(self) -> SchemaRef {
        match self {
            SchemaSpec::Opaque => SchemaRef::Opaque,
            SchemaSpec::Named(name) => SchemaRef::named(name),
        }
    }
}

/// Compile-time node specification produced by macros.
#[derive(Debug, Clone)]
pub struct NodeSpec {
    /// Stable identifier (generally module path + function/struct name).
    pub identifier: &'static str,
    /// Human friendly name surfaced to authors.
    pub name: &'static str,
    /// Node category.
    pub kind: NodeKind,
    /// Optional short description.
    pub summary: Option<&'static str>,
    /// Input schema information.
    pub in_schema: SchemaSpec,
    /// Output schema information.
    pub out_schema: SchemaSpec,
    /// Declared effects metadata.
    pub effects: Effects,
    /// Declared determinism metadata.
    pub determinism: Determinism,
    /// Determinism hints emitted by macros or plugins.
    pub determinism_hints: &'static [&'static str],
    /// Effect hints emitted by macros or plugins.
    pub effect_hints: &'static [&'static str],
}

impl NodeSpec {
    /// Helper for inline nodes.
    pub const fn inline(
        identifier: &'static str,
        name: &'static str,
        in_schema: SchemaSpec,
        out_schema: SchemaSpec,
        effects: Effects,
        determinism: Determinism,
        summary: Option<&'static str>,
    ) -> Self {
        Self::inline_with_hints(
            identifier,
            name,
            in_schema,
            out_schema,
            effects,
            determinism,
            summary,
            &[],
            &[],
        )
    }

    /// Helper for inline nodes with explicit determinism hints.
    pub const fn inline_with_hints(
        identifier: &'static str,
        name: &'static str,
        in_schema: SchemaSpec,
        out_schema: SchemaSpec,
        effects: Effects,
        determinism: Determinism,
        summary: Option<&'static str>,
        determinism_hints: &'static [&'static str],
        effect_hints: &'static [&'static str],
    ) -> Self {
        Self {
            identifier,
            name,
            kind: NodeKind::Inline,
            summary,
            in_schema,
            out_schema,
            effects,
            determinism,
            determinism_hints,
            effect_hints,
        }
    }
}

/// Canonical Flow IR structure serialised to JSON/dot/etc.
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct FlowIR {
    /// Unique workflow identifier.
    pub id: FlowId,
    /// Workflow display name.
    pub name: String,
    /// Semantic version.
    #[serde(with = "version_serde")]
    #[schemars(with = "String")]
    pub version: Version,
    /// Target profile.
    pub profile: Profile,
    /// Optional human readable summary.
    #[serde(skip_serializing_if = "Option::is_none")]
    pub summary: Option<String>,
    /// Nodes contained in the workflow.
    #[serde(default)]
    pub nodes: Vec<NodeIR>,
    /// Edges describing the DAG.
    #[serde(default)]
    pub edges: Vec<EdgeIR>,
    /// Control surface metadata (branching/loops/etc).
    #[serde(default)]
    pub control_surfaces: Vec<ControlSurfaceIR>,
    /// Declared checkpoints.
    #[serde(default)]
    pub checkpoints: Vec<CheckpointIR>,
    /// Policy metadata.
    #[serde(default)]
    pub policies: FlowPolicies,
    /// Documentation and tag metadata.
    #[serde(default)]
    pub metadata: FlowMetadata,
    /// Associated artifact references (DOT, WIT, etc.).
    #[serde(default)]
    pub artifacts: Vec<ArtifactRef>,
}

impl FlowIR {
    /// Convenience accessor to find a node by alias.
    pub fn node(&self, alias: &str) -> Option<&NodeIR> {
        self.nodes.iter().find(|n| n.alias == alias)
    }
}

/// Node entry within the Flow IR.
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct NodeIR {
    /// Stable identifier derived from the spec.
    pub id: NodeId,
    /// Alias used within the workflow definition.
    pub alias: String,
    /// Fully-qualified implementation identifier used for runtime lookup.
    pub identifier: String,
    /// Human readable name.
    pub name: String,
    /// Node kind.
    pub kind: NodeKind,
    /// Optional summary/description.
    #[serde(skip_serializing_if = "Option::is_none")]
    pub summary: Option<String>,
    /// Input schema reference.
    pub in_schema: SchemaRef,
    /// Output schema reference.
    pub out_schema: SchemaRef,
    /// Declared effects metadata.
    pub effects: Effects,
    /// Declared determinism metadata.
    pub determinism: Determinism,
    /// Optional idempotency configuration.
    #[serde(default)]
    pub idempotency: IdempotencySpec,
    /// Determinism hints recorded during macro expansion.
    #[serde(rename = "determinismHints", default)]
    pub determinism_hints: Vec<String>,
    /// Effect hints recorded during macro expansion.
    #[serde(rename = "effectHints", default)]
    pub effect_hints: Vec<String>,
}

/// Edge entry within the Flow IR.
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct EdgeIR {
    /// Source node alias.
    pub from: String,
    /// Destination node alias.
    pub to: String,
    /// Delivery semantics.
    pub delivery: Delivery,
    /// Ordering semantics.
    pub ordering: Ordering,
    /// Optional partition key expression.
    #[serde(skip_serializing_if = "Option::is_none")]
    pub partition_key: Option<String>,
    /// Optional timeout in milliseconds.
    #[serde(skip_serializing_if = "Option::is_none")]
    pub timeout_ms: Option<u64>,
    /// Buffer policy.
    pub buffer: BufferPolicy,
}

impl Default for EdgeIR {
    fn default() -> Self {
        Self {
            from: String::new(),
            to: String::new(),
            delivery: Delivery::AtLeastOnce,
            ordering: Ordering::Ordered,
            partition_key: None,
            timeout_ms: None,
            buffer: BufferPolicy::default(),
        }
    }
}

/// Delivery semantics enumeration.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize, JsonSchema)]
#[serde(rename_all = "snake_case")]
pub enum Delivery {
    /// At least once delivery (default).
    AtLeastOnce,
    /// At most once delivery.
    AtMostOnce,
    /// Exactly once delivery (requires dedupe).
    ExactlyOnce,
}

impl Default for Delivery {
    fn default() -> Self {
        Delivery::AtLeastOnce
    }
}

/// Edge ordering semantics.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize, JsonSchema)]
#[serde(rename_all = "snake_case")]
pub enum Ordering {
    /// FIFO semantics per partition.
    Ordered,
    /// No ordering guarantees.
    Unordered,
}

impl Default for Ordering {
    fn default() -> Self {
        Ordering::Ordered
    }
}

/// Buffering behaviour metadata.
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct BufferPolicy {
    /// Optional max items held in memory.
    #[serde(skip_serializing_if = "Option::is_none")]
    pub max_items: Option<u32>,
    /// Optional spill threshold in bytes.
    #[serde(skip_serializing_if = "Option::is_none")]
    pub spill_threshold_bytes: Option<u64>,
    /// Optional spill tier identifier.
    #[serde(skip_serializing_if = "Option::is_none")]
    pub spill_tier: Option<String>,
    /// Drop behaviour description.
    #[serde(skip_serializing_if = "Option::is_none")]
    pub on_drop: Option<String>,
}

impl Default for BufferPolicy {
    fn default() -> Self {
        Self {
            max_items: None,
            spill_threshold_bytes: None,
            spill_tier: None,
            on_drop: None,
        }
    }
}

/// Control surface metadata for branching/looping constructs.
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct ControlSurfaceIR {
    /// Stable identifier for the control surface.
    pub id: String,
    /// Control surface type.
    pub kind: ControlSurfaceKind,
    /// Target node/edge aliases.
    pub targets: Vec<String>,
    /// JSON payload describing surface-specific configuration.
    #[serde(default, skip_serializing_if = "serde_json::Value::is_null")]
    pub config: serde_json::Value,
}

impl Default for ControlSurfaceIR {
    fn default() -> Self {
        Self {
            id: String::new(),
            kind: ControlSurfaceKind::Switch,
            targets: Vec::new(),
            config: serde_json::Value::Null,
        }
    }
}

/// Supported control surface variants.
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
#[serde(rename_all = "snake_case")]
pub enum ControlSurfaceKind {
    /// Switch/multi-branch control flow.
    Switch,
    /// Binary branching (if).
    If,
    /// Loop construct.
    Loop,
    /// For-each iteration construct.
    ForEach,
    /// Windowing configuration.
    Window,
    /// Partition description.
    Partition,
    /// Timeout/latency guard.
    Timeout,
    /// Rate limit guard.
    RateLimit,
    /// Error-handling surface.
    ErrorHandler,
}

impl Default for ControlSurfaceKind {
    fn default() -> Self {
        ControlSurfaceKind::Switch
    }
}

/// Checkpoint definition.
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema, Default)]
pub struct CheckpointIR {
    /// Checkpoint identifier.
    pub id: String,
    /// Optional summary.
    #[serde(skip_serializing_if = "Option::is_none")]
    pub summary: Option<String>,
}

/// Policy lint configuration.
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema, Default)]
pub struct FlowPolicies {
    /// Lint configuration.
    #[serde(default)]
    pub lint: PolicyLintSettings,
}

/// Lint-specific policy settings.
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema, Default)]
pub struct PolicyLintSettings {
    /// Whether control surface hints are required.
    pub require_control_hints: bool,
}

/// Arbitrary metadata describing the workflow.
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema, Default)]
pub struct FlowMetadata {
    /// Optional tags associated with the workflow.
    #[serde(default)]
    pub tags: Vec<String>,
}

/// Artifact reference bundled alongside Flow IR.
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct ArtifactRef {
    /// Artifact kind (dot, wit, json-schema, etc.).
    pub kind: String,
    /// Relative path to the artifact.
    pub path: String,
    /// Optional format identifier.
    #[serde(skip_serializing_if = "Option::is_none")]
    pub format: Option<String>,
}

/// Idempotency metadata for a node.
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema, Default)]
pub struct IdempotencySpec {
    /// Canonical idempotency key expression.
    #[serde(skip_serializing_if = "Option::is_none")]
    pub key: Option<String>,
    /// Scope for the idempotency key.
    #[serde(skip_serializing_if = "Option::is_none")]
    pub scope: Option<IdempotencyScope>,
    /// Optional TTL (in milliseconds) for dedupe reservations.
    #[serde(rename = "ttlMs", skip_serializing_if = "Option::is_none")]
    pub ttl_ms: Option<u64>,
}

/// Supported idempotency scopes.
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
#[serde(rename_all = "snake_case")]
pub enum IdempotencyScope {
    /// Key applies to a specific node.
    Node,
    /// Key applies to a specific edge.
    Edge,
    /// Key applies to a partition.
    Partition,
}
````

## File: crates/dag-core/src/lib.rs
````rust
//! Core types, diagnostics, and Flow IR structures for LatticeFlow.

mod builder;
pub mod determinism;
mod diagnostics;
mod effects;
pub mod effects_registry;
mod ir;

pub use builder::{EdgeHandle, FlowBuilder, FlowBuilderError, NodeHandle};
pub use diagnostics::{DIAGNOSTIC_CODES, Diagnostic, DiagnosticCode, Severity, diagnostic_codes};
pub use effects::{Determinism, Effects, NodeError, NodeResult};
pub use ir::*;

/// Convenient prelude re-exporting the most commonly used items.
pub mod prelude {
    pub use crate::builder::{FlowBuilder, FlowBuilderError};
    pub use crate::effects::{Determinism, Effects, NodeError, NodeResult};
    pub use crate::ir::{
        BufferPolicy, Delivery, FlowIR, FlowId, NodeId, NodeKind, NodeSpec, Profile, SchemaRef,
        SchemaSpec,
    };
    pub use semver::Version;
}

#[cfg(test)]
mod tests {
    use std::fs;
    use std::path::PathBuf;

    use semver::Version;
    use serde_json::json;

    use super::*;

    #[test]
    fn diagnostics_registry_matches_doc() {
        let mut doc_path = PathBuf::from(env!("CARGO_MANIFEST_DIR"));
        doc_path.pop(); // crates/
        doc_path.push("../impl-docs/error-codes.md");
        let registry_doc = fs::read_to_string(doc_path).expect("read error-code registry doc");

        for code in diagnostic_codes() {
            assert!(
                registry_doc.contains(code.code),
                "diagnostic code `{}` missing from registry document",
                code.code
            );
        }
    }

    #[test]
    fn flow_builder_constructs_serializable_ir() {
        let version = Version::parse("1.0.0").expect("parse version");
        let mut builder = FlowBuilder::new("echo_norm", version, Profile::Web);
        builder.summary(Some("Simple echo workflow"));

        let trigger_spec = NodeSpec {
            identifier: "examples::s1_echo::http_trigger",
            name: "HttpTrigger",
            kind: NodeKind::Trigger,
            summary: Some("Entry point for HTTP requests"),
            in_schema: SchemaSpec::Opaque,
            out_schema: SchemaSpec::Named("EchoRequest"),
            effects: Effects::ReadOnly,
            determinism: Determinism::Strict,
            determinism_hints: &[],
            effect_hints: &[],
        };

        let normalize_spec = NodeSpec::inline(
            "examples::s1_echo::normalize",
            "Normalize",
            SchemaSpec::Named("EchoRequest"),
            SchemaSpec::Named("EchoResponse"),
            Effects::Pure,
            Determinism::Strict,
            Some("Trim and lowercase payload"),
        );

        let respond_spec = NodeSpec::inline(
            "examples::s1_echo::respond",
            "Respond",
            SchemaSpec::Named("EchoResponse"),
            SchemaSpec::Named("HttpResponse"),
            Effects::Effectful,
            Determinism::BestEffort,
            Some("Write HTTP response"),
        );

        let trigger = builder
            .add_node("trigger", &trigger_spec)
            .expect("add trigger");
        let normalize = builder
            .add_node("normalize", &normalize_spec)
            .expect("add normalize");
        let respond = builder
            .add_node("respond", &respond_spec)
            .expect("add respond");
        builder.connect(&trigger, &normalize);
        builder.connect(&normalize, &respond);

        let flow = builder.build();

        assert_eq!(flow.nodes.len(), 3);
        assert_eq!(flow.edges.len(), 2);

        let json = serde_json::to_value(&flow).expect("serialize flow");
        assert_eq!(
            json["profile"],
            json!(Profile::Web),
            "profile should serialize correctly"
        );
    }
}
````

## File: crates/kernel-plan/src/lib.rs
````rust
use std::collections::{HashMap, HashSet};

use dag_core::{Delivery, Diagnostic, Effects, FlowIR, SchemaRef, diagnostic_codes};

const MIN_EXACTLY_ONCE_TTL_MS: u64 = 300_000;
const DEDUPE_HINT_PREFIX: &str = "resource::dedupe";

/// Result of a successful validation run.
#[derive(Debug, Clone)]
pub struct ValidatedIR {
    flow: FlowIR,
}

impl ValidatedIR {
    /// Access the validated Flow IR.
    pub fn flow(&self) -> &FlowIR {
        &self.flow
    }

    /// Consume the validated wrapper and return the underlying Flow IR.
    pub fn into_inner(self) -> FlowIR {
        self.flow
    }
}

/// Validate a flow and return diagnostics if issues are discovered.
pub fn validate(flow: &FlowIR) -> Result<ValidatedIR, Vec<Diagnostic>> {
    let mut diagnostics = Vec::new();

    check_duplicate_aliases(flow, &mut diagnostics);
    check_edge_references(flow, &mut diagnostics);
    check_cycles(flow, &mut diagnostics);
    check_port_compatibility(flow, &mut diagnostics);
    check_effectful_idempotency(flow, &mut diagnostics);
    check_effect_conflicts(flow, &mut diagnostics);
    check_determinism_conflicts(flow, &mut diagnostics);
    check_exactly_once_requirements(flow, &mut diagnostics);
    check_spill_requirements(flow, &mut diagnostics);

    if diagnostics.is_empty() {
        Ok(ValidatedIR { flow: flow.clone() })
    } else {
        Err(diagnostics)
    }
}

fn check_exactly_once_requirements(flow: &FlowIR, diagnostics: &mut Vec<Diagnostic>) {
    let nodes: HashMap<_, _> = flow
        .nodes
        .iter()
        .map(|node| (node.alias.as_str(), node))
        .collect();

    for edge in &flow.edges {
        if edge.delivery == Delivery::ExactlyOnce {
            let Some(target) = nodes.get(edge.to.as_str()) else {
                continue;
            };

            if !has_dedupe_binding(target) {
                diagnostics.push(diagnostic(
                    "EXACT001",
                    format!(
                        "edge `{}` -> `{}` requests Delivery::ExactlyOnce but node `{}` does not bind a dedupe capability (hint `{}` expected)",
                        edge.from,
                        edge.to,
                        target.alias,
                        DEDUPE_HINT_PREFIX
                    ),
                ));
            }

            if target.idempotency.key.is_none() {
                diagnostics.push(diagnostic(
                    "EXACT002",
                    format!(
                        "edge `{}` -> `{}` requests Delivery::ExactlyOnce but node `{}` has no idempotency key",
                        edge.from, edge.to, target.alias
                    ),
                ));
            }

            match target.idempotency.ttl_ms {
                Some(ttl) if ttl >= MIN_EXACTLY_ONCE_TTL_MS => {}
                Some(ttl) => diagnostics.push(diagnostic(
                    "EXACT003",
                    format!(
                        "edge `{}` -> `{}` requests Delivery::ExactlyOnce but node `{}` declares dedupe TTL {}ms (minimum {}ms)",
                        edge.from,
                        edge.to,
                        target.alias,
                        ttl,
                        MIN_EXACTLY_ONCE_TTL_MS
                    ),
                )),
                None => diagnostics.push(diagnostic(
                    "EXACT003",
                    format!(
                        "edge `{}` -> `{}` requests Delivery::ExactlyOnce but node `{}` does not declare a dedupe TTL (minimum {}ms)",
                        edge.from,
                        edge.to,
                        target.alias,
                        MIN_EXACTLY_ONCE_TTL_MS
                    ),
                )),
            }
        }
    }
}

fn has_dedupe_binding(node: &dag_core::NodeIR) -> bool {
    node.effect_hints
        .iter()
        .any(|hint| hint.starts_with(DEDUPE_HINT_PREFIX))
}

fn check_duplicate_aliases(flow: &FlowIR, diagnostics: &mut Vec<Diagnostic>) {
    let mut seen = HashSet::new();
    for node in &flow.nodes {
        if !seen.insert(&node.alias) {
            diagnostics.push(diagnostic(
                "DAG205",
                format!("duplicate node alias `{}`", node.alias),
            ));
        }
    }
}

fn check_edge_references(flow: &FlowIR, diagnostics: &mut Vec<Diagnostic>) {
    let aliases: HashSet<_> = flow.nodes.iter().map(|node| node.alias.as_str()).collect();
    for edge in &flow.edges {
        if !aliases.contains(edge.from.as_str()) {
            diagnostics.push(diagnostic(
                "DAG202",
                format!("unknown node alias `{}` referenced as source", edge.from),
            ));
        }
        if !aliases.contains(edge.to.as_str()) {
            diagnostics.push(diagnostic(
                "DAG202",
                format!("unknown node alias `{}` referenced as target", edge.to),
            ));
        }
        if !aliases.contains(edge.from.as_str()) || !aliases.contains(edge.to.as_str()) {
            continue;
        }
    }
}

fn check_cycles(flow: &FlowIR, diagnostics: &mut Vec<Diagnostic>) {
    let mut adjacency: HashMap<&str, Vec<&str>> = HashMap::new();
    for edge in &flow.edges {
        adjacency
            .entry(edge.from.as_str())
            .or_default()
            .push(edge.to.as_str());
    }

    let mut visiting = HashSet::new();
    let mut visited = HashSet::new();

    for node in &flow.nodes {
        if dfs_cycle(node.alias.as_str(), &adjacency, &mut visiting, &mut visited) {
            diagnostics.push(diagnostic("DAG200", "cycle detected in workflow"));
            break;
        }
    }
}

fn dfs_cycle<'a>(
    node: &'a str,
    adjacency: &HashMap<&'a str, Vec<&'a str>>,
    visiting: &mut HashSet<&'a str>,
    visited: &mut HashSet<&'a str>,
) -> bool {
    if visiting.contains(node) {
        return true;
    }
    if visited.contains(node) {
        return false;
    }

    visiting.insert(node);
    if let Some(neighbours) = adjacency.get(node) {
        for &next in neighbours {
            if dfs_cycle(next, adjacency, visiting, visited) {
                return true;
            }
        }
    }
    visiting.remove(node);
    visited.insert(node);
    false
}

fn check_port_compatibility(flow: &FlowIR, diagnostics: &mut Vec<Diagnostic>) {
    let nodes: HashMap<_, _> = flow
        .nodes
        .iter()
        .map(|node| (node.alias.as_str(), node))
        .collect();
    for edge in &flow.edges {
        let source = match nodes.get(edge.from.as_str()) {
            Some(node) => node,
            None => continue,
        };
        let target = match nodes.get(edge.to.as_str()) {
            Some(node) => node,
            None => continue,
        };
        if !schemas_compatible(&source.out_schema, &target.in_schema) {
            diagnostics.push(diagnostic(
                "DAG201",
                format!(
                    "port type mismatch: `{}` -> `{}` ({:?} -> {:?})",
                    edge.from, edge.to, source.out_schema, target.in_schema
                ),
            ));
        }
    }
}

fn schemas_compatible(source: &SchemaRef, target: &SchemaRef) -> bool {
    match (source, target) {
        (SchemaRef::Named { name: a }, SchemaRef::Named { name: b }) => a == b,
        _ => true,
    }
}

fn check_effectful_idempotency(flow: &FlowIR, diagnostics: &mut Vec<Diagnostic>) {
    for node in &flow.nodes {
        if node.effects == Effects::Effectful && node.idempotency.key.is_none() {
            diagnostics.push(diagnostic(
                "DAG004",
                format!("effectful node `{}` missing idempotency key", node.alias),
            ));
        }
    }
}

fn check_effect_conflicts(flow: &FlowIR, diagnostics: &mut Vec<Diagnostic>) {
    for node in &flow.nodes {
        for hint in &node.effect_hints {
            if let Some(conflict) = dag_core::effects_registry::constraint_for_hint(hint) {
                if !node.effects.is_at_least(conflict.minimum) {
                    diagnostics.push(diagnostic(
                        "EFFECT201",
                        format!(
                            "node `{}` declares effects {} but resource `{}` requires at least {}: {}",
                            node.alias,
                            node.effects.as_str(),
                            hint,
                            conflict.minimum.as_str(),
                            conflict.guidance
                        ),
                    ));
                }
            }
        }
    }
}

fn check_determinism_conflicts(flow: &FlowIR, diagnostics: &mut Vec<Diagnostic>) {
    for node in &flow.nodes {
        for hint in &node.determinism_hints {
            if let Some(conflict) = dag_core::determinism::constraint_for_hint(hint) {
                if !node.determinism.is_at_least(conflict.minimum) {
                    diagnostics.push(diagnostic(
                        "DET302",
                        format!(
                            "node `{}` declares determinism {} but resource `{}` requires at least {}: {}",
                            node.alias,
                            node.determinism.as_str(),
                            hint,
                            conflict.minimum.as_str(),
                            conflict.guidance
                        ),
                    ));
                }
            }
        }
    }
}

fn check_spill_requirements(flow: &FlowIR, diagnostics: &mut Vec<Diagnostic>) {
    let has_blob_hint = flow.nodes.iter().any(|node| {
        node.effect_hints
            .iter()
            .any(|hint| hint.starts_with("resource::blob::"))
    });

    let mut emitted_blob_diagnostic = false;

    for edge in &flow.edges {
        if let Some(tier) = &edge.buffer.spill_tier {
            if edge.buffer.max_items.is_none() {
                diagnostics.push(diagnostic(
                    "SPILL001",
                    format!(
                        "edge `{}` -> `{}` configures `spill_tier = {tier}` without bounding `max_items`",
                        edge.from, edge.to
                    ),
                ));
            }

            if !has_blob_hint && !emitted_blob_diagnostic {
                diagnostics.push(diagnostic(
                    "SPILL002",
                    format!(
                        "edge `{}` -> `{}` configures `spill_tier = {tier}` but no node declares a blob capability hint",
                        edge.from, edge.to
                    ),
                ));
                emitted_blob_diagnostic = true;
            }
        }
    }
}

fn diagnostic(code: &str, message: impl Into<String>) -> Diagnostic {
    let entry = diagnostic_codes()
        .iter()
        .find(|item| item.code == code)
        .unwrap_or_else(|| panic!("unknown diagnostic code `{code}`"));
    Diagnostic::new(entry, message)
}

#[cfg(test)]
mod tests {
    use super::*;
    use capabilities::{blob, db, http, kv, queue};
    use dag_core::IdempotencyScope;
    use dag_core::NodeResult;
    use dag_core::prelude::*;
    use dag_macros::node;
    use proptest::prelude::*;
    use proptest::sample::select;
    use std::collections::BTreeSet;

    fn build_sample_flow() -> FlowIR {
        let mut builder = FlowBuilder::new("sample", Version::new(1, 0, 0), Profile::Web);

        let producer_spec = NodeSpec::inline(
            "tests::producer",
            "Producer",
            SchemaSpec::Opaque,
            SchemaSpec::Named("String"),
            Effects::Pure,
            Determinism::Strict,
            None,
        );
        let consumer_spec = NodeSpec::inline(
            "tests::consumer",
            "Consumer",
            SchemaSpec::Named("String"),
            SchemaSpec::Opaque,
            Effects::ReadOnly,
            Determinism::Stable,
            None,
        );

        let producer = builder
            .add_node("producer", &producer_spec)
            .expect("add producer");
        let consumer = builder
            .add_node("consumer", &consumer_spec)
            .expect("add consumer");
        builder.connect(&producer, &consumer);

        builder.build()
    }

    fn downgrade_effect(level: Effects) -> Option<Effects> {
        match level {
            Effects::Effectful => Some(Effects::ReadOnly),
            Effects::ReadOnly => Some(Effects::Pure),
            Effects::Pure => None,
        }
    }

    fn downgrade_determinism(level: Determinism) -> Option<Determinism> {
        match level {
            Determinism::Nondeterministic => Some(Determinism::BestEffort),
            Determinism::BestEffort => Some(Determinism::Stable),
            Determinism::Stable => Some(Determinism::Strict),
            Determinism::Strict => None,
        }
    }

    const DEDUPE_HINT_WRITE: &str = "resource::dedupe::write";

    fn set_idempotency(flow: &mut FlowIR, alias: &str) {
        if let Some(node) = flow.nodes.iter_mut().find(|n| n.alias == alias) {
            node.idempotency.key = Some("prop.case".to_string());
            node.idempotency.scope = Some(IdempotencyScope::Node);
            node.idempotency.ttl_ms = Some(MIN_EXACTLY_ONCE_TTL_MS);
        }
    }

    fn ensure_dedupe_hint(flow: &mut FlowIR, alias: &str) {
        if let Some(node) = flow.nodes.iter_mut().find(|n| n.alias == alias) {
            if !node
                .effect_hints
                .iter()
                .any(|hint| hint == DEDUPE_HINT_WRITE)
            {
                node.effect_hints.push(DEDUPE_HINT_WRITE.to_string());
            }
        }
    }

    fn register_all_hints() {
        http::ensure_registered();
        db::ensure_registered();
        kv::ensure_registered();
        blob::ensure_registered();
        queue::ensure_registered();
        capabilities::clock::ensure_registered();
        capabilities::rng::ensure_registered();
    }

    #[test]
    fn exactly_once_requires_dedupe_binding() {
        let mut flow = build_sample_flow();
        if let Some(edge) = flow.edges.first_mut() {
            edge.delivery = Delivery::ExactlyOnce;
        }
        set_idempotency(&mut flow, "consumer");

        let diagnostics = validate(&flow).err().expect("expected validation errors");
        assert!(diagnostics.iter().any(|d| d.code.code == "EXACT001"));
    }

    #[test]
    fn exactly_once_requires_idempotency_key() {
        let mut flow = build_sample_flow();
        if let Some(edge) = flow.edges.first_mut() {
            edge.delivery = Delivery::ExactlyOnce;
        }
        set_idempotency(&mut flow, "consumer");
        ensure_dedupe_hint(&mut flow, "consumer");
        if let Some(node) = flow.nodes.iter_mut().find(|n| n.alias == "consumer") {
            node.idempotency.key = None;
        }

        let diagnostics = validate(&flow).err().expect("expected validation errors");
        assert!(diagnostics.iter().any(|d| d.code.code == "EXACT002"));
        // Without a key TTL is irrelevant; ensure no panic when missing.
    }

    #[test]
    fn exactly_once_requires_ttl() {
        let mut flow = build_sample_flow();
        if let Some(edge) = flow.edges.first_mut() {
            edge.delivery = Delivery::ExactlyOnce;
        }
        set_idempotency(&mut flow, "consumer");
        ensure_dedupe_hint(&mut flow, "consumer");
        if let Some(node) = flow.nodes.iter_mut().find(|n| n.alias == "consumer") {
            node.idempotency.ttl_ms = None;
        }

        let diagnostics = validate(&flow).err().expect("expected validation errors");
        assert!(diagnostics.iter().any(|d| d.code.code == "EXACT003"));
    }

    #[test]
    fn exactly_once_requires_minimum_ttl() {
        let mut flow = build_sample_flow();
        if let Some(edge) = flow.edges.first_mut() {
            edge.delivery = Delivery::ExactlyOnce;
        }
        set_idempotency(&mut flow, "consumer");
        ensure_dedupe_hint(&mut flow, "consumer");
        if let Some(node) = flow.nodes.iter_mut().find(|n| n.alias == "consumer") {
            node.idempotency.ttl_ms = Some(MIN_EXACTLY_ONCE_TTL_MS - 1);
        }

        let diagnostics = validate(&flow).err().expect("expected validation errors");
        assert!(diagnostics.iter().any(|d| d.code.code == "EXACT003"));
    }

    #[test]
    fn exactly_once_succeeds_when_prerequisites_met() {
        let mut flow = build_sample_flow();
        if let Some(edge) = flow.edges.first_mut() {
            edge.delivery = Delivery::ExactlyOnce;
        }
        set_idempotency(&mut flow, "consumer");
        ensure_dedupe_hint(&mut flow, "consumer");
        if let Some(node) = flow.nodes.iter_mut().find(|n| n.alias == "consumer") {
            node.idempotency.ttl_ms = Some(MIN_EXACTLY_ONCE_TTL_MS);
        }

        let result = validate(&flow);
        assert!(result.is_ok(), "unexpected diagnostics: {result:?}");
    }

    fn dedup_hints(hints: Vec<&'static str>) -> Vec<&'static str> {
        let mut set = BTreeSet::new();
        for hint in hints {
            set.insert(hint);
        }
        set.into_iter().collect()
    }

    #[test]
    fn validate_accepts_well_formed_flow() {
        let flow = build_sample_flow();
        let result = validate(&flow);
        assert!(result.is_ok(), "unexpected diagnostics: {result:?}");
    }

    #[test]
    fn detect_type_mismatch() {
        let mut flow = build_sample_flow();
        if let Some(node) = flow.nodes.iter_mut().find(|n| n.alias == "consumer") {
            node.in_schema = SchemaRef::Named {
                name: "Other".to_string(),
            };
        }
        let result = validate(&flow);
        assert!(result.is_err());
        let diagnostics = result.err().unwrap();
        assert!(diagnostics.iter().any(|d| d.code.code == "DAG201"));
    }

    #[test]
    fn detect_cycles() {
        let mut flow = build_sample_flow();
        flow.edges.push(dag_core::EdgeIR {
            from: "consumer".to_string(),
            to: "producer".to_string(),
            ..dag_core::EdgeIR::default()
        });
        let diagnostics = validate(&flow).err().expect("expected cycle diagnostic");
        assert!(diagnostics.iter().any(|d| d.code.code == "DAG200"));
    }

    #[test]
    fn detect_unknown_aliases() {
        let mut flow = build_sample_flow();
        flow.edges.push(dag_core::EdgeIR {
            from: "missing".to_string(),
            to: "consumer".to_string(),
            ..dag_core::EdgeIR::default()
        });
        flow.edges.push(dag_core::EdgeIR {
            from: "producer".to_string(),
            to: "absent".to_string(),
            ..dag_core::EdgeIR::default()
        });
        let diagnostics = validate(&flow).err().expect("expected alias diagnostics");
        let mut seen_source = false;
        let mut seen_target = false;
        for diag in diagnostics {
            if diag.code.code == "DAG202" && diag.message.contains("source") {
                seen_source = true;
            }
            if diag.code.code == "DAG202" && diag.message.contains("target") {
                seen_target = true;
            }
        }
        assert!(seen_source, "missing source alias diagnostic not emitted");
        assert!(seen_target, "missing target alias diagnostic not emitted");
    }

    #[test]
    fn fuzz_registry_hint_enforcement() {
        register_all_hints();
        let effect_universe: Vec<&'static str> = dag_core::effects_registry::all_constraints()
            .into_iter()
            .map(|c| c.hint)
            .collect();
        let determinism_universe: Vec<&'static str> = dag_core::determinism::all_constraints()
            .into_iter()
            .map(|c| c.hint)
            .collect();

        let effect_levels = vec![Effects::Pure, Effects::ReadOnly, Effects::Effectful];
        let determinism_levels = vec![
            Determinism::Strict,
            Determinism::Stable,
            Determinism::BestEffort,
            Determinism::Nondeterministic,
        ];

        let mut runner = proptest::test_runner::TestRunner::new(ProptestConfig {
            cases: 64,
            ..ProptestConfig::default()
        });

        let strategy = (
            proptest::collection::vec(select(effect_universe.clone()), 0..=3),
            proptest::collection::vec(select(determinism_universe.clone()), 0..=3),
            select(effect_levels.clone()),
            select(determinism_levels.clone()),
            proptest::bool::ANY,
            proptest::bool::ANY,
        );

        runner
            .run(
                &strategy,
                |(
                    effect_hints_case,
                    det_hints_case,
                    base_effect,
                    base_det,
                    degrade_effect_flag,
                    degrade_det_flag,
                )| {
                    let effect_vec = dedup_hints(effect_hints_case);
                    let det_vec = dedup_hints(det_hints_case);

                    let required_effect = effect_vec
                        .iter()
                        .filter_map(|hint| {
                            dag_core::effects_registry::constraint_for_hint(hint).map(|c| c.minimum)
                        })
                        .fold(None::<Effects>, |acc, next| match acc {
                            Some(current) if current.rank() >= next.rank() => Some(current),
                            Some(_) => Some(next),
                            None => Some(next),
                        });

                    let required_det = det_vec
                        .iter()
                        .filter_map(|hint| {
                            dag_core::determinism::constraint_for_hint(hint).map(|c| c.minimum)
                        })
                        .fold(None::<Determinism>, |acc, next| match acc {
                            Some(current) if current.rank() >= next.rank() => Some(current),
                            Some(_) => Some(next),
                            None => Some(next),
                        });

                    let declared_effect = if degrade_effect_flag {
                        downgrade_effect(base_effect).unwrap_or(base_effect)
                    } else {
                        base_effect
                    };
                    let declared_det = if degrade_det_flag {
                        downgrade_determinism(base_det).unwrap_or(base_det)
                    } else {
                        base_det
                    };

                    let effect_slice: &'static [&'static str] =
                        Box::leak(effect_vec.clone().into_boxed_slice());
                    let det_slice: &'static [&'static str] =
                        Box::leak(det_vec.clone().into_boxed_slice());

                    let spec_box = Box::new(NodeSpec::inline_with_hints(
                        "tests::prop_validator",
                        "PropValidator",
                        SchemaSpec::Opaque,
                        SchemaSpec::Opaque,
                        declared_effect,
                        declared_det,
                        None,
                        det_slice,
                        effect_slice,
                    ));
                    let spec: &'static NodeSpec = Box::leak(spec_box);

                    let mut builder =
                        FlowBuilder::new("prop_validation", Version::new(0, 0, 1), Profile::Web);
                    builder.add_node("entry", spec).expect("add node");
                    let mut flow = builder.build();
                    if declared_effect == Effects::Effectful {
                        set_idempotency(&mut flow, "entry");
                    }

                    let result = validate(&flow);

                    let violates_effect = required_effect
                        .map(|req| !declared_effect.is_at_least(req))
                        .unwrap_or(false);
                    let violates_det = required_det
                        .map(|req| !declared_det.is_at_least(req))
                        .unwrap_or(false);

                    if violates_effect || violates_det {
                        let diagnostics = result.err().expect("expected validation errors");
                        if violates_effect {
                            prop_assert!(
                                diagnostics.iter().any(|d| d.code.code == "EFFECT201"),
                                "expected EFFECT201 diagnostic"
                            );
                        }
                        if violates_det {
                            prop_assert!(
                                diagnostics.iter().any(|d| d.code.code == "DET302"),
                                "expected DET302 diagnostic"
                            );
                        }
                    } else {
                        prop_assert!(
                            result.is_ok(),
                            "expected validation success when declared policies satisfy hints"
                        );
                    }
                    Ok(())
                },
            )
            .unwrap();
    }

    #[test]
    fn detect_effect_conflicts() {
        let mut builder = FlowBuilder::new("effect_conflict", Version::new(1, 0, 0), Profile::Web);
        let writer = builder
            .add_node(
                "writer",
                &NodeSpec::inline_with_hints(
                    "tests::writer",
                    "Writer",
                    SchemaSpec::Opaque,
                    SchemaSpec::Opaque,
                    Effects::Pure,
                    Determinism::Strict,
                    None,
                    &[],
                    &["resource::http::write"],
                ),
            )
            .expect("add writer node");
        let sink = builder
            .add_node(
                "sink",
                &NodeSpec::inline(
                    "tests::sink",
                    "Sink",
                    SchemaSpec::Opaque,
                    SchemaSpec::Opaque,
                    Effects::ReadOnly,
                    Determinism::BestEffort,
                    None,
                ),
            )
            .expect("add sink node");
        builder.connect(&writer, &sink);
        let flow = builder.build();

        let diagnostics = validate(&flow).err().expect("expected effect diagnostic");
        assert!(diagnostics.iter().any(|d| d.code.code == "EFFECT201"));
    }

    #[test]
    fn detect_missing_idempotency() {
        let mut flow = build_sample_flow();
        if let Some(node) = flow.nodes.iter_mut().find(|n| n.alias == "consumer") {
            node.effects = Effects::Effectful;
            node.idempotency.key = None;
        }
        let diagnostics = validate(&flow)
            .err()
            .expect("expected idempotency diagnostic");
        assert!(diagnostics.iter().any(|d| d.code.code == "DAG004"));
    }

    #[test]
    fn spill_requires_max_items_bound() {
        let mut builder = FlowBuilder::new("spill_no_bound", Version::new(1, 0, 0), Profile::Queue);
        let trigger = builder
            .add_node(
                "trigger",
                &NodeSpec::inline(
                    "tests::trigger",
                    "Trigger",
                    SchemaSpec::Opaque,
                    SchemaSpec::Opaque,
                    Effects::Pure,
                    Determinism::Strict,
                    None,
                ),
            )
            .unwrap();
        let worker = builder
            .add_node(
                "worker",
                &NodeSpec::inline(
                    "tests::worker",
                    "Worker",
                    SchemaSpec::Opaque,
                    SchemaSpec::Opaque,
                    Effects::Pure,
                    Determinism::Strict,
                    None,
                ),
            )
            .unwrap();
        builder.connect(&trigger, &worker);

        let mut flow = builder.build();
        for edge in &mut flow.edges {
            edge.buffer.spill_tier = Some("local".into());
            edge.buffer.max_items = None;
        }

        let diagnostics = validate(&flow).err().expect("expected spill diagnostic");
        assert!(diagnostics.iter().any(|d| d.code.code == "SPILL001"));
    }

    #[test]
    fn spill_requires_blob_hint() {
        let mut builder =
            FlowBuilder::new("spill_blob_hint", Version::new(1, 0, 0), Profile::Queue);
        let trigger = builder
            .add_node(
                "trigger",
                &NodeSpec::inline(
                    "tests::trigger",
                    "Trigger",
                    SchemaSpec::Opaque,
                    SchemaSpec::Opaque,
                    Effects::Pure,
                    Determinism::Strict,
                    None,
                ),
            )
            .unwrap();
        let worker = builder
            .add_node(
                "worker",
                &NodeSpec::inline(
                    "tests::worker",
                    "Worker",
                    SchemaSpec::Opaque,
                    SchemaSpec::Opaque,
                    Effects::Pure,
                    Determinism::Strict,
                    None,
                ),
            )
            .unwrap();
        builder.connect(&trigger, &worker);

        let mut flow = builder.build();
        for edge in &mut flow.edges {
            edge.buffer.spill_tier = Some("local".into());
            edge.buffer.max_items = Some(1);
        }

        let diagnostics = validate(&flow)
            .err()
            .expect("expected blob hint diagnostic");
        assert!(diagnostics.iter().any(|d| d.code.code == "SPILL002"));
    }

    #[test]
    fn spill_passes_when_blob_hint_declared() {
        let mut builder = FlowBuilder::new("spill_blob_ok", Version::new(1, 0, 0), Profile::Queue);
        let trigger = builder
            .add_node(
                "trigger",
                &NodeSpec::inline(
                    "tests::trigger",
                    "Trigger",
                    SchemaSpec::Opaque,
                    SchemaSpec::Opaque,
                    Effects::Pure,
                    Determinism::Strict,
                    None,
                ),
            )
            .unwrap();
        let worker_spec = NodeSpec::inline_with_hints(
            "tests::worker",
            "Worker",
            SchemaSpec::Opaque,
            SchemaSpec::Opaque,
            Effects::Effectful,
            Determinism::BestEffort,
            None,
            &[],
            &["resource::blob::write"],
        );
        let worker = builder.add_node("worker", &worker_spec).unwrap();
        builder.connect(&trigger, &worker);

        let mut flow = builder.build();
        for edge in &mut flow.edges {
            edge.buffer.spill_tier = Some("local".into());
            edge.buffer.max_items = Some(1);
        }
        set_idempotency(&mut flow, "worker");

        let result = validate(&flow);
        if let Err(diags) = &result {
            let codes: Vec<&str> = diags.iter().map(|d| d.code.code).collect();
            panic!(
                "spill validation should succeed when blob hints are present, diagnostics: {:?}",
                codes
            );
        }
    }

    #[test]
    fn detect_determinism_conflicts() {
        let mut builder = FlowBuilder::new("det_conflict", Version::new(1, 0, 0), Profile::Web);
        let clock = builder
            .add_node(
                "clock",
                &NodeSpec::inline_with_hints(
                    "tests::clock",
                    "Clock",
                    SchemaSpec::Opaque,
                    SchemaSpec::Opaque,
                    Effects::ReadOnly,
                    Determinism::Strict,
                    None,
                    &["resource::clock"],
                    &[],
                ),
            )
            .expect("add clock node");
        let sink = builder
            .add_node(
                "sink",
                &NodeSpec::inline(
                    "tests::sink",
                    "Sink",
                    SchemaSpec::Opaque,
                    SchemaSpec::Opaque,
                    Effects::ReadOnly,
                    Determinism::BestEffort,
                    None,
                ),
            )
            .expect("add sink node");
        builder.connect(&clock, &sink);
        let flow = builder.build();

        let diagnostics = validate(&flow)
            .err()
            .expect("expected determinism diagnostic");
        assert!(diagnostics.iter().any(|d| d.code.code == "DET302"));
    }

    #[test]
    fn respect_registered_custom_conflicts() {
        const HINT: &str = "test::custom";
        dag_core::determinism::register_determinism_constraint(
            dag_core::determinism::DeterminismConstraint::new(
                HINT,
                Determinism::Stable,
                "Custom resource requires Stable determinism",
            ),
        );

        let mut builder = FlowBuilder::new("custom_conflict", Version::new(1, 0, 0), Profile::Web);
        let source = builder
            .add_node(
                "source",
                &NodeSpec::inline_with_hints(
                    "tests::source",
                    "Source",
                    SchemaSpec::Opaque,
                    SchemaSpec::Opaque,
                    Effects::ReadOnly,
                    Determinism::Strict,
                    None,
                    &[HINT],
                    &[],
                ),
            )
            .expect("add source node");
        let sink = builder
            .add_node(
                "sink",
                &NodeSpec::inline(
                    "tests::sink",
                    "Sink",
                    SchemaSpec::Opaque,
                    SchemaSpec::Opaque,
                    Effects::ReadOnly,
                    Determinism::BestEffort,
                    None,
                ),
            )
            .expect("add sink node");
        builder.connect(&source, &sink);
        let flow = builder.build();

        let diagnostics = validate(&flow)
            .err()
            .expect("expected determinism diagnostic");
        assert!(diagnostics.iter().any(|d| d.code.code == "DET302"));
    }

    mod auto_hint_validation {
        use super::*;
        use dag_core::IdempotencyScope;

        struct HttpWrite;
        struct TestClock;
        struct DbHandle;
        struct Noop;

        #[node(
            name = "HttpWriter",
            effects = "Effectful",
            determinism = "BestEffort",
            resources(http(HttpWrite))
        )]
        async fn http_writer(_: ()) -> NodeResult<()> {
            Ok(())
        }

        #[node(
            name = "ClockBestEffort",
            effects = "ReadOnly",
            determinism = "BestEffort",
            resources(clock(TestClock))
        )]
        async fn clock_best_effort(_: ()) -> NodeResult<()> {
            Ok(())
        }

        #[node(
            name = "DbWriter",
            effects = "Effectful",
            determinism = "BestEffort",
            resources(db_writer(DbHandle))
        )]
        async fn db_writer(_: ()) -> NodeResult<()> {
            Ok(())
        }

        #[node(name = "NoResources", effects = "Pure", determinism = "Strict")]
        async fn no_resources(_: ()) -> NodeResult<()> {
            Ok(())
        }

        fn single_node_flow(alias: &str, spec: &'static NodeSpec) -> FlowIR {
            let mut builder = FlowBuilder::new(alias, Version::new(1, 0, 0), Profile::Web);
            builder.add_node(alias, spec).expect("add node");
            builder.build()
        }

        fn ensure_idempotency(flow: &mut FlowIR, alias: &str) {
            if let Some(node) = flow.nodes.iter_mut().find(|n| n.alias == alias) {
                node.idempotency.key = Some("request.id".to_string());
                node.idempotency.scope = Some(IdempotencyScope::Node);
            }
        }

        #[test]
        fn validator_flags_effect_conflict_from_registry_hint() {
            capabilities::http::ensure_registered();
            let mut flow = single_node_flow("writer", http_writer_node_spec());
            ensure_idempotency(&mut flow, "writer");
            if let Some(node) = flow.nodes.iter_mut().find(|n| n.alias == "writer") {
                node.effects = Effects::Pure;
            }
            let diagnostics = validate(&flow)
                .err()
                .expect("expected effect mismatch diagnostic");
            assert!(
                diagnostics.iter().any(|d| d.code.code == "EFFECT201"),
                "expected EFFECT201, got: {:?}",
                diagnostics
            );
        }

        #[test]
        fn validator_accepts_effectful_node_with_registry_hint() {
            capabilities::http::ensure_registered();
            let flow = single_node_flow("writer_ok", http_writer_node_spec());
            // effectful nodes require idempotency; clone and set before validation
            let mut flow = flow;
            ensure_idempotency(&mut flow, "writer_ok");
            assert!(
                validate(&flow).is_ok(),
                "effectful http writer should validate cleanly"
            );
        }

        #[test]
        fn validator_flags_determinism_conflict_from_registry_hint() {
            capabilities::clock::ensure_registered();
            let mut flow = single_node_flow("clock", clock_best_effort_node_spec());
            if let Some(node) = flow.nodes.iter_mut().find(|n| n.alias == "clock") {
                node.determinism = Determinism::Strict;
            }
            let diagnostics = validate(&flow)
                .err()
                .expect("expected determinism mismatch diagnostic");
            assert!(
                diagnostics.iter().any(|d| d.code.code == "DET302"),
                "expected DET302, got: {:?}",
                diagnostics
            );
        }

        #[test]
        fn fallback_hints_still_trigger_conflicts() {
            let mut flow = single_node_flow("db_writer", db_writer_node_spec());
            ensure_idempotency(&mut flow, "db_writer");
            if let Some(node) = flow.nodes.iter_mut().find(|n| n.alias == "db_writer") {
                node.effects = Effects::Pure;
            }
            let diagnostics = validate(&flow)
                .err()
                .expect("expected effect conflict from fallback hints");
            assert!(
                diagnostics.iter().any(|d| d.code.code == "EFFECT201"),
                "expected EFFECT201 from fallback hints, got: {:?}",
                diagnostics
            );
        }

        #[test]
        fn nodes_without_resources_remain_hint_free() {
            let spec = no_resources_node_spec();
            assert!(
                spec.effect_hints.is_empty() && spec.determinism_hints.is_empty(),
                "expected no hints for resource-free node"
            );
            let flow = single_node_flow("noop", spec);
            assert!(
                validate(&flow).is_ok(),
                "resource-free nodes with pure/strict defaults should validate"
            );
        }
    }
}
````

## File: impl-docs/rust-workflow-tdd-rfc.md
````markdown
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

### 4.9 Code-first control-flow hints
- Authors continue to write idiomatic Rust (`match`, `if let`, `for`, `while`) but opt into richer orchestration semantics by wrapping those constructs with lightweight helpers. Each helper emits a `ControlSurfaceIR` entry (§5.2.2) so schedulers, UI, and policy engines see explicit branch labels, partition keys, and retry semantics.
- When a helper is recommended but not used, the compiler emits `CTRL001` with guidance (“Wrap this loop in `for_each!` to unlock partition-aware batching and policy enforcement”). Warnings stay informational unless `policies.lint.require_control_hints` elevates them.
- Import/export tooling never rewrites author code; it consumes the IR surfaces. Rust remains the source of truth while n8n-style features (visual branching, metrics, linting) stay available.

#### 4.9.1 Branching surfaces (`switch!`, `if!`)
- `switch!(expr, cases = { label if predicate => edge, ... })` declares multi-way branching with explicit labels. The macro expands to normal Rust, so `expr` can be any value and predicates are just boolean expressions.
- `if!(cond, then = edge_a, else = edge_b)` is the single-branch shorthand; it still records the control surface so case labels show up in Studio and policy can reason about fall-through.
- Attribute sugar preserves raw `match` ergonomics: `let route = #[flow::switch(risk)] match risk.score { s if s > 80.0 => Risk::High, _ => Risk::Low };` desugars to `switch!` while keeping Rust’s pattern matching, guards, and exhaustiveness checking.

**Simple** — binary branch with connector nodes:
```rust
let route = switch!(risk, cases = {
    "ManualReview" if risk.score > 70.0 => manual_review,
    "Auto" => auto_charge,
});
connect!(route.case("ManualReview") -> connectors::slack::SendAlert::new());
connect!(route.case("Auto") -> connectors::stripe::Charge::new());
```

**Complex** — pattern guards with native `match` syntax:
```rust
let decision = #[flow::switch(order)] match &order.channel {
    Channel::Marketplace(v) if v.is_high_value() => Case::MarketplaceHigh,
    Channel::Direct(_) if order.total > Decimal::from(500) => Case::DirectHigh,
    _ => Case::Standard,
};
connect!(decision.case("MarketplaceHigh") -> escalate);
connect!(decision.case("DirectHigh") -> capture_id_check);
connect!(decision.case("Standard") -> fulfill);
```

#### 4.9.2 Looping and iteration (`for_each!`, `loop!`)
- `for_each!(items, item => node, key = expr)` wraps a `for` loop, emitting a control surface that documents the iteration source, concurrency hints, and optional partition key. Bodies remain pure Rust; the macro only annotates the loop for scheduling.
- `loop!(label, break_on = expr, body = { ... })` is the cooperative retry helper: it surfaces the loop bounds while authoring continues to use `while`/`loop` internally.
- Attribute form mirrors the branching sugar: `#[flow::for_each(key = |c| c.customer_id)] for chunk in order.items.chunks(25) { ... }` lowers to `for_each!` but keeps direct access to Rust iteration adapters.

**Simple** — batching items with inline logic:
```rust
let sanitize = for_each!(order.items, item => SanitizeItem, key = |i| i.sku.clone(), batch = 50);
connect!(sanitize -> connectors::inventory::Reserve::new());
```

**Complex** — native syntax with partition-aware loop:
```rust
let bucket_stats = #[flow::for_each(key = |bucket: &Bucket| bucket.customer.clone(), concurrency = 4)]
for bucket in buckets(order) {
    compute_custom_stats(&bucket)                        // arbitrary Rust
};
connect!(bucket_stats -> connectors::warehouse::UpsertMetrics::new());
```

#### 4.9.3 Temporal, rate, and partition surfaces
- `partition_by!(target, key = expr, strategy = ChildWorkflow|LocalShard)` documents sharding so queue/Temporal hosts can spawn per-partition activities and policy can enforce fairness.
- `window!(...)` (or the `Window::*` nodes) already expose tumbling/sliding/session semantics; when combined with `partition_by!` they give schedulers enough metadata to place child workflows and configure timers (§4.8.1).
- `timeout!(scope, duration, behaviour = Cancel|Retry|DeadLetter)` records watchdog policy for nodes, edges, or subflows. Runtime hooks it to kernel timers; Temporal maps it to `ScheduleToClose`.
- `rate_limit!(scope, qps, burst, smoothing)` converts to a `ControlSurfaceKind::RateLimit`, allowing per-scope throttles to merge with tenant budgets.

**Example**:
```rust
partition_by!(upsert, key = |row: &MetricRow| row.customer_id.clone(), strategy = ChildWorkflow);
window!(agg, kind = "tumbling", size = "PT5M", allowed_lateness = "PT2M", watermark = "max(ts)-PT2M");
timeout!(scope = upsert, duration = "PT30S", behaviour = Retry);
rate_limit!(scope = workflow, qps = 2000, burst = 4000);
```

#### 4.9.4 Reliability surfaces (`on_error!`, `delivery!`, `compensate`)
- `delivery!(edge, ExactlyOnce|AtLeastOnce|AtMostOnce)` declares delivery guarantees and binds to dedupe/control metadata. Exactly-once edges require a `DedupeStore` capability and an idempotent sink contract (§5.4).
- `on_error!(scope, strategy = Retry {...} | DeadLetter {...} | Compensate { upstream = node_id } | Halt)` surfaces retry ownership and unwind strategies so both Temporal and queue hosts make identical decisions.
- `compensate` remains a node attribute (`#[node(..., compensate=Refund)]` or connector manifest field). The control surface records the relationship so policy can mandate compensators for critical capabilities.

##### Generic nodes & trait bounds

The `#[node]` / `subflow!` macros can support Rust generics and trait bounds (e.g., `#[node] pub async fn map<T: Serialize + JsonSchema>(...)`). At expansion time we can monomorphise each instantiation, but a few caveats apply:

- **Schema emission:** Flow IR must capture concrete JSON Schemas. Using open-ended trait bounds (`T: Serialize`) doesn’t give us enough to emit a stable schema; requiring `T: JsonSchema` or concrete enum wrappers keeps schemas deterministic.
- **Importer/Studio expectations:** Agents and tooling expect a finite set of inputs/outputs. Allowing “any `T`” makes Flow IR harder to reason about and complicates code generation for connectors.
- **Best practice:** prefer explicit enums or struct wrappers for the supported payloads (e.g., `enum ExtractionInput { Labs(LabPage), Imaging(ImagingPage) }`). Each variant carries `JsonSchema`, and the Flow IR exports a `oneOf` schema naturally. This matches policy-driven validation and keeps Studio UX predictable.
- **Helper macro:** use `#[flow_enum]` (from `dag-macros`) to tag these enums with the canonical derives (`serde::Serialize`, `serde::Deserialize`, `schemars::JsonSchema`) and `#[serde(tag = "type")]`. This keeps runtime schemas consistent and ensures agents do not forget the required annotations.

We can still offer generic helpers for author ergonomics, but workflow definitions should instantiate them with concrete types so Flow IR remains fully specified.

**Example**:
```rust
#[flow_enum]
pub enum ExtractionInput {
    Labs(LabPage),
    Imaging(ImagingPage),
    Notes { physician: String, contents: String },
}

#[node(name = "RouteExtraction", effects = "Pure", determinism = "Strict")]
async fn route(input: ExtractionInput) -> NodeResult<ExtractionInput> {
    Ok(input)
}

delivery!(charge_edge, ExactlyOnce);
on_error!(scope = fulfill, strategy = Compensate { upstream = charge });
on_error!(scope = notify, strategy = DeadLetter { queue = "notifications_dlq" });
```

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
    control_surfaces: Vec<ControlSurfaceIR>,
    policies: FlowPolicies,
    metadata: FlowMetadata,
    artifacts: Vec<ArtifactRef>,
}

> Canonical serialization lives in `schemas/flow_ir.schema.json` (JSON Schema draft 2020-12). The schema ships with a representative example (`examples[0]`) covering partitions, windows, compensation, and policy bindings so downstream tooling can validate without linking against the Rust types. A concrete artifact is checked in at `schemas/examples/etl_logs.flow_ir.json` for contract tests and sample agent interactions.

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
    determinism_hints: Vec<String>,
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

struct ControlSurfaceIR {
    id: SurfaceId,
    kind: ControlSurfaceKind,
    targets: Vec<TargetRef>,          // node ids or edge ids
    config: serde_json::Value,        // type-specific payload
    origin: SourceSpan,               // macro span for diagnostics
    lint: Option<LintHint>,           // opt-in suggestions (e.g., to wrap raw loops)
}

struct ArtifactRef {
    kind: ArtifactKind,               // dot, wit, json-schema, etc.
    path: PathBuf,
    format: Option<String>,
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
- `IdempotencySpec`: `key: KeyExpr`, `scope: IdemScope` (Node | Edge | Partition), `ttl_ms: u64`. Keys must reference deterministic fields only; TTL must satisfy host minimums (currently 300_000ms for queue bridges).
- `IdempotencyKeySpec`: canonical description of key encoding/hashing and deterministic field selection; see §5.2.1.
- `StateModel`: `None`, `Ephemeral`, or `Durable { consistency: Local | Global, ttl: Option<Duration>, version: SchemaVersion }`. Durable nodes require migration hooks (§5.5).
- `CompensationSpec`: maps to a compensating node/subflow plus timeout budget for SAGA unwinds.
- `CostEstimate`: static or learned estimator `Cost { cpu_ms, mem_mb, storage_bytes, tokens, currency }` with confidence interval.
- `CacheSpec`: optional cache description for Strict/Stable nodes including deterministic key, tier, TTL, and invalidation triggers.
- `CacheTier`: enumerates storage tiers (`Memory`, `Disk`, `Remote(region)`); policy may restrict tiers per profile.
- `CacheInvalidate`: `SchemaBump`, `EgressChange`, `CapabilityUpgrade`, `PolicyRule(Name)`.
- `ControlSurfaceKind`: `Switch`, `If`, `Loop`, `ForEach`, `Window`, `Partition`, `Merge`, `Join`, `Zip`, `ErrorHandler`, `Timeout`, `RateLimit`; each entry references the nodes/edges it governs plus macro origin metadata.
- `ArtifactKind`: exported artifacts persisted with the IR (`dot`, `wit`, `json-schema`, provider manifests) consumed by registry and Studio tooling.
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

### 5.2.2 Control-flow surfaces (opt-in)
- Control constructs are captured in `control_surfaces: Vec<ControlSurfaceIR>` so planners, Studio, and import/export tooling can reason about branching, looping, and temporal semantics without re-parsing Rust. Surfaces are emitted only when the related macro or helper is used; plain Rust loops/branches still work, but the validator can surface a `CTRL001 MissingControlSurfaceHint` warning when `policies.lint.require_control_hints == true`.
- **Switch/If**: produced by `Switch::<T>::new`, `If::<T>::new`, or `match!` helper macros. Config: `{ cases: [{ label, predicate, edges[] }], default_case }`. Drives UI widgets for branch editing and importer mapping from n8n “Split in Batches/If”.
- **Loop/ForEach**: optional `loop!`/`for_each!` wrappers annotate `for`/`while` constructs with `{ kind: While|Until|ForEach, break_on, concurrency_hint }`. When absent, analysis treats the loop as opaque Rust and may downgrade determinism if it detects unbounded iteration.
- **Partition**: `partition_by!` annotates edges with `{ hash, expression, routing = ChildWorkflow|LocalShard }` and binds shard-count hints used by queue/Temporal hosts (§5.5).
- **Window**: `window!` or `Window::tumbling/sliding/session` nodes register `{ window, size, allowed_lateness, watermark, lateness_output }` so Temporal lowering can emit timers.
- **Merge/Join/Zip**: emitted by corresponding std combinators. Config: `{ type: Merge|Join|Zip, keyed: bool, key_expr, ordering_contract }`; allows determinism and backpressure rules to validate fan-in/fan-out.
- **ErrorHandler**: `on_error!` produces surfaces describing `{ scope, strategy, retry, dead_letter }` to keep runtime/orchestrator retries aligned.
- **Timeout**: `timeout!` writes `{ scope, duration, behaviour = Cancel|Retry|DeadLetter }`; Temporal maps to `ScheduleToClose` while queue host wires watchdogs.
- **RateLimit**: `rate_limit!` surfaces `{ scope, qps, burst, smoothing }` so policy can merge runtime quotas with tenant budgets.
- **Respond policy**: `HttpTrigger::respond(...)` and `Respond(stream=...)` register surfaces describing handshake deadlines, streaming semantics, and cancellation wiring.
- Additional macros (`hitl!`, `delivery!`, `vars!`) map to existing `CheckpointIR`, `EdgeIR.delivery`, and `FlowPolicies` respectively and do not get their own control surface entries.

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
- `capabilities` crate ships in-memory scaffolds (`MemoryKv`, `MemoryBlobStore`, `MemoryQueue`, `MemoryCache`) for unit tests and local runs. Production adapters (Workers KV/R2, Neon/Postgres, durable queues) plug into the same traits so Flow IR metadata and validator hints stay consistent regardless of backend.

#### 6.3.1 Capability versioning & compatibility
- Capabilities advertise `(name, semver)`; Flow IR records required ranges on each node binding. Hosts must provide compatible versions at deploy time; otherwise validator `CAPV101 IncompatibleCapabilityVersion` fires.
- Registry stores capability compatibility matrices; upgrades triggering major version bumps require certification reruns for dependent nodes.
- Policy gates can block automatic upgrades or enforce staged rollouts with per-tenant overrides.

### 6.4 Host adapters
- **Tokio engine:** In-process scheduler, fluent for dev and unit tests.
- **Queue/Redis:** HTTP/webhook front-end enqueues items; workers use `ActivityRuntime` to run nodes, ensure idempotency, ack results.
- **Axum adapter:** Binds triggers to HTTP routes; injects `RequestScope`; handles `Respond` bridging with oneshot channels; allows session/middleware outside kernel. This adapter is a fully-fledged web host—teams can deploy it as their own HTTP server. For platform-managed offerings we layer an invocation gateway (see below) that fronts the same executor while adding tenancy-aware routing, throttling, audit, and metering.
- **Invocation gateway (managed ingress):** Optional control-plane service we operate for paid tiers. Provides elastic ingress endpoints (invoke, poll, stream) that forward to the appropriate runtime profile (inline Axum, queue workers, WASM edge). Handles authentication, per-tenant quotas, cost attribution, and automatic cold-start mitigation. Self-hosted deployments can skip this layer and run the Axum adapter directly; managed tenants get the gateway as their hardened entrypoint.
- **WASM host:** Executes WIT-defined nodes; capabilities exposed via WIT handles; strict allowlist (no network unless declared).
- **Python/gRPC host:** `plugin-python` crate starts gRPC subprocess; nodes call remote functions (with schema handshake).
- **Temporal adapter:** Converts Flow IR pipelines to Temporal workflows (Go/TS) plus activities; we run a Rust activity worker using our Node runtime.

#### 6.4.1 Web streaming & cancellation
- Web profile supports streaming responses via Server-Sent Events (SSE), chunked HTTP/1.1 transfer, and WebSocket bridging. `Respond(stream = true)` wires edge backpressure to HTTP body writes with 64 KiB default chunks.
- Client disconnects propagate a cancellation token; nodes observe `ctx.is_cancelled()` and should cease work promptly. Cancel events emit `WEB201 ClientDisconnected` telemetry.
- Kernel returns an `ExecutionResult` (`Value` | `Stream`). `StreamHandle` owns the active `FlowInstance`, closes over the node cancellation token, and drops the semaphore permit only when the consumer finishes or disconnects. Hosts map the handle into transport-specific streaming (Axum → SSE body).
- CLI adds `--stream` for local runs; streaming examples (e.g., S2 site telemetry) error when invoked without the flag to avoid accidental truncation. `flows run serve` exposes `/site/stream` and forwards JSON payloads to the trigger before switching the HTTP response into SSE mode.
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
- Shared registry: `dag_core::determinism` exposes the canonical map of resource hints (e.g., `resource::clock`, `resource::rng`) to minimum determinism levels. Macros and plugins record hints when they detect sensitive resources; the validator raises `DET302` if a node claims a stricter level than the registry permits. Crates can extend the registry at runtime to cover new capability families without modifying the core runtime.
- Effect registry: `dag_core::effects_registry` mirrors the determinism flow for side-effects. Built-in hints cover HTTP/database writes; connectors register additional hints (e.g., `connector::stripe::charge`). Validation emits `EFFECT201` when a node claims `Pure`/`ReadOnly` despite the registry requiring `Effectful`. Defaults remain pessimistic; authors must opt in to stronger guarantees consciously.
- Canonical hints seeded in the runtime today: `resource::http::{read,write}`, `resource::db::{read,write}`, `resource::kv::{read,write}`, `resource::blob::{read,write}`, `resource::queue::{publish,consume}`, `resource::clock`, and `resource::rng`. Platform adapters (Workers KV/R2, Neon/Postgres, Durable Objects, etc.) register additional aliases so validators and Studio surface accurate policies automatically.
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
- Instrumentation catalog lives in `impl-docs/metrics.md`; all names are prefixed `latticeflow.*`.
- Executors emit gauges/counters/histograms for active nodes, queue depth, node latency, cancellations, capture backpressure, and stream client counts (`latticeflow.executor.*`).
- Hosts expose HTTP request totals/latency, in-flight gauges, SSE client gauges, and deadline breaches (`latticeflow.host.*`).
- CLI runs surface per-node success/failure counters and capture totals plus an aggregated table/`--json` snapshot (`latticeflow.cli.*`).
- Telemetry rides OpenTelemetry + `metrics` and can export to Prometheus/DataDog. Default recorder is no-op; features unlock exporters.
- Cardinality controls: metric label allowlist prohibits raw IDs; exemplars carry hashed run IDs instead. Profiles cap labels (≤100 distinct values per 5 min); overages drop labels and emit WARN `METRIC201 CardinalityExceeded` while keeping exemplars.

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
````




# Instruction
