# AMP — Agent Memory Protocol

> Standardize how AI agents remember.

## What is AMP?

AMP (Agent Memory Protocol) is an open protocol that standardizes how AI agents store, retrieve, share, and forget memories. Like MCP standardized tool calling, AMP standardizes memory.

It defines:

1. **Types** — Four memory types grounded in cognitive science (episodic, semantic, procedural, working)
2. **Lifecycle** — How memories are created, reinforced, consolidated, decayed, and deleted
3. **Schema** — A standard JSON format for memory records (MemoryRecord)
4. **Strategies** — Pluggable, composable storage/retrieval algorithms
5. **Sharing** — Safe memory exchange between agents

```
Agent → [AMP Provider] → Memory Store
                ↕
        Other Agents (via AMP Interop)
```

## Why?

AI agents today have **amnesia**. Each framework implements memory differently, and nothing is portable or interoperable.

- **Before AMP**: Agent A learns your preferences. Agent B has no idea. Switch frameworks? Start over.
- **After AMP**: Memories are typed, scored, versioned, and portable. Any agent, any backend.

### Where AMP fits

```
MCP  → Standardizes tool calling        (what agents CAN DO)
M2C  → Standardizes media analysis      (what agents UNDERSTAND)
AMP  → Standardizes memory              (what agents REMEMBER)  ← NEW
A2A  → Standardizes agent communication (how agents TALK)
```

No existing standard covers **agent memory**. AMP fills this gap.

## Core Concepts

### Memory Types

| Type | Description | Example |
|------|-------------|---------|
| **Episodic** | Events, conversations, experiences | "User debugged CORS on March 15" |
| **Semantic** | Facts, knowledge, preferences | "User prefers dark mode" |
| **Procedural** | How-to, workflows, patterns | "Deploy: `npm run build && cargo tauri build`" |
| **Working** | Temporary, session-scoped | "Currently editing video #42" |

### Memory Lifecycle

```
Encode → Store → Retrieve → Reinforce → Consolidate → Decay → Delete
                                            ↓
                                        Transform
```

Memories are **living objects** — they strengthen with use, decay without reinforcement, and consolidate into stronger knowledge over time.

### Confidence Scoring

Every memory has a confidence score (0.0–1.0) based on:

- **Source reliability**: User statement (0.9) > Observation (0.8) > Tool result (0.7) > Inference (0.5)
- **Reinforcement**: Each confirmation boosts confidence
- **Recency**: Recent memories score higher
- **Decay**: Unreinforced memories fade over time

### Strategy System

Different tasks need different memory strategies:

| Strategy | Best For | Example Backend |
|----------|---------|-----------------|
| **Vector** | "Find similar memories" | Qdrant, Pinecone |
| **Graph** | "What's related to X?" | Neo4j, SQLite |
| **Hierarchical** | "What happened this week?" | SQLite FTS5 |
| **Buffer** | Working memory | In-memory, Redis |

Strategies are **composable** — combine Vector + Graph for semantic search with relation expansion.

### Access Control

```
private → agent → team → public
```

Built-in PII classification: `none` / `personal` / `sensitive` / `restricted`

### Agent Interop

Agents can share memories with full provenance tracking. Every memory carries its complete history — who created it, when, and through what operations.

## Architecture

```
┌──────────────────────────────────────────────┐
│                 AMP Layer                     │
│                                              │
│  Agent A ──┐                    ┌── Agent B  │
│            │                    │            │
│            ▼                    ▼            │
│  ┌──────────────────────────────────┐       │
│  │         Strategy Router           │       │
│  │  (selects best strategy per query)│       │
│  └──────┬──────┬──────┬──────┬──────┘       │
│         │      │      │      │              │
│     Vector  Graph  Hier.  Buffer            │
│         │      │      │      │              │
│  ┌──────┴──────┴──────┴──────┴──────┐       │
│  │         AMP Provider              │       │
│  │  (MemU, Mem0, LangMem, custom)   │       │
│  └──────────────────────────────────┘       │
│                                              │
│  Memory Exchange ←→ Other AMP Providers      │
└──────────────────────────────────────────────┘
```

## Specification

**Latest:** v0.1 (stable), v0.2-draft (AKARI-OS alignment delta)

```
spec/
├── v0.1/
│   ├── protocol.md      ← Core protocol specification (stable)
│   ├── schema.json      ← MemoryRecord JSON Schema
│   └── ja/
│       └── protocol.md  ← Japanese translation
└── v0.2/
    └── protocol.md      ← Delta spec (draft): Pool interop,
                            7-agent schemas, Shell APIs
```

## Design Principles

| Principle | Description |
|-----------|-------------|
| **Memory First** | Remember before re-discovering |
| **Typed** | Every memory has a cognitive type |
| **Lifecycle-Aware** | Memories strengthen, decay, consolidate |
| **Scored** | Confidence scoring on everything |
| **Local-First** | Fully local operation supported |
| **Interoperable** | Any agent framework, any backend |
| **Privacy-Aware** | Access control and PII classification built-in |

## Known Implementations

See [IMPLEMENTATIONS.md](./IMPLEMENTATIONS.md) for the current list of known implementations and providers.

## License

Apache 2.0

## Links

- [Protocol Spec (v0.1, stable)](spec/v0.1/protocol.md)
- [Protocol Spec (v0.2, draft — delta)](spec/v0.2/protocol.md)
- [JSON Schema](spec/v0.1/schema.json)

### Inspiration

- [MCP (Model Context Protocol)](https://modelcontextprotocol.io)
- [M2C (Media to Context Protocol)](https://github.com/)
- [Mem0 — The Memory Layer for AI](https://mem0.ai/)
- [CoALA — Cognitive Architectures for Language Agents](https://arxiv.org/abs/2309.02427)
- [Memory in the Age of AI Agents (Liu et al., 2025)](https://arxiv.org/abs/2512.13564)
