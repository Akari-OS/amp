# AMP — Known Implementations

This document lists known AMP implementations and providers.
Moved from v0.1 spec §11.2 to a separate file in v0.2.

## Reference Implementations

Currently none. AMP v0.2 is in draft.
First reference implementation is planned alongside AkariAgents' Memorist agent.

## Community Providers

None yet. Contributions welcome.

## How to Add Your Implementation

Submit a PR adding your implementation to the table below. Include:

- Name / Link
- AMP version supported (v0.1 / v0.2)
- Language / Runtime
- Storage backend (SQLite / Postgres / Redis / etc.)
- License
- Status (alpha / beta / production)

## Planned Implementations

| Name | Language | Planned AMP Version | Notes |
|---|---|---|---|
| AkariAgents (Memorist) | TypeScript | v0.2 | Reference implementation for `preference_bundle` type |
| AkariAgents (Analyst) | TypeScript | v0.2 | Uses `experiment_record` schema |

## Inspiration

- [MemU](https://memu.pro/) — Memory service for AI agents
- [Mem0](https://mem0.ai/) — The Memory Layer for AI
- [LangMem](https://github.com/langchain-ai/langmem) — LangChain's memory tools
