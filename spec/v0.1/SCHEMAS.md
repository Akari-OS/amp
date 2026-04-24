# AMP v0.1 — Schema Files

本ディレクトリに同梱される JSON Schema ファイルの一覧。コード生成（codegen）や ランタイム validation から直接参照できるようスタンドアロンで保持する。

| ファイル | 対象 | 出所（spec 本文） |
|---|---|---|
| `schema.json` | `MemoryRecord`（コアメモリレコード、§4.1） | `protocol.md` §4.1 Core Schema |
| `error.schema.json` | エラー応答エンベロープ（§14.2） | `protocol.md` §14.2 Error Response |
| `mcp-tools.schema.json` | 7 MCP ツールの入力スキーマ（§12） | `protocol.md` §12.1 tool 表 + §3.2 operations TypeScript 定義から派生 |

## 追記・更新ルール

- spec 本文の JSON 断片（`protocol.md` 内の ```` ```json ```` ブロック）を更新したら、対応する schema ファイルも同一 PR で更新する
- codegen（`akari-sdk/packages/sdk-types/`）のパイプライン整備は P3 #5 Phase 2（未着手）で実施予定。現時点では schema ファイルは**参照用**に留まり、SDK の手書き TypeScript と drift する可能性がある点に留意
- schema に `$id` を付けているのは将来のリゾルバ統合のため。URL は実在させていないので `$ref` 解決は相対パスで行うこと

## 既知の限界（v0.1 時点）

- `mcp-tools.schema.json` の各 tool definition は spec §3.2 の TypeScript interface から派生させたもので、**`provenance` / `access` / `relations` の詳細型は `MemoryRecord` 本体スキーマを参照する形では完全に繋げていない**（`additionalProperties: true` 扱い）。Phase 2 codegen 導入時に `$ref` で本体スキーマを参照する形に差し替える
- `amp_info` は入力パラメータなしのため空 object 扱い
