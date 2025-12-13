Status: Draft
Purpose: epic
Owner: Core
Last reviewed: 2025-12-12

# Epic 04: Resend Email Sequencing (0.1 Candidate)

Goal
- Ship one real workflow on Workers:
  - inbound webhook event -> sequencing state -> Resend send mechanics

Value
- Validates: triggers, Workers host, effectful HTTP connector, idempotency, durable state, observability.

Scope
- One connector (Resend) implemented by hand (no OpenAPI codegen required).
- One stateful sequencing strategy (Durable Objects recommended).

Non-goals
- Full n8n importer.
- Full marketplace connector packaging.

Dependencies
- Epic 02 Workers host.
- Epic 03 Cloudflare minimal capabilities.
- Epic 01 control surface + invocation ABI contracts.

Phases (ticket-sized)

04.1 Resend webhook trigger adapter
- Verify webhook signature.
- Normalize payload into a stable event struct.
- Populate invocation metadata:
  - `lf.event_id`
  - `lf.run_id`

Acceptance gates
- Invalid signatures are rejected.
- Event id extraction is deterministic.

04.2 Sequencing state machine
- Use DO state per conversation/thread key.
- Ensure strict per-key ordering.

Acceptance gates
- Duplicate/out-of-order deliveries do not produce double-sends.

04.3 Resend send connector (HTTP write)
- Implement send call through the HTTP capability.
- Use idempotency key strategy (header-based) and/or dedupe ledger.

Acceptance gates
- Duplicate injection yields exactly one external side-effect.

04.4 Observability
- Emit metrics/logs keyed by `lf.run_id` and event id.

Acceptance gates
- Operator can correlate webhook -> flow run -> send attempt.

References
- Scenario pack: `impl-docs/user-stories.md` (conceptual), plus a new scenario specific to Resend (to be added).
