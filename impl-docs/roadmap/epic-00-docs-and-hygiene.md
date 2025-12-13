Status: Draft
Purpose: epic
Owner: Core
Last reviewed: 2025-12-12

# Epic 00: Docs and Baseline Hygiene

Goal
- Consolidate documentation so the repo has one canonical roadmap, a small normative spec set, ADRs for forever decisions, and scenarios as acceptance targets.
- Ensure the repo is in a "clean baseline" state (fmt + tests green) so future epics are test-driven and incremental.

Why this matters
- Without doc consolidation, every implementation decision risks duplicating or contradicting a mega-plan.
- Without a green baseline, every new epic becomes archaeology and breaks trust in diagnostics.

Scope
- Documentation organization and consolidation.
- Minimal repo hygiene to restore a known-green baseline.

Non-goals
- No new runtime features.
- No new capability providers.

Phases (ticket-sized)

00.1 Doc taxonomy + audit tagging
- Add the standard header block to `impl-docs/*.md` and `impl-docs/*/*.md`:
  - Status: Canonical | Draft | Superseded | Archived
  - Purpose: spec | adr | epic | scenario | notes
  - Owner and Last reviewed
- Acceptance gate: every doc opens with a correct header and can be triaged instantly.

00.2 RFC umbrella + archive longform
- Convert `impl-docs/rust-workflow-tdd-rfc.md` into an umbrella that primarily links into:
  - `impl-docs/spec/*`
  - `impl-docs/adrs/*`
  - `impl-docs/roadmap/*`
- Preserve the longform RFC under `impl-docs/archive/`.
- Acceptance gate: readers can find canonical semantics in <= 2 clicks.

00.3 Replace monolithic impl-plan with epic docs
- Convert `impl-docs/impl-plan.md` into:
  - `impl-docs/roadmap/epics.md` (index)
  - one epic file per epic with phase-level tickets
- Preserve the longform plan under `impl-docs/archive/`.
- Acceptance gate: every planned item has a home epic + phase with an acceptance test.

00.4 Restore green baseline
- Fix fmt drift and compile/test failures.
- Acceptance gate:
  - `cargo fmt --check`
  - `cargo test --workspace`

Deliverables
- Canonical doc spine (`impl-docs/spec`, `impl-docs/adrs`, `impl-docs/roadmap`, `impl-docs/scenarios`, `impl-docs/archive`).
- A minimal normative 0.1 contract spec set.
- Initial ADR batch for forever decisions.

References
- Epic index: `impl-docs/roadmap/epics.md`
- Scenarios: `impl-docs/user-stories.md`
