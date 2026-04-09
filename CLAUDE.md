# AMP (Agent Memory Protocol)

> **親**: [Akari-OS エコシステム](https://github.com/Akari-OS/.github) — 全体ビジョン・ロードマップ・行動規範
> **Memory Architecture**: [docs/memory.md](https://github.com/Akari-OS/.github/blob/main/docs/memory.md) — AMP が参照する 4 層記憶モデル

## 概要

AIエージェントの記憶管理を標準化するオープンプロトコル。
MCPがツール呼び出しを標準化したように、AMPは記憶の保存・検索・共有・忘却を標準化する。

Akari-OS エコシステムの **Core 層**（プロトコル仕様）として、M2C（入力側）と
AKARI Desktop / AkariPool（実装先）と連携する。

## 技術スタック

- プロトコル仕様: Markdown + JSON Schema
- 型定義: TypeScript（informative）
- ライセンス: Apache 2.0

## ディレクトリ構成

```
spec/
├── v0.1/
│   ├── protocol.md      ← コアプロトコル仕様
│   ├── schema.json      ← MemoryRecord JSON Schema
│   └── ja/              ← 日本語訳
│       └── protocol.md
examples/                ← 利用例
docs/                    ← 補足ドキュメント
```

## 開発ルール

- 英語が正本、日本語は翻訳
- M2Cと用語・構造の一貫性を保つ
- JSON Schemaが protocol.md の型定義と常に一致すること
