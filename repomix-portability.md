# Files

## File: crates/capabilities/src/hints.rs
````rust
//! Static mapping between capability aliases and the resource hints they imply.
//!
//! This module is consumed at compile time by `dag-macros` so that node
//! declarations automatically inherit the correct effect/determinism hints
//! without bespoke wiring in every crate.

use crate::{blob, clock, db, dedupe, http, kv, queue, rng};

/// Canonical set of hints exported for a capability.
#[derive(Debug, Clone, Copy)]
pub struct ResourceHintSet {
    pub effect_hints: &'static [&'static str],
    pub determinism_hints: &'static [&'static str],
}

impl ResourceHintSet {
    pub const fn new(
        effect_hints: &'static [&'static str],
        determinism_hints: &'static [&'static str],
    ) -> Self {
        Self {
            effect_hints,
            determinism_hints,
        }
    }

    pub const EMPTY: ResourceHintSet = ResourceHintSet::new(&[], &[]);

    pub fn is_empty(self) -> bool {
        self.effect_hints.is_empty() && self.determinism_hints.is_empty()
    }
}

const HTTP_READ_EFFECT: [&str; 1] = [http::HINT_HTTP_READ];
const HTTP_WRITE_EFFECT: [&str; 1] = [http::HINT_HTTP_WRITE];
const HTTP_DETERMINISM: [&str; 1] = [http::HINT_HTTP];

const CLOCK_DETERMINISM: [&str; 1] = [clock::HINT_CLOCK];
const RNG_DETERMINISM: [&str; 1] = [rng::HINT_RNG];

const DB_READ_EFFECT: [&str; 1] = [db::HINT_DB_READ];
const DB_WRITE_EFFECT: [&str; 1] = [db::HINT_DB_WRITE];
const DB_DETERMINISM: [&str; 1] = [db::HINT_DB];

const KV_READ_EFFECT: [&str; 1] = [kv::HINT_KV_READ];
const KV_WRITE_EFFECT: [&str; 1] = [kv::HINT_KV_WRITE];
const KV_DETERMINISM: [&str; 1] = [kv::HINT_KV];

const QUEUE_PUBLISH_EFFECT: [&str; 1] = [queue::HINT_QUEUE_PUBLISH];
const QUEUE_CONSUME_EFFECT: [&str; 1] = [queue::HINT_QUEUE_CONSUME];
const QUEUE_DETERMINISM: [&str; 1] = [queue::HINT_QUEUE];

const BLOB_READ_EFFECT: [&str; 1] = [blob::HINT_BLOB_READ];
const BLOB_WRITE_EFFECT: [&str; 1] = [blob::HINT_BLOB_WRITE];
const BLOB_DETERMINISM: [&str; 1] = [blob::HINT_BLOB];

const DEDUPE_EFFECT: [&str; 1] = [dedupe::HINT_DEDUPE_WRITE];
const DEDUPE_DETERMINISM: [&str; 1] = [dedupe::HINT_DEDUPE];

/// Infer canonical hints for a capability declaration based on its alias and identifier.
pub fn infer(alias: &str, capability_ident: &str) -> ResourceHintSet {
    let alias_lower = alias.to_ascii_lowercase();
    let ident_lower = capability_ident.to_ascii_lowercase();

    if alias_lower.is_empty() && ident_lower.is_empty() {
        return ResourceHintSet::EMPTY;
    }

    if alias_lower.contains("clock") || ident_lower.contains("clock") {
        clock::ensure_registered();
        return ResourceHintSet::new(&[], &CLOCK_DETERMINISM);
    }

    if alias_lower.contains("rng")
        || alias_lower.contains("random")
        || ident_lower.contains("rng")
        || ident_lower.contains("random")
    {
        rng::ensure_registered();
        return ResourceHintSet::new(&[], &RNG_DETERMINISM);
    }

    if alias_lower.contains("http") || ident_lower.contains("http") {
        http::ensure_registered();
        let write_tokens = [
            "write", "post", "put", "patch", "delete", "send", "emit", "publish", "producer",
            "upsert", "insert", "update",
        ];
        if write_tokens
            .iter()
            .any(|token| alias_lower.contains(token) || ident_lower.contains(token))
        {
            return ResourceHintSet::new(&HTTP_WRITE_EFFECT, &HTTP_DETERMINISM);
        }

        let read_tokens = [
            "read", "fetch", "get", "load", "receive", "consumer", "listen",
        ];
        if read_tokens
            .iter()
            .any(|token| alias_lower.contains(token) || ident_lower.contains(token))
        {
            return ResourceHintSet::new(&HTTP_READ_EFFECT, &HTTP_DETERMINISM);
        }

        return ResourceHintSet::new(&[], &HTTP_DETERMINISM);
    }

    if alias_lower.contains("db")
        || alias_lower.contains("sql")
        || alias_lower.contains("postgres")
        || alias_lower.contains("pg")
        || ident_lower.contains("db")
        || ident_lower.contains("sql")
        || ident_lower.contains("postgres")
        || ident_lower.contains("pg")
    {
        db::ensure_registered();
        let read_tokens = ["read", "select", "fetch", "query"];
        if read_tokens
            .iter()
            .any(|token| alias_lower.contains(token) || ident_lower.contains(token))
        {
            return ResourceHintSet::new(&DB_READ_EFFECT, &DB_DETERMINISM);
        }
        return ResourceHintSet::new(&DB_WRITE_EFFECT, &DB_DETERMINISM);
    }

    if alias_lower.contains("kv")
        || alias_lower.contains("cache")
        || ident_lower.contains("kv")
        || ident_lower.contains("workers_kv")
    {
        kv::ensure_registered();
        let read_tokens = ["read", "get", "fetch", "lookup"];
        if read_tokens
            .iter()
            .any(|token| alias_lower.contains(token) || ident_lower.contains(token))
        {
            return ResourceHintSet::new(&KV_READ_EFFECT, &KV_DETERMINISM);
        }
        return ResourceHintSet::new(&KV_WRITE_EFFECT, &KV_DETERMINISM);
    }

    if alias_lower.contains("queue") || ident_lower.contains("queue") {
        queue::ensure_registered();
        let consume_tokens = ["consume", "dequeue", "receive", "poll"];
        if consume_tokens
            .iter()
            .any(|token| alias_lower.contains(token) || ident_lower.contains(token))
        {
            return ResourceHintSet::new(&QUEUE_CONSUME_EFFECT, &QUEUE_DETERMINISM);
        }
        return ResourceHintSet::new(&QUEUE_PUBLISH_EFFECT, &QUEUE_DETERMINISM);
    }

    if alias_lower.contains("dedupe")
        || alias_lower.contains("idem")
        || ident_lower.contains("dedupe")
        || ident_lower.contains("idempot")
    {
        dedupe::ensure_registered();
        return ResourceHintSet::new(&DEDUPE_EFFECT, &DEDUPE_DETERMINISM);
    }

    if alias_lower.contains("blob")
        || alias_lower.contains("object")
        || ident_lower.contains("blob")
        || ident_lower.contains("object")
        || ident_lower.contains("storage")
    {
        blob::ensure_registered();
        let read_tokens = ["read", "fetch", "get", "download"];
        if read_tokens
            .iter()
            .any(|token| alias_lower.contains(token) || ident_lower.contains(token))
        {
            return ResourceHintSet::new(&BLOB_READ_EFFECT, &BLOB_DETERMINISM);
        }
        return ResourceHintSet::new(&BLOB_WRITE_EFFECT, &BLOB_DETERMINISM);
    }

    ResourceHintSet::EMPTY
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn http_write_derives_effect_and_determinism() {
        let hints = infer("http_sender", "HttpWrite");
        assert_eq!(hints.effect_hints, &HTTP_WRITE_EFFECT);
        assert_eq!(hints.determinism_hints, &HTTP_DETERMINISM);
    }

    #[test]
    fn clock_derives_clock_hint() {
        let hints = infer("clock", "SystemClock");
        assert!(hints.effect_hints.is_empty());
        assert_eq!(hints.determinism_hints, &CLOCK_DETERMINISM);
    }

    #[test]
    fn unknown_alias_defaults_to_empty() {
        let hints = infer("custom", "CustomCapability");
        assert!(hints.effect_hints.is_empty());
        assert!(hints.determinism_hints.is_empty());
    }

    #[test]
    fn db_read_and_write_infer_correctly() {
        let read = infer("db_read_pool", "PostgresRead");
        assert_eq!(read.effect_hints, &DB_READ_EFFECT);
        assert_eq!(read.determinism_hints, &DB_DETERMINISM);

        let write = infer("postgres_writer", "PgWrite");
        assert_eq!(write.effect_hints, &DB_WRITE_EFFECT);
        assert_eq!(write.determinism_hints, &DB_DETERMINISM);
    }

    #[test]
    fn kv_read_and_write_infer_correctly() {
        let read = infer("kv_get", "WorkersKv");
        assert_eq!(read.effect_hints, &KV_READ_EFFECT);
        assert_eq!(read.determinism_hints, &KV_DETERMINISM);

        let write = infer("kv_put", "WorkersKv");
        assert_eq!(write.effect_hints, &KV_WRITE_EFFECT);
        assert_eq!(write.determinism_hints, &KV_DETERMINISM);
    }

    #[test]
    fn queue_publish_and_consume_infer_correctly() {
        let publish = infer("queue_publish", "QueueProducer");
        assert_eq!(publish.effect_hints, &QUEUE_PUBLISH_EFFECT);
        assert_eq!(publish.determinism_hints, &QUEUE_DETERMINISM);

        let consume = infer("QUEUE_CONSUME", "QueueConsumer");
        assert_eq!(consume.effect_hints, &QUEUE_CONSUME_EFFECT);
        assert_eq!(consume.determinism_hints, &QUEUE_DETERMINISM);
    }

    #[test]
    fn blob_read_and_write_infer_correctly() {
        let read = infer("blob_fetch", "BlobReader");
        assert_eq!(read.effect_hints, &BLOB_READ_EFFECT);
        assert_eq!(read.determinism_hints, &BLOB_DETERMINISM);

        let write = infer("blob_write", "BlobWriter");
        assert_eq!(write.effect_hints, &BLOB_WRITE_EFFECT);
        assert_eq!(write.determinism_hints, &BLOB_DETERMINISM);
    }

    #[test]
    fn dedupe_infers_effectful_hint() {
        let hints = infer("dedupe_store", "RedisDedupe");
        assert_eq!(hints.effect_hints, &DEDUPE_EFFECT);
        assert_eq!(hints.determinism_hints, &DEDUPE_DETERMINISM);
    }
}
````

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

## File: crates/kernel-exec/src/lib.rs
````rust
//! In-process executor for validated Flow IR graphs.
//!
//! This module provides a cooperative Tokio scheduler that wires node handlers
//! via bounded channels. It honours buffer capacities expressed on edges,
//! propagates cancellation on failures or deadlines, and exposes a thin
//! interface for hosts to drive requests (e.g. HTTP triggers).

use std::collections::HashMap;
use std::fmt;
use std::future::Future;
use std::marker::PhantomData;
use std::path::PathBuf;
use std::pin::Pin;
use std::sync::Arc;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::task::{Context, Poll};
use std::time::{Duration, Instant};

use anyhow::{Context as AnyhowContext, Result as AnyhowResult};
use async_trait::async_trait;
use capabilities::{ResourceAccess, ResourceBag, context};
use dag_core::{NodeError, NodeResult, Profile};
use futures::{Stream, StreamExt};
use kernel_plan::ValidatedIR;
use serde::Serialize;
use serde::de::DeserializeOwned;
use serde_json::Value as JsonValue;
use tempfile::{Builder as TempDirBuilder, TempDir};
use thiserror::Error;
use tokio::fs;
use tokio::sync::{
    OwnedSemaphorePermit, Semaphore, mpsc,
    mpsc::error::{SendError, TrySendError},
};
use tokio::time;
use tokio_stream::{StreamMap, wrappers::ReceiverStream};
use tokio_util::sync::CancellationToken;
use tracing::{debug, error, instrument, trace, warn};
use uuid::Uuid;

const DEFAULT_EDGE_CAPACITY: usize = 32;
const DEFAULT_TRIGGER_CAPACITY: usize = 8;
const DEFAULT_CAPTURE_CAPACITY: usize = 8;

/// Registry mapping node identifiers to executable handlers.
#[derive(Default)]
pub struct NodeRegistry {
    handlers: HashMap<&'static str, Arc<dyn NodeHandler>>,
}

impl NodeRegistry {
    /// Create an empty registry.
    pub fn new() -> Self {
        Self {
            handlers: HashMap::new(),
        }
    }
    /// Register a new async handler identified by the fully-qualified node identifier.
    pub fn register_fn<F, Fut, In, Out>(
        &mut self,
        identifier: &'static str,
        handler: F,
    ) -> Result<(), RegistryError>
    where
        F: Fn(In) -> Fut + Send + Sync + 'static,
        Fut: Future<Output = NodeResult<Out>> + Send + 'static,
        In: DeserializeOwned + Send + Sync + 'static,
        Out: Serialize + Send + Sync + 'static,
    {
        if self.handlers.contains_key(identifier) {
            return Err(RegistryError::Duplicate(identifier));
        }
        let wrapper = FunctionHandler::new(handler);
        self.handlers
            .insert(identifier, Arc::new(wrapper) as Arc<dyn NodeHandler>);
        Ok(())
    }

    /// Register a streaming handler returning an async stream of payloads.
    pub fn register_stream_fn<F, Fut, In, S, Item>(
        &mut self,
        identifier: &'static str,
        handler: F,
    ) -> Result<(), RegistryError>
    where
        F: Fn(In) -> Fut + Send + Sync + 'static,
        Fut: Future<Output = NodeResult<S>> + Send + 'static,
        In: DeserializeOwned + Send + Sync + 'static,
        S: Stream<Item = NodeResult<Item>> + Send + 'static,
        Item: Serialize + Send + Sync + 'static,
    {
        if self.handlers.contains_key(identifier) {
            return Err(RegistryError::Duplicate(identifier));
        }
        let wrapper = StreamingFunctionHandler::new(handler);
        self.handlers
            .insert(identifier, Arc::new(wrapper) as Arc<dyn NodeHandler>);
        Ok(())
    }

    /// Lookup a handler by implementation identifier.
    pub(crate) fn handler(&self, identifier: &str) -> Option<Arc<dyn NodeHandler>> {
        self.handlers.get(identifier).cloned()
    }
}

/// Errors produced when registering node handlers.
#[derive(Debug, Error, PartialEq, Eq)]
pub enum RegistryError {
    /// Handler already registered.
    #[error("node `{0}` already registered")]
    Duplicate(&'static str),
}

#[async_trait]
trait NodeHandler: Send + Sync {
    async fn invoke(&self, input: JsonValue, ctx: &NodeContext) -> NodeResult<NodeOutput>;
}

struct FunctionHandler<F, Fut, In, Out>
where
    F: Fn(In) -> Fut + Send + Sync + 'static,
    Fut: Future<Output = NodeResult<Out>> + Send + 'static,
    In: DeserializeOwned + Send + 'static,
    Out: Serialize + Send + 'static,
{
    inner: F,
    _marker: PhantomData<(In, Out)>,
}

impl<F, Fut, In, Out> FunctionHandler<F, Fut, In, Out>
where
    F: Fn(In) -> Fut + Send + Sync + 'static,
    Fut: Future<Output = NodeResult<Out>> + Send + 'static,
    In: DeserializeOwned + Send + Sync + 'static,
    Out: Serialize + Send + Sync + 'static,
{
    fn new(inner: F) -> Self {
        Self {
            inner,
            _marker: PhantomData,
        }
    }
}

#[async_trait]
impl<F, Fut, In, Out> NodeHandler for FunctionHandler<F, Fut, In, Out>
where
    F: Fn(In) -> Fut + Send + Sync + 'static,
    Fut: Future<Output = NodeResult<Out>> + Send + 'static,
    In: DeserializeOwned + Send + Sync + 'static,
    Out: Serialize + Send + Sync + 'static,
{
    async fn invoke(&self, input: JsonValue, _ctx: &NodeContext) -> NodeResult<NodeOutput> {
        let deserialised: In = serde_json::from_value(input.clone())
            .map_err(|err| NodeError::new(format!("failed to deserialize node input: {err}")))?;
        let output = (self.inner)(deserialised).await?;
        let json = serde_json::to_value(output)
            .map_err(|err| NodeError::new(format!("failed to serialise node output: {err}")))?;
        Ok(NodeOutput::Value(json))
    }
}

struct StreamingFunctionHandler<F, Fut, In, S, Item>
where
    F: Fn(In) -> Fut + Send + Sync + 'static,
    Fut: Future<Output = NodeResult<S>> + Send + 'static,
    In: DeserializeOwned + Send + Sync + 'static,
    S: Stream<Item = NodeResult<Item>> + Send + 'static,
    Item: Serialize + Send + Sync + 'static,
{
    inner: F,
    _marker: PhantomData<(In, Item)>,
}

impl<F, Fut, In, S, Item> StreamingFunctionHandler<F, Fut, In, S, Item>
where
    F: Fn(In) -> Fut + Send + Sync + 'static,
    Fut: Future<Output = NodeResult<S>> + Send + 'static,
    In: DeserializeOwned + Send + Sync + 'static,
    S: Stream<Item = NodeResult<Item>> + Send + 'static,
    Item: Serialize + Send + Sync + 'static,
{
    fn new(inner: F) -> Self {
        Self {
            inner,
            _marker: PhantomData,
        }
    }
}

#[async_trait]
impl<F, Fut, In, S, Item> NodeHandler for StreamingFunctionHandler<F, Fut, In, S, Item>
where
    F: Fn(In) -> Fut + Send + Sync + 'static,
    Fut: Future<Output = NodeResult<S>> + Send + 'static,
    In: DeserializeOwned + Send + Sync + 'static,
    S: Stream<Item = NodeResult<Item>> + Send + 'static,
    Item: Serialize + Send + Sync + 'static,
{
    async fn invoke(&self, input: JsonValue, _ctx: &NodeContext) -> NodeResult<NodeOutput> {
        let deserialised: In = serde_json::from_value(input.clone())
            .map_err(|err| NodeError::new(format!("failed to deserialize node input: {err}")))?;
        let stream = (self.inner)(deserialised).await?;
        let json_stream = stream.map(|item| {
            item.and_then(|value| {
                serde_json::to_value(value).map_err(|err| {
                    NodeError::new(format!("failed to serialise stream item: {err}"))
                })
            })
        });
        Ok(NodeOutput::Stream(json_stream.boxed()))
    }
}

/// Lightweight execution context passed to nodes.
#[derive(Clone)]
pub struct NodeContext {
    cancellation: CancellationToken,
    resources: Arc<dyn ResourceAccess>,
}

impl NodeContext {
    fn new(cancellation: CancellationToken, resources: Arc<dyn ResourceAccess>) -> Self {
        Self {
            cancellation,
            resources,
        }
    }

    /// Returns `true` if execution has been cancelled.
    pub fn is_cancelled(&self) -> bool {
        self.cancellation.is_cancelled()
    }

    /// Access the underlying cancellation token.
    pub fn token(&self) -> CancellationToken {
        self.cancellation.clone()
    }

    /// Borrow the resource access collection for capability lookups.
    pub fn resources(&self) -> &dyn ResourceAccess {
        self.resources.as_ref()
    }

    /// Clone the underlying resource access handle.
    pub fn resource_handle(&self) -> Arc<dyn ResourceAccess> {
        self.resources.clone()
    }
}

enum CapturedPayload {
    Value(JsonValue),
    Stream(StreamingCapture),
}

struct StreamingCapture {
    alias: String,
    stream: futures::stream::BoxStream<'static, NodeResult<JsonValue>>,
    cancellation: CancellationToken,
    permit: Option<OwnedSemaphorePermit>,
}

impl StreamingCapture {
    fn new(
        alias: String,
        stream: futures::stream::BoxStream<'static, NodeResult<JsonValue>>,
        cancellation: CancellationToken,
        permit: Option<OwnedSemaphorePermit>,
    ) -> Self {
        Self {
            alias,
            stream,
            cancellation,
            permit,
        }
    }
}

/// Result produced by a node handler.
pub enum NodeOutput {
    /// Single JSON value to forward downstream.
    Value(JsonValue),
    /// Stream of JSON payloads emitted incrementally.
    Stream(futures::stream::BoxStream<'static, NodeResult<JsonValue>>),
    /// Node intentionally emitted no output.
    None,
}

impl fmt::Debug for NodeOutput {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            NodeOutput::Value(value) => f.debug_tuple("Value").field(value).finish(),
            NodeOutput::Stream(_) => f.write_str("Stream(..)"),
            NodeOutput::None => f.write_str("None"),
        }
    }
}

#[derive(Debug)]
struct ExecutorMetrics {
    flow: Arc<str>,
    profile: &'static str,
}

impl ExecutorMetrics {
    fn new(flow: String, profile: Profile) -> Self {
        Self {
            flow: Arc::from(flow),
            profile: profile_label(profile),
        }
    }

    fn flow(&self) -> &str {
        self.flow.as_ref()
    }

    fn profile(&self) -> &'static str {
        self.profile
    }
}

impl ExecutorMetrics {
    fn queue_tracker(self: &Arc<Self>, edge: Arc<str>, capacity: usize) -> Arc<QueueDepthTracker> {
        Arc::new(QueueDepthTracker::new(Arc::clone(self), edge, capacity))
    }

    fn track_node(self: &Arc<Self>, node: &str) -> NodeRunGuard {
        let flow_label = self.flow().to_string();
        let node_label = node.to_string();
        let profile_label = self.profile();
        metrics::gauge!(
            "latticeflow.executor.active_nodes",
            "flow" => flow_label,
            "node" => node_label,
            "profile" => profile_label
        )
        .increment(1.0);
        NodeRunGuard::new(Arc::clone(self), node.to_string())
    }

    fn record_node_error(self: &Arc<Self>, node: &str, error_kind: &str) {
        let flow_label = self.flow().to_string();
        let node_label = node.to_string();
        let error_label = error_kind.to_string();
        let profile_label = self.profile();
        metrics::counter!(
            "latticeflow.executor.node_errors_total",
            "flow" => flow_label,
            "node" => node_label,
            "profile" => profile_label,
            "error_kind" => error_label
        )
        .increment(1);
    }

    fn record_cancellation(self: &Arc<Self>, node: &str, reason: &str) {
        let flow_label = self.flow().to_string();
        let node_label = node.to_string();
        let reason_label = reason.to_string();
        let profile_label = self.profile();
        metrics::counter!(
            "latticeflow.executor.cancellations_total",
            "flow" => flow_label,
            "node" => node_label,
            "profile" => profile_label,
            "reason" => reason_label
        )
        .increment(1);
    }

    fn observe_capture_backpressure(self: &Arc<Self>, capture: &str, waited: Duration) {
        let millis = waited.as_secs_f64() * 1_000.0;
        let flow_label = self.flow().to_string();
        let capture_label = capture.to_string();
        let profile_label = self.profile();
        metrics::histogram!(
            "latticeflow.executor.capture_backpressure_ms",
            "flow" => flow_label,
            "capture" => capture_label,
            "profile" => profile_label
        )
        .record(millis);
    }

    fn record_stream_spawn(self: &Arc<Self>, node: &str) {
        let flow_label = self.flow().to_string();
        let node_label = node.to_string();
        let profile_label = self.profile();
        metrics::counter!(
            "latticeflow.executor.stream_clients_total",
            "flow" => flow_label,
            "node" => node_label,
            "profile" => profile_label
        )
        .increment(1);
    }
}

#[derive(Debug, Error)]
enum SpillError {
    #[error("failed to serialise payload for spill: {0}")]
    Serialize(#[from] serde_json::Error),
    #[error("failed to persist/load spill file: {0}")]
    Io(#[from] std::io::Error),
}

#[derive(Debug)]
struct BlobSpill {
    root: PathBuf,
    _guard: TempDir,
}

impl BlobSpill {
    fn new(flow_id: &str, tier: &str) -> AnyhowResult<Self> {
        let dir = TempDirBuilder::new()
            .prefix(&format!("lf-spill-{flow_id}-{tier}-"))
            .tempdir()
            .context("create spill directory")?;
        let root = dir.path().to_path_buf();
        Ok(Self { root, _guard: dir })
    }

    async fn persist(&self, payload: &JsonValue) -> Result<SpillRecord, SpillError> {
        let bytes = serde_json::to_vec(payload)?;
        let key = format!("{}.json", Uuid::new_v4());
        let path = self.root.join(&key);
        fs::write(&path, &bytes).await?;
        Ok(SpillRecord { key })
    }

    async fn load(&self, key: &str) -> Result<JsonValue, SpillError> {
        let path = self.root.join(key);
        let bytes = fs::read(&path).await?;
        let value = serde_json::from_slice(&bytes)?;
        Ok(value)
    }

    async fn remove(&self, key: &str) -> Result<(), SpillError> {
        let path = self.root.join(key);
        if path.exists() {
            fs::remove_file(path).await?;
        }
        Ok(())
    }
}

#[derive(Debug)]
struct SpillRecord {
    key: String,
}

#[derive(Clone)]
struct SpillContext {
    storage: Arc<BlobSpill>,
    threshold_bytes: Option<u64>,
}

impl SpillContext {
    fn new(storage: Arc<BlobSpill>, threshold_bytes: Option<u64>) -> Self {
        Self {
            storage,
            threshold_bytes,
        }
    }

    fn exceeds_threshold(&self, payload: &JsonValue) -> bool {
        match self.threshold_bytes {
            Some(limit) => match serde_json::to_vec(payload) {
                Ok(bytes) => bytes.len() as u64 >= limit,
                Err(_) => false,
            },
            None => false,
        }
    }

    async fn spill_and_send(
        &self,
        sender: &mpsc::Sender<FlowMessage>,
        tracker: Arc<QueueDepthTracker>,
        message: FlowMessage,
    ) -> Result<(), SendError<FlowMessage>> {
        let FlowMessage::Data {
            payload,
            permit,
            queue_tracker,
        } = message
        else {
            return Err(SendError(FlowMessage::Data {
                payload: JsonValue::Null,
                permit: None,
                queue_tracker: None,
            }));
        };

        let record = match self.storage.persist(&payload).await {
            Ok(record) => record,
            Err(_) => {
                return Err(SendError(FlowMessage::Data {
                    payload,
                    permit,
                    queue_tracker,
                }));
            }
        };

        drop(payload);

        let spilled = FlowMessage::Spilled {
            key: record.key.clone(),
            storage: Arc::clone(&self.storage),
            permit,
            queue_tracker,
        };

        match sender.reserve().await {
            Ok(permit) => {
                tracker.increment();
                permit.send(spilled);
                Ok(())
            }
            Err(_) => {
                let _ = self.storage.remove(&record.key).await;
                Err(SendError(spilled))
            }
        }
    }
}

struct SpillManager {
    storages: HashMap<String, Arc<BlobSpill>>,
}

impl SpillManager {
    fn new(flow: &dag_core::FlowIR) -> AnyhowResult<Self> {
        let mut storages = HashMap::new();
        let flow_id = flow.id.as_str().to_string();
        for edge in &flow.edges {
            if let Some(tier) = &edge.buffer.spill_tier {
                if !storages.contains_key(tier) {
                    let storage = Arc::new(BlobSpill::new(&flow_id, tier)?);
                    storages.insert(tier.clone(), storage);
                }
            }
        }
        Ok(Self { storages })
    }

    fn context_for(&self, policy: &dag_core::BufferPolicy) -> Option<Arc<SpillContext>> {
        policy.spill_tier.as_ref().and_then(|tier| {
            self.storages.get(tier).map(|storage| {
                Arc::new(SpillContext::new(
                    Arc::clone(storage),
                    policy.spill_threshold_bytes,
                ))
            })
        })
    }
}

#[derive(Clone)]
struct InstrumentedSender {
    sender: mpsc::Sender<FlowMessage>,
    tracker: Arc<QueueDepthTracker>,
    spill: Option<Arc<SpillContext>>,
}

impl InstrumentedSender {
    fn new(
        sender: mpsc::Sender<FlowMessage>,
        tracker: Arc<QueueDepthTracker>,
        spill: Option<Arc<SpillContext>>,
    ) -> Self {
        Self {
            sender,
            tracker,
            spill,
        }
    }

    async fn send(
        &self,
        payload: JsonValue,
        permit: Option<OwnedSemaphorePermit>,
    ) -> Result<(), mpsc::error::SendError<FlowMessage>> {
        let tracker = self.tracker.clone();
        let mut message = FlowMessage::Data {
            payload,
            permit,
            queue_tracker: Some(tracker.clone()),
        };

        if let Some(spill) = &self.spill {
            if matches!(&message, FlowMessage::Data { payload, .. } if spill.exceeds_threshold(payload))
            {
                return spill.spill_and_send(&self.sender, tracker, message).await;
            }

            match self.sender.try_send(message) {
                Ok(()) => {
                    tracker.increment();
                    return Ok(());
                }
                Err(TrySendError::Full(returned)) => {
                    message = returned;
                    return spill.spill_and_send(&self.sender, tracker, message).await;
                }
                Err(TrySendError::Closed(returned)) => return Err(SendError(returned)),
            }
        }

        let result = self.sender.send(message).await;
        if result.is_ok() {
            tracker.increment();
        }
        result
    }
}

struct QueueDepthTracker {
    metrics: Arc<ExecutorMetrics>,
    edge: Arc<str>,
    depth: AtomicUsize,
    capacity: usize,
}

impl QueueDepthTracker {
    fn new(metrics: Arc<ExecutorMetrics>, edge: Arc<str>, capacity: usize) -> Self {
        let tracker = Self {
            metrics,
            edge,
            depth: AtomicUsize::new(0),
            capacity,
        };
        tracker.publish(0);
        tracker
    }

    fn increment(&self) {
        let depth = self.depth.fetch_add(1, Ordering::AcqRel) + 1;
        self.publish(depth.min(self.capacity));
    }

    fn decrement(&self) {
        let prev = self.depth.fetch_sub(1, Ordering::AcqRel);
        let depth = prev.saturating_sub(1).min(self.capacity);
        self.publish(depth);
    }

    fn publish(&self, depth: usize) {
        let flow_label = self.metrics.flow().to_string();
        let edge_label = self.edge.to_string();
        let profile_label = self.metrics.profile();
        metrics::gauge!(
            "latticeflow.executor.queue_depth",
            "flow" => flow_label,
            "edge" => edge_label,
            "profile" => profile_label
        )
        .set(depth as f64);
    }
}

impl fmt::Debug for QueueDepthTracker {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_struct("QueueDepthTracker")
            .field("edge", &self.edge)
            .field("capacity", &self.capacity)
            .field("depth", &self.depth.load(Ordering::Relaxed))
            .finish()
    }
}

struct NodeRunGuard {
    metrics: Arc<ExecutorMetrics>,
    node: String,
    start: Instant,
}

impl NodeRunGuard {
    fn new(metrics: Arc<ExecutorMetrics>, node: String) -> Self {
        Self {
            metrics,
            node,
            start: Instant::now(),
        }
    }
}

impl Drop for NodeRunGuard {
    fn drop(&mut self) {
        let elapsed = self.start.elapsed().as_secs_f64() * 1_000.0;
        let flow_label = self.metrics.flow().to_string();
        let node_label = self.node.to_string();
        let profile_label = self.metrics.profile();
        metrics::histogram!(
            "latticeflow.executor.node_latency_ms",
            "flow" => flow_label.clone(),
            "node" => node_label.clone(),
            "profile" => profile_label
        )
        .record(elapsed);
        metrics::gauge!(
            "latticeflow.executor.active_nodes",
            "flow" => flow_label,
            "node" => node_label,
            "profile" => profile_label
        )
        .decrement(1.0);
    }
}

fn profile_label(profile: Profile) -> &'static str {
    match profile {
        Profile::Web => "web",
        Profile::Queue => "queue",
        Profile::Temporal => "temporal",
        Profile::Wasm => "wasm",
        Profile::Dev => "dev",
    }
}

/// High-level executor responsible for instantiating graph runs.
#[derive(Clone)]
pub struct FlowExecutor {
    registry: Arc<NodeRegistry>,
    edge_capacity: usize,
    trigger_capacity: usize,
    capture_capacity: usize,
    resources: Arc<dyn ResourceAccess>,
}

impl FlowExecutor {
    /// Construct a new executor backed by the supplied registry.
    pub fn new(registry: Arc<NodeRegistry>) -> Self {
        let default_resources: Arc<ResourceBag> = Arc::new(ResourceBag::new());
        Self {
            registry,
            edge_capacity: DEFAULT_EDGE_CAPACITY,
            trigger_capacity: DEFAULT_TRIGGER_CAPACITY,
            capture_capacity: DEFAULT_CAPTURE_CAPACITY,
            resources: default_resources,
        }
    }

    /// Override default edge channel capacity.
    pub fn with_edge_capacity(mut self, capacity: usize) -> Self {
        self.edge_capacity = capacity.max(1);
        self
    }

    /// Override capture channel capacity (defaults to 8).
    pub fn with_capture_capacity(mut self, capacity: usize) -> Self {
        self.capture_capacity = capacity.max(1);
        self
    }

    /// Provide a custom resource access handle used for node execution.
    pub fn with_resource_access<T>(mut self, resources: Arc<T>) -> Self
    where
        T: ResourceAccess,
    {
        self.resources = resources;
        self
    }

    /// Provide a convenient builder for resource bags.
    pub fn with_resource_bag(mut self, bag: ResourceBag) -> Self {
        let resources: Arc<ResourceBag> = Arc::new(bag);
        self.resources = resources;
        self
    }

    /// Instantiate a fresh flow instance ready to receive inputs.
    pub fn instantiate(
        &self,
        ir: &ValidatedIR,
        capture_alias: impl Into<String>,
    ) -> Result<FlowInstance, ExecutionError> {
        let capture_alias = capture_alias.into();
        let flow = ir.flow().clone();
        let metrics = Arc::new(ExecutorMetrics::new(flow.name.clone(), flow.profile));

        let mut handlers: HashMap<String, Arc<dyn NodeHandler>> = HashMap::new();
        for node in &flow.nodes {
            let identifier = node.identifier.clone();
            let handler = self
                .registry
                .handler(&identifier)
                .ok_or_else(|| ExecutionError::UnregisteredNode { identifier })?;
            handlers.insert(node.alias.clone(), handler);
        }

        let mut inbound: HashMap<String, Vec<mpsc::Receiver<FlowMessage>>> = HashMap::new();
        let mut outbound: HashMap<String, Vec<InstrumentedSender>> = HashMap::new();

        for node in &flow.nodes {
            inbound.insert(node.alias.clone(), Vec::new());
            outbound.insert(node.alias.clone(), Vec::new());
        }

        let spill_manager = SpillManager::new(&flow).map_err(ExecutionError::SpillSetup)?;

        for edge in &flow.edges {
            let capacity = edge
                .buffer
                .max_items
                .map(|v| usize::try_from(v).unwrap_or(1))
                .unwrap_or(self.edge_capacity)
                .max(1);
            let (tx, rx) = mpsc::channel(capacity);
            let edge_label: Arc<str> =
                Arc::from(format!("{}->{}", edge.from.as_str(), edge.to.as_str()));
            let tracker = metrics.queue_tracker(edge_label, capacity);
            let spill_context = spill_manager.context_for(&edge.buffer);
            outbound
                .get_mut(&edge.from)
                .expect("source node exists")
                .push(InstrumentedSender::new(tx, tracker, spill_context));
            inbound
                .get_mut(&edge.to)
                .expect("target node exists")
                .push(rx);
        }

        let mut trigger_inputs: HashMap<String, mpsc::Sender<FlowMessage>> = HashMap::new();
        for node in &flow.nodes {
            if inbound
                .get(&node.alias)
                .map(|r| r.is_empty())
                .unwrap_or(false)
            {
                let (tx, rx) = mpsc::channel(self.trigger_capacity);
                inbound
                    .get_mut(&node.alias)
                    .expect("node registered")
                    .push(rx);
                trigger_inputs.insert(node.alias.clone(), tx);
            }
        }

        if !flow.nodes.iter().any(|n| n.alias == capture_alias) {
            return Err(ExecutionError::UnknownCapture {
                alias: capture_alias,
            });
        }

        let (capture_tx, capture_rx) = mpsc::channel::<CapturedOutput>(self.capture_capacity);
        let cancellation = CancellationToken::new();
        let permits = Arc::new(Semaphore::new(self.capture_capacity));
        let capture_tracker = metrics.queue_tracker(
            Arc::from(format!("capture::{}", capture_alias.as_str())),
            self.capture_capacity,
        );

        let mut tasks = Vec::with_capacity(flow.nodes.len());
        for node in flow.nodes {
            let handler = handlers.get(&node.alias).expect("handler resolved").clone();
            let inputs = inbound.remove(&node.alias).expect("inputs allocated");
            let outputs = outbound.remove(&node.alias).expect("outputs allocated");
            let capture_sender = capture_tx.clone();
            let capture_target = capture_alias.clone();
            let token = cancellation.child_token();
            let alias = node.alias.clone();
            let resource_handle = self.resources.clone();
            tasks.push(tokio::spawn(run_node(
                alias,
                capture_target,
                handler,
                inputs,
                outputs,
                capture_sender,
                metrics.clone(),
                capture_tracker.clone(),
                NodeContext::new(token, resource_handle),
            )));
        }

        drop(capture_tx);

        Ok(FlowInstance {
            triggers: trigger_inputs,
            capture: capture_rx,
            cancellation,
            permits,
            tasks,
            metrics,
        })
    }

    /// Convenience helper to execute a flow once and await the first response.
    #[instrument(skip_all, fields(trigger = %trigger_alias, capture = %capture_alias))]
    pub async fn run_once(
        &self,
        ir: &ValidatedIR,
        trigger_alias: &str,
        payload: JsonValue,
        capture_alias: &str,
        deadline: Option<Duration>,
    ) -> Result<ExecutionResult, ExecutionError> {
        let mut instance = self.instantiate(ir, capture_alias)?;
        instance.send(trigger_alias, payload).await?;

        let start = Instant::now();
        let result: Result<CaptureResult, ExecutionError> = if let Some(duration) = deadline {
            match time::timeout(duration, instance.next()).await {
                Ok(Some(result)) => result,
                Ok(None) => Err(ExecutionError::MissingOutput {
                    alias: capture_alias.to_string(),
                }),
                Err(_) => {
                    instance.cancel_with_reason("deadline");
                    Err(ExecutionError::DeadlineExceeded { elapsed: duration })
                }
            }
        } else {
            match instance.next().await {
                Some(result) => result,
                None => Err(ExecutionError::MissingOutput {
                    alias: capture_alias.to_string(),
                }),
            }
        };

        let elapsed = start.elapsed();
        if let Err(err) = &result {
            debug!(?elapsed, "flow execution ended with error: {err:?}");
        } else {
            trace!(?elapsed, "flow execution completed successfully");
        }

        match result {
            Ok(CaptureResult::Value(value)) => {
                instance.shutdown().await?;
                Ok(ExecutionResult::Value(value))
            }
            Ok(CaptureResult::Stream(stream)) => {
                Ok(ExecutionResult::Stream(stream.into_handle(instance)))
            }
            Err(err) => {
                instance.shutdown().await?;
                Err(err)
            }
        }
    }
}

/// Active per-run state for a workflow.
pub struct FlowInstance {
    triggers: HashMap<String, mpsc::Sender<FlowMessage>>,
    capture: mpsc::Receiver<CapturedOutput>,
    cancellation: CancellationToken,
    permits: Arc<Semaphore>,
    tasks: Vec<tokio::task::JoinHandle<()>>,
    metrics: Arc<ExecutorMetrics>,
}

impl FlowInstance {
    /// Enqueue a payload onto a trigger node.
    pub async fn send(
        &self,
        trigger_alias: &str,
        payload: JsonValue,
    ) -> Result<(), ExecutionError> {
        let sender =
            self.triggers
                .get(trigger_alias)
                .ok_or_else(|| ExecutionError::UnknownTrigger {
                    alias: trigger_alias.to_string(),
                })?;
        let permit = self
            .permits
            .clone()
            .acquire_owned()
            .await
            .map_err(|_| ExecutionError::Cancelled)?;
        sender
            .send(FlowMessage::Data {
                payload,
                permit: Some(permit),
                queue_tracker: None,
            })
            .await
            .map_err(|_| ExecutionError::Cancelled)?;
        Ok(())
    }

    /// Fetch the next captured result from the configured sink.
    pub(crate) async fn next(&mut self) -> Option<Result<CaptureResult, ExecutionError>> {
        match self.capture.recv().await {
            Some(CapturedOutput {
                alias,
                result,
                permit,
                queue_tracker,
            }) => {
                if let Some(tracker) = queue_tracker {
                    tracker.decrement();
                }
                let outcome = match result {
                    Ok(CapturedPayload::Value(value)) => Ok(CaptureResult::Value(value)),
                    Ok(CapturedPayload::Stream(stream)) => Ok(CaptureResult::Stream(stream)),
                    Err(err) => Err(ExecutionError::NodeFailed { alias, source: err }),
                };
                drop(permit);
                Some(outcome)
            }
            None => None,
        }
    }

    /// Signal cooperative cancellation.
    pub fn cancel(&self) {
        self.cancel_with_reason("external");
    }

    /// Signal cancellation with an explicit reason (used for metrics labels).
    pub fn cancel_with_reason(&self, reason: &str) {
        self.metrics.record_cancellation("__instance", reason);
        self.cancellation.cancel();
    }

    /// Gracefully tear down the instance, awaiting node tasks.
    pub async fn shutdown(self) -> Result<(), ExecutionError> {
        self.cancellation.cancel();
        for handle in self.tasks {
            if let Err(join_err) = handle.await {
                error!("node task panicked: {join_err}");
            }
        }
        Ok(())
    }

    #[cfg(test)]
    pub(crate) fn available_permits(&self) -> usize {
        self.permits.available_permits()
    }
}

/// Result captured from a workflow execution.
pub(crate) enum CaptureResult {
    /// Single JSON value completion.
    Value(JsonValue),
    /// Streaming handle producing incremental events.
    Stream(StreamingCapture),
}

/// Public outcome returned by `FlowExecutor::run_once`.
pub enum ExecutionResult {
    /// Single JSON value response.
    Value(JsonValue),
    /// Streaming response handle.
    Stream(StreamHandle),
}

pub struct StreamHandle {
    alias: String,
    stream: futures::stream::BoxStream<'static, NodeResult<JsonValue>>,
    cancellation: CancellationToken,
    permit: Option<OwnedSemaphorePermit>,
    instance: Option<FlowInstance>,
}

impl StreamHandle {
    fn new(
        alias: String,
        stream: futures::stream::BoxStream<'static, NodeResult<JsonValue>>,
        cancellation: CancellationToken,
        permit: Option<OwnedSemaphorePermit>,
        instance: FlowInstance,
    ) -> Self {
        Self {
            alias,
            stream,
            cancellation,
            permit,
            instance: Some(instance),
        }
    }

    /// Clone the cancellation token associated with the stream.
    pub fn cancellation_token(&self) -> CancellationToken {
        self.cancellation.clone()
    }

    fn finalize(&mut self, cancel: bool) {
        if let Some(permit) = self.permit.take() {
            drop(permit);
        }
        if let Some(instance) = self.instance.take() {
            if cancel {
                instance.cancel_with_reason("stream_cancelled");
            }
            tokio::spawn(async move {
                if let Err(err) = instance.shutdown().await {
                    error!("stream shutdown failed: {err}");
                }
            });
        }
    }
}

impl Stream for StreamHandle {
    type Item = Result<JsonValue, ExecutionError>;

    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>> {
        let this = Pin::into_inner(self);
        match Pin::new(&mut this.stream).poll_next(cx) {
            Poll::Ready(Some(Ok(value))) => Poll::Ready(Some(Ok(value))),
            Poll::Ready(Some(Err(err))) => {
                let alias = this.alias.clone();
                this.finalize(true);
                Poll::Ready(Some(Err(ExecutionError::NodeFailed { alias, source: err })))
            }
            Poll::Ready(None) => {
                this.finalize(false);
                Poll::Ready(None)
            }
            Poll::Pending => Poll::Pending,
        }
    }
}

impl Drop for StreamHandle {
    fn drop(&mut self) {
        self.cancellation.cancel();
        self.finalize(true);
    }
}

impl StreamingCapture {
    fn into_handle(self, instance: FlowInstance) -> StreamHandle {
        StreamHandle::new(
            self.alias,
            self.stream,
            self.cancellation,
            self.permit,
            instance,
        )
    }
}

struct CapturedOutput {
    alias: String,
    result: NodeResult<CapturedPayload>,
    permit: Option<OwnedSemaphorePermit>,
    queue_tracker: Option<Arc<QueueDepthTracker>>,
}

#[derive(Debug)]
enum FlowMessage {
    Data {
        payload: JsonValue,
        permit: Option<OwnedSemaphorePermit>,
        queue_tracker: Option<Arc<QueueDepthTracker>>,
    },
    Spilled {
        key: String,
        storage: Arc<BlobSpill>,
        permit: Option<OwnedSemaphorePermit>,
        queue_tracker: Option<Arc<QueueDepthTracker>>,
    },
}

#[instrument(
    level = "trace",
    skip_all,
    fields(node = %alias)
)]
async fn run_node(
    alias: String,
    capture_alias: String,
    handler: Arc<dyn NodeHandler>,
    mut inputs: Vec<mpsc::Receiver<FlowMessage>>,
    outputs: Vec<InstrumentedSender>,
    capture: mpsc::Sender<CapturedOutput>,
    metrics: Arc<ExecutorMetrics>,
    capture_tracker: Arc<QueueDepthTracker>,
    ctx: NodeContext,
) {
    if inputs.is_empty() {
        warn!("node `{alias}` has no inputs; dropping without execution");
        return;
    }

    let mut streams = StreamMap::new();
    for (idx, receiver) in inputs.drain(..).enumerate() {
        streams.insert(idx, ReceiverStream::new(receiver));
    }

    while !ctx.is_cancelled() {
        let cancel_token = ctx.token();
        let maybe_message = tokio::select! {
            biased;
            _ = cancel_token.cancelled() => {
                debug!("node `{alias}` observed cancellation");
                metrics.record_cancellation(alias.as_str(), "propagated");
                break;
            }
            message = streams.next() => message,
        };

        let (_, message) = match maybe_message {
            Some(pair) => pair,
            None => break,
        };

        let Some((mut permit, queue_tracker, payload)) = (match message {
            FlowMessage::Data {
                payload,
                permit,
                queue_tracker,
            } => Some((permit, queue_tracker, payload)),
            FlowMessage::Spilled {
                key,
                storage,
                permit,
                queue_tracker,
            } => match storage.load(&key).await {
                Ok(value) => {
                    if let Err(err) = storage.remove(&key).await {
                        warn!("failed to delete spilled payload `{key}` for node `{alias}`: {err}");
                    }
                    Some((permit, queue_tracker, value))
                }
                Err(err) => {
                    error!("node `{alias}` failed to hydrate spilled payload `{key}`: {err}");
                    metrics.record_node_error(alias.as_str(), "spill_load_failed");
                    let failure = NodeError::new(format!(
                        "failed to load spilled payload `{key}` from blob storage: {err}"
                    ));
                    let captured_permit = permit;
                    let send_result = capture
                        .send(CapturedOutput {
                            alias: alias.clone(),
                            result: Err(failure),
                            permit: captured_permit,
                            queue_tracker: queue_tracker.clone(),
                        })
                        .await;
                    if let Some(tracker) = &queue_tracker {
                        tracker.decrement();
                    }
                    if send_result.is_ok() {
                        capture_tracker.increment();
                        metrics
                            .observe_capture_backpressure(capture_alias.as_str(), Duration::ZERO);
                    }
                    metrics.record_cancellation(alias.as_str(), "spill_load_failed");
                    ctx.token().cancel();
                    None
                }
            },
        }) else {
            continue;
        };

        if let Some(tracker) = &queue_tracker {
            tracker.decrement();
        }
        let _guard = metrics.track_node(alias.as_str());

        let ctx_for_handler = ctx.clone();
        let resources = ctx_for_handler.resource_handle();
        let handler_clone = handler.clone();
        let result = context::with_resources(resources.clone(), async move {
            handler_clone.invoke(payload, &ctx_for_handler).await
        })
        .await;

        match result {
            Ok(NodeOutput::Value(value)) => {
                if alias == capture_alias {
                    let captured_permit = permit.take();
                    let start = Instant::now();
                    let send_result = capture
                        .send(CapturedOutput {
                            alias: alias.clone(),
                            result: Ok(CapturedPayload::Value(value.clone())),
                            permit: captured_permit,
                            queue_tracker: Some(capture_tracker.clone()),
                        })
                        .await;
                    if send_result.is_ok() {
                        capture_tracker.increment();
                        metrics
                            .observe_capture_backpressure(capture_alias.as_str(), start.elapsed());
                    }
                }
                if outputs.is_empty() {
                    // Leaf node; any outstanding permit will be released below.
                } else {
                    for (idx, sender) in outputs.iter().enumerate() {
                        let downstream_permit = if idx == 0 { permit.take() } else { None };
                        if let Err(err) = sender.send(value.clone(), downstream_permit).await {
                            debug!("downstream receiver for node `{alias}` dropped: {err}");
                        }
                    }
                }
            }
            Ok(NodeOutput::Stream(stream)) => {
                if alias != capture_alias {
                    error!(
                        "node `{alias}` emitted a stream but capture alias is `{capture_alias}`; \
                        streaming outputs are only supported on the configured capture node"
                    );
                    metrics.record_node_error(alias.as_str(), "invalid_stream_target");
                    let captured_permit = permit.take();
                    let send_result = capture
                        .send(CapturedOutput {
                            alias: alias.clone(),
                            result: Err(NodeError::new(
                                "stream outputs are only supported on the capture node",
                            )),
                            permit: captured_permit,
                            queue_tracker: Some(capture_tracker.clone()),
                        })
                        .await;
                    if send_result.is_ok() {
                        capture_tracker.increment();
                        metrics
                            .observe_capture_backpressure(capture_alias.as_str(), Duration::ZERO);
                    }
                    metrics.record_cancellation(alias.as_str(), "invalid_stream_target");
                    ctx.token().cancel();
                    break;
                }

                let cancellation = ctx.token();
                let scoped_stream = stream.then({
                    let resources = resources.clone();
                    move |item| {
                        let resources = resources.clone();
                        async move { context::with_resources(resources, async move { item }).await }
                    }
                });
                let boxed_stream = scoped_stream
                    .take_until(cancellation.clone().cancelled_owned())
                    .boxed();
                let streaming =
                    StreamingCapture::new(alias.clone(), boxed_stream, cancellation, permit.take());
                let start = Instant::now();
                let send_result = capture
                    .send(CapturedOutput {
                        alias: alias.clone(),
                        result: Ok(CapturedPayload::Stream(streaming)),
                        permit: None,
                        queue_tracker: Some(capture_tracker.clone()),
                    })
                    .await;
                if send_result.is_ok() {
                    capture_tracker.increment();
                    metrics.observe_capture_backpressure(capture_alias.as_str(), start.elapsed());
                }
                metrics.record_stream_spawn(alias.as_str());
                break;
            }
            Ok(NodeOutput::None) => {
                trace!("node `{alias}` produced no output");
            }
            Err(err) => {
                error!("node `{alias}` failed: {err}");
                metrics.record_node_error(alias.as_str(), "handler_error");
                let captured_permit = permit.take();
                let send_result = capture
                    .send(CapturedOutput {
                        alias: alias.clone(),
                        result: Err(err),
                        permit: captured_permit,
                        queue_tracker: Some(capture_tracker.clone()),
                    })
                    .await;
                if send_result.is_ok() {
                    capture_tracker.increment();
                    metrics.observe_capture_backpressure(capture_alias.as_str(), Duration::ZERO);
                }
                metrics.record_cancellation(alias.as_str(), "node_error");
                ctx.token().cancel();
                break;
            }
        }
        if let Some(permit) = permit {
            drop(permit);
        }
    }

    debug!("node `{alias}` exiting");
}

/// Errors surfaced during flow execution.
#[derive(Debug, Error)]
pub enum ExecutionError {
    /// Unknown trigger alias or missing input queue.
    #[error("trigger alias `{alias}` is not part of the workflow")]
    UnknownTrigger { alias: String },
    /// Capture alias not found.
    #[error("capture alias `{alias}` is not part of the workflow")]
    UnknownCapture { alias: String },
    /// Execution cancelled before completion.
    #[error("execution cancelled")]
    Cancelled,
    /// Deadline exceeded.
    #[error("deadline exceeded after {elapsed:?}")]
    DeadlineExceeded { elapsed: Duration },
    /// Flow finished without producing a captured output.
    #[error("capture `{alias}` did not yield an output")]
    MissingOutput { alias: String },
    /// Node handler failed.
    #[error("node `{alias}` failed: {source}")]
    NodeFailed {
        alias: String,
        #[source]
        source: NodeError,
    },
    /// Handler not registered for node identifier.
    #[error("no handler registered for node `{identifier}`")]
    UnregisteredNode { identifier: String },
    /// Spill storage could not be initialised.
    #[error("failed to configure spill storage: {0}")]
    SpillSetup(anyhow::Error),
}

#[cfg(test)]
mod tests {
    use super::*;
    use capabilities::{ResourceBag, context, kv::MemoryKv};
    use dag_core::prelude::*;
    use futures::StreamExt;
    use kernel_plan::validate;
    use metrics_util::debugging::{DebugValue, DebuggingRecorder, Snapshotter};
    use proptest::prelude::*;
    use serde_json::json;
    use std::sync::{Arc, OnceLock};
    use std::time::Duration;
    use tokio::runtime::Builder as RuntimeBuilder;
    use tokio::time::{sleep, timeout};

    fn metrics_snapshotter() -> &'static Snapshotter {
        static SNAPSHOTTER: OnceLock<Snapshotter> = OnceLock::new();
        SNAPSHOTTER.get_or_init(|| {
            let recorder = DebuggingRecorder::new();
            let snapshotter = recorder.snapshotter();
            metrics::set_global_recorder(recorder)
                .unwrap_or_else(|_| panic!("metrics recorder already installed"));
            snapshotter
        })
    }

    fn reset_metrics() {
        let _ = metrics_snapshotter().snapshot();
    }

    #[tokio::test]
    async fn emits_executor_metrics_on_success() {
        reset_metrics();

        const FORWARD_SPEC: NodeSpec = NodeSpec::inline(
            "tests::forward",
            "Forward",
            SchemaSpec::Opaque,
            SchemaSpec::Opaque,
            Effects::Pure,
            Determinism::Strict,
            None,
        );

        let mut builder = FlowBuilder::new("metrics_flow", Version::new(0, 1, 0), Profile::Dev);
        let trigger = builder.add_node("trigger", &FORWARD_SPEC).unwrap();
        let capture = builder.add_node("capture", &FORWARD_SPEC).unwrap();
        builder.connect(&trigger, &capture);
        let flow_ir = builder.build();
        let validated = validate(&flow_ir).expect("flow should validate");

        let mut registry = NodeRegistry::new();
        registry
            .register_fn(
                "tests::forward",
                |input: JsonValue| async move { Ok(input) },
            )
            .unwrap();
        let executor = FlowExecutor::new(Arc::new(registry));

        let payload = json!({"ping": "pong"});
        let result = executor
            .run_once(&validated, "trigger", payload.clone(), "capture", None)
            .await
            .expect("run succeeds");
        match result {
            ExecutionResult::Value(value) => assert_eq!(value, payload),
            ExecutionResult::Stream(_) => panic!("expected non-streaming result"),
        }

        let snapshot = metrics_snapshotter().snapshot().into_vec();
        let mut saw_latency = false;
        let mut saw_queue = false;
        for (key, _unit, _desc, value) in snapshot.into_iter() {
            let name = key.key().name();
            match (name, value) {
                ("latticeflow.executor.node_latency_ms", DebugValue::Histogram(vals)) => {
                    assert!(!vals.is_empty(), "latency histogram recorded no samples");
                    saw_latency = true;
                }
                ("latticeflow.executor.queue_depth", DebugValue::Gauge(_)) => {
                    saw_queue = true;
                }
                _ => {}
            }
        }

        assert!(saw_latency, "expected node latency histogram to be emitted");
        assert!(saw_queue, "expected queue depth gauge to be emitted");
    }

    #[test]
    fn flow_instance_respects_capture_capacity() {
        let mut runner = proptest::test_runner::TestRunner::new(ProptestConfig {
            cases: 32,
            ..ProptestConfig::default()
        });
        let strategy = (1usize..=4, 1usize..=5);

        runner
            .run(&strategy, |(capacity, cycles)| {
                let runtime = RuntimeBuilder::new_multi_thread()
                    .worker_threads(2)
                    .enable_all()
                    .build()
                    .expect("tokio runtime");

                runtime.block_on(async move {
                    let mut registry = NodeRegistry::new();
                    registry
                        .register_fn(
                            "tests::forward",
                            |input: JsonValue| async move { Ok(input) },
                        )
                        .unwrap();

                    let mut builder =
                        FlowBuilder::new("prop_backpressure", Version::new(0, 0, 1), Profile::Dev);
                    let trigger = builder
                        .add_node(
                            "trigger",
                            &NodeSpec::inline(
                                "tests::forward",
                                "Trigger",
                                SchemaSpec::Opaque,
                                SchemaSpec::Opaque,
                                Effects::Pure,
                                Determinism::Strict,
                                None,
                            ),
                        )
                        .unwrap();
                    let capture = builder
                        .add_node(
                            "capture",
                            &NodeSpec::inline(
                                "tests::forward",
                                "Capture",
                                SchemaSpec::Opaque,
                                SchemaSpec::Opaque,
                                Effects::Pure,
                                Determinism::Strict,
                                None,
                            ),
                        )
                        .unwrap();
                    builder.connect(&trigger, &capture);

                    let flow = builder.build();
                    let validated = validate(&flow).expect("flow should validate");

                    let executor =
                        FlowExecutor::new(Arc::new(registry)).with_capture_capacity(capacity);
                    let mut instance = executor
                        .instantiate(&validated, "capture")
                        .expect("instance creation succeeds");

                    for idx in 0..capacity {
                        tokio::time::timeout(
                            std::time::Duration::from_millis(50),
                            instance.send("trigger", json!({ "prefill": idx })),
                        )
                        .await
                        .expect("prefill send completes")
                        .expect("prefill send succeeds");
                    }

                    prop_assert_eq!(
                        instance.available_permits(),
                        0,
                        "all permits should be consumed after prefill"
                    );

                    for cycle in 0..cycles {
                        let payload = json!({ "cycle": cycle });
                        let blocked = tokio::time::timeout(
                            std::time::Duration::from_millis(20),
                            instance.send("trigger", payload.clone()),
                        )
                        .await;
                        prop_assert!(
                            blocked.is_err(),
                            "send should block when permits are exhausted"
                        );

                        let drained = tokio::time::timeout(
                            std::time::Duration::from_millis(100),
                            instance.next(),
                        )
                        .await
                        .expect("drain future completes");
                        let result = drained.expect("capture channel open")?;
                        match result {
                            CaptureResult::Value(_) => {}
                            CaptureResult::Stream(_) => {
                                prop_assert!(false, "unexpected stream capture during drain cycle");
                            }
                        }

                        tokio::time::timeout(
                            std::time::Duration::from_millis(100),
                            instance.send("trigger", payload),
                        )
                        .await
                        .expect("send should succeed after draining one result")
                        .expect("send after drain succeeds");

                        prop_assert_eq!(
                            instance.available_permits(),
                            0,
                            "permits should be fully leased after refill"
                        );
                    }

                    for _ in 0..capacity {
                        let drained = tokio::time::timeout(
                            std::time::Duration::from_millis(100),
                            instance.next(),
                        )
                        .await
                        .expect("tail drain future completes");
                        let result = drained.expect("capture channel open while draining tail")?;
                        match result {
                            CaptureResult::Value(_) => {}
                            CaptureResult::Stream(_) => {
                                prop_assert!(
                                    false,
                                    "unexpected stream capture while draining tail"
                                );
                            }
                        }
                    }

                    prop_assert_eq!(
                        instance.available_permits(),
                        capacity,
                        "all permits should be restored after final drain"
                    );

                    instance.cancel();
                    instance
                        .shutdown()
                        .await
                        .expect("instance shutdown succeeds");

                    Ok(())
                })
            })
            .unwrap();
    }

    #[tokio::test]
    async fn backpressure_applies_when_capture_is_full() {
        let mut registry = NodeRegistry::new();
        registry
            .register_fn("tests::trigger", |val: JsonValue| async move { Ok(val) })
            .unwrap();
        registry
            .register_fn("tests::sink", |val: JsonValue| async move { Ok(val) })
            .unwrap();

        let mut builder = FlowBuilder::new("capture_flow", Version::new(1, 0, 0), Profile::Dev);
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
        let sink = builder
            .add_node(
                "sink",
                &NodeSpec::inline(
                    "tests::sink",
                    "Sink",
                    SchemaSpec::Opaque,
                    SchemaSpec::Opaque,
                    Effects::Pure,
                    Determinism::Strict,
                    None,
                ),
            )
            .unwrap();
        builder.connect(&trigger, &sink);

        let mut flow = builder.build();
        for edge in &mut flow.edges {
            edge.buffer.max_items = Some(1);
        }
        let validated = validate(&flow).expect("flow should validate");

        let executor = FlowExecutor::new(Arc::new(registry)).with_capture_capacity(1);
        let mut instance = executor.instantiate(&validated, "sink").unwrap();

        instance
            .send("trigger", JsonValue::from(1))
            .await
            .expect("first send");

        match timeout(
            Duration::from_millis(50),
            instance.send("trigger", JsonValue::from(2)),
        )
        .await
        {
            Err(_) => {} // expected timeout indicates capture queue saturation
            Ok(result) => panic!("send should block until capture drains, got {result:?}"),
        }

        let first = match instance.next().await {
            Some(Ok(CaptureResult::Value(value))) => value,
            _ => panic!("unexpected capture result"),
        };
        assert_eq!(first, JsonValue::from(1));

        instance
            .send("trigger", JsonValue::from(2))
            .await
            .expect("second send completes once capture drained");
        let second = match instance.next().await {
            Some(Ok(CaptureResult::Value(value))) => value,
            _ => panic!("unexpected capture result"),
        };
        assert_eq!(second, JsonValue::from(2));

        instance.shutdown().await.unwrap();
    }

    #[tokio::test]
    async fn resource_bag_available_during_execution() {
        let kv_store = Arc::new(MemoryKv::new());

        let mut registry = NodeRegistry::new();
        registry
            .register_fn("tests::trigger", |val: JsonValue| async move { Ok(val) })
            .unwrap();
        registry
            .register_fn("tests::worker", |val: JsonValue| async move {
                context::with_current_async(|resources| async move {
                    let kv = resources.kv().expect("kv capability available");
                    kv.put("resource-test", b"value", None)
                        .await
                        .expect("kv put succeeds");
                })
                .await
                .expect("resource context scoped during node execution");
                Ok(val)
            })
            .unwrap();
        registry
            .register_fn("tests::capture", |val: JsonValue| async move { Ok(val) })
            .unwrap();

        let mut builder = FlowBuilder::new("resource_flow", Version::new(1, 0, 0), Profile::Dev);
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
                    Effects::Effectful,
                    Determinism::BestEffort,
                    None,
                ),
            )
            .unwrap();
        let capture = builder
            .add_node(
                "capture",
                &NodeSpec::inline(
                    "tests::capture",
                    "Capture",
                    SchemaSpec::Opaque,
                    SchemaSpec::Opaque,
                    Effects::Pure,
                    Determinism::Strict,
                    None,
                ),
            )
            .unwrap();
        builder.connect(&trigger, &worker);
        builder.connect(&worker, &capture);

        let flow = builder.build();
        let validated = validate(&flow).expect("flow validates");

        let executor = FlowExecutor::new(Arc::new(registry))
            .with_resource_bag(ResourceBag::new().with_kv(kv_store.clone()));

        let result = executor
            .run_once(&validated, "trigger", JsonValue::Null, "capture", None)
            .await
            .expect("execution succeeds");

        assert!(matches!(result, ExecutionResult::Value(JsonValue::Null)));
        assert_eq!(
            kv_store
                .get("resource-test")
                .expect("kv read after execution"),
            Some(b"value".to_vec())
        );
    }

    #[tokio::test]
    async fn spill_to_blob_handles_queue_overflow() {
        reset_metrics();

        const TRIGGER_SPEC: NodeSpec = NodeSpec::inline(
            "tests::trigger_spill",
            "Trigger",
            SchemaSpec::Opaque,
            SchemaSpec::Opaque,
            Effects::Pure,
            Determinism::Strict,
            None,
        );
        const WORKER_SPEC: NodeSpec = NodeSpec::inline(
            "tests::slow_worker",
            "SlowWorker",
            SchemaSpec::Opaque,
            SchemaSpec::Opaque,
            Effects::Pure,
            Determinism::Strict,
            None,
        );
        const CAPTURE_SPEC: NodeSpec = NodeSpec::inline(
            "tests::capture_spill",
            "Capture",
            SchemaSpec::Opaque,
            SchemaSpec::Opaque,
            Effects::Pure,
            Determinism::Strict,
            None,
        );

        let mut builder = FlowBuilder::new("spill_flow", Version::new(1, 1, 0), Profile::Dev);
        let trigger = builder.add_node("trigger", &TRIGGER_SPEC).unwrap();
        let worker = builder.add_node("worker", &WORKER_SPEC).unwrap();
        let capture = builder.add_node("capture", &CAPTURE_SPEC).unwrap();
        builder.connect(&trigger, &worker);
        builder.connect(&worker, &capture);

        let mut flow_ir = builder.build();
        for edge in &mut flow_ir.edges {
            if edge.from == "trigger" && edge.to == "worker" {
                edge.buffer.max_items = Some(1);
                edge.buffer.spill_tier = Some("local".to_string());
            }
        }
        let validated = validate(&flow_ir).expect("flow should validate");

        let mut registry = NodeRegistry::new();
        registry
            .register_fn("tests::trigger_spill", |input: JsonValue| async move {
                Ok(input)
            })
            .unwrap();
        registry
            .register_fn("tests::slow_worker", |input: JsonValue| async move {
                sleep(Duration::from_millis(100)).await;
                Ok(input)
            })
            .unwrap();
        registry
            .register_fn("tests::capture_spill", |input: JsonValue| async move {
                Ok(input)
            })
            .unwrap();

        let executor = FlowExecutor::new(Arc::new(registry));
        let mut instance = executor.instantiate(&validated, "capture").unwrap();

        let first = json!({"seq": 1});
        instance
            .send("trigger", first.clone())
            .await
            .expect("first payload enqueued");

        let second = json!({"seq": 2});
        timeout(
            Duration::from_millis(50),
            instance.send("trigger", second.clone()),
        )
        .await
        .expect("spill prevents sender from blocking")
        .expect("second payload enqueued via spill");

        let first_out = match instance.next().await {
            Some(Ok(CaptureResult::Value(value))) => value,
            Some(Ok(CaptureResult::Stream(_))) => {
                panic!("unexpected streaming capture result for first payload")
            }
            Some(Err(err)) => panic!("capture failed: {err}"),
            None => panic!("capture channel closed unexpectedly"),
        };
        assert_eq!(first_out, first);

        let second_out = match instance.next().await {
            Some(Ok(CaptureResult::Value(value))) => value,
            Some(Ok(CaptureResult::Stream(_))) => {
                panic!("unexpected streaming capture result for second payload")
            }
            Some(Err(err)) => panic!("capture failed: {err}"),
            None => panic!("capture channel closed unexpectedly"),
        };
        assert_eq!(second_out, second);

        instance.shutdown().await.unwrap();
    }

    #[tokio::test]
    async fn cancellation_propagates_on_error() {
        let mut registry = NodeRegistry::new();
        registry
            .register_fn("tests::trigger", |val: JsonValue| async move { Ok(val) })
            .unwrap();
        registry
            .register_fn("tests::fails", |_val: JsonValue| async move {
                Err::<JsonValue, NodeError>(NodeError::new("boom"))
            })
            .unwrap();
        registry
            .register_fn("tests::sink", |val: JsonValue| async move { Ok(val) })
            .unwrap();

        let mut builder = FlowBuilder::new("cancel", Version::new(1, 0, 0), Profile::Dev);
        let trigger = builder
            .add_node(
                "trigger",
                &NodeSpec::inline(
                    "tests::trigger",
                    "Trig",
                    SchemaSpec::Opaque,
                    SchemaSpec::Opaque,
                    Effects::Pure,
                    Determinism::Strict,
                    None,
                ),
            )
            .unwrap();
        let failing = builder
            .add_node(
                "fails",
                &NodeSpec::inline(
                    "tests::fails",
                    "Fail",
                    SchemaSpec::Opaque,
                    SchemaSpec::Opaque,
                    Effects::Pure,
                    Determinism::Strict,
                    None,
                ),
            )
            .unwrap();
        let sink = builder
            .add_node(
                "sink",
                &NodeSpec::inline(
                    "tests::sink",
                    "Sink",
                    SchemaSpec::Opaque,
                    SchemaSpec::Opaque,
                    Effects::Pure,
                    Determinism::Strict,
                    None,
                ),
            )
            .unwrap();
        builder.connect(&trigger, &failing);
        builder.connect(&failing, &sink);

        let flow = builder.build();
        let validated = validate(&flow).expect("flow should validate");

        let executor = FlowExecutor::new(Arc::new(registry));
        let mut instance = executor.instantiate(&validated, "sink").unwrap();
        instance
            .send("trigger", JsonValue::from(1))
            .await
            .expect("send");

        match instance.next().await {
            Some(Err(ExecutionError::NodeFailed { alias, .. })) => {
                assert_eq!(alias, "fails");
            }
            _ => panic!("unexpected result"),
        }

        instance.shutdown().await.unwrap();
    }

    #[tokio::test]
    async fn streaming_capture_emits_events() {
        let mut registry = NodeRegistry::new();
        registry
            .register_fn("tests::trigger", |val: JsonValue| async move { Ok(val) })
            .unwrap();
        registry
            .register_stream_fn("tests::stream", |_val: JsonValue| async move {
                let events = vec![
                    Ok(JsonValue::from(1)),
                    Ok(JsonValue::from(2)),
                    Ok(JsonValue::from(3)),
                ];
                Ok(futures::stream::iter(events))
            })
            .unwrap();

        let mut builder = FlowBuilder::new("streaming", Version::new(1, 0, 0), Profile::Dev);
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
        let capture = builder
            .add_node(
                "stream",
                &NodeSpec::inline(
                    "tests::stream",
                    "StreamCapture",
                    SchemaSpec::Opaque,
                    SchemaSpec::Opaque,
                    Effects::ReadOnly,
                    Determinism::Stable,
                    Some("Emits incremental updates"),
                ),
            )
            .unwrap();
        builder.connect(&trigger, &capture);

        let flow = builder.build();
        let validated = validate(&flow).expect("flow should validate");

        let executor = FlowExecutor::new(Arc::new(registry));
        let result = executor
            .run_once(
                &validated,
                "trigger",
                JsonValue::from(serde_json::json!({"seed": 1})),
                "stream",
                None,
            )
            .await
            .expect("run once succeeds");

        let mut stream = match result {
            ExecutionResult::Stream(handle) => handle,
            ExecutionResult::Value(_) => panic!("expected streaming response"),
        };

        let mut collected = Vec::new();
        while let Some(event) = stream.next().await {
            let value = event.expect("stream event ok");
            collected.push(value);
        }

        assert_eq!(collected.len(), 3);
        assert_eq!(collected[0], JsonValue::from(1));
        assert_eq!(collected[2], JsonValue::from(3));
    }

    #[tokio::test]
    async fn dropping_stream_cancels_run() {
        let mut registry = NodeRegistry::new();
        registry
            .register_fn("tests::trigger", |val: JsonValue| async move { Ok(val) })
            .unwrap();
        registry
            .register_stream_fn("tests::stream", |_val: JsonValue| async move {
                Ok(futures::stream::unfold(0, |state| async move {
                    Some((Ok(JsonValue::from(state)), state + 1))
                }))
            })
            .unwrap();

        let mut builder = FlowBuilder::new("stream_cancel", Version::new(1, 0, 0), Profile::Dev);
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
        let capture = builder
            .add_node(
                "stream",
                &NodeSpec::inline(
                    "tests::stream",
                    "StreamCapture",
                    SchemaSpec::Opaque,
                    SchemaSpec::Opaque,
                    Effects::ReadOnly,
                    Determinism::Stable,
                    Some("Infinite stream for cancellation test"),
                ),
            )
            .unwrap();
        builder.connect(&trigger, &capture);

        let flow = builder.build();
        let validated = validate(&flow).expect("flow should validate");

        let executor = FlowExecutor::new(Arc::new(registry));
        let result = executor
            .run_once(&validated, "trigger", JsonValue::Null, "stream", None)
            .await
            .expect("run once succeeds");

        let handle = match result {
            ExecutionResult::Stream(stream) => stream,
            ExecutionResult::Value(_) => panic!("expected streaming response"),
        };
        let cancel_token = handle.cancellation_token();
        drop(handle);

        tokio::time::timeout(Duration::from_millis(50), cancel_token.cancelled())
            .await
            .expect("cancellation should fire promptly");
    }
}
````

## File: impl-docs/impl-plan.md
````markdown
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

## File: impl-docs/surface-and-buildout.md
````markdown
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
````




# Instruction
# Agent Briefing: Kernel Portability & Spill Refactor

You are advising on how to bring LatticeFlow's runtime and capability system to a "build once, run anywhere" posture spanning containers, WASI/wasmtime, and Cloudflare Workers. Study the bundled sources and propose concrete paths to:
- Refactor spill handling into a capability-backed abstraction (durable stores, tier policies, error semantics) so kernels avoid direct filesystem calls and survive constrained hosts.
- Stabilize the capability trait surface (HTTP/KV/Blob/Queue/Spill/etc.) while letting hosts supply custom implementations or facets (e.g., mTLS, zero-copy) without breaking portability guarantees.
- Map Flow IR buffer metadata (spill tiers, thresholds, determinism) into host-specific resource choices and validation rules that fail fast when a host cannot honor a requirement.
- Outline the build/feature matrix, WASI vs container vs Workers boundaries, and the incremental migration plan mentioned in the chat.

Deliver a step-by-step architecture plan, risks with mitigations, open questions for ops/platform owners, and quick-win prototypes/tests that de-risk the changes.

Read every file below carefully; do not assume context beyond what is provided here or in the chat summary.

---
