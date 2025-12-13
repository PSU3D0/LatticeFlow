Status: Draft
Purpose: epic
Owner: Core
Last reviewed: 2025-12-12

# Epic 05: Connector Certification Harness (Pre-marketplace)

Goal
- Produce evidence-driven confidence for effectful connectors and flows.

Value
- Marketplace and large-scale agent maintenance require proofs, not manual QA.

Scope
- Extend idempotency harness into a cert runner.
- Emit machine-readable evidence artifacts.

Non-goals
- Full registry/publish workflows (Epic 07).

Dependencies
- Existing harness: `crates/testing-harness-idem`
- Error taxonomy + codes:
  - `impl-docs/error-taxonomy.md`
  - `impl-docs/error-codes.md`

Phases (ticket-sized)

05.1 Evidence schema
- Define the minimal artifact format:
  - flow id + version
  - connector id/version
  - test vectors
  - outcomes (pass/fail) with diagnostics

Acceptance gates
- Artifact is stable and diffable.

05.2 Idempotency injection runner
- Duplicate deliveries
- Reordering
- Retry storms

Acceptance gates
- Resend slice passes with single-side-effect guarantee.

05.3 Contract/error mapping tests
- Minimum HTTP contract tests:
  - 200 OK
  - 401/403
  - 429
  - 5xx
- Ensure errors map into stable taxonomy.

Acceptance gates
- Deterministic classification of errors.

References
- Scenario acceptance criteria patterns: `impl-docs/user-stories.md`
