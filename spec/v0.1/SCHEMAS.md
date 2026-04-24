# AMP v0.1 — Schema Files

本ディレクトリに同梱される JSON Schema ファイルの一覧。コード生成（codegen）や ランタイム validation から直接参照できるようスタンドアロンで保持する。

| ファイル | 対象 | 出所（spec 本文） |
|---|---|---|
| `schema.json` | `MemoryRecord`（コアメモリレコード、§4.1） | `protocol.md` §4.1 Core Schema |
| `error.schema.json` | エラー応答エンベロープ（§14.2） | `protocol.md` §14.2 Error Response |
| `mcp-tools.schema.json` | 7 MCP ツールの入力スキーマ（§12） | `protocol.md` §12.1 tool 表 + §3.2 operations TypeScript 定義から派生 |

## 追記・更新ルール

- spec 本文の JSON 断片（`protocol.md` 内の ```` ```json ```` ブロック）を更新したら、対応する schema ファイルも同一 PR で更新する
- codegen（`akari-sdk/packages/sdk-types/`）のパイプライン整備は ✅ **2026-04-24 に Phase 2+3 完了**（akari-sdk `3f5e1ee` / `7555e29`）。上流 spec の `.schema.json` は akari-sdk 側 `schemas/amp/v0.1/` に vendored copy として取り込まれ、`pnpm codegen` で `packages/sdk-types/src/generated/amp-v0-1.ts` に反映される。同期手順は `akari-sdk/docs/codegen.md` を参照
- schema の `$id` は codegen 時の cross-schema `$ref` 解決に使われる。URL 自体は実在しないが quicktype-core が `InputData` 内で名前解決する

## 既知の限界（v0.1 時点）

- `mcp-tools.schema.json` の各 tool definition は spec §3.2 の TypeScript interface から派生させたもの。**`provenanceInput` / `accessPolicy` / `relationInput` は 2026-04-24 に MemoryRecord 本体スキーマの `$defs.Provenance` / `.AccessPolicy` / `.Relation` に `$ref` 接続済み**（akari-amp `de734ce`、codegen 出力で具体型にバインドされることを確認）
- `amp_info` は入力パラメータなしのため空 object 扱い
