---
spec-id: AMP-002
version: 0.2.0-draft
status: draft
created: 2026-04-14
predecessor: v0.1
ai-context: claude-code
related-specs:
  - AKARI-AMP-001
  - AKARI-HUB-017
  - AKARI-HUB-018
  - AMP-V0.1
  - ACE-V0.1
  - M2C-V0.2
---

# Agent Memory Protocol v0.2 (Delta Spec)

> **This is a delta spec.** It describes **only the changes from v0.1**. For the base protocol (types, lifecycle, schema, strategies, provider interface, MCP compatibility, etc.), see [`spec/v0.1/protocol.md`](../v0.1/protocol.md).
>
> Anything not mentioned here is unchanged and still normative as written in v0.1.

---

## Why v0.2?

v0.1 established AMP as a **generic, framework-agnostic memory protocol**. It covered the fundamentals: four memory types, a lifecycle, a scored schema, pluggable strategies, and a provider interface. It is deliberately neutral with respect to which agents use it and what they use it for.

v0.2 is a targeted alignment pass for the **AKARI-OS ecosystem**. While keeping v0.1 fully intact as the generic base, v0.2 adds the ecosystem-specific glue that real deployments inside AKARI need:

- **Pool interop** — AkariPool is AKARI's Knowledge Store (raw assets, metadata, relations, M2C context). Without explicit role boundaries, AMP and Pool would overlap and duplicate each other. v0.2 draws the line: Pool owns *what exists*; AMP owns *what was learned about it*.
- **7-agent model** — AKARI's Shell runs seven named agents (Partner, Memorist, Analyst, Studio, Operator, Researcher, Guardian). Some of them have specialized memory needs (the Memorist's preference bundles, the Analyst's experiment records) that justify typed schemas on top of the generic `semantic` / `episodic` types.
- **Platform integration** — The AKARI Shell ships a Memory Viewer Module and Analyst Reports Module. v0.2 defines the minimum API surface these modules rely on, so multiple Shell implementations stay consistent.
- **Multi-agent orchestration** — v0.1 defined memory exchange (§10) and conflict resolution (§10.3) in the abstract. v0.2 adds concrete flows between named AKARI agents and ties them to the v0.1 scope model.

v0.2 is backwards-compatible with v0.1. A v0.1-only provider MAY ignore the v0.2 additions; a v0.2 provider MUST still satisfy every v0.1 requirement.

---

## §0.5 Position in AKARI OS

AMP is the **Memory Layer** protocol within the AKARI OS 5-layer architecture:

```
┌─────────────────────────────────────────────┐
│ アプリ層（Module）                            │
│   Writer / Video / Publishing Modules / ...  │
├─────────────────────────────────────────────┤
│ Shell（器）— UI chrome、Module の実行場       │
├─────────────────────────────────────────────┤
│ Agent Runtime — Reference Agents (7) + α     │
│   (reference defaults; apps may add agents)  │
├─────────────────────────────────────────────┤
│ Memory Layer  ← AMP lives here               │
│   Pool (Knowledge Store) + AMP (this spec)   │
├─────────────────────────────────────────────┤
│ Semantic Layer — M2C + ACE                   │
├─────────────────────────────────────────────┤
│ Protocol Suite — MCP / M2C / AMP / ACE       │
└─────────────────────────────────────────────┘
```

AKARI agents write all state exclusively to the Memory Layer (Pool + AMP). The Agent Runtime is ephemeral — state is only persisted here.

> Full ecosystem context: [AKARI VISION](https://github.com/Akari-OS/.github)

---

## 1. Pool Interop (new chapter)

### 1.1 Role Boundary

AKARI-OS has two long-term storage layers with overlapping surface area: **AkariPool** (the Knowledge Store) and **AMP** (the Memory Store). Without a clear line between them, both systems would end up storing preferences, experiment logs, file metadata, and user notes, and neither would be authoritative.

The boundary is drawn by **level of abstraction**, not by content type:

| Layer | Owner | Description | Examples |
|---|---|---|---|
| Physical (raw bytes) | **Pool** | Files, blobs, binary assets on disk or remote storage | MP4 files, PNG images, PDF documents |
| Structural (item + metadata) | **Pool** | Logical items with structured attributes | `pool_items` rows, `ai_tags`, `ai_summary`, dimensions, duration |
| Semantic (M2C context) | **Pool** via Analyzer | Extracted meaning attached to an item | `context_json` produced by an M2C Analyzer |
| Experiential (who did what) | **AMP** | Episodic memories of agent/user actions | "Studio edited video #42 on 2026-04-13 at 14:20" |
| Preferential (learned patterns) | **AMP** | Semantic memories of user preferences | "Memorist learned: user prefers concise tone" |
| Causal (action → outcome) | **AMP** | Experimental records linking cause to effect | "Posting at 09:00 → 2× engagement vs. 21:00" |

**Rule of thumb:** if the fact is *"what exists"*, it belongs in Pool. If the fact is *"what was learned about, by, or because of something"*, it belongs in AMP.

Concretely:
- A 4K video file and its extracted metadata? **Pool.**
- The observation that the user tends to reuse that video's B-roll in morning posts? **AMP.**
- A transcript produced by M2C and attached to the video? **Pool.**
- The Memorist's semantic preference "user writes sharper hooks after reading transcripts"? **AMP.**

Both systems MAY reference the other, but neither copies the other's data. References are resolved at query time.

### 1.2 New Source Kind: `pool_item`

v0.1 §4.2 (Provenance) defines a `sources[]` array where each entry has a `kind` string identifying where the memory came from. v0.2 defines one new well-known kind, `pool_item`, for memories that were derived from or are about a Pool-owned item:

```json
{
  "kind": "pool_item",
  "workspaceId": "default",
  "itemId": "018f5a7c-3b21-7e4e-9f10-9b7a1c5d2e33",
  "itemVersion": "2026-04-13T14:20:00Z"
}
```

Fields:

| Field | Required | Description |
|---|---|---|
| `kind` | yes | MUST be the literal string `"pool_item"`. |
| `workspaceId` | yes | Pool workspace identifier. Defaults to `"default"` if the Pool deployment is single-workspace. |
| `itemId` | yes | Pool's stable UUID for the item. |
| `itemVersion` | no | Optional ISO 8601 timestamp of the Pool item version the memory was derived from. Useful for detecting staleness after Pool-side edits. |

A single MemoryRecord MAY include multiple `pool_item` sources (e.g., a preference derived from a pattern across many items).

Implementations that do not integrate with Pool MAY ignore this kind; it is additive and does not change the shape of `sources[]`.

### 1.3 Integrity Rules

Pool items have their own lifecycle, independent of the memories that reference them. v0.2 specifies how AMP reacts when a referenced Pool item is changed or deleted.

**Pool item deleted:**

- **Episodic memories** referencing the item MUST be retained by default. The episode ("Studio edited item X") is historically true even if X no longer exists. Implementations MUST set a top-level boolean `orphaned: true` on such records and SHOULD record a follow-up audit entry.
- **Semantic memories** that cite the deleted item among multiple sources MUST drop that source from `sources[]` but MUST NOT be deleted themselves, provided at least one other source remains. The learned preference is not invalidated by one source disappearing.
- **Semantic memories** whose only source was the deleted item SHOULD have their `confidence` decremented by a provider-defined amount (default: 0.2) and be flagged for review, but MUST NOT be automatically deleted.
- An audit trail entry MUST be appended describing the Pool-side deletion, so the chain of custody remains reconstructible.

**Pool item version bumped:**

- AMP providers MAY mark dependent semantic memories as `staleness: "source-updated"` to prompt reinforcement or re-derivation. This is advisory; decay logic (v0.1 §5.2, §3.2.6) does not fire automatically on source updates.

**Pool item moved between workspaces:**

- AMP records MUST be rewritten to reflect the new `workspaceId`. This is a metadata update, not a content change, and does not affect confidence.

These rules keep AMP's principle that memories are *experiences*, not *copies*: Pool can lose a file without AMP losing the fact that the file once existed and was used.

---

## 2. Agent Capability Schemas (new chapter)

v0.1 defines four generic memory types (episodic, semantic, procedural, working) that are sufficient for any agent. In AKARI, two of the seven agents — the Memorist and the Analyst — use memory so intensively and in such a structured way that typed sub-schemas improve interoperability and tooling. v0.2 specifies these as *refinements* of v0.1's `semantic` type, using the existing `schema` and `type` fields; they are not new top-level types.

### 2.1 Partner

The Partner is AKARI's conversational front door. It relies primarily on **working memory** (the current conversation state) and acts as a router into the other agents. For the Partner, v0.2 adds no new schema: the generic types suffice. However, the Partner MUST have read access to the shared scope of all other agents so it can make routing decisions with full context. Providers SHOULD optimize retrieval for the Partner's common patterns (last-N conversation turns, user identity, open tasks).

### 2.2 Memorist — `preference_bundle`

The Memorist's job is to learn and maintain the user's preferences across many dimensions. Its canonical output is a **preference bundle**: a semantic memory whose `type` is `"semantic"` and whose `schema` is `"amp.preference_bundle/v1"`.

```json
{
  "id": "mem_0193...",
  "type": "semantic",
  "schema": "amp.preference_bundle/v1",
  "payload": {
    "type": "preference_bundle",
    "dimensions": {
      "style": "casual",
      "tone": "warm-direct",
      "format": "short-paragraphs",
      "language": "ja"
    },
    "sources": ["mem_ep_018f...", "mem_ep_018f...b"],
    "confidence": 0.78,
    "lastReinforced": "2026-04-12T09:30:00Z"
  }
}
```

Field notes:

- `dimensions` is an open object. Well-known keys are `style`, `tone`, `format`, `language`, but implementations MAY add domain-specific dimensions (e.g., `videoPacing`).
- `sources` references the episodic memories that produced each dimension. This preserves explainability: any preference can be traced back to the moments it was learned from.
- `confidence` here shadows the MemoryRecord-level `confidence`; the two MUST remain in sync. Providers are free to derive one from the other.
- `lastReinforced` allows the Memorist to tell the user "I last confirmed this preference 3 days ago" — a common Shell UX need.

### 2.3 Analyst — `experiment_record`

The Analyst runs ongoing experiments about what works (what posts perform, what edits the user keeps, which recommendations are accepted). Its canonical record is an **experiment record**: a semantic memory whose `schema` is `"amp.experiment_record/v1"`.

```json
{
  "id": "mem_0193...",
  "type": "semantic",
  "schema": "amp.experiment_record/v1",
  "payload": {
    "type": "experiment_record",
    "input": {
      "action": "social_post",
      "params": { "platform": "x", "timeOfDay": "09:00", "length": 120 }
    },
    "output": {
      "metrics": { "impressions": 4200, "engagementRate": 0.062 }
    },
    "contextAtTime": {
      "timeOfDay": "morning",
      "dayOfWeek": "Mon",
      "platform": "x",
      "userState": "active"
    },
    "relatedExperiments": ["mem_exp_018e..."]
  }
}
```

Field notes:

- `input` and `output` are intentionally loose objects; the Analyst's experiments are not pre-registered and their shape evolves. The `action` key is the only required field in `input` and acts as the experiment family.
- `contextAtTime` is a frozen snapshot. The point of an experiment record is to preserve the context for future causal reasoning; mutating it would destroy the record's scientific value.
- `relatedExperiments` forms an explicit graph used by the Analyst Reports Module (§3.2) to build cohorts.

### 2.4 Other agents

The remaining four agents use v0.1's generic types without refinement in v0.2. Their typical usage patterns:

- **Studio** (video/asset editing): heavy **procedural** memory (reusable edit recipes, transition presets). Shell SHOULD surface these as reusable templates.
- **Operator** (task execution, posting, scheduling): heavy **episodic** memory (what was posted, when, with what result). Frequently paired with Analyst experiment records via `relations`.
- **Researcher** (external information gathering): heavy **semantic** memory (extracted facts, summaries). Provenance is critical here; Researcher memories SHOULD always include a URL or document source.
- **Guardian** (safety, brand, policy checks): mixed episodic (past flags) and semantic (learned rules). Guardian memories are typically `scope: "system"` and read-only for other agents.

v0.3 MAY promote some of these to typed schemas if usage proves the need.

---

## 3. Platform Integration (new chapter)

v0.2 defines the minimum API surface the AKARI Shell needs in order to ship the Memory Viewer Module and Analyst Reports Module across providers. Providers MAY offer richer APIs; they MUST offer at least these.

### 3.1 Memory Viewer Module API

The Memory Viewer lets the user browse, filter, and inspect their agent memories. It needs list, detail, and graph endpoints:

```
GET /memories
  ?agentId=<id>
  &types=<comma-separated amp types>
  &timeRange=<fromISO>,<toISO>
  &relatedTo=<poolItemId | memoryId>
  &scope=<private|shared|system>
  &cursor=<opaque>
  &limit=<1..200, default 50>
→ 200 OK
  { "items": MemoryRecord[], "nextCursor": string | null }
```

```
GET /memory/{id}
→ 200 OK MemoryRecord (full, including sources and relations)
→ 404 Not Found
```

```
GET /memory/{id}/graph?depth=<1..3, default 2>
→ 200 OK
  {
    "nodes": [{ "id": string, "type": string, "label": string }],
    "edges": [{ "from": string, "to": string, "kind": string }]
  }
```

The graph endpoint is used to render relation visualizations (which memories reinforce which, which derive from which). Depth is capped to prevent runaway traversal.

### 3.2 Analyst Reports Module API

The Analyst Reports Module aggregates experiment records across time to answer questions like "which posting time performs best?"

```
GET /experiments
  ?family=<action string>
  &metric=<metric name>
  &timeRange=<fromISO>,<toISO>
  &groupBy=<contextAtTime key>
→ 200 OK
  {
    "groups": [
      { "key": string, "n": integer, "metric": { "mean": number, "p50": number, "p90": number } }
    ]
  }
```

Responses MUST be reproducible — the same parameters against the same memory store MUST yield identical groupings — so downstream UI can cache them.

### 3.3 Shell Host Contract

In AKARI, the Shell is the host process that owns scheduling and side effects. Providers MUST expose the following three host-facing operations so the Shell can drive the lifecycle without knowing provider internals:

- `amp.encode(record) → { id }` — persist a new MemoryRecord. Returns the assigned id. Wraps v0.1 §3.2.1.
- `amp.retrieve(query) → MemoryRecord[]` — search according to v0.1 §3.2.2.
- `amp.decay() → { processed: number }` — run the decay pass described in v0.1 §3.2.6. Idempotent within a reasonable interval (implementation-defined, default 1h).

The Shell, not the provider, owns **when** `amp.decay()` runs (typically as a low-priority background task). Providers MUST NOT decay memories unprompted; this makes behaviour predictable across Shell versions.

---

## 4. Multi-Agent Orchestration (new chapter)

v0.1 §10 defines memory exchange and conflict resolution in the abstract. v0.2 grounds these in concrete AKARI flows.

### 4.1 Shared Memory Flow

A common flow crosses three of the seven agents in sequence:

1. **Memorist** observes user behaviour across sessions and encodes a preference bundle (`amp.preference_bundle/v1`), scope `shared`.
2. **Analyst** reads the shared preference via `amp.retrieve` when scoring experiment outcomes. For example, an experiment whose output aligns with the user's known style dimension gets a confidence bonus.
3. **Guardian** reads both the preference and the experiment record when performing a safety/brand check on a proposed action. If the action conflicts with either, Guardian encodes a new episodic memory documenting the block and the reason.

All three agents use the same v0.1 schema; only `provenance.agentId` differs. This keeps the cross-agent chain legible to the Memory Viewer (§3.1) without special casing.

### 4.2 Conflict Resolution

When two agents encode semantic memories that make contradictory claims about the same subject (determined by matching `payload.type` and a subject key — e.g., the `style` dimension in a preference bundle), providers MUST apply the following resolution, extending v0.1 §10.3:

1. **Higher confidence wins.** The lower-confidence record is demoted to `status: "superseded"` and linked via `relations` with kind `"supersededBy"`.
2. **Tie on confidence → more recent wins.** `updatedAt` breaks ties.
3. **Persistent conflict** — defined as three or more supersede events in a rolling 30-day window on the same subject key — MUST be flagged for user review via a record whose `type` is `episodic` and `scope` is `system`. The Shell SHOULD surface this in the Memory Viewer.

Superseded memories are retained, not deleted, to preserve the learning trajectory. Reinforcement of a superseded record MAY promote it back above its supersessor; this is the mechanism by which the user's changing mind is captured correctly.

---

## 5. Deprecations from v0.1

v0.2 deprecates the following v0.1 content, removing it from the normative spec but keeping it available in the v0.1 file:

- **v0.1 §11.2 Known Providers (Informative).** Moved to a separate living document, `IMPLEMENTATIONS.md`, at the repo root. The spec should not enumerate implementations; that list ages badly and the v0.1 listing is already out of date.
- **v0.1 §13 Security, encryption requirements.** The normative "providers SHOULD encrypt memory stores at rest" clause is relaxed to "providers SHOULD respect OS-level filesystem permissions and MAY offer encryption". In practice, all known AKARI deployments rely on local filesystem permissions, and mandating encryption added compliance overhead without a credible threat model. Deployments with a real threat model MAY add encryption on top; v0.2 no longer mandates it.

Neither deprecation changes wire formats or provider APIs. v0.1 providers remain conformant under v0.2.

---

## 6. Changelog

- **v0.2.0-draft (2026-04-14)** — Initial delta spec. Added four new chapters (Pool Interop, Agent Capability Schemas, Platform Integration, Multi-Agent Orchestration), deprecated two informative sections of v0.1, introduced the `pool_item` provenance kind and two typed semantic schemas (`amp.preference_bundle/v1`, `amp.experiment_record/v1`).

---

## Appendix: Relationship to v0.1

v0.2 does not modify any v0.1 section. It only adds. The following v0.1 sections are referenced from v0.2 but remain authoritative in v0.1:

- §3 Memory Lifecycle (all operations)
- §4 Memory Schema (MemoryRecord core, provenance, access control, relations)
- §5 Confidence & Scoring
- §7 Strategy System
- §10 Agent Interop (v0.2 §4 narrows, does not replace)
- §11.1 AMPProvider interface
- §12 MCP Compatibility
- §14 Error Handling
- §15 Versioning

When a v0.2 implementation encounters a situation not covered by this delta, v0.1 is the fallback.
