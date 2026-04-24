---
spec-id: AMP-V0.1
version: 0.1.0
status: stable
created: 2026-04-01
updated: 2026-04-22
related-specs:
  - AKARI-AMP-001
  - AKARI-HUB-017
  - AKARI-HUB-018
ai-context: claude-code
---

# AMP Protocol Specification v0.1

> Status: Stable (v0.1.0 released 2026-04-01, ref impl planned for v0.2)
> Date: 2026-04-01

## 1. Overview

AMP (Agent Memory Protocol) is an open protocol that standardizes how AI agents store, retrieve, share, and forget memories. It defines a common language for memory operations so that any agent, any backend, and any strategy can interoperate.

It defines:

1. **Types** — What kinds of memories exist (episodic, semantic, procedural, working)
2. **Lifecycle** — How memories are created, reinforced, consolidated, decayed, and deleted
3. **Schema** — How memories are structured (MemoryRecord JSON)
4. **Strategies** — How memories are stored and retrieved (pluggable, composable)
5. **Sharing** — How agents exchange memories safely

```
Agent → [AMP Provider] → Memory Store
                ↕
        Other Agents (via AMP Interop)
```

### 1.1 Design Principles

| Principle | Description |
|-----------|-------------|
| **Memory First** | Agents SHOULD remember before re-discovering. Memory is the first source of truth. |
| **Typed** | Every memory has a type. Types enable strategy selection and lifecycle management. |
| **Lifecycle-Aware** | Memories are living objects — they strengthen, decay, consolidate, and transform. |
| **Scored** | Every memory has a confidence score. Low-confidence memories decay faster. |
| **Local-First** | The protocol MUST support fully local operation. Cloud is optional. |
| **Interoperable** | Any agent framework can implement AMP. Any backend can serve as a provider. |
| **Privacy-Aware** | Access control and PII classification are built into the schema, not bolted on. |

### 1.2 Terminology

| Term | Definition |
|------|-----------|
| **Memory** | A discrete unit of knowledge, experience, or procedure retained by an agent. |
| **MemoryRecord** | The standardized JSON representation of a memory. |
| **Provider** | A backend implementation that stores and retrieves MemoryRecords. |
| **Strategy** | An algorithm that determines how memories are encoded, indexed, and retrieved. |
| **Lifecycle** | The stages a memory passes through from creation to deletion. |
| **Provenance** | Metadata tracking who created a memory, when, and from what source. |
| **Confidence** | A score (0.0–1.0) indicating how reliable a memory is. |
| **Decay** | The process by which unreinforced memories lose confidence over time. |
| **Consolidation** | Merging multiple related memories into a stronger, unified memory. |
| **Transform** | Converting a memory from one form to another (e.g., detailed → summary). |

### 1.3 Relationship to Other Protocols

| Protocol | Scope | AMP Relationship |
|----------|-------|-----------------|
| **MCP** (Model Context Protocol) | Tool calling for LLMs | AMP providers MAY be exposed as MCP tool servers |
| **M2C** (Media to Context) | Media analysis standardization | M2C produces context; AMP stores it as memory |
| **A2A** (Agent-to-Agent) | Agent communication | AMP interop enables memory sharing over A2A |

```
[Media] →(M2C)→ [Context] →(AMP)→ [Memory Store]
                               ↕
                          [Other Agents via A2A]
                               ↕
                          [Tools via MCP]
```

---

## 2. Memory Type System

AMP defines four memory types, grounded in cognitive science (Atkinson-Shiffrin model, Tulving's taxonomy, CoALA framework).

### 2.1 Core Types

| Type | Description | Examples | Default Decay |
|------|-------------|---------|---------------|
| **episodic** | Events, conversations, experiences. Time-stamped and contextual. | "User debugged a CORS issue on 2026-03-15" | Medium (30 days) |
| **semantic** | Facts, knowledge, preferences. Timeless and declarative. | "User prefers TypeScript over JavaScript" | Slow (90 days) |
| **procedural** | How-to knowledge, workflows, patterns. Action-oriented. | "To deploy: run `npm run build && cargo tauri build`" | Slow (90 days) |
| **working** | Temporary, task-scoped context. Exists only during a session. | "Currently editing video #42, timeline at 3:20" | Immediate (session end) |

### 2.2 Type Selection Guidelines

Implementations SHOULD classify memories using these heuristics:

| Signal | Type |
|--------|------|
| Contains a timestamp or "when" | `episodic` |
| Describes a fact, preference, or identity | `semantic` |
| Describes steps, commands, or "how to" | `procedural` |
| Only relevant to the current task/session | `working` |

When ambiguous, prefer `semantic` (most general).

### 2.3 Type Transitions

Memories MAY transition between types through the Transform lifecycle operation:

```
episodic → semantic    (Experience crystallizes into knowledge)
episodic → procedural  (Repeated actions become procedures)
working  → episodic    (Temporary context becomes a remembered event)
working  → semantic    (Session insight becomes lasting knowledge)
```

Transitions MUST preserve provenance chain (see Section 6).

---

## 3. Memory Lifecycle

Every memory follows a lifecycle from creation to deletion. AMP defines seven lifecycle operations.

### 3.1 Lifecycle Overview

```
                    ┌─── Reinforce ←──┐
                    │                  │
Encode → Store → Retrieve ──→ Consolidate
                    │                  │
                    │              Transform
                    │                  │
                    └──→ Decay ──→ Delete
```

### 3.2 Operations

#### 3.2.1 Encode

Create a new MemoryRecord from raw input.

- The provider MUST assign a unique `id`.
- The provider MUST set `createdAt` to the current UTC time.
- The provider MUST calculate initial `confidence` based on source reliability (see Section 5).
- The provider SHOULD detect duplicates and merge via Consolidate instead of creating new records.

```typescript
encode(input: EncodeInput): Promise<MemoryRecord>;

interface EncodeInput {
  /** Memory type */
  type: 'episodic' | 'semantic' | 'procedural' | 'working';
  /** Memory content (natural language) */
  content: string;
  /** Source provenance */
  provenance: ProvenanceInput;
  /** Access policy */
  access?: AccessPolicy;
  /** Tags for categorization */
  tags?: string[];
  /** Relations to other memories */
  relations?: RelationInput[];
  /** Type-specific metadata */
  metadata?: Record<string, unknown>;
}
```

#### 3.2.2 Retrieve

Search for memories matching a query.

- Providers MUST support semantic (embedding-based) search.
- Providers SHOULD support structured filters (type, tags, time range, confidence threshold).
- Results MUST be scored by relevance and sorted descending.
- Results MUST exclude memories below the requested `minConfidence`.

```typescript
retrieve(query: RetrieveQuery): Promise<RetrieveResult>;

interface RetrieveQuery {
  /** Semantic search query (natural language) */
  text?: string;
  /** Filter by memory type */
  types?: ('episodic' | 'semantic' | 'procedural' | 'working')[];
  /** Filter by tags */
  tags?: string[];
  /** Filter by time range */
  timeRange?: { after?: string; before?: string };
  /** Minimum confidence threshold (default: 0.3) */
  minConfidence?: number;
  /** Maximum number of results (default: 10) */
  limit?: number;
  /** Filter by access scope */
  scope?: AccessScope;
  /** Agent ID filter (for multi-agent retrieval) */
  agentId?: string;
}

interface RetrieveResult {
  /** Matched memories, sorted by relevance */
  memories: ScoredMemory[];
  /** Total count (may be greater than returned results) */
  totalCount: number;
  /** Query execution time in milliseconds */
  durationMs: number;
}

interface ScoredMemory {
  /** The memory record */
  record: MemoryRecord;
  /** Relevance score for this query (0.0–1.0) */
  relevance: number;
}
```

#### 3.2.3 Reinforce

Strengthen a memory that has been re-encountered or re-confirmed.

- Providers MUST increment the `reinforcement` counter.
- Providers MUST recalculate `confidence` (see Section 5.3).
- Providers MUST update `updatedAt`.
- Reinforcement MAY also update `content` if new information supplements the existing memory.

```typescript
reinforce(id: string, input?: ReinforceInput): Promise<MemoryRecord>;

interface ReinforceInput {
  /** Optional supplementary content */
  additionalContent?: string;
  /** Source of reinforcement */
  provenance: ProvenanceInput;
}
```

#### 3.2.4 Consolidate

Merge multiple related memories into a single, stronger memory.

- The provider MUST create a new MemoryRecord combining the source memories.
- The new memory MUST reference the source memories in its provenance chain.
- Source memories SHOULD be marked as `consolidated` (not deleted) for audit.
- Confidence of the consolidated memory MUST be >= the highest source confidence.

```typescript
consolidate(ids: string[]): Promise<MemoryRecord>;
```

#### 3.2.5 Transform

Convert a memory from one representation to another.

- Common transforms: detailed → summary, episodic → semantic, multi-memory → digest.
- The provider MUST preserve the provenance chain.
- The provider MUST set `provenance.transformedFrom` on the new record.

```typescript
transform(id: string, target: TransformTarget): Promise<MemoryRecord>;

interface TransformTarget {
  /** Target memory type (for type transitions) */
  type?: 'episodic' | 'semantic' | 'procedural';
  /** Target representation */
  representation?: 'summary' | 'detailed' | 'structured';
  /** Custom transform instructions */
  instructions?: string;
}
```

#### 3.2.6 Decay

Reduce confidence of unreinforced memories over time.

- Decay SHOULD run as a background process (not triggered per-request).
- Decay rate is determined by memory type (see Section 2.1) and `decayRate` field.
- Memories that decay below a threshold (default: 0.1) are candidates for deletion.
- `working` memories MUST be decayed to 0 at session end.

The decay formula:

```
confidence(t) = confidence(t₀) × e^(-decayRate × Δt)
```

Where:

- `t₀` = last reinforcement or creation time
- `Δt` = time elapsed (in days)
- `decayRate` = per-type decay rate (configurable)

Default decay rates:

| Type | decayRate | Half-life |
|------|-----------|-----------|
| `working` | ∞ (immediate at session end) | 0 |
| `episodic` | 0.023 | ~30 days |
| `semantic` | 0.008 | ~90 days |
| `procedural` | 0.008 | ~90 days |

Reinforcement resets the decay clock: each reinforce sets `t₀` to now.

```typescript
decay(options?: DecayOptions): Promise<DecayResult>;

interface DecayOptions {
  /** Only decay memories older than this (ISO 8601) */
  olderThan?: string;
  /** Only decay specific types */
  types?: ('episodic' | 'semantic' | 'procedural' | 'working')[];
  /** Dry run — report what would decay without changing anything */
  dryRun?: boolean;
}

interface DecayResult {
  /** Number of memories affected */
  decayed: number;
  /** Number of memories below deletion threshold */
  markedForDeletion: number;
}
```

#### 3.2.7 Delete

Remove a memory permanently.

- Providers MUST support explicit deletion by ID.
- Providers SHOULD support bulk deletion by filter (type, tags, time range).
- Deletion MUST be permanent (no soft-delete in the protocol; implementations MAY soft-delete internally).
- Deletion of memories with `access.scope: 'team'` or `'public'` SHOULD require confirmation.

```typescript
delete(id: string): Promise<void>;
deleteBulk(filter: DeleteFilter): Promise<{ deleted: number }>;

interface DeleteFilter {
  /** Delete by type */
  types?: ('episodic' | 'semantic' | 'procedural' | 'working')[];
  /** Delete by tags */
  tags?: string[];
  /** Delete memories older than this */
  olderThan?: string;
  /** Delete memories with confidence below this */
  maxConfidence?: number;
}
```

---

## 4. Memory Schema (MemoryRecord)

### 4.1 Core Schema

```typescript
interface MemoryRecord {
  /** Unique identifier (UUID v7 recommended for time-ordering) */
  id: string;

  /** Memory type */
  type: 'episodic' | 'semantic' | 'procedural' | 'working';

  /** Human-readable content (natural language) */
  content: string;

  /** Confidence score (0.0–1.0) */
  confidence: number;

  /** Reinforcement count (starts at 0) */
  reinforcement: number;

  /** Decay rate override (type default if omitted) */
  decayRate?: number;

  /** Provenance metadata */
  provenance: Provenance;

  /** Access control policy */
  access: AccessPolicy;

  /** Categorization tags */
  tags: string[];

  /** Relations to other memories or external entities */
  relations: Relation[];

  /** Creation timestamp (ISO 8601 UTC) */
  createdAt: string;

  /** Last update timestamp (ISO 8601 UTC) */
  updatedAt: string;

  /** Status */
  status: 'active' | 'consolidated' | 'archived';

  /** Type-specific structured data */
  metadata?: Record<string, unknown>;

  /** Extension fields (namespaced) */
  extensions?: Record<string, unknown>;
}
```

### 4.2 Provenance

```typescript
interface Provenance {
  /** Agent that created this memory */
  agent: AgentIdentity;

  /** How this memory was created */
  source: MemorySource;

  /** Session/conversation ID */
  sessionId?: string;

  /** If this memory was transformed from another, the source record ID */
  transformedFrom?: string;

  /** If this memory was consolidated from others, the source record IDs */
  consolidatedFrom?: string[];

  /** Full provenance chain (for deep audit) */
  chain?: ProvenanceEntry[];
}

interface AgentIdentity {
  /** Agent ID (unique across the system) */
  id: string;
  /** Human-readable agent name */
  name: string;
  /** Agent framework or platform */
  platform?: string;  // e.g., "claude-code", "langchain", "crewai"
}

type MemorySource =
  | { kind: 'user_statement'; confidence: number }      // User said it directly
  | { kind: 'observation'; confidence: number }          // Agent observed it
  | { kind: 'inference'; confidence: number }            // Agent inferred it
  | { kind: 'tool_result'; toolName: string; confidence: number }  // Tool returned it
  | { kind: 'consolidation'; sourceIds: string[] }       // Merged from others
  | { kind: 'transform'; sourceId: string; method: string }  // Transformed from another
  | { kind: 'import'; externalSystem: string }           // Imported from external system

interface ProvenanceEntry {
  /** Timestamp of this provenance event */
  timestamp: string;
  /** Operation that produced this entry */
  operation: 'encode' | 'reinforce' | 'consolidate' | 'transform';
  /** Agent that performed the operation */
  agent: AgentIdentity;
  /** Human-readable description */
  description?: string;
}
```

### 4.3 Access Control

```typescript
interface AccessPolicy {
  /** Visibility scope */
  scope: AccessScope;

  /** Read permissions (empty = scope default) */
  readers?: string[];  // Agent IDs

  /** Write permissions (empty = creator only) */
  writers?: string[];  // Agent IDs

  /** PII classification */
  pii: PIILevel;
}

type AccessScope =
  | 'private'   // Only the creating agent
  | 'agent'     // The creating agent + explicitly listed agents
  | 'team'      // All agents in the same team/workspace
  | 'public';   // Any agent with AMP access

type PIILevel =
  | 'none'       // No personal information
  | 'personal'   // Contains personal preferences, habits
  | 'sensitive'  // Contains PII (name, email, location)
  | 'restricted'; // Contains highly sensitive data (financial, medical)
```

### 4.4 Relations

```typescript
interface Relation {
  /** Target memory ID or external URI */
  target: string;

  /** Relation type */
  type: RelationType;

  /** Relation strength (0.0–1.0) */
  strength: number;

  /** Human-readable label */
  label?: string;
}

type RelationType =
  | 'related_to'     // General association
  | 'derived_from'   // This memory was derived from target
  | 'contradicts'    // This memory contradicts target
  | 'supersedes'     // This memory replaces target
  | 'part_of'        // This memory is part of a larger concept
  | 'depends_on'     // This memory depends on target being true
  | 'co_occurred';   // These memories were formed in the same context
```

---

## 5. Confidence & Scoring

### 5.1 Initial Confidence

When a memory is encoded, its initial confidence is determined by the source:

| Source Kind | Base Confidence | Rationale |
|------------|----------------|-----------|
| `user_statement` | 0.9 | User directly stated it |
| `observation` | 0.8 | Agent observed it firsthand |
| `tool_result` | 0.7 | Tool returned it (may have errors) |
| `inference` | 0.5 | Agent inferred it (uncertain) |
| `import` | 0.6 | External system (unknown reliability) |
| `consolidation` | max(sources) | At least as confident as the best source |
| `transform` | source × 0.9 | Slight loss from transformation |

Implementations MAY adjust these base values.

### 5.2 Recency Weighting

For retrieval, confidence is adjusted by recency:

```
effectiveConfidence = confidence × recencyBoost(Δt)

recencyBoost(Δt) = 1.0                    if Δt < 1 day
                 = 1.0 - 0.1 × log₂(Δt)  if 1 day ≤ Δt ≤ 365 days
                 = 0.5                     if Δt > 365 days
```

Where `Δt` is time since last reinforcement in days.

### 5.3 Reinforcement Boost

Each reinforcement increases confidence:

```
newConfidence = min(1.0, confidence + 0.05 × (1.0 - confidence))
```

This gives diminishing returns: first reinforcements matter most.

### 5.4 Quality Gates

| Gate | Threshold | Action |
|------|-----------|--------|
| **Active** | confidence >= 0.3 | Memory is retrievable and usable |
| **Fading** | 0.1 <= confidence < 0.3 | Memory is retrievable but flagged as low-confidence |
| **Forgotten** | confidence < 0.1 | Memory is not returned in retrieval (candidate for deletion) |

---

## 6. Provenance

Provenance tracks the complete history of a memory from creation through every modification.

### 6.1 Requirements

- Every MemoryRecord MUST include a `provenance` field.
- Every lifecycle operation (encode, reinforce, consolidate, transform) MUST append to the `provenance.chain`.
- Provenance MUST be immutable — entries can be appended but never modified or deleted.
- When memories are shared between agents, provenance MUST travel with the memory.

### 6.2 Provenance Chain Example

```json
{
  "provenance": {
    "agent": { "id": "agent-akari-partner", "name": "AKARI Partner", "platform": "akari" },
    "source": { "kind": "user_statement", "confidence": 0.9 },
    "sessionId": "session-2026-04-01-001",
    "chain": [
      {
        "timestamp": "2026-04-01T10:00:00Z",
        "operation": "encode",
        "agent": { "id": "agent-akari-partner", "name": "AKARI Partner", "platform": "akari" },
        "description": "User stated preference for TypeScript"
      },
      {
        "timestamp": "2026-04-05T14:30:00Z",
        "operation": "reinforce",
        "agent": { "id": "agent-akari-partner", "name": "AKARI Partner", "platform": "akari" },
        "description": "User chose TypeScript over JavaScript again in code review"
      },
      {
        "timestamp": "2026-04-10T09:00:00Z",
        "operation": "consolidate",
        "agent": { "id": "agent-akari-partner", "name": "AKARI Partner", "platform": "akari" },
        "description": "Merged with 2 related memories about coding preferences"
      }
    ]
  }
}
```

### 6.3 Cross-Agent Provenance

When a memory is shared from Agent A to Agent B:

1. Agent B MUST preserve the original provenance chain.
2. Agent B MUST append a new entry with `operation: "import"` and its own agent identity.
3. The memory's `access` policy MUST be re-evaluated by Agent B's provider.

---

## 7. Strategy System

The Strategy System allows different memory storage and retrieval algorithms to coexist and be composed.

### 7.1 Strategy Interface

```typescript
interface MemoryStrategy {
  /** Unique strategy identifier */
  id: string;

  /** Human-readable name */
  name: string;

  /** Semantic version */
  version: string;

  /** Capabilities this strategy provides */
  capabilities: StrategyCapability[];

  /** Store a memory */
  encode(record: MemoryRecord): Promise<void>;

  /** Retrieve memories matching a query */
  retrieve(query: RetrieveQuery): Promise<ScoredMemory[]>;

  /** Remove a memory from the strategy's store */
  remove(id: string): Promise<void>;

  /** Estimate storage/retrieval cost */
  estimate(operation: 'encode' | 'retrieve', input: unknown): Promise<StrategyEstimate>;
}

type StrategyCapability =
  | 'semantic_search'       // Embedding-based similarity search
  | 'graph_traversal'       // Follow relations between memories
  | 'temporal_query'        // Query by time range
  | 'structured_filter'     // Filter by type, tags, metadata
  | 'full_text_search'      // Keyword-based search
  | 'aggregation'           // Aggregate/summarize memory sets
  | 'real_time';            // Sub-100ms retrieval

interface StrategyEstimate {
  /** Estimated cost (implementation-defined unit) */
  cost: number;
  /** Cost unit label */
  costUnit: string;
  /** Estimated time in milliseconds */
  timeMs: number;
}
```

### 7.2 Built-in Strategies

Implementations SHOULD provide at least one of these strategies:

| Strategy | Capabilities | Best For | Example Backends |
|----------|-------------|----------|-----------------|
| **Vector** | semantic_search, real_time | "Find memories similar to X" | Qdrant, Pinecone, ChromaDB |
| **Graph** | graph_traversal, structured_filter | "What's related to X?" | Neo4j, SQLite + relations |
| **Hierarchical** | temporal_query, structured_filter, aggregation | "What happened this week?" + layered storage | SQLite FTS5, custom L1/L2/L3 |
| **Buffer** | real_time, structured_filter | Working memory during a session | In-memory, Redis |

### 7.3 Strategy Router

The Strategy Router selects the best strategy for each operation based on the query characteristics.

```typescript
interface StrategyRouter {
  /** Select the best strategy for a retrieve query */
  route(query: RetrieveQuery): MemoryStrategy;

  /** Register a strategy */
  register(strategy: MemoryStrategy): void;

  /** List registered strategies */
  list(): StrategyInfo[];
}
```

Default routing rules:

| Query Characteristics | Selected Strategy |
|----------------------|-------------------|
| `text` provided (semantic search) | Vector |
| `relations` or graph traversal needed | Graph |
| `timeRange` provided | Hierarchical |
| `types: ['working']` | Buffer |
| Multiple characteristics | Composite (see 7.4) |

### 7.4 Strategy Composition

Strategies MAY be composed to combine capabilities:

```typescript
interface CompositeStrategy extends MemoryStrategy {
  /** Sub-strategies in priority order */
  strategies: MemoryStrategy[];

  /** How to merge results from multiple strategies */
  mergePolicy: 'union' | 'intersection' | 'weighted';
}
```

Example: A `Vector + Graph` composite first retrieves semantically similar memories via Vector, then expands results by following graph relations.

```json
{
  "id": "vector-graph-composite",
  "strategies": ["vector-qdrant", "graph-sqlite"],
  "mergePolicy": "union"
}
```

---

## 8. Access Control

### 8.1 Scope Model

AMP uses a four-tier scope model:

```
private → agent → team → public
  (most restrictive)    (least restrictive)
```

| Scope | Who Can Read | Who Can Write |
|-------|-------------|---------------|
| `private` | Creating agent only | Creating agent only |
| `agent` | Creating agent + `readers` list | Creating agent + `writers` list |
| `team` | All agents in the same team | Creating agent + `writers` list |
| `public` | Any agent with AMP access | Creating agent + `writers` list |

### 8.2 PII Protection

Memories containing personal information MUST be classified:

| PIILevel | Description | Requirements |
|----------|-------------|-------------|
| `none` | No personal data | No special handling |
| `personal` | Preferences, habits, opinions | MUST NOT be shared beyond `agent` scope without consent |
| `sensitive` | Name, email, location, identifiers | MUST NOT leave local storage unless encrypted. MUST NOT be shared beyond `private` scope without explicit consent |
| `restricted` | Financial, medical, credentials | MUST be encrypted at rest. MUST NOT be shared. MUST be deleted on user request |

### 8.3 Enforcement

- Providers MUST check access policy on every `retrieve()` call.
- Providers MUST reject `encode()` calls that attempt to set a broader scope than the source memory (e.g., cannot share a `private` source as `public`).
- Providers SHOULD log access attempts to `sensitive` and `restricted` memories for audit.

---

## 9. Memory Transform

Transform operations convert memories from one representation to another.

### 9.1 Transform Types

| Transform | Input | Output | Use Case |
|-----------|-------|--------|----------|
| **Compress** | Detailed memory | Summary | Reduce token usage |
| **Expand** | Summary memory | Detailed | Recover detail from provenance |
| **Type Transition** | Episodic | Semantic/Procedural | Crystallize experience into knowledge |
| **Merge** | Multiple memories | Single consolidated memory | Reduce redundancy |
| **Format** | Natural language | Structured JSON | Machine-readable extraction |

### 9.2 Transform Quality

- Transforms MUST preserve the original memory's `id` in the new memory's `provenance.transformedFrom`.
- Transforms SHOULD reduce confidence slightly (multiply by 0.9) to reflect potential information loss.
- Implementations MAY use LLMs for intelligent transformation (summarization, extraction).

### 9.3 Automatic Transforms

Providers MAY implement automatic transforms as background processes:

| Trigger | Transform | Purpose |
|---------|-----------|---------|
| 5+ related episodic memories | Consolidate → semantic | Crystallize recurring patterns |
| Memory older than 60 days, never reinforced | Compress | Reduce storage |
| 3+ procedural memories for same task | Consolidate | Unified procedure |
| Session end | Working → episodic/semantic | Preserve session insights |

---

## 10. Agent Interop

### 10.1 Memory Exchange Format

Agents exchange memories using the standard MemoryRecord JSON format. The exchange payload adds transport metadata:

```typescript
interface MemoryExchange {
  /** Protocol version */
  protocolVersion: string;

  /** Sending agent */
  sender: AgentIdentity;

  /** Receiving agent */
  receiver: AgentIdentity;

  /** Memories being shared */
  memories: MemoryRecord[];

  /** Exchange metadata */
  exchange: {
    /** Unique exchange ID */
    id: string;
    /** Timestamp */
    timestamp: string;
    /** Reason for sharing */
    reason?: string;
    /** Whether receiver may further share these memories */
    reshareAllowed: boolean;
  };
}
```

### 10.2 Sharing Rules

1. Only memories with `access.scope` of `agent`, `team`, or `public` can be shared.
2. `private` memories MUST NOT be included in exchanges.
3. Memories with `pii: 'sensitive'` or `'restricted'` MUST NOT be shared unless the receiver is on the `readers` list.
4. The receiver MUST re-evaluate access policies upon import.
5. The receiver MUST preserve the original provenance chain.

### 10.3 Conflict Resolution

When an imported memory conflicts with an existing memory:

| Conflict Type | Resolution Strategy |
|--------------|-------------------|
| **Contradiction** | Keep both; add `contradicts` relation. Flag for human review |
| **Duplicate** | Merge via Consolidate; take higher confidence |
| **Update** | If provenance shows same source, update. If different source, keep both |

### 10.4 A2A Integration

AMP memories can be shared over the Google A2A protocol:

- Memories are serialized as JSON `Part` objects in A2A messages.
- The A2A `TaskArtifact` carries the `MemoryExchange` payload.
- A2A's opaque-agent principle is respected: agents share memories voluntarily, never forcibly.

---

## 11. Provider Interface

### 11.1 AMPProvider

Every AMP-compatible backend MUST implement this interface:

```typescript
interface AMPProvider {
  /** Provider metadata */
  info(): ProviderInfo;

  /** Lifecycle operations */
  encode(input: EncodeInput): Promise<MemoryRecord>;
  retrieve(query: RetrieveQuery): Promise<RetrieveResult>;
  reinforce(id: string, input?: ReinforceInput): Promise<MemoryRecord>;
  consolidate(ids: string[]): Promise<MemoryRecord>;
  transform(id: string, target: TransformTarget): Promise<MemoryRecord>;
  decay(options?: DecayOptions): Promise<DecayResult>;
  delete(id: string): Promise<void>;
  deleteBulk(filter: DeleteFilter): Promise<{ deleted: number }>;

  /** Strategy management */
  strategies(): StrategyInfo[];
  setStrategy(strategyId: string): void;
}

interface ProviderInfo {
  /** Provider name */
  name: string;
  /** Provider version */
  version: string;
  /** Supported AMP protocol version */
  protocolVersion: string;
  /** Available strategies */
  strategies: StrategyInfo[];
  /** Supported memory types */
  supportedTypes: ('episodic' | 'semantic' | 'procedural' | 'working')[];
  /** Supported access scopes */
  supportedScopes: AccessScope[];
  /** Whether this provider supports interop (exchange) */
  interop: boolean;
  /** Storage locality */
  locality: 'local' | 'lan' | 'cloud';
}

interface StrategyInfo {
  id: string;
  name: string;
  version: string;
  capabilities: StrategyCapability[];
  isDefault: boolean;
}
```

### 11.2 Known Providers (Informative)

The following systems can implement `AMPProvider`:

| System | Type | Strategy Mapping |
|--------|------|-----------------|
| **MemU** | MCP Server | Vector (semantic search) |
| **Mem0** | SDK / Hosted | Vector + Graph composite |
| **LangMem** | SDK | Vector + Hierarchical |
| **Letta** | Framework | Hierarchical (MemGPT-style) |
| **SQLite + FTS5** | Local DB | Hierarchical + full_text_search |
| **AKARI Memory** | Tauri/Rust | Hierarchical (L1 file / L2 SQLite) |

---

## 12. MCP Compatibility

### 12.1 AMP as MCP Tools

AMP lifecycle operations can be exposed as MCP tools:

| MCP Tool Name | AMP Operation | Description |
|--------------|---------------|-------------|
| `amp_encode` | `encode()` | Store a new memory |
| `amp_retrieve` | `retrieve()` | Search memories |
| `amp_reinforce` | `reinforce()` | Strengthen a memory |
| `amp_consolidate` | `consolidate()` | Merge memories |
| `amp_transform` | `transform()` | Convert a memory |
| `amp_delete` | `delete()` | Remove a memory |
| `amp_info` | `info()` | Provider metadata |

### 12.2 MCP Tool Schema Example

```json
{
  "name": "amp_retrieve",
  "description": "Search agent memories using AMP (Agent Memory Protocol)",
  "inputSchema": {
    "type": "object",
    "properties": {
      "text": {
        "type": "string",
        "description": "Semantic search query"
      },
      "types": {
        "type": "array",
        "items": { "type": "string", "enum": ["episodic", "semantic", "procedural", "working"] }
      },
      "tags": {
        "type": "array",
        "items": { "type": "string" }
      },
      "minConfidence": {
        "type": "number",
        "minimum": 0.0,
        "maximum": 1.0,
        "default": 0.3
      },
      "limit": {
        "type": "integer",
        "minimum": 1,
        "default": 10
      }
    },
    "required": ["text"]
  }
}
```

### 12.3 Migration from Memory MCP Servers

Existing Memory MCP Servers (like the official MCP Knowledge Graph Memory Server) can be wrapped as AMP providers:

1. Map MCP `create_entities` → AMP `encode()` (type: semantic)
2. Map MCP `search_nodes` → AMP `retrieve()` (text search)
3. Map MCP `create_relations` → AMP relation fields
4. Add provenance, confidence, and lifecycle metadata around existing operations

This enables gradual migration without breaking existing MCP-based memory systems.

---

## 13. Security

### 13.1 Principles

1. **Local-First**: AMP MUST support fully local operation. No data leaves the device by default.
2. **Encryption at Rest**: Memories with `pii: 'sensitive'` or `'restricted'` MUST be encrypted at rest.
3. **Encryption in Transit**: Memory exchanges between agents MUST use encrypted transport (TLS 1.3+).
4. **Minimal Exposure**: Providers MUST NOT return more data than requested. Retrieve results MUST respect access policies.
5. **Audit Trail**: Providers SHOULD maintain an audit log of access to sensitive memories.

### 13.2 Transport Options

Following M2C's pattern:

| Topology | Description | Use Case |
|----------|-------------|----------|
| **Local** | Provider runs on the same machine | Default. Desktop apps, privacy-sensitive data |
| **LAN** | Provider on local network | Shared team memory on a NAS or server |
| **Cloud** | Provider on remote servers | Cross-device sync, team collaboration |

### 13.3 User Control

- Users MUST be able to view all memories stored about them.
- Users MUST be able to delete any memory.
- Users MUST be able to export all their memories in standard AMP JSON format.
- These rights are non-negotiable regardless of provider implementation.

---

## 14. Error Handling

### 14.1 Error Model

AMP uses JSON-RPC error codes, consistent with MCP and M2C:

| Code | Meaning |
|------|---------|
| `-32600` | Invalid request |
| `-32601` | Unknown operation |
| `-32602` | Invalid parameters |
| `-32603` | Internal error |
| `-32700` | Parse error |
| `-34001` | Memory not found |
| `-34002` | Access denied |
| `-34003` | Confidence below threshold |
| `-34004` | Strategy not available |
| `-34005` | Duplicate memory detected |
| `-34006` | Consolidation conflict |
| `-34007` | Transform failed |
| `-34008` | Exchange rejected (access policy violation) |

### 14.2 Error Response

```json
{
  "error": {
    "code": -34002,
    "message": "Access denied: memory is private to agent-akari-partner",
    "data": {
      "memoryId": "mem_abc123",
      "requiredScope": "private",
      "requestedBy": "agent-external-001"
    }
  }
}
```

---

## 15. Versioning

- Protocol version: date-based (`YYYY-MM-DD`), consistent with M2C
- Schema version: same date-based format in `MemoryRecord` extensions
- Strategy versions: independent semver
- Provider versions: independent semver
- Backward compatibility: consumers MUST ignore unknown fields

### 15.1 Version Negotiation

Following the MCP pattern:

1. Consumer sends its supported AMP version
2. Provider responds with the same version (if supported) or its latest
3. If incompatible, the connection is rejected

### 15.2 Extension Principles

1. **Optional**: Extensions MUST NOT be required for core functionality
2. **Additive**: Extensions MUST only add, never remove or modify core behavior
3. **Composable**: Multiple extensions MUST work independently
4. **Independently versioned**: Each extension has its own version

### 15.3 Application-Level Record Categorization (Informative)

The core `type` field (§2.1) is a fixed 4-value enum (`episodic` / `semantic` / `procedural` / `working`) and MUST NOT be extended by implementations.

However, applications often need a second axis of categorization orthogonal to `type` — e.g. `"style-preference"`, `"tone"`, `"writing-voice"` — to group memories by domain concept rather than cognitive kind. AMP providers MAY expose an application-level categorization field for this purpose, under the following constraints:

- **Field location**: MUST be placed in `fields` (the extensible property bag, §4.1) or as a well-known optional property outside the core record keys
- **Orthogonality**: MUST NOT conflict with or shadow the core `type` enum
- **Naming**: SDK bindings SHOULD use `kind` (common in existing AKARI SDK bindings) or `category`. Implementations MUST document their chosen name in provider-specific docs
- **Wire format**: Unknown values MUST be round-trippable (forwarded, not stripped) by intermediaries

Reference implementation: the AKARI SDK (`@akari-os/schema-panel`) uses `record.kind: string` and query parameter `record_kind` as a user-defined category alongside the core `type` enum. This is a SDK-layer convention; downstream providers are not required to adopt the name `kind`.

---

## Appendix A: Cognitive Science Foundations

AMP's memory type system is grounded in established cognitive science models:

| Model | Contribution to AMP |
|-------|-------------------|
| **Atkinson-Shiffrin** (1968) | Working → Long-term memory pipeline. AMP's working type + decay model |
| **Tulving** (1972) | Episodic vs. semantic memory distinction. AMP's type system |
| **Anderson (ACT-R)** (1983) | Procedural memory and production rules. AMP's procedural type |
| **CoALA** (Sumers et al., 2023) | Cognitive architecture for language agents. AMP's overall framework |
| **Liu et al.** (2025) | 3D taxonomy (Forms/Functions/Dynamics). AMP's lifecycle model |

---

## Appendix B: Related Specifications

| Specification | Relationship |
|--------------|-------------|
| [MCP](https://modelcontextprotocol.io) | AMP providers can be MCP tool servers |
| [M2C](https://github.com/Akari-OS/m2c) | M2C context can be stored as AMP memories |
| [A2A](https://a2a-protocol.org/) | AMP memories can be exchanged over A2A |
| [Mem0](https://mem0.ai/) | Can implement AMPProvider interface |
| [LangMem](https://github.com/langchain-ai/langmem) | Can implement AMPProvider interface |

---

## Appendix C: Full Example — Memory Lifecycle

A complete example showing a memory through its full lifecycle:

### Step 1: Encode

User says: "I prefer dark mode in all my apps"

```json
{
  "id": "mem_001",
  "type": "semantic",
  "content": "User prefers dark mode in all applications",
  "confidence": 0.9,
  "reinforcement": 0,
  "provenance": {
    "agent": { "id": "agent-akari", "name": "AKARI Partner", "platform": "akari" },
    "source": { "kind": "user_statement", "confidence": 0.9 },
    "sessionId": "sess_001",
    "chain": [{
      "timestamp": "2026-04-01T10:00:00Z",
      "operation": "encode",
      "agent": { "id": "agent-akari", "name": "AKARI Partner", "platform": "akari" }
    }]
  },
  "access": { "scope": "private", "pii": "personal" },
  "tags": ["preference", "ui", "dark-mode"],
  "relations": [],
  "createdAt": "2026-04-01T10:00:00Z",
  "updatedAt": "2026-04-01T10:00:00Z",
  "status": "active"
}
```

### Step 2: Reinforce (Day 5)

User sets dark mode in another context.

```json
{
  "confidence": 0.945,
  "reinforcement": 1,
  "updatedAt": "2026-04-06T14:00:00Z"
}
```

### Step 3: Consolidate (Day 10)

Three related UI preference memories are merged.

```json
{
  "id": "mem_010",
  "type": "semantic",
  "content": "User's UI preferences: dark mode, minimal UI, large fonts, high contrast",
  "confidence": 0.95,
  "provenance": {
    "source": { "kind": "consolidation", "sourceIds": ["mem_001", "mem_004", "mem_007"] },
    "consolidatedFrom": ["mem_001", "mem_004", "mem_007"]
  }
}
```

### Step 4: Retrieve (Day 30)

Query: "What does the user prefer for UI settings?"

```json
{
  "memories": [{
    "record": { "id": "mem_010", "content": "User's UI preferences: dark mode, minimal UI, large fonts, high contrast", "confidence": 0.95 },
    "relevance": 0.92
  }],
  "totalCount": 1,
  "durationMs": 12
}
```
