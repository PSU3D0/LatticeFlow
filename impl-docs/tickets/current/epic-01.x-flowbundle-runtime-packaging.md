Status: Draft
Owner: Runtime + Tooling
Epic: 01 (0.1 contract) / 02 (Workers host)
Phase: 01.x (ergonomics)

# Ticket: FlowBundle abstraction (remove executor() boilerplate)

## Background
Today each example crate defines:
- `flow()` -> Flow IR
- `validated_ir()` -> kernel-plan validation
- `executor()` -> manual registry wiring
- host-specific entrypoint constants (`TRIGGER_ALIAS`, `CAPTURE_ALIAS`, `ROUTE_PATH`, `DEADLINE`)

This is workable but verbose and hard to scale. We want a library-first authoring story that still keeps:
- binding and entrypoints out-of-band
- plugin nodes / remote nodes open for future use
- host adapters thin

## Goal
Introduce a FlowBundle abstraction that packages Flow IR + node bindings + entrypoint metadata so hosts and CLI can run workflows without explicit `executor()` helpers in example crates.

## Scope
- Define a `FlowBundle` type that encapsulates:
  - **ValidatedIR** (not raw FlowIR — validation happens at bundle construction)
  - node binding / resolver handle
  - entrypoint defaults (trigger/capture aliases, route metadata)
- Update examples to use the bundle and eliminate custom `executor()` functions.
- Update CLI to consume FlowBundle instead of hard-coded example wiring.

## Non-goals
- Do not change Flow IR schema.
- Do not embed secrets or provider bindings in the bundle (still use `bindings.lock.json`).
- Do not build a full packaging format or deployment bundle yet (future `flows pack`).

## Design Goals
1) **Ergonomics**: no `executor()` per example, minimal boilerplate.
2) **Safety**: allow hosts to reject arbitrary Flow IR when identifiers are not allowed.
3) **Portability**: flows remain host-agnostic, bindings remain out-of-band.
4) **Extensibility**: plugin nodes, remote nodes, or registries remain possible.

---

## Resolved Architectural Decisions

### Crate Location: `host-inproc` (not `dag-core`)

**Decision:** `FlowBundle`, `NodeResolver`, and `FlowEntrypoint` **must** live in `crates/host-inproc`.

**Rationale:**
- `NodeResolver::resolve()` returns `Arc<dyn NodeHandler>`.
- `NodeHandler` is defined in `kernel-exec`.
- `kernel-exec` depends on `dag-core`.
- Placing these types in `dag-core` would create a circular dependency.
- `host-inproc` already depends on `kernel-exec` and `kernel-plan`, making it the correct location.

### Bundle Contains ValidatedIR (not FlowIR)

**Decision:** `FlowBundle` holds `ValidatedIR`, not raw `FlowIR`.

**Rationale:**
- The bundle represents a "ready to run" artifact.
- Validation should happen at bundle construction time (compile-time via the macro).
- This catches errors early and simplifies host logic.
- `ValidatedIR` lives in `kernel-plan`; `host-inproc` already depends on it.

### Method Type: String (not axum-specific)

**Decision:** `FlowEntrypoint.method` is `Option<String>` (e.g., `"GET"`, `"POST"`).

**Rationale:**
- `host-inproc` must remain host-agnostic (no axum dependency).
- Hosts convert the string to their framework's method type.
- Simple and portable.

---

## Proposed Abstraction

### Core Types (in `crates/host-inproc`)
```rust
use kernel_plan::ValidatedIR;
use kernel_exec::NodeHandler;
use std::sync::Arc;
use std::time::Duration;

pub struct FlowBundle {
    pub validated_ir: ValidatedIR,
    pub entrypoints: Vec<FlowEntrypoint>,
    pub resolver: Arc<dyn NodeResolver>,
    pub node_contracts: Vec<NodeContract>,
    pub environment_plugins: Vec<Arc<dyn EnvironmentPlugin>>,
}

pub struct FlowEntrypoint {
    pub trigger_alias: String,
    pub capture_alias: String,
    pub route_path: Option<String>,
    pub method: Option<String>,  // "GET", "POST", etc.
    pub deadline: Option<Duration>,
}

pub trait NodeResolver: Send + Sync {
    fn resolve(&self, identifier: &str) -> Option<Arc<dyn NodeHandler>>;
    fn known_identifiers(&self) -> Vec<String> { Vec::new() }
}

pub struct NodeContract {
    pub identifier: String,
    pub contract_hash: Option<String>,
    pub source: NodeSource,
}

pub enum NodeSource {
    Local,
    Plugin,
    Remote,
}
```

Notes:
- `NodeResolver` is the abstraction boundary. A resolver can be backed by a local `NodeRegistry`, a plugin host, or a remote execution proxy.
- `node_contracts` is an allowlist: a host can ensure the Flow IR references only identifiers present in this list.
- `contract_hash` is optional; it keeps the door open for a future node binding lock without forcing it today.

---

## Macro Design: Handler-to-Spec Coupling

### Current State
The existing `#[node]` macro emits:
- `<fn_name>_node_spec()` — returns `&'static NodeSpec`
- `<FN_NAME>_NODE_SPEC` — const with `identifier: concat!(module_path!(), "::", stringify!(#fn_name))`

The existing `workflow!` macro:
- References specs via `let trigger = http_trigger_node_spec();`
- Does NOT know about handler functions
- Does NOT generate registry registration

Manual `executor()` functions currently do:
```rust
registry.register_fn("example_s1_echo::http_trigger", http_trigger)
```

### Problem
The `workflow_bundle!` macro needs to generate registry registration code, which requires:
1. The handler function name (e.g., `http_trigger`)
2. The identifier string (e.g., `"example_s1_echo::http_trigger"`)

### Solution: Extend `#[node]` to Emit Registration Helper

**Decision:** Extend `#[node]` macro to emit a registration helper function.

The `#[node]` macro will additionally emit:
```rust
pub fn <fn_name>_register(registry: &mut NodeRegistry) -> Result<(), RegistryError> {
    registry.register_fn(
        concat!(module_path!(), "::", stringify!(<fn_name>)),
        <fn_name>
    )
}
```

This allows `workflow_bundle!` to generate:
```rust
fn __register_nodes(registry: &mut NodeRegistry) {
    http_trigger_register(registry).expect("register http_trigger");
    normalize_register(registry).expect("register normalize");
    responder_register(registry).expect("register responder");
}
```

### Bundle Macro Syntax
```rust
workflow_bundle! {
    name: s1_echo_flow,
    version: "1.0.0",
    profile: Web,
    summary: "S1 echo example";

    // Node bindings (spec accessor, handler register function inferred by convention)
    let trigger = http_trigger_node_spec();
    let normalize = normalize_node_spec();
    let responder = responder_node_spec();

    connect!(trigger -> normalize);
    connect!(normalize -> responder);

    // Entrypoint metadata (new)
    entrypoint! {
        trigger: "trigger",
        capture: "responder",
        route: "/echo",
        method: "POST",
        deadline_ms: 250,
    };
}
```

The macro will:
1. Generate `flow()` returning `FlowIR` (existing behavior)
2. Generate `validated_ir()` returning `ValidatedIR`
3. Generate `bundle()` returning `FlowBundle` with:
   - `validated_ir` from step 2
   - `resolver` backed by a `NodeRegistry` with all nodes registered
   - `entrypoints` from the `entrypoint!` blocks
   - `node_contracts` derived from the bound nodes
   - `environment_plugins` (empty by default; can add syntax later)

### Convention: Spec-to-Register Derivation
Given `foo_node_spec()`, the macro assumes `foo_register()` exists.
This is guaranteed by the `#[node]` macro extension.

---

## Host Integration
- Hosts and CLI accept `FlowBundle` rather than `ValidatedIR + NodeRegistry`.
- Suggested API:
```rust
impl FlowBundle {
    pub fn build_runtime(&self, resources: ResourceBag) -> HostRuntime { ... }
    
    pub fn executor(&self) -> FlowExecutor {
        // Build executor from resolver
    }
}
```
- Hosts remain responsible for:
  - selecting entrypoints (from the bundle)
  - binding resources from `bindings.lock.json`
  - enforcing preflight and policy

### Plugin / Remote Nodes
- `NodeResolver` allows non-local nodes. A bundle can include only node contracts while delegating resolution to a resolver chain:
  - local registry
  - plugin registry
  - remote proxy
- A host can enforce a policy such as:
  - allow only `NodeSource::Local`
  - allow local + plugin, reject remote
- This keeps future plugin execution and registry-based hosting open.

### Flow IR and Safety
- The bundle does not change Flow IR. Instead:
  - the bundle provides an allowlist of identifiers (`node_contracts`)
  - hosts reject flows whose identifiers are not in the allowlist
- This preserves the ability to use auto-discovery internally without allowing arbitrary IR to call any linked node.

### Multi-flow Support
- Define `FlowBundleCatalog`:
```rust
pub struct FlowBundleCatalog {
    pub bundles: Vec<FlowBundle>,
}
```
- CLI uses the catalog to select a bundle by name.
- Hosts can mount multiple entrypoints for multiple bundles.

---

## Implementation Plan

### Phase 1: Extend `#[node]` macro
1. Update `crates/dag-macros/src/lib.rs` to emit `<fn_name>_register()` helper.
2. Ensure existing examples still compile (non-breaking addition).

### Phase 2: Define bundle types in `host-inproc`
1. Add `FlowBundle`, `FlowEntrypoint`, `NodeResolver`, `NodeContract`, `NodeSource` to `crates/host-inproc/src/lib.rs`.
2. Implement `NodeResolver` for `NodeRegistry` (adapter).
3. Add `FlowBundle::executor()` and `FlowBundle::build_runtime()` methods.

### Phase 3: Implement `workflow_bundle!` macro
1. Extend `crates/dag-macros/src/lib.rs` with `workflow_bundle!` proc-macro.
2. Parse `entrypoint!` blocks within the macro.
3. Generate `bundle()` function that constructs `FlowBundle`.

### Phase 4: Refactor examples
1. Replace manual `executor()` with `workflow_bundle!` usage.
2. Remove `TRIGGER_ALIAS`, `CAPTURE_ALIAS`, `ROUTE_PATH`, `DEADLINE` constants (now in entrypoint).
3. Keep `environment_plugins()` pattern or integrate into macro.

### Phase 5: Refactor CLI
1. Update `crates/cli/src/main.rs` to consume `bundle()` instead of `executor()` + constants.
2. Remove `load_example()` match statement; use bundle metadata.

### Phase 6: Add allowlist enforcement
1. Host runtime validates that all node identifiers in `ValidatedIR` are present in `node_contracts`.
2. Add error code for unknown identifier (e.g., `BUNDLE101`).

---

## Acceptance Gates (Definition of Done)
- [ ] `#[node]` macro emits `<fn_name>_register()` helper.
- [ ] `FlowBundle` and related types exist in `host-inproc`.
- [ ] All examples compile using `workflow_bundle!` without defining `executor()`.
- [ ] CLI can run `flows run local --example sX_*` using bundle metadata only.
- [ ] Flow IR schema unchanged; bindings remain out-of-band.
- [ ] Host runtime rejects flows whose node identifiers are not in the bundle allowlist.

---

## Files to Touch (expected)
- `crates/dag-macros/src/lib.rs` — extend `#[node]`, add `workflow_bundle!`
- `crates/host-inproc/src/lib.rs` — add `FlowBundle`, `NodeResolver`, etc.
- `crates/cli/src/main.rs` — refactor to use bundles
- `examples/s1_echo/src/lib.rs` — migrate to `workflow_bundle!`
- `examples/s2_site/src/lib.rs` — migrate
- `examples/s3_branching/src/lib.rs` — migrate
- `examples/s4_preflight/src/lib.rs` — migrate
- `examples/s5_unsupported_surface/src/lib.rs` — migrate
- `examples/s6_spill/src/lib.rs` — migrate

---

## Risks / Considerations

### Streaming Nodes
The `#[node]` macro currently handles regular async functions. Streaming nodes use `register_stream_fn`. The registration helper needs to detect or be told whether the node is streaming.

**Mitigation:** Add an optional `streaming` attribute to `#[node]`:
```rust
#[node(name = "StreamNode", streaming)]
async fn stream_node(input: Input) -> NodeResult<impl Stream<Item = NodeResult<Output>>> { ... }
```
The macro emits `<fn_name>_register_stream()` or a unified helper that picks the right registration method.

### Environment Plugins
Current examples expose `environment_plugins()` separately. This could be integrated into the bundle macro later, but for now, `FlowBundle` can have an `environment_plugins` field that examples populate manually or via a builder.

### Contract Hash
Deferred to future node binding lock work. `contract_hash` remains `None` for now.

### Resolver Composition
For local-only bundles, the resolver is simply the registry. Plugin/remote composition is out of scope for this ticket but the trait boundary supports it.

---

## Notes
- Example dependencies (`kernel-exec`, `kernel-plan`, `host-inproc`) are already in place — no Cargo.toml changes needed for examples.
- The `workflow!` macro remains available for cases where only `FlowIR` is needed (no bundle).
- Keep a path open for a future `flows pack` CLI that serializes bundle metadata for deployment.
