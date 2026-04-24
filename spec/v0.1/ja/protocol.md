# AMP Protocol Specification v0.1

> Status: Draft
> Date: 2026-04-01
>
> 言語: 日本語訳（正本は [英語版](../protocol.md)）

## 1. Overview

AMP（Agent Memory Protocol）は、AIエージェントが記憶を保存・検索・共有・忘却する方法を標準化するオープンプロトコルである。あらゆるエージェント、あらゆるバックエンド、あらゆる Strategy が相互運用できるよう、記憶操作の共通言語を定義する。

定義する内容：

1. **Types** — どのような種類の記憶が存在するか（episodic、semantic、procedural、working）
2. **Lifecycle** — 記憶がどのように生成・強化・統合・減衰・削除されるか
3. **Schema** — 記憶がどのように構造化されるか（MemoryRecord JSON）
4. **Strategies** — 記憶がどのように保存・検索されるか（プラガブル、コンポーザブル）
5. **Sharing** — エージェント間で安全に記憶を交換する方法

```
Agent → [AMP Provider] → Memory Store
                ↕
        Other Agents (via AMP Interop)
```

### 1.1 Design Principles

| 原則 | 説明 |
|------|------|
| **Memory First** | エージェントは再発見する前に、まず記憶を参照すべきである（SHOULD）。記憶が第一の情報源である。 |
| **Typed** | すべての記憶は型を持つ。型により Strategy の選択とライフサイクル管理が可能になる。 |
| **Lifecycle-Aware** | 記憶は生きたオブジェクトである — 強化され、減衰し、統合され、変換される。 |
| **Scored** | すべての記憶は信頼度スコアを持つ。低信頼度の記憶はより速く減衰する。 |
| **Local-First** | プロトコルは完全にローカルでの動作をサポートしなければならない（MUST）。クラウドはオプション。 |
| **Interoperable** | あらゆるエージェントフレームワークが AMP を実装可能。あらゆるバックエンドが Provider となれる。 |
| **Privacy-Aware** | アクセス制御と PII 分類はスキーマに組み込まれており、後付けではない。 |

### 1.2 Terminology

| 用語 | 定義 |
|------|------|
| **Memory** | エージェントが保持する知識・経験・手順の離散的な単位。 |
| **MemoryRecord** | Memory の標準化された JSON 表現。 |
| **Provider** | MemoryRecord を保存・検索するバックエンド実装。 |
| **Strategy** | 記憶のエンコード・インデックス・検索方法を決定するアルゴリズム。 |
| **Lifecycle** | 記憶が生成から削除までに経る段階。 |
| **Provenance** | 誰が、いつ、どのソースから記憶を作成したかを追跡するメタデータ。 |
| **Confidence** | 記憶の信頼性を示すスコア（0.0〜1.0）。 |
| **Decay** | 強化されない記憶が時間とともに信頼度を失うプロセス。 |
| **Consolidation** | 複数の関連する記憶を、より強固で統一された1つの記憶に統合すること。 |
| **Transform** | 記憶をある形式から別の形式に変換すること（例：詳細 → 要約）。 |

### 1.3 Relationship to Other Protocols

| プロトコル | スコープ | AMP との関係 |
|-----------|---------|-------------|
| **MCP**（Model Context Protocol） | LLM 向けツール呼び出し | AMP Provider は MCP ツールサーバーとして公開してもよい（MAY） |
| **M2C**（Media to Context） | メディア分析の標準化 | M2C がコンテキストを生成し、AMP がそれを記憶として保存する |
| **A2A**（Agent-to-Agent） | エージェント間通信 | AMP Interop により A2A 上で記憶共有が可能になる |

```
[Media] →(M2C)→ [Context] →(AMP)→ [Memory Store]
                               ↕
                          [Other Agents via A2A]
                               ↕
                          [Tools via MCP]
```

---

## 2. Memory Type System

AMP は認知科学（Atkinson-Shiffrin モデル、Tulving の分類、CoALA フレームワーク）に基づく4つの記憶型を定義する。

### 2.1 Core Types

| 型 | 説明 | 例 | デフォルト減衰 |
|----|------|-----|--------------|
| **episodic** | イベント、会話、体験。タイムスタンプ付きで文脈的。 | "ユーザーは 2026-03-15 に CORS の問題をデバッグした" | 中程度（30日） |
| **semantic** | 事実、知識、好み。時間に依存せず宣言的。 | "ユーザーは JavaScript より TypeScript を好む" | 遅い（90日） |
| **procedural** | ハウツー知識、ワークフロー、パターン。行動指向。 | "デプロイ手順: `npm run build && cargo tauri build` を実行" | 遅い（90日） |
| **working** | 一時的でタスクスコープのコンテキスト。セッション中のみ存在。 | "現在動画 #42 を編集中、タイムラインは 3:20" | 即時（セッション終了時） |

### 2.2 Type Selection Guidelines

実装は以下のヒューリスティクスを用いて記憶を分類すべきである（SHOULD）：

| シグナル | 型 |
|---------|-----|
| タイムスタンプまたは「いつ」を含む | `episodic` |
| 事実、好み、またはアイデンティティを記述している | `semantic` |
| 手順、コマンド、または「方法」を記述している | `procedural` |
| 現在のタスク/セッションにのみ関連する | `working` |

曖昧な場合は `semantic`（最も汎用的）を優先する。

### 2.3 Type Transitions

記憶は Transform ライフサイクル操作を通じて型間を遷移してもよい（MAY）：

```
episodic → semantic    (経験が知識として結晶化する)
episodic → procedural  (繰り返された行動が手順になる)
working  → episodic    (一時的なコンテキストが記憶されたイベントになる)
working  → semantic    (セッション中の洞察が恒久的な知識になる)
```

遷移は Provenance チェーンを保持しなければならない（MUST）（セクション6参照）。

---

## 3. Memory Lifecycle

すべての記憶は生成から削除までのライフサイクルに従う。AMP は7つのライフサイクル操作を定義する。

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

生の入力から新しい MemoryRecord を作成する。

- Provider は一意の `id` を割り当てなければならない（MUST）。
- Provider は `createdAt` を現在の UTC 時刻に設定しなければならない（MUST）。
- Provider はソースの信頼性に基づいて初期 `confidence` を計算しなければならない（MUST）（セクション5参照）。
- Provider は重複を検出し、新しいレコードを作成する代わりに Consolidate で統合すべきである（SHOULD）。

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

クエリに一致する記憶を検索する。

- Provider はセマンティック（埋め込みベース）検索をサポートしなければならない（MUST）。
- Provider は構造化フィルタ（型、タグ、時間範囲、信頼度閾値）をサポートすべきである（SHOULD）。
- 結果は関連度でスコアリングされ、降順にソートされなければならない（MUST）。
- 結果は要求された `minConfidence` を下回る記憶を除外しなければならない（MUST）。

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

再び遭遇または再確認された記憶を強化する。

- Provider は `reinforcement` カウンタをインクリメントしなければならない（MUST）。
- Provider は `confidence` を再計算しなければならない（MUST）（セクション5.3参照）。
- Provider は `updatedAt` を更新しなければならない（MUST）。
- Reinforce は、新しい情報が既存の記憶を補完する場合、`content` を更新してもよい（MAY）。

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

複数の関連する記憶を単一の、より強固な記憶に統合する。

- Provider はソース記憶を結合した新しい MemoryRecord を作成しなければならない（MUST）。
- 新しい記憶はその Provenance チェーン内でソース記憶を参照しなければならない（MUST）。
- ソース記憶は監査のために `consolidated`（削除ではなく）としてマークされるべきである（SHOULD）。
- 統合された記憶の Confidence は、ソースの最高 Confidence 以上でなければならない（MUST）。

```typescript
consolidate(ids: string[]): Promise<MemoryRecord>;
```

#### 3.2.5 Transform

記憶をある表現から別の表現に変換する。

- 一般的な変換：詳細 → 要約、episodic → semantic、複数記憶 → ダイジェスト。
- Provider は Provenance チェーンを保持しなければならない（MUST）。
- Provider は新しいレコードに `provenance.transformedFrom` を設定しなければならない（MUST）。

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

時間の経過とともに強化されない記憶の Confidence を低下させる。

- Decay はバックグラウンドプロセスとして実行されるべきである（SHOULD）（リクエストごとにトリガーされるのではなく）。
- 減衰率は記憶型（セクション2.1参照）と `decayRate` フィールドによって決定される。
- 閾値（デフォルト: 0.1）を下回った記憶は削除候補となる。
- `working` 記憶はセッション終了時に 0 に減衰されなければならない（MUST）。

減衰の公式：

```
confidence(t) = confidence(t₀) × e^(-decayRate × Δt)
```

ここで：

- `t₀` = 最後の強化または作成時刻
- `Δt` = 経過時間（日単位）
- `decayRate` = 型ごとの減衰率（設定可能）

デフォルトの減衰率：

| 型 | decayRate | 半減期 |
|----|-----------|--------|
| `working` | ∞（セッション終了時に即時） | 0 |
| `episodic` | 0.023 | 約30日 |
| `semantic` | 0.008 | 約90日 |
| `procedural` | 0.008 | 約90日 |

Reinforce は減衰クロックをリセットする：各 Reinforce は `t₀` を現在時刻に設定する。

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

記憶を永久に削除する。

- Provider は ID による明示的な削除をサポートしなければならない（MUST）。
- Provider はフィルタ（型、タグ、時間範囲）による一括削除をサポートすべきである（SHOULD）。
- 削除は永久でなければならない（MUST）（プロトコル上のソフトデリートはない。実装は内部的にソフトデリートしてもよい（MAY））。
- `access.scope: 'team'` または `'public'` の記憶の削除は確認を要求すべきである（SHOULD）。

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

記憶がエンコードされる際、初期 Confidence はソースによって決定される：

| ソース種別 | 基本 Confidence | 根拠 |
|-----------|----------------|------|
| `user_statement` | 0.9 | ユーザーが直接述べた |
| `observation` | 0.8 | エージェントが直接観察した |
| `tool_result` | 0.7 | ツールが返した（エラーの可能性あり） |
| `inference` | 0.5 | エージェントが推論した（不確実） |
| `import` | 0.6 | 外部システム（信頼性不明） |
| `consolidation` | max(sources) | 最も信頼できるソース以上 |
| `transform` | source × 0.9 | 変換による情報損失を反映 |

実装はこれらの基本値を調整してもよい（MAY）。

### 5.2 Recency Weighting

検索時、Confidence は新しさにより調整される：

```
effectiveConfidence = confidence × recencyBoost(Δt)

recencyBoost(Δt) = 1.0                    if Δt < 1 day
                 = 1.0 - 0.1 × log₂(Δt)  if 1 day ≤ Δt ≤ 365 days
                 = 0.5                     if Δt > 365 days
```

ここで `Δt` は最後の強化からの経過日数。

### 5.3 Reinforcement Boost

各 Reinforce は Confidence を増加させる：

```
newConfidence = min(1.0, confidence + 0.05 × (1.0 - confidence))
```

これは収穫逓減を生む：最初の数回の強化が最も効果的である。

### 5.4 Quality Gates

| ゲート | 閾値 | アクション |
|--------|------|-----------|
| **Active** | confidence >= 0.3 | 記憶は検索可能で使用可能 |
| **Fading** | 0.1 <= confidence < 0.3 | 記憶は検索可能だが低信頼度としてフラグ付き |
| **Forgotten** | confidence < 0.1 | 記憶は検索結果に返されない（削除候補） |

---

## 6. Provenance

Provenance は、記憶の作成から全ての変更に至るまでの完全な履歴を追跡する。

### 6.1 Requirements

- すべての MemoryRecord は `provenance` フィールドを含まなければならない（MUST）。
- すべてのライフサイクル操作（encode、reinforce、consolidate、transform）は `provenance.chain` に追記しなければならない（MUST）。
- Provenance は不変でなければならない（MUST）— エントリの追加は可能だが、変更や削除は不可。
- エージェント間で記憶が共有される際、Provenance は記憶とともに移動しなければならない（MUST）。

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

Agent A から Agent B に記憶が共有される場合：

1. Agent B は元の Provenance チェーンを保持しなければならない（MUST）。
2. Agent B は `operation: "import"` と自身のエージェント ID を持つ新しいエントリを追記しなければならない（MUST）。
3. 記憶の `access` ポリシーは Agent B の Provider によって再評価されなければならない（MUST）。

---

## 7. Strategy System

Strategy System は、異なる記憶の保存・検索アルゴリズムが共存し、組み合わせ可能にする仕組みである。

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

実装は以下の Strategy のうち少なくとも1つを提供すべきである（SHOULD）：

| Strategy | Capabilities | 適したユースケース | バックエンド例 |
|----------|-------------|-------------------|---------------|
| **Vector** | semantic_search, real_time | "X に似た記憶を見つけて" | Qdrant, Pinecone, ChromaDB |
| **Graph** | graph_traversal, structured_filter | "X に関連するものは？" | Neo4j, SQLite + relations |
| **Hierarchical** | temporal_query, structured_filter, aggregation | "今週何があった？" + 階層型ストレージ | SQLite FTS5, カスタム L1/L2/L3 |
| **Buffer** | real_time, structured_filter | セッション中のワーキングメモリ | In-memory, Redis |

### 7.3 Strategy Router

Strategy Router は、クエリの特性に基づいて各操作に最適な Strategy を選択する。

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

デフォルトのルーティングルール：

| クエリ特性 | 選択される Strategy |
|-----------|-------------------|
| `text` が指定されている（セマンティック検索） | Vector |
| `relations` またはグラフ探索が必要 | Graph |
| `timeRange` が指定されている | Hierarchical |
| `types: ['working']` | Buffer |
| 複数の特性が該当 | Composite（7.4 参照） |

### 7.4 Strategy Composition

Strategy は機能を組み合わせるためにコンポーズしてもよい（MAY）：

```typescript
interface CompositeStrategy extends MemoryStrategy {
  /** Sub-strategies in priority order */
  strategies: MemoryStrategy[];

  /** How to merge results from multiple strategies */
  mergePolicy: 'union' | 'intersection' | 'weighted';
}
```

例：`Vector + Graph` コンポジットは、まず Vector でセマンティックに類似した記憶を検索し、次にグラフのリレーションをたどって結果を拡張する。

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

AMP は4段階のスコープモデルを採用する：

```
private → agent → team → public
  (最も制限的)         (最も開放的)
```

| スコープ | 読み取り可能 | 書き込み可能 |
|---------|------------|------------|
| `private` | 作成したエージェントのみ | 作成したエージェントのみ |
| `agent` | 作成したエージェント + `readers` リスト | 作成したエージェント + `writers` リスト |
| `team` | 同じチーム/ワークスペースの全エージェント | 作成したエージェント + `writers` リスト |
| `public` | AMP アクセスを持つすべてのエージェント | 作成したエージェント + `writers` リスト |

### 8.2 PII Protection

個人情報を含む記憶は分類されなければならない（MUST）：

| PIILevel | 説明 | 要件 |
|----------|------|------|
| `none` | 個人データなし | 特別な対応不要 |
| `personal` | 好み、習慣、意見 | 同意なく `agent` スコープを超えて共有してはならない（MUST NOT） |
| `sensitive` | 氏名、メール、所在地、識別子 | 暗号化せずにローカルストレージ外に出してはならない（MUST NOT）。明示的な同意なく `private` スコープを超えて共有してはならない（MUST NOT） |
| `restricted` | 金融、医療、認証情報 | 保存時に暗号化しなければならない（MUST）。共有してはならない（MUST NOT）。ユーザーの要求に応じて削除しなければならない（MUST） |

### 8.3 Enforcement

- Provider はすべての `retrieve()` 呼び出しでアクセスポリシーをチェックしなければならない（MUST）。
- Provider はソース記憶より広いスコープを設定しようとする `encode()` 呼び出しを拒否しなければならない（MUST）（例：`private` ソースを `public` として共有できない）。
- Provider は `sensitive` および `restricted` 記憶へのアクセス試行を監査のためにログに記録すべきである（SHOULD）。

---

## 9. Memory Transform

Transform 操作は記憶をある表現から別の表現に変換する。

### 9.1 Transform Types

| Transform | 入力 | 出力 | ユースケース |
|-----------|------|------|------------|
| **Compress** | 詳細な記憶 | 要約 | トークン使用量の削減 |
| **Expand** | 要約記憶 | 詳細 | Provenance からの詳細復元 |
| **Type Transition** | Episodic | Semantic/Procedural | 経験を知識として結晶化 |
| **Merge** | 複数の記憶 | 単一の統合記憶 | 冗長性の削減 |
| **Format** | 自然言語 | 構造化 JSON | 機械可読な抽出 |

### 9.2 Transform Quality

- Transform は元の記憶の `id` を新しい記憶の `provenance.transformedFrom` に保持しなければならない（MUST）。
- Transform は情報損失の可能性を反映して Confidence をわずかに低下させるべきである（SHOULD）（0.9 を乗算）。
- 実装はインテリジェントな変換（要約、抽出）に LLM を使用してもよい（MAY）。

### 9.3 Automatic Transforms

Provider はバックグラウンドプロセスとして自動 Transform を実装してもよい（MAY）：

| トリガー | Transform | 目的 |
|---------|-----------|------|
| 5つ以上の関連する episodic 記憶 | Consolidate → semantic | 繰り返されるパターンの結晶化 |
| 60日以上経過し、一度も強化されていない記憶 | Compress | ストレージの削減 |
| 同じタスクに対する3つ以上の procedural 記憶 | Consolidate | 統一された手順 |
| セッション終了 | Working → episodic/semantic | セッション中の洞察の保存 |

---

## 10. Agent Interop

### 10.1 Memory Exchange Format

エージェントは標準の MemoryRecord JSON フォーマットを使用して記憶を交換する。交換ペイロードにはトランスポートメタデータが追加される：

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

1. `access.scope` が `agent`、`team`、または `public` の記憶のみ共有可能。
2. `private` の記憶は交換に含めてはならない（MUST NOT）。
3. `pii: 'sensitive'` または `'restricted'` の記憶は、受信者が `readers` リストに含まれていない限り共有してはならない（MUST NOT）。
4. 受信者はインポート時にアクセスポリシーを再評価しなければならない（MUST）。
5. 受信者は元の Provenance チェーンを保持しなければならない（MUST）。

### 10.3 Conflict Resolution

インポートされた記憶が既存の記憶と競合する場合：

| 競合タイプ | 解決戦略 |
|-----------|---------|
| **Contradiction** | 両方を保持し、`contradicts` リレーションを追加。人間のレビュー用にフラグを付ける |
| **Duplicate** | Consolidate で統合。より高い Confidence を採用 |
| **Update** | Provenance が同一ソースを示す場合は更新。異なるソースの場合は両方保持 |

### 10.4 A2A Integration

AMP の記憶は Google A2A プロトコル上で共有できる：

- 記憶は A2A メッセージ内の JSON `Part` オブジェクトとしてシリアライズされる。
- A2A の `TaskArtifact` が `MemoryExchange` ペイロードを運搬する。
- A2A の不透明エージェント原則が尊重される：エージェントは自発的に記憶を共有し、強制的に共有されることはない。

---

## 11. Provider Interface

### 11.1 AMPProvider

すべての AMP 互換バックエンドはこのインターフェースを実装しなければならない（MUST）：

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

### 11.2 Known Providers (参考)

以下のシステムは `AMPProvider` を実装可能である：

| システム | タイプ | Strategy マッピング |
|---------|--------|-------------------|
| **MemU** | MCP Server | Vector（セマンティック検索） |
| **Mem0** | SDK / Hosted | Vector + Graph コンポジット |
| **LangMem** | SDK | Vector + Hierarchical |
| **Letta** | Framework | Hierarchical（MemGPT スタイル） |
| **SQLite + FTS5** | Local DB | Hierarchical + full_text_search |
| **AKARI Memory** | Tauri/Rust | Hierarchical（L1 ファイル / L2 SQLite） |

---

## 12. MCP Compatibility

### 12.1 AMP as MCP Tools

AMP のライフサイクル操作は MCP ツールとして公開できる：

| MCP ツール名 | AMP 操作 | 説明 |
|-------------|---------|------|
| `amp_encode` | `encode()` | 新しい記憶を保存する |
| `amp_retrieve` | `retrieve()` | 記憶を検索する |
| `amp_reinforce` | `reinforce()` | 記憶を強化する |
| `amp_consolidate` | `consolidate()` | 記憶を統合する |
| `amp_transform` | `transform()` | 記憶を変換する |
| `amp_delete` | `delete()` | 記憶を削除する |
| `amp_info` | `info()` | Provider メタデータ |

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

既存の Memory MCP Server（公式 MCP Knowledge Graph Memory Server 等）は AMP Provider としてラップできる：

1. MCP `create_entities` → AMP `encode()`（type: semantic）にマッピング
2. MCP `search_nodes` → AMP `retrieve()`（テキスト検索）にマッピング
3. MCP `create_relations` → AMP relation フィールドにマッピング
4. 既存の操作に Provenance、Confidence、ライフサイクルメタデータを追加

これにより、既存の MCP ベースの記憶システムを壊すことなく段階的な移行が可能になる。

---

## 13. Security

### 13.1 Principles

1. **Local-First**: AMP は完全にローカルでの動作をサポートしなければならない（MUST）。デフォルトではデータはデバイス外に出ない。
2. **Encryption at Rest**: `pii: 'sensitive'` または `'restricted'` の記憶は保存時に暗号化されなければならない（MUST）。
3. **Encryption in Transit**: エージェント間の記憶交換は暗号化されたトランスポート（TLS 1.3 以上）を使用しなければならない（MUST）。
4. **Minimal Exposure**: Provider は要求された以上のデータを返してはならない（MUST NOT）。検索結果はアクセスポリシーを遵守しなければならない（MUST）。
5. **Audit Trail**: Provider は機密記憶へのアクセスの監査ログを維持すべきである（SHOULD）。

### 13.2 Transport Options

M2C のパターンに準拠：

| トポロジ | 説明 | ユースケース |
|---------|------|------------|
| **Local** | Provider が同一マシン上で稼働 | デフォルト。デスクトップアプリ、プライバシー重視のデータ |
| **LAN** | Provider がローカルネットワーク上 | NAS やサーバー上のチーム共有記憶 |
| **Cloud** | Provider がリモートサーバー上 | クロスデバイス同期、チームコラボレーション |

### 13.3 User Control

- ユーザーは自分について保存されたすべての記憶を閲覧できなければならない（MUST）。
- ユーザーは任意の記憶を削除できなければならない（MUST）。
- ユーザーは標準の AMP JSON フォーマットですべての記憶をエクスポートできなければならない（MUST）。
- これらの権利は Provider の実装に関わらず交渉不可である。

---

## 14. Error Handling

### 14.1 Error Model

AMP は MCP および M2C と一貫した JSON-RPC エラーコードを使用する：

| コード | 意味 |
|--------|------|
| `-32600` | 不正なリクエスト |
| `-32601` | 不明な操作 |
| `-32602` | 不正なパラメータ |
| `-32603` | 内部エラー |
| `-32700` | パースエラー |
| `-34001` | 記憶が見つからない |
| `-34002` | アクセス拒否 |
| `-34003` | Confidence が閾値を下回っている |
| `-34004` | Strategy が利用できない |
| `-34005` | 重複した記憶が検出された |
| `-34006` | 統合の競合 |
| `-34007` | Transform に失敗 |
| `-34008` | 交換が拒否された（アクセスポリシー違反） |

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

- プロトコルバージョン：日付ベース（`YYYY-MM-DD`）、M2C と一貫
- スキーマバージョン：同じ日付ベースフォーマットで `MemoryRecord` extensions に記載
- Strategy バージョン：独立した semver
- Provider バージョン：独立した semver
- 後方互換性：コンシューマは不明なフィールドを無視しなければならない（MUST）

### 15.1 Version Negotiation

MCP のパターンに準拠：

1. コンシューマがサポートする AMP バージョンを送信
2. Provider が同じバージョン（サポートしている場合）または最新バージョンで応答
3. 互換性がない場合、接続は拒否される

### 15.2 Extension Principles

1. **Optional**: 拡張はコア機能に必須であってはならない（MUST NOT）
2. **Additive**: 拡張は追加のみ可能で、コアの動作を削除または変更してはならない（MUST NOT）
3. **Composable**: 複数の拡張は独立して動作しなければならない（MUST）
4. **Independently versioned**: 各拡張は独自のバージョンを持つ

---

## Appendix A: Cognitive Science Foundations

AMP の記憶型システムは確立された認知科学モデルに基づいている：

| モデル | AMP への貢献 |
|--------|-------------|
| **Atkinson-Shiffrin**（1968） | ワーキング → 長期記憶のパイプライン。AMP の working 型 + 減衰モデル |
| **Tulving**（1972） | エピソード記憶 vs. 意味記憶の区別。AMP の型システム |
| **Anderson (ACT-R)**（1983） | 手続き記憶とプロダクションルール。AMP の procedural 型 |
| **CoALA**（Sumers et al., 2023） | 言語エージェントのための認知アーキテクチャ。AMP の全体的フレームワーク |
| **Liu et al.**（2025） | 3D分類法（Forms/Functions/Dynamics）。AMP のライフサイクルモデル |

---

## Appendix B: 関連仕様

| 仕様 | 関係 |
|------|------|
| [MCP](https://modelcontextprotocol.io) | AMP Provider は MCP ツールサーバーとして公開できる |
| [M2C](https://github.com/Akari-OS/m2c) | M2C が生成したコンテキストを AMP 記憶として保存できる |
| [A2A](https://a2a-protocol.org/) | AMP 記憶は A2A 上でエージェント間交換できる |
| [Mem0](https://mem0.ai/) | AMPProvider インターフェースを実装可能 |
| [LangMem](https://github.com/langchain-ai/langmem) | AMPProvider インターフェースを実装可能 |

---

## Appendix C: 完全な例 — 記憶ライフサイクル

記憶がライフサイクル全体を経る完全な例を示す。

### Step 1: Encode（エンコード）

ユーザーが「すべてのアプリでダークモードにしたい」と発言。

```json
{
  "id": "mem_001",
  "type": "semantic",
  "content": "ユーザーはすべてのアプリでダークモードを好む",
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

### Step 2: Reinforce（強化）— 5日後

ユーザーが別のコンテキストでダークモードを設定。

```json
{
  "confidence": 0.945,
  "reinforcement": 1,
  "updatedAt": "2026-04-06T14:00:00Z"
}
```

### Step 3: Consolidate（統合）— 10日後

UI 設定に関する3件の関連記憶をひとつに統合。

```json
{
  "id": "mem_010",
  "type": "semantic",
  "content": "ユーザーの UI 設定好み: ダークモード、ミニマル UI、大きなフォント、ハイコントラスト",
  "confidence": 0.95,
  "provenance": {
    "source": { "kind": "consolidation", "sourceIds": ["mem_001", "mem_004", "mem_007"] },
    "consolidatedFrom": ["mem_001", "mem_004", "mem_007"]
  }
}
```

### Step 4: Retrieve（検索）— 30日後

クエリ：「ユーザーの UI 設定の好みは？」

```json
{
  "memories": [{
    "record": { "id": "mem_010", "content": "ユーザーの UI 設定好み: ダークモード、ミニマル UI、大きなフォント、ハイコントラスト", "confidence": 0.95 },
    "relevance": 0.92
  }],
  "totalCount": 1,
  "durationMs": 12
}
```
