---
spec-id: AKARI-AMP-001
version: 0.1.0
status: implemented
created: 2026-04-22
updated: 2026-04-22
related-specs:
  - AKARI-HUB-017
  - AKARI-HUB-018
ai-context: claude-code
---

# AKARI-AMP-001: Agent Memory Protocol v0.1 (Reverse Reference)

## Purpose

This is a thin wrapper spec that serves as a stable anchor for Hub-side references (ADR-014 and related-specs entries such as AKARI-HUB-017 and AKARI-HUB-018). The normative content lives in [`spec/v0.1/protocol.md`](../../spec/v0.1/protocol.md). This document does not duplicate that content; it summarises scope, key interfaces, and entry points so Hub tooling can resolve the `related-specs` field without loading the full spec.

## Protocol Summary

**AMP (Agent Memory Protocol)** standardises how AI agents store, retrieve, share, and forget memories. It defines:

- **Four memory types** — episodic, semantic, procedural, working — grounded in Atkinson-Shiffrin / Tulving / CoALA frameworks.
- **MemoryRecord schema** — a JSON object carrying `id`, `type`, `content`, `confidence`, `createdAt`, `updatedAt`, provenance, access-control, and relations fields. The authoritative JSON Schema is at [`spec/v0.1/schema.json`](../../spec/v0.1/schema.json).
- **Memory lifecycle** — encode → retrieve → reinforce → consolidate → decay → delete. Decay is confidence-weighted; high-confidence memories decay more slowly.
- **AMPProvider interface** — a pluggable backend contract. Any storage backend that implements `encode`, `retrieve`, `reinforce`, `consolidate`, `transform`, `decay`, `delete`, and `deleteBulk` is a conformant provider.
- **MCP compatibility** — AMP providers can be exposed as MCP tool servers (Appendix A of the spec), making AMP memory accessible from any MCP-aware agent runtime.

## Status

Spec v0.1 is **stable** (released 2026-04-01). All sections are normative unless marked informative. 参照実装は現時点で未リリース（[IMPLEMENTATIONS.md](../../IMPLEMENTATIONS.md) 参照）。

## Key Entry Points

| What | Where |
|---|---|
| Full normative spec | [`spec/v0.1/protocol.md`](../../spec/v0.1/protocol.md) |
| JSON Schema | [`spec/v0.1/schema.json`](../../spec/v0.1/schema.json) |
| Japanese translation | [`spec/v0.1/ja/protocol.md`](../../spec/v0.1/ja/protocol.md) |
| v0.2 delta (AKARI-specific) | [`spec/v0.2/protocol.md`](../../spec/v0.2/protocol.md) |
| Known implementations | [`IMPLEMENTATIONS.md`](../../IMPLEMENTATIONS.md) |

## Relation to AKARI-HUB-017 / AKARI-HUB-018

AKARI-HUB-017 and AKARI-HUB-018 reference AMP as the Memory Layer contract in the AKARI OS 5-layer architecture (see `spec/v0.2/protocol.md §0.5`). This spec-id (AKARI-AMP-001) is the resolvable anchor for those `related-specs` entries. Any Hub ADR listing `AKARI-AMP-001` in its front matter should be read as: "this decision depends on or is constrained by AMP v0.1 as described in `spec/v0.1/protocol.md`."
