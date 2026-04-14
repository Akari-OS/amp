# Changelog

All notable changes to the AMP (Agent Memory Protocol) specification are documented here.
The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and AMP
follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html) at the spec level.

## [v0.2.0-draft] — 2026-04-14

Delta spec aligning AMP with the AKARI-OS ecosystem. **Fully backwards-compatible with v0.1**;
v0.2 only adds on top of v0.1 and deprecates two informative sections.

See [`spec/v0.2/protocol.md`](spec/v0.2/protocol.md) for the full delta.

### Added

- **Pool interop (§1).** Defines the role boundary between AkariPool (physical /
  structural / semantic layers) and AMP (experiential / preferential / causal layers),
  a new `pool_item` provenance source kind, and integrity rules for when referenced
  Pool items are deleted, updated, or moved.
- **Agent capability schemas (§2).** Two typed refinements of v0.1's `semantic` type:
  - `amp.preference_bundle/v1` — the Memorist's canonical output.
  - `amp.experiment_record/v1` — the Analyst's causal record format.
  Brief usage notes for the other five AKARI agents (Partner, Studio, Operator,
  Researcher, Guardian) using v0.1's generic types.
- **Platform integration (§3).** Minimum API surface for the AKARI Shell's Memory
  Viewer Module (`/memories`, `/memory/{id}`, `/memory/{id}/graph`) and Analyst
  Reports Module (`/experiments`), plus the Shell Host contract (`amp.encode`,
  `amp.retrieve`, `amp.decay`).
- **Multi-agent orchestration (§4).** Concrete shared-memory flow
  (Memorist → Analyst → Guardian) and an extended conflict resolution rule
  (supersede-on-confidence, tie-break on recency, persistent-conflict review flag)
  that narrows v0.1 §10.3 without replacing it.

### Deprecated

- v0.1 §11.2 *Known Providers (Informative)* — moved to a living `IMPLEMENTATIONS.md`
  outside the normative spec.
- v0.1 §13 encryption mandates — relaxed from "SHOULD encrypt at rest" to
  "SHOULD respect OS filesystem permissions; MAY offer encryption".

### Unchanged

All v0.1 normative content (memory types, lifecycle operations, MemoryRecord schema,
strategy system, confidence scoring, provider interface, MCP compatibility, error
handling, versioning) remains authoritative. v0.2 implementations MUST satisfy every
v0.1 requirement.

## [v0.1.0] — 2026-04-01

Initial draft of the Agent Memory Protocol. See [`spec/v0.1/protocol.md`](spec/v0.1/protocol.md).

### Added

- Four-type cognitive memory system (episodic, semantic, procedural, working).
- Memory lifecycle (encode, retrieve, reinforce, consolidate, transform, decay, delete).
- `MemoryRecord` JSON schema with provenance, access control, and relations.
- Confidence & scoring model (initial, recency, reinforcement, quality gates).
- Strategy system (vector / graph / hierarchical / buffer + router + composition).
- Agent interop: memory exchange format, sharing rules, conflict resolution, A2A integration.
- `AMPProvider` interface + MCP compatibility mapping.
- Security principles, transport options, user control.
- Error model and versioning policy.
