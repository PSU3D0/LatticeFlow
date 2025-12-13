Status: Draft
Purpose: epic
Owner: Capabilities
Last reviewed: 2025-12-12

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
