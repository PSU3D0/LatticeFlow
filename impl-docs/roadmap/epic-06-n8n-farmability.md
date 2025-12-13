Status: Draft
Purpose: epic
Owner: Core
Last reviewed: 2025-12-12

# Epic 06: n8n Farmability (Importer + Coverage + HITL)

Goal
- Import n8n JSON workflows into Flow IR for the common subset.
- Produce deterministic reports for unsupported patterns.

Value
- Unlocks the agent-native loop: import -> validate -> repair -> run.

Scope
- Implement `crates/importer-n8n` for a conservative core subset.
- Expression lowering strategy for `={{...}}`.

Non-goals
- Full n8n parity.
- Browser automation and UI-dependent actions.

Dependencies
- Epic 01 stable surfaces (IR + control surfaces).
- Validation diagnostics stability.

Phases (ticket-sized)

06.1 Importer skeleton + core node mapping
- Map:
  - webhook/cron triggers -> Trigger nodes
  - HTTP request nodes -> HTTP capability nodes
  - set/map nodes -> pure transform nodes
  - if/switch -> control surfaces

Acceptance gates
- A representative subset imports into a validating Flow IR.

06.2 Expression lowering
- Provide typed adapters for common patterns:
  - jsonpath-like field selection
  - templating/string interpolation
- Fallback to sandboxed compute node when needed (explicit determinism downgrade).

Acceptance gates
- Imported flows clearly label nondeterminism rather than hiding it.

06.3 Corpus runner + HITL report
- Run a template corpus:
  - conversion rate
  - unsupported nodes list
  - deterministic HITL diff output

Acceptance gates
- Same corpus produces same coverage numbers and reasons.

References
- n8n importer crate: `crates/importer-n8n`
