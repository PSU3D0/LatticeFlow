Status: Draft
Purpose: epic
Owner: Core
Last reviewed: 2025-12-12

# Epic 07: Marketplace/Registry (Publish + Policy Gates)

Goal
- Publish connectors/subflows with certification evidence and enforceable policy gates.

Value
- Enables reuse, versioned upgrades, and safe sharing across teams/tenants.

Scope
- Registry metadata model and CLI surfaces.
- Evidence-required publish gating.

Non-goals
- Full Studio UI.

Dependencies
- Epic 05 evidence harness.

Phases (ticket-sized)

07.1 Metadata model
- Define connector/subflow identity, versions, and compatibility declarations.

Acceptance gates
- Deterministic identity/version rules.

07.2 Publish/consume CLI
- Publish artifacts with evidence attached.
- Consume and verify evidence.

Acceptance gates
- Publish fails without required evidence.

07.3 Policy gates
- Define minimal policy checks based on evidence.

Acceptance gates
- Policy blocks unsafe connectors deterministically.

References
- Planned crates: `crates/registry-client`, `crates/registry-cert`, `crates/policy-engine`
