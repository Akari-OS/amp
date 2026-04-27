---
spec-id: AMP-003
version: 0.3.0-draft
status: draft
created: 2026-04-27
predecessor: v0.2
ai-context: claude-code
related-specs:
  - AMP-V0.1
  - AMP-V0.2
  - AKARI-AMP-001
  - AKARI-HUB-017
  - ADR-076
---

# Agent Memory Protocol v0.3 (Delta Spec)

> **This is a delta spec.** It describes **only the changes from v0.2**. For the base protocol (types, lifecycle, schema, strategies, provider interface, MCP compatibility, agent schemas, etc.), see [`spec/v0.2/protocol.md`](../v0.2/protocol.md).
>
> Anything not mentioned here is unchanged and still normative as written in v0.2.

---

## Why v0.3?

v0.2 established AKARI ecosystem integration and the Memory Viewer / Analyst Reports APIs. However, the **Strategy Router** — the core mechanism that selects which strategy to use for a given query — remained underspecified. v0.1 §7.3 defined "best strategy" intuitively but gave no algorithm. Implementations varied, and graph-based queries were not reliably routed to the Graph strategy.

v0.3 makes three targeted additions:

1. **Strategy Router Decision Tree** — Explicit algorithm that routes queries to Vector, Graph, Tag, or Hybrid strategy based on keyword detection and intent flags. Normative requirement.
2. **Deduplication Configuration** — Normative integration of ADR-076's threshold values, allowing providers to override embedding similarity thresholds and conflict resolution rules.
3. **Forward Compatibility Signaling** — Informative note that graph walk will become a first-class AMP API in v0.4 (coordinated with POOL-005 transitive relation query).

v0.3 is fully backward-compatible with v0.2. A v0.2-only provider MAY use its existing routing heuristics; a v0.3 provider MUST implement the decision tree as normative.

---

## §6. Strategy Router Decision Tree (Normative)

> **Replaces and extends** v0.1 §7.3. The routing logic described here is **normative** and MUST be implemented by all v0.3 providers.

### 6.1 Routing Algorithm

The Strategy Router determines which memory strategy to use for a given retrieve query by examining the query's textual content, structured fields, and explicit intent flag. The decision tree is deterministic and reproducible.

#### 6.1.1 Query Analysis Phase

Before routing, the provider MUST analyze the retrieve query for the following signals:

| Signal | Detection Method | Example |
|--------|------------------|---------|
| **Causal Keywords** | Text search for `causal:`, `because:`, `led_to:`, `caused`, `effect`, `outcome`, `result`, `consequence` (case-insensitive) | `"What effect did posting at 9am have?"` |
| **Similarity Keywords** | Text search for `similar`, `like`, `related to`, `analogue`, `equivalent`, `comparable` | `"Find memories similar to this video"` |
| **Tag/Topic Query** | Presence of `topic:` prefix or `tags` filter in structured query | `"topic:writing-voice"` or `tags: ["ui", "preference"]` |
| **Intent Flag** | Explicit `intent` field in query (v0.3 extension) | `intent: 'causal'` or `intent: 'similarity'` |
| **Graph Traversal Marker** | Presence of `relations` filter or `depth` parameter | `relations: [{ type: 'reinforces', target: 'mem_abc' }]` |

#### 6.1.2 Decision Tree

Apply the following checks **in order**. The first condition that matches determines the selected strategy. If none match, default to **Hybrid**.

```
1. IF query.intent === 'causal' OR (causal keywords detected AND text length > 5 words)
   THEN → Graph Strategy with depth ≥ 2
   
2. ELSE IF query.intent === 'similarity' OR similarity keywords detected
   THEN → Vector Strategy
   
3. ELSE IF query.intent === 'tag' OR query.tags present OR topic: prefix detected
   THEN → Tag Strategy
   
4. ELSE IF query.relations present OR query.depth present
   THEN → Graph Strategy with specified depth (default 2)
   
5. ELSE IF query.text present (semantic search)
   THEN → Vector Strategy
   
6. ELSE (no recognized signals)
   THEN → Hybrid Strategy (vector-first fallback, see §6.1.3)
```

#### 6.1.3 Hybrid Strategy Composition

When the decision tree resolves to **Hybrid**, the provider MUST implement a two-phase approach:

1. **Phase 1 (Vector)**: Execute a semantic search on the provided text, retrieving top-K memories.
2. **Phase 2 (Graph Expansion)**: For each result, follow relations (up to depth=1 by default) to find supplementary memories.
3. **Merge Policy**: Use `weighted` merge — vector-matched memories scored by relevance, relation-expanded memories scored by edge strength × parent relevance.

### 6.2 Provider Override Hook

Providers MAY expose a hook to override routing decisions on a per-query basis. The hook signature is:

```typescript
interface MemoryProvider {
  /**
   * Override routing decision for a specific query.
   * Return null to use default decision tree.
   * Implementations SHOULD log all overrides for auditability.
   */
  resolveStrategy(query: RetrieveQuery): MemoryStrategy | null;
}
```

**Providers MUST document** any custom routing logic in implementation-specific docs. Overrides MUST be traceable (logged with query and decision).

### 6.3 Routing Decision Logging (Optional but Recommended)

Providers SHOULD return optional debug metadata in the retrieve result to aid in troubleshooting and UX transparency:

```typescript
interface RetrieveResult {
  // ... existing fields ...
  
  /** Optional: Debug information about routing decision (v0.3+) */
  _debug?: {
    /** Which routing condition was matched */
    matched: 'causal-keyword' | 'causal-intent' | 'similarity-keyword' | 'similarity-intent' 
           | 'tag-query' | 'graph-marker' | 'vector-fallback' | 'default-hybrid';
    /** Selected strategy name */
    strategy: string;
    /** Effective depth for graph-based strategies */
    depth?: number;
    /** Execution time in milliseconds */
    durationMs?: number;
  };
}
```

This field MUST NOT be present in production responses sent to agents (only for developer/debug modes).

### 6.4 Query Examples and Expected Routing

| Query | Detection | Selected Strategy | Reasoning |
|-------|-----------|-------------------|-----------|
| `{ text: "What caused the drop in engagement after the redesign?" }` | causal keyword: `caused` | Graph (depth ≥ 2) | Explicit cause-effect question requires traversing causal relations |
| `{ intent: "causal", text: "timeline of decisions leading to this outcome" }` | causal intent flag | Graph (depth ≥ 2) | Explicit intent overrides keyword detection |
| `{ text: "Find posts similar to the morning format" }` | similarity keyword: `similar` | Vector | Semantic search is primary strategy for similarity queries |
| `{ tags: ["writing-voice", "tone"], scope: "shared" }` | tag query + tags filter | Tag | Categorical filtering is most efficient for tag-based queries |
| `{ text: "AI agents in 2026", topic: "research" }` | topic prefix | Tag | Explicit topic takes precedence over free-text search |
| `{ text: "memories related to project X", relations: [{ type: "about", target: "proj_x" }] }` | graph traversal marker | Graph (depth 2) | Relation filter indicates graph strategy needed |
| `{ timeRange: { after: "2026-04-20" }, text: "what happened last week" }` | temporal context only | Hierarchical | Time-bounded queries use hierarchical (temporal) strategy |
| `{ text: "summarize my learning style" }` | unrecognized | Hybrid (vector → graph) | No specific signal → default composition |

---

## §7. Deduplication Configuration (Normative)

> **Extends** v0.2 §3.2.1 (Encode operation). Normatively references ADR-076 for threshold definitions and conflict resolution.

### 7.1 dedupConfig Structure

The `encode()` operation's `EncodeInput` now MAY include a provider-level or query-level deduplication configuration:

```typescript
interface EncodeInput {
  // ... existing fields ...
  
  /** v0.3: Deduplication strategy configuration */
  dedupConfig?: DedupConfig;
}

interface DedupConfig {
  /** Enable deduplication check before encoding */
  enabled: boolean;  // default: true
  
  /** Semantic similarity threshold (0.0-1.0) */
  semanticThreshold?: number;  // default: 0.85 (see ADR-076 §3.1)
  
  /** Near-exact similarity threshold (0.0-1.0) */
  nearExactThreshold?: number;  // default: 0.95 (see ADR-076 §3.1)
  
  /** Exact match detection (text hash equality) */
  exactMatchDetection?: boolean;  // default: true
  
  /** What to do with duplicates: 'overwrite' | 'consolidate' | 'relate' */
  conflictResolution: 'overwrite' | 'consolidate' | 'relate';
}
```

### 7.2 Threshold Semantics (Normative Reference to ADR-076)

Providers MUST apply the following threshold-based classification, as defined in ADR-076:

| Classification | Threshold | Action |
|---|---|---|
| **Exact Match** | embedding cosine = 1.0 (text hash collision) | Overwrite existing memory with new (if `conflictResolution: 'overwrite'`) or consolidate (if `consolidate`) |
| **Near-Exact** | embedding cosine ≥ 0.95 | Consolidate: merge into single memory, increment confidence |
| **Semantic Duplicate** | embedding cosine ≥ 0.85 | Relate: keep both, create `related_to` relation with strength = cosine value |
| **Unrelated** | embedding cosine < 0.85 | Encode as new memory |

**Normative reference**: Full rationale, embedding methodology, and cultural context are in ADR-076. This spec defines only the normative thresholds and actions.

### 7.3 Confidence Update on Consolidation

When two memories are consolidated due to near-exact or semantic match:

```
confidence_new = max(confidence_a, confidence_b) + 0.1 * min(confidence_a, confidence_b)
```

This reflects that the second observation strengthens the memory without overstating certainty.

### 7.4 Provider Override

Providers MAY support query-time override of dedup thresholds via a `DedupConfig` parameter passed to `encode()`. Implementations SHOULD document threshold customization in provider-specific docs.

---

## §8. Forward Compatibility: Graph Walk as First-Class API (Informative)

This section signals planned v0.4 changes and is **informative, not normative**.

### 8.1 Planned Graph Walk API

v0.4 is expected to expose graph traversal as a dedicated AMP operation, not just a Strategy Router internal. The signature would be:

```typescript
/** Planned for v0.4 (informative) */
async graphWalk(
  startId: string,
  options: GraphWalkOptions
): Promise<GraphWalkResult>;

interface GraphWalkOptions {
  depth: number;                    // 1-3 (normative limit)
  relationTypes?: string[];         // Filter by relation kind
  maxBreadth?: number;              // Max children per node (default 20)
  visitedSet?: Set<string>;         // Cycle detection
}

interface GraphWalkResult {
  nodes: Array<{ id: string; type: string; confidence: number }>;
  edges: Array<{ from: string; to: string; kind: string; strength: number }>;
}
```

### 8.2 Coordination with Pool

This API is being coordinated with POOL-005 (transitive relation query) to ensure that when a memory references a Pool item, the graph walk can seamlessly traverse both AMP relations and Pool relations via M2C context bridges. Implementation details are under discussion.

### 8.3 Why Not v0.3?

Graph walk is kept out of v0.3's normative scope because:

1. The Strategy Router (§6) achieves routing to Graph strategy without exposing walk as a first-class operation.
2. Providers need time to optimize cycle detection and breadth limits.
3. Pool interop semantics (cross-system graph traversal) need more validation.

v0.3 does not restrict providers from offering graph walk as an extension; this section simply indicates it will become normative in v0.4.

---

## §9. Deprecations from v0.2

None. v0.3 adds only, does not deprecate.

---

## §10. Changelog

- **v0.3.0-draft (2026-04-27)** — Initial delta spec. Added three new chapters (Strategy Router Decision Tree, Deduplication Configuration, Forward Compatibility signaling). Strategy Router decision tree is normative and MUST be implemented by all v0.3 providers.

---

## Appendix: Relationship to v0.2 and v0.1

v0.3 does not modify any v0.2 or v0.1 section. All previous normative content remains in effect. The following sections from v0.1 and v0.2 are directly referenced from v0.3 but remain authoritative in their original specs:

- v0.1 §7.1–7.2 (Strategy types and their characteristics)
- v0.2 §3.2.1 (Encode operation base definition)
- ADR-076 (Dedup threshold rationale and conflict resolution rules)

When a v0.3 implementation encounters a situation not covered by this delta, v0.2 is the fallback, then v0.1.
