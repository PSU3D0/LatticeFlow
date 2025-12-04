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

## File: examples/s1_echo/src/auth.rs
````rust
use std::cell::RefCell;
use std::sync::Arc;

use host_inproc::{EnvironmentPlugin, InvocationMetadata};
use serde::{Deserialize, Serialize};
use serde_json;

thread_local! {
    static AUTH_CONTEXT: RefCell<Option<AuthUser>> = RefCell::new(None);
}

#[derive(Clone, Debug, Serialize, Deserialize, PartialEq, Eq)]
pub struct AuthUser {
    pub sub: String,
    pub email: Option<String>,
}

pub fn current_user() -> Option<AuthUser> {
    AUTH_CONTEXT.with(|ctx| ctx.borrow().clone())
}

fn set_current_user(user: Option<AuthUser>) {
    AUTH_CONTEXT.with(|ctx| *ctx.borrow_mut() = user);
}

#[derive(Default)]
pub struct AuthEnvironmentPlugin;

impl EnvironmentPlugin for AuthEnvironmentPlugin {
    fn before_execute(&self, metadata: &InvocationMetadata) {
        let user = metadata
            .extensions()
            .get("auth.user")
            .and_then(|value| serde_json::from_value::<AuthUser>(value.clone()).ok());
        set_current_user(user);
    }

    fn after_execute(
        &self,
        _metadata: &InvocationMetadata,
        _outcome: Result<&host_inproc::HostExecutionResult, &host_inproc::HostExecutionError>,
    ) {
        set_current_user(None);
    }
}

pub fn environment_plugins() -> Vec<Arc<dyn EnvironmentPlugin>> {
    vec![Arc::new(AuthEnvironmentPlugin::default())]
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

## File: examples/s1_echo/Cargo.toml
````toml
[package]
name = "example-s1-echo"
version = "0.1.0"
authors.workspace = true
edition.workspace = true
rust-version.workspace = true
license.workspace = true
repository.workspace = true
homepage.workspace = true
description = "S1 echo workflow example built with LatticeFlow macros."

[dependencies]
dag-core = { path = "../../crates/dag-core" }
dag-macros = { path = "../../crates/dag-macros" }
kernel-exec = { path = "../../crates/kernel-exec" }
kernel-plan = { path = "../../crates/kernel-plan" }
host-inproc = { path = "../../crates/host-inproc" }
serde.workspace = true
serde_json.workspace = true
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

## File: crates/dag-macros/src/lib.rs
````rust
// TODO
use capabilities::hints;
use proc_macro::TokenStream;
use proc_macro2::{Span, TokenStream as TokenStream2};
use quote::{ToTokens, format_ident, quote};
use semver::Version;
use std::collections::HashSet;
use syn::parse::{Parse, ParseStream};
use syn::punctuated::Punctuated;
use syn::{
    Attribute, Expr, ExprLit, Ident, ItemEnum, ItemFn, Lit, LitStr, Macro, Meta, MetaNameValue,
    Path, Result, Token, parse_macro_input, parse_quote, spanned::Spanned,
};

/// Attribute macro for defining workflow nodes.
#[proc_macro_attribute]
pub fn node(attr: TokenStream, item: TokenStream) -> TokenStream {
    expand_node(attr, item, NodeDefaults::node())
}

/// Attribute macro for defining workflow triggers.
#[proc_macro_attribute]
pub fn trigger(attr: TokenStream, item: TokenStream) -> TokenStream {
    expand_node(attr, item, NodeDefaults::trigger())
}

/// Declarative workflow macro producing Flow IR at compile time.
#[proc_macro]
pub fn workflow(input: TokenStream) -> TokenStream {
    let parsed = parse_macro_input!(input as WorkflowInput);
    match parsed.expand() {
        Ok(tokens) => tokens.into(),
        Err(err) => err.to_compile_error().into(),
    }
}

/// Attribute helper for Flow-value enums used to constrain generics.
#[proc_macro_attribute]
pub fn flow_enum(attr: TokenStream, item: TokenStream) -> TokenStream {
    if !attr.is_empty() {
        let err = syn::Error::new(
            Span::call_site(),
            "#[flow_enum] does not accept attribute arguments",
        );
        return TokenStream::from(err.to_compile_error());
    }

    let item_tokens: TokenStream2 = item.clone().into();
    let mut enum_item = match syn::parse2::<ItemEnum>(item_tokens.clone()) {
        Ok(item) => item,
        Err(_) => {
            let err = syn::Error::new_spanned(
                item_tokens,
                "#[flow_enum] can only be applied to enum definitions",
            );
            return TokenStream::from(err.to_compile_error());
        }
    };

    match ensure_flow_enum_attrs(&mut enum_item) {
        Ok(()) => TokenStream::from(quote!(#enum_item)),
        Err(err) => TokenStream::from(err.to_compile_error()),
    }
}

struct NodeDefaults {
    kind: &'static str,
    effects: EffectLevel,
    determinism: DeterminismLevel,
}

impl NodeDefaults {
    fn node() -> Self {
        Self {
            kind: "Inline",
            effects: EffectLevel::Pure,
            determinism: DeterminismLevel::BestEffort,
        }
    }

    fn trigger() -> Self {
        Self {
            kind: "Trigger",
            effects: EffectLevel::ReadOnly,
            determinism: DeterminismLevel::Strict,
        }
    }
}

#[derive(Copy, Clone, Debug)]
enum EffectLevel {
    Pure,
    ReadOnly,
    Effectful,
}

impl EffectLevel {
    fn parse(value: &str, span: Span) -> Result<Self> {
        match value {
            "Pure" | "pure" => Ok(EffectLevel::Pure),
            "ReadOnly" | "readonly" => Ok(EffectLevel::ReadOnly),
            "Effectful" | "effectful" => Ok(EffectLevel::Effectful),
            other => Err(syn::Error::new(
                span,
                format!("[EFFECT201] unknown effects value `{other}`"),
            )),
        }
    }

    fn to_tokens(self, span: Span) -> TokenStream2 {
        match self {
            EffectLevel::Pure => enum_expr("Effects", "Pure", span),
            EffectLevel::ReadOnly => enum_expr("Effects", "ReadOnly", span),
            EffectLevel::Effectful => enum_expr("Effects", "Effectful", span),
        }
    }

    fn to_runtime(self) -> dag_core::Effects {
        match self {
            EffectLevel::Pure => dag_core::Effects::Pure,
            EffectLevel::ReadOnly => dag_core::Effects::ReadOnly,
            EffectLevel::Effectful => dag_core::Effects::Effectful,
        }
    }

    fn as_str(self) -> &'static str {
        match self {
            EffectLevel::Pure => "Pure",
            EffectLevel::ReadOnly => "ReadOnly",
            EffectLevel::Effectful => "Effectful",
        }
    }
}

#[derive(Copy, Clone, Debug)]
enum DeterminismLevel {
    Strict,
    Stable,
    BestEffort,
    Nondeterministic,
}

impl DeterminismLevel {
    fn parse(value: &str, span: Span) -> Result<Self> {
        match value {
            "Strict" | "strict" => Ok(DeterminismLevel::Strict),
            "Stable" | "stable" => Ok(DeterminismLevel::Stable),
            "BestEffort" | "best_effort" | "besteffort" => Ok(DeterminismLevel::BestEffort),
            "Nondeterministic" | "non_deterministic" | "nondet" => {
                Ok(DeterminismLevel::Nondeterministic)
            }
            other => Err(syn::Error::new(
                span,
                format!("[DET301] unknown determinism value `{other}`"),
            )),
        }
    }

    fn to_tokens(self, span: Span) -> TokenStream2 {
        match self {
            DeterminismLevel::Strict => enum_expr("Determinism", "Strict", span),
            DeterminismLevel::Stable => enum_expr("Determinism", "Stable", span),
            DeterminismLevel::BestEffort => enum_expr("Determinism", "BestEffort", span),
            DeterminismLevel::Nondeterministic => {
                enum_expr("Determinism", "Nondeterministic", span)
            }
        }
    }

    fn to_runtime(self) -> dag_core::Determinism {
        match self {
            DeterminismLevel::Strict => dag_core::Determinism::Strict,
            DeterminismLevel::Stable => dag_core::Determinism::Stable,
            DeterminismLevel::BestEffort => dag_core::Determinism::BestEffort,
            DeterminismLevel::Nondeterministic => dag_core::Determinism::Nondeterministic,
        }
    }

    fn as_str(self) -> &'static str {
        match self {
            DeterminismLevel::Strict => "Strict",
            DeterminismLevel::Stable => "Stable",
            DeterminismLevel::BestEffort => "BestEffort",
            DeterminismLevel::Nondeterministic => "Nondeterministic",
        }
    }
}

struct ParsedEffects {
    tokens: TokenStream2,
    level: EffectLevel,
}

impl ParsedEffects {
    fn new(level: EffectLevel, span: Span) -> Self {
        Self {
            tokens: level.to_tokens(span),
            level,
        }
    }
}

struct ParsedDeterminism {
    tokens: TokenStream2,
    level: DeterminismLevel,
}

impl ParsedDeterminism {
    fn new(level: DeterminismLevel, span: Span) -> Self {
        Self {
            tokens: level.to_tokens(span),
            level,
        }
    }
}

struct ResourceSpec {
    alias: Ident,
    capability: Path,
    span: Span,
}

impl Parse for ResourceSpec {
    fn parse(input: ParseStream) -> Result<Self> {
        let alias: Ident = input.parse()?;
        let span = alias.span();
        let content;
        syn::parenthesized!(content in input);
        let capability: Path = content.parse()?;
        Ok(ResourceSpec {
            alias,
            capability,
            span,
        })
    }
}

struct ResourceList {
    entries: Vec<ResourceSpec>,
}

impl Parse for ResourceList {
    fn parse(input: ParseStream) -> Result<Self> {
        let punctuated = Punctuated::<ResourceSpec, Token![,]>::parse_terminated(input)?;
        Ok(ResourceList {
            entries: punctuated.into_iter().collect(),
        })
    }
}

struct NodeArgs {
    name: Option<LitStr>,
    summary: Option<LitStr>,
    effects: Option<ParsedEffects>,
    determinism: Option<ParsedDeterminism>,
    kind: Option<TokenStream2>,
    input_schema: Option<LitStr>,
    output_schema: Option<LitStr>,
    resources: Vec<ResourceSpec>,
}

impl NodeArgs {
    fn parse(args: Punctuated<Meta, Token![,]>, defaults: &NodeDefaults) -> Result<Self> {
        let mut parsed = NodeArgs {
            name: None,
            summary: None,
            effects: None,
            determinism: None,
            kind: None,
            input_schema: None,
            output_schema: None,
            resources: Vec::new(),
        };

        for meta in args {
            match meta {
                Meta::NameValue(MetaNameValue { path, value, .. }) => {
                    let ident = path
                        .get_ident()
                        .ok_or_else(|| syn::Error::new(path.span(), "expected identifier"))?;
                    let lit = match value {
                        Expr::Lit(ExprLit { lit, .. }) => lit,
                        _ => return Err(syn::Error::new(value.span(), "expected literal value")),
                    };
                    match ident.to_string().as_str() {
                        "name" => match lit {
                            Lit::Str(s) => parsed.name = Some(s),
                            _ => {
                                return Err(syn::Error::new(
                                    lit.span(),
                                    "name must be string literal",
                                ));
                            }
                        },
                        "summary" => match lit {
                            Lit::Str(s) => parsed.summary = Some(s),
                            _ => {
                                return Err(syn::Error::new(
                                    lit.span(),
                                    "summary must be string literal",
                                ));
                            }
                        },
                        "effects" => parsed.effects = Some(parse_effects(&lit)?),
                        "determinism" => parsed.determinism = Some(parse_determinism(&lit)?),
                        "kind" => parsed.kind = Some(parse_node_kind(&lit)?),
                        "in" => match lit {
                            Lit::Str(s) => parsed.input_schema = Some(s),
                            _ => {
                                return Err(syn::Error::new(
                                    lit.span(),
                                    "in must be string literal",
                                ));
                            }
                        },
                        "out" => match lit {
                            Lit::Str(s) => parsed.output_schema = Some(s),
                            _ => {
                                return Err(syn::Error::new(
                                    lit.span(),
                                    "out must be string literal",
                                ));
                            }
                        },
                        other => {
                            return Err(syn::Error::new(
                                ident.span(),
                                format!("[DAG003] unknown attribute `{other}`"),
                            ));
                        }
                    }
                }
                Meta::List(list) => {
                    let ident = list
                        .path
                        .get_ident()
                        .ok_or_else(|| syn::Error::new(list.path.span(), "expected identifier"))?;
                    match ident.to_string().as_str() {
                        "resources" => {
                            let entries = syn::parse2::<ResourceList>(list.tokens.clone())?.entries;
                            parsed.resources.extend(entries.into_iter());
                        }
                        other => {
                            return Err(syn::Error::new(
                                ident.span(),
                                format!("[DAG003] unknown attribute `{other}`"),
                            ));
                        }
                    }
                }
                Meta::Path(path) => {
                    return Err(syn::Error::new(
                        path.span(),
                        "expected key = \"value\" pairs in attribute",
                    ));
                }
            }
        }

        if parsed.effects.is_none() {
            parsed.effects = Some(ParsedEffects::new(defaults.effects, Span::call_site()));
        }
        if parsed.determinism.is_none() {
            parsed.determinism = Some(ParsedDeterminism::new(
                defaults.determinism,
                Span::call_site(),
            ));
        }
        if parsed.kind.is_none() {
            parsed.kind = Some(enum_expr("NodeKind", defaults.kind, Span::call_site()));
        }

        Ok(parsed)
    }
}

fn expand_node(attr: TokenStream, item: TokenStream, defaults: NodeDefaults) -> TokenStream {
    let args = parse_macro_input!(attr with Punctuated::<Meta, Token![,]>::parse_terminated);
    let function = parse_macro_input!(item as ItemFn);
    match node_impl(args, function, defaults) {
        Ok(tokens) => tokens.into(),
        Err(err) => err.to_compile_error().into(),
    }
}

fn node_impl(
    args: Punctuated<Meta, Token![,]>,
    function: ItemFn,
    defaults: NodeDefaults,
) -> Result<TokenStream2> {
    if function.sig.asyncness.is_none() {
        return Err(syn::Error::new(
            function.sig.span(),
            "[DAG001] #[node] requires an async function",
        ));
    }

    let config = NodeArgs::parse(args, &defaults)?;
    let name_lit = config.name.as_ref().cloned().ok_or_else(|| {
        syn::Error::new(
            function.sig.span(),
            "[DAG001] #[node] requires `name = \"...\"`",
        )
    })?;
    let summary_expr = config
        .summary
        .as_ref()
        .map(|s| quote!(Some(#s)))
        .unwrap_or_else(|| quote!(None));

    let (input_schema_expr, output_schema_expr) = infer_schemas(&function, &config)?;

    let fn_name = &function.sig.ident;
    let spec_ident = format_ident!("{}_NODE_SPEC", fn_name.to_string().to_uppercase());
    let accessor_ident = format_ident!("{}_node_spec", fn_name);

    let effects = config
        .effects
        .as_ref()
        .expect("effects default should be populated");
    let determinism = config
        .determinism
        .as_ref()
        .expect("determinism default should be populated");
    let kind_expr = config
        .kind
        .as_ref()
        .expect("node kind default should be populated");

    let (determinism_hints, effect_hints) = compute_resource_hints(&config.resources);
    validate_effect_hints(&effect_hints, effects)?;
    validate_determinism_hints(&determinism_hints, determinism)?;
    let determinism_hints_expr = hint_array_tokens(&determinism_hints);
    let effect_hints_expr = hint_array_tokens(&effect_hints);
    let effects_expr = &effects.tokens;
    let determinism_expr = &determinism.tokens;

    Ok(quote! {
        #function

        #[allow(non_upper_case_globals)]
        pub const #spec_ident: ::dag_core::NodeSpec = ::dag_core::NodeSpec {
            identifier: concat!(module_path!(), "::", stringify!(#fn_name)),
            name: #name_lit,
            kind: #kind_expr,
            summary: #summary_expr,
            in_schema: #input_schema_expr,
            out_schema: #output_schema_expr,
            effects: #effects_expr,
            determinism: #determinism_expr,
            determinism_hints: #determinism_hints_expr,
            effect_hints: #effect_hints_expr,
        };

        #[allow(dead_code)]
        pub fn #accessor_ident() -> &'static ::dag_core::NodeSpec {
            &#spec_ident
        }
    })
}

struct HintSpec {
    value: String,
    span: Span,
    origin: String,
}

fn compute_resource_hints(resources: &[ResourceSpec]) -> (Vec<HintSpec>, Vec<HintSpec>) {
    let mut determinism = Vec::new();
    let mut effects = Vec::new();
    let mut determinism_seen = HashSet::new();
    let mut effects_seen = HashSet::new();

    for resource in resources {
        let alias = resource.alias.to_string();
        let alias_lower = alias.to_ascii_lowercase();
        let cap_ident = resource
            .capability
            .segments
            .last()
            .map(|segment| segment.ident.to_string())
            .unwrap_or_default();
        let cap_lower = cap_ident.to_ascii_lowercase();

        let inferred = hints::infer(&alias_lower, &cap_lower);

        for hint in inferred.determinism_hints {
            push_hint(
                &mut determinism,
                &mut determinism_seen,
                hint,
                &alias,
                resource.span,
            );
        }

        for hint in inferred.effect_hints {
            push_hint(&mut effects, &mut effects_seen, hint, &alias, resource.span);
        }

        if inferred.is_empty() {
            let namespace = canonical_namespace(&alias_lower, &cap_lower);
            push_determinism_hint(
                &mut determinism,
                &mut determinism_seen,
                &alias,
                &alias_lower,
                &cap_lower,
                resource.span,
            );
            push_effect_hints(
                &mut effects,
                &mut effects_seen,
                &alias,
                &alias_lower,
                &cap_lower,
                &namespace,
                resource.span,
            );
        }
    }

    (determinism, effects)
}

fn canonical_namespace(alias: &str, cap_lower: &str) -> String {
    for domain in ["http", "kv", "db", "queue", "blob"] {
        if alias.contains(domain) || cap_lower.contains(domain) {
            return domain.to_string();
        }
    }
    alias
        .split('_')
        .next()
        .map(|part| part.to_ascii_lowercase())
        .unwrap_or_else(|| alias.to_string())
}

fn push_determinism_hint(
    accumulator: &mut Vec<HintSpec>,
    seen: &mut HashSet<String>,
    alias: &str,
    alias_lower: &str,
    cap_lower: &str,
    span: Span,
) {
    if alias_lower.contains("clock") || cap_lower.contains("clock") {
        push_hint(accumulator, seen, "resource::clock", alias, span);
    }

    if alias_lower.contains("rng")
        || alias_lower.contains("random")
        || cap_lower.contains("rng")
        || cap_lower.contains("random")
    {
        push_hint(accumulator, seen, "resource::rng", alias, span);
    }

    if alias_lower.contains("http") || cap_lower.contains("http") {
        push_hint(accumulator, seen, "resource::http", alias, span);
    }

    if alias_lower.contains("kv") || cap_lower.contains("kv") {
        push_hint(accumulator, seen, "resource::kv", alias, span);
    }

    if alias_lower.contains("db") || cap_lower.contains("db") {
        push_hint(accumulator, seen, "resource::db", alias, span);
    }
}

fn push_effect_hints(
    accumulator: &mut Vec<HintSpec>,
    seen: &mut HashSet<String>,
    alias: &str,
    alias_lower: &str,
    cap_lower: &str,
    namespace: &str,
    span: Span,
) {
    if let Some(hint) = classify_read_hint(alias_lower, cap_lower, namespace) {
        push_hint(accumulator, seen, hint.as_str(), alias, span);
    }

    if let Some(hint) = classify_write_hint(alias_lower, cap_lower, namespace) {
        push_hint(accumulator, seen, hint.as_str(), alias, span);
    }
}

fn classify_read_hint(alias_lower: &str, cap_lower: &str, namespace: &str) -> Option<String> {
    const TOKENS: &[&str] = &["read", "fetch", "load", "get"];
    if TOKENS
        .iter()
        .any(|token| cap_lower.contains(token) || alias_lower.contains(token))
    {
        return Some(format!("resource::{}::read", namespace));
    }
    None
}

fn classify_write_hint(alias_lower: &str, cap_lower: &str, namespace: &str) -> Option<String> {
    const TOKENS: &[&str] = &[
        "write",
        "producer",
        "publish",
        "publisher",
        "sender",
        "send",
        "emit",
        "upsert",
        "insert",
        "delete",
        "update",
        "post",
        "put",
        "patch",
    ];
    if TOKENS
        .iter()
        .any(|token| cap_lower.contains(token) || alias_lower.contains(token))
    {
        return Some(format!("resource::{}::write", namespace));
    }
    None
}

fn push_hint(
    accumulator: &mut Vec<HintSpec>,
    seen: &mut HashSet<String>,
    hint: &str,
    origin: &str,
    span: Span,
) {
    let value = hint.to_string();
    if seen.insert(value.clone()) {
        accumulator.push(HintSpec {
            value,
            span,
            origin: origin.to_string(),
        });
    }
}

fn hint_array_tokens(hints: &[HintSpec]) -> TokenStream2 {
    if hints.is_empty() {
        quote!(&[] as &[&'static str])
    } else {
        let values = hints.iter().map(|hint| {
            let lit = LitStr::new(&hint.value, Span::call_site());
            quote!(#lit)
        });
        quote!(&[#(#values),*] as &[&'static str])
    }
}

fn validate_effect_hints(hints: &[HintSpec], effects: &ParsedEffects) -> Result<()> {
    for hint in hints {
        if let Some(constraint) =
            dag_core::effects_registry::constraint_for_hint(hint.value.as_str())
        {
            if !effects.level.to_runtime().is_at_least(constraint.minimum) {
                return Err(syn::Error::new(
                    hint.span,
                    format!(
                        "[EFFECT201] resource `{}` (from `{}`) requires effects >= {}, but node declares {}. {}",
                        hint.value,
                        hint.origin,
                        constraint.minimum.as_str(),
                        effects.level.as_str(),
                        constraint.guidance,
                    ),
                ));
            }
        }
    }
    Ok(())
}

fn validate_determinism_hints(hints: &[HintSpec], determinism: &ParsedDeterminism) -> Result<()> {
    for hint in hints {
        if let Some(constraint) = dag_core::determinism::constraint_for_hint(hint.value.as_str()) {
            if !determinism
                .level
                .to_runtime()
                .is_at_least(constraint.minimum)
            {
                return Err(syn::Error::new(
                    hint.span,
                    format!(
                        "[DET302] resource `{}` (from `{}`) requires determinism >= {}, but node declares {}. {}",
                        hint.value,
                        hint.origin,
                        constraint.minimum.as_str(),
                        determinism.level.as_str(),
                        constraint.guidance,
                    ),
                ));
            }
        }
    }
    Ok(())
}

fn infer_schemas(function: &ItemFn, config: &NodeArgs) -> Result<(TokenStream2, TokenStream2)> {
    let input_schema = if let Some(custom) = &config.input_schema {
        quote!(::dag_core::SchemaSpec::Named(#custom))
    } else {
        let arg = function.sig.inputs.first().ok_or_else(|| {
            syn::Error::new(
                function.sig.span(),
                "[DAG001] nodes must accept exactly one argument",
            )
        })?;
        let ty = match arg {
            syn::FnArg::Typed(pt) => &pt.ty,
            syn::FnArg::Receiver(_) => {
                return Err(syn::Error::new(
                    arg.span(),
                    "[DAG001] first parameter must be typed (use `fn foo(input: T)`)",
                ));
            }
        };
        schema_from_type(ty)
    };

    let output_schema = if let Some(custom) = &config.output_schema {
        quote!(::dag_core::SchemaSpec::Named(#custom))
    } else {
        let ty = match &function.sig.output {
            syn::ReturnType::Type(_, ty) => ty.as_ref(),
            syn::ReturnType::Default => {
                return Err(syn::Error::new(
                    function.sig.output.span(),
                    "[DAG001] nodes must return dag_core::NodeResult<T>",
                ));
            }
        };
        schema_from_return(ty)?
    };

    Ok((input_schema, output_schema))
}

fn schema_from_type(ty: &syn::Type) -> TokenStream2 {
    let ty_str = ty.to_token_stream().to_string();
    if ty_str == "()" {
        quote!(::dag_core::SchemaSpec::Opaque)
    } else {
        let lit = LitStr::new(&ty_str, ty.span());
        quote!(::dag_core::SchemaSpec::Named(#lit))
    }
}

fn schema_from_return(ty: &syn::Type) -> Result<TokenStream2> {
    if let syn::Type::Path(path) = ty {
        if let Some(last) = path.path.segments.last() {
            if last.ident == "NodeResult" {
                if let syn::PathArguments::AngleBracketed(args) = &last.arguments {
                    if let Some(syn::GenericArgument::Type(inner)) = args.args.first() {
                        return Ok(schema_from_type(inner));
                    }
                }
            }
        }
    }
    Err(syn::Error::new(
        ty.span(),
        "[DAG001] return type must be dag_core::NodeResult<Output>",
    ))
}

fn ensure_flow_enum_attrs(item: &mut ItemEnum) -> Result<()> {
    ensure_flow_enum_derives(item)?;
    ensure_flow_enum_tag(item);
    Ok(())
}

fn ensure_flow_enum_derives(item: &mut ItemEnum) -> Result<()> {
    let required_traits: [(&str, Path); 5] = [
        ("Debug", parse_quote!(Debug)),
        ("Clone", parse_quote!(Clone)),
        ("Serialize", parse_quote!(::serde::Serialize)),
        ("Deserialize", parse_quote!(::serde::Deserialize)),
        ("JsonSchema", parse_quote!(::schemars::JsonSchema)),
    ];

    if let Some(index) = item
        .attrs
        .iter()
        .position(|attr| attr.path().is_ident("derive"))
    {
        let original = item.attrs.remove(index);
        let mut traits: Vec<Path> = original
            .parse_args_with(Punctuated::<Path, Token![,]>::parse_terminated)?
            .into_iter()
            .collect();

        for (ident, path) in &required_traits {
            let already_present = traits.iter().any(|existing| {
                existing
                    .segments
                    .last()
                    .map(|segment| segment.ident == *ident)
                    .unwrap_or(false)
            });
            if !already_present {
                traits.push(path.clone());
            }
        }

        let updated: Attribute = parse_quote! {
            #[derive(#(#traits),*)]
        };
        item.attrs.insert(index, updated);
    } else {
        let attr: Attribute = parse_quote! {
            #[derive(Debug, Clone, ::serde::Serialize, ::serde::Deserialize, ::schemars::JsonSchema)]
        };
        item.attrs.insert(0, attr);
    }

    Ok(())
}

fn ensure_flow_enum_tag(item: &mut ItemEnum) {
    let mut has_tag = false;
    for attr in &item.attrs {
        if !attr.path().is_ident("serde") {
            continue;
        }

        if let Ok(meta) = attr.parse_args_with(Punctuated::<Meta, Token![,]>::parse_terminated) {
            for entry in meta {
                if let Meta::NameValue(name_value) = entry {
                    if name_value.path.is_ident("tag") {
                        if let Expr::Lit(ExprLit {
                            lit: Lit::Str(value),
                            ..
                        }) = name_value.value
                        {
                            if value.value() == "type" {
                                has_tag = true;
                            }
                        }
                    }
                }
            }
        }
    }

    if !has_tag {
        let attr: Attribute = parse_quote! {
            #[serde(tag = "type")]
        };
        item.attrs.push(attr);
    }
}

fn parse_effects(lit: &Lit) -> Result<ParsedEffects> {
    let value = lit_to_string(lit)?;
    let level = EffectLevel::parse(&value, lit.span())?;
    Ok(ParsedEffects::new(level, lit.span()))
}

fn parse_determinism(lit: &Lit) -> Result<ParsedDeterminism> {
    let value = lit_to_string(lit)?;
    let level = DeterminismLevel::parse(&value, lit.span())?;
    Ok(ParsedDeterminism::new(level, lit.span()))
}

fn parse_node_kind(lit: &Lit) -> Result<TokenStream2> {
    let value = lit_to_string(lit)?;
    match value.as_str() {
        "Trigger" | "trigger" => Ok(enum_expr("NodeKind", "Trigger", lit.span())),
        "Inline" | "inline" => Ok(enum_expr("NodeKind", "Inline", lit.span())),
        "Activity" | "activity" => Ok(enum_expr("NodeKind", "Activity", lit.span())),
        "Subflow" | "subflow" => Ok(enum_expr("NodeKind", "Subflow", lit.span())),
        other => Err(syn::Error::new(
            lit.span(),
            format!("[DAG003] unknown node kind `{other}`"),
        )),
    }
}

fn enum_expr(enum_name: &str, variant: &str, span: Span) -> TokenStream2 {
    let enum_ident = Ident::new(enum_name, span);
    let variant_ident = Ident::new(variant, span);
    quote!(::dag_core::#enum_ident::#variant_ident)
}

fn lit_to_string(lit: &Lit) -> Result<String> {
    if let Lit::Str(s) = lit {
        Ok(s.value())
    } else {
        Err(syn::Error::new(
            lit.span(),
            "value must be specified as string literal",
        ))
    }
}

struct WorkflowInput {
    name: Ident,
    version: LitStr,
    profile: Ident,
    summary: Option<LitStr>,
    bindings: Vec<Binding>,
    connects: Vec<ConnectEntry>,
}

struct Binding {
    alias: Ident,
    expr: Expr,
}

struct ConnectEntry {
    from: Ident,
    to: Ident,
}

struct ConnectArgs {
    from: Ident,
    to: Ident,
}

impl Parse for ConnectArgs {
    fn parse(input: ParseStream) -> Result<Self> {
        let from: Ident = input.parse()?;
        input.parse::<Token![->]>()?;
        let to: Ident = input.parse()?;
        if !input.is_empty() {
            return Err(syn::Error::new(
                input.span(),
                "connect! currently supports only `from -> to` syntax",
            ));
        }
        Ok(Self { from, to })
    }
}

impl Parse for WorkflowInput {
    fn parse(input: ParseStream) -> Result<Self> {
        let mut name = None;
        let mut version = None;
        let mut profile = None;
        let mut summary = None;
        let mut bindings = Vec::new();
        let mut connects = Vec::new();

        while !input.is_empty() {
            let key: Ident = input.parse()?;
            input.parse::<Token![:]>()?;
            match key.to_string().as_str() {
                "name" => {
                    if name.is_some() {
                        return Err(syn::Error::new(key.span(), "duplicate `name` field"));
                    }
                    name = Some(input.parse()?);
                }
                "version" => {
                    if version.is_some() {
                        return Err(syn::Error::new(key.span(), "duplicate `version` field"));
                    }
                    version = Some(input.parse()?);
                }
                "profile" => {
                    if profile.is_some() {
                        return Err(syn::Error::new(key.span(), "duplicate `profile` field"));
                    }
                    profile = Some(input.parse()?);
                }
                "summary" => {
                    summary = Some(input.parse()?);
                }
                other => {
                    return Err(syn::Error::new(
                        key.span(),
                        format!("unknown workflow field `{other}`"),
                    ));
                }
            }

            if input.peek(Token![;]) {
                input.parse::<Token![;]>()?;
                break;
            } else if input.peek(Token![,]) {
                input.parse::<Token![,]>()?;
            } else {
                return Err(syn::Error::new(
                    input.span(),
                    "expected `,` or `;` after workflow metadata",
                ));
            }
        }

        while !input.is_empty() {
            if input.peek(Token![let]) {
                input.parse::<Token![let]>()?;
                let alias: Ident = input.parse()?;
                input.parse::<Token![=]>()?;
                let expr: Expr = input.parse()?;
                input.parse::<Token![;]>()?;
                bindings.push(Binding { alias, expr });
                continue;
            }

            if input.peek(Token![if])
                || input.peek(Token![match])
                || input.peek(Token![while])
                || input.peek(Token![for])
            {
                return Err(syn::Error::new(
                    input.span(),
                    "workflow! does not support Rust control-flow statements; use the flow::switch/for_each helpers",
                ));
            }

            let mac: Macro = input.parse()?;
            if !mac.path.is_ident("connect") {
                return Err(syn::Error::new(
                    mac.span(),
                    "workflow! body currently supports only `let` bindings and `connect!` statements",
                ));
            }
            let args = syn::parse2::<ConnectArgs>(mac.tokens)?;
            if input.peek(Token![;]) {
                input.parse::<Token![;]>()?;
            } else {
                return Err(syn::Error::new(
                    input.span(),
                    "expected `;` after connect! statement",
                ));
            }
            connects.push(ConnectEntry {
                from: args.from,
                to: args.to,
            });
        }

        Ok(WorkflowInput {
            name: name
                .ok_or_else(|| syn::Error::new(Span::call_site(), "workflow! requires `name`"))?,
            version: version.ok_or_else(|| {
                syn::Error::new(Span::call_site(), "workflow! requires `version`")
            })?,
            profile: profile.ok_or_else(|| {
                syn::Error::new(Span::call_site(), "workflow! requires `profile`")
            })?,
            summary,
            bindings,
            connects,
        })
    }
}

impl WorkflowInput {
    fn expand(&self) -> Result<TokenStream2> {
        let fn_name = &self.name;
        let flow_name = self.name.to_string();
        let flow_name_lit = LitStr::new(&flow_name, self.name.span());
        let profile_ident = &self.profile;
        let version_literal = &self.version;

        Version::parse(&version_literal.value()).map_err(|err| {
            syn::Error::new(
                version_literal.span(),
                format!("invalid semver literal: {err}"),
            )
        })?;

        let mut alias_set = HashSet::new();
        for binding in &self.bindings {
            let alias_str = binding.alias.to_string();
            if !alias_set.insert(alias_str.clone()) {
                return Err(syn::Error::new(
                    binding.alias.span(),
                    format!("[DAG205] duplicate node alias `{}`", binding.alias),
                ));
            }
        }

        for connect in &self.connects {
            let from = connect.from.to_string();
            let to = connect.to.to_string();
            if !alias_set.contains(&from) {
                return Err(syn::Error::new(
                    connect.from.span(),
                    format!("[DAG202] unknown node alias `{}`", connect.from),
                ));
            }
            if !alias_set.contains(&to) {
                return Err(syn::Error::new(
                    connect.to.span(),
                    format!("[DAG202] unknown node alias `{}`", connect.to),
                ));
            }
        }

        let summary_stmt = if let Some(summary) = &self.summary {
            quote!(builder.summary(Some(#summary));)
        } else {
            quote!()
        };

        let binding_statements = self.bindings.iter().map(|binding| {
            let alias = &binding.alias;
            let alias_str = binding.alias.to_string();
            let expr = &binding.expr;
            quote! {
                let #alias = builder
                    .add_node(#alias_str, #expr)
                    .expect(concat!(
                        "[DAG205] duplicate node alias `",
                        stringify!(#alias),
                        "`"
                    ));
            }
        });

        let connect_statements = self.connects.iter().map(|connect| {
            let from = &connect.from;
            let to = &connect.to;
            quote! {
                builder.connect(&#from, &#to);
            }
        });

        Ok(quote! {
            pub fn #fn_name() -> ::dag_core::FlowIR {
                let version = ::dag_core::prelude::Version::parse(#version_literal)
                    .expect("workflow!: invalid semver literal");
                let mut builder = ::dag_core::FlowBuilder::new(
                    #flow_name_lit,
                    version,
                    ::dag_core::Profile::#profile_ident,
                );

                #summary_stmt
                #(#binding_statements)*
                #(#connect_statements)*

                builder.build()
            }
        })
    }
}
````

## File: examples/s1_echo/src/lib.rs
````rust
use std::sync::Arc;
use std::time::Duration;

use dag_core::{FlowIR, NodeResult};
use dag_macros::{node, trigger};
use host_inproc::EnvironmentPlugin;
use kernel_exec::{FlowExecutor, NodeRegistry};
use kernel_plan::{ValidatedIR, validate};
use serde::{Deserialize, Serialize};

mod auth;
use auth::AuthUser;

#[derive(Clone, Debug, Serialize, Deserialize)]
pub struct EchoRequest {
    pub value: String,
}

#[derive(Clone, Debug, Serialize, Deserialize)]
pub struct EchoResponse {
    pub value: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub user: Option<AuthUser>,
}

#[trigger(
    name = "HttpTrigger",
    summary = "Ingress HTTP trigger for the echo route"
)]
async fn http_trigger(request: EchoRequest) -> NodeResult<EchoRequest> {
    Ok(request)
}

#[node(
    name = "Normalize",
    summary = "Trim whitespace and lowercase the payload",
    effects = "Pure",
    determinism = "Strict"
)]
async fn normalize(input: EchoRequest) -> NodeResult<EchoResponse> {
    let normalized = EchoResponse {
        value: input.value.trim().to_lowercase(),
        user: None,
    };
    Ok(normalized)
}

#[node(
    name = "Responder",
    summary = "Finalize the HTTP response",
    effects = "Pure",
    determinism = "Strict"
)]
async fn responder(mut payload: EchoResponse) -> NodeResult<EchoResponse> {
    payload.user = auth::current_user();
    Ok(payload)
}

dag_macros::workflow! {
    name: s1_echo_flow,
    version: "1.0.0",
    profile: Web,
    summary: "Implements the S1 webhook echo example";
    let trigger = http_trigger_node_spec();
    let normalize = normalize_node_spec();
    let responder = responder_node_spec();
    connect!(trigger -> normalize);
    connect!(normalize -> responder);
}

/// Convenience helper to fetch the Flow IR for the example workflow.
pub fn flow() -> FlowIR {
    s1_echo_flow()
}

pub const TRIGGER_ALIAS: &str = "trigger";
pub const CAPTURE_ALIAS: &str = "responder";
pub const ROUTE_PATH: &str = "/echo";
pub const DEADLINE: Duration = Duration::from_millis(250);

/// Construct a node registry with the example node implementations registered.
pub fn executor() -> FlowExecutor {
    let mut registry = NodeRegistry::new();
    registry
        .register_fn("example_s1_echo::http_trigger", http_trigger)
        .expect("register http_trigger");
    registry
        .register_fn("example_s1_echo::normalize", normalize)
        .expect("register normalize");
    registry
        .register_fn("example_s1_echo::responder", responder)
        .expect("register responder");
    FlowExecutor::new(Arc::new(registry))
}

/// Validated Flow IR for the example workflow.
pub fn validated_ir() -> ValidatedIR {
    validate(&flow()).expect("S1 flow should validate")
}

pub fn environment_plugins() -> Vec<Arc<dyn EnvironmentPlugin>> {
    auth::environment_plugins()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn flow_contains_expected_nodes() {
        let ir = flow();
        let aliases: Vec<_> = ir.nodes.iter().map(|node| node.alias.as_str()).collect();
        assert_eq!(aliases, vec!["trigger", "normalize", "responder"]);
        assert_eq!(ir.edges.len(), 2);
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
# Agent Briefing: Host Adapters & Entrypoint Macros

You are designing the developer surface that lets the same LatticeFlow flow run in containers, WASI/wasmtime, and Cloudflare Workers. Use the bundled sources to reason about:
- How dag-macros, capability hints, and example flows should evolve to offer #[entry_*] macros that emit host-specific bootstraps while keeping user code host-agnostic.
- Patterns for default vs explicit capability selection ("caps=auto" vs per-domain overrides) and how to surface optional facets (e.g., HTTP mTLS) safely.
- What validation, docs, and scaffolding (examples/tests) are required so engineers understand portability guarantees and host-specific opt-ins.
- How to stage the work: macro API design, example rewrites, CI matrix, developer ergonomics.

Produce recommended API designs (with macro signatures and expansion sketches), migration notes for existing flows, risk ledger, and next steps for tooling and documentation.

Read the repository snapshot in full before you answer; do not assume any context beyond this briefing and the selected files.

---
