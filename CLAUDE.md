# AMP (Agent Memory Protocol)

## 概要

AIエージェントの記憶管理を標準化するオープンプロトコル。
MCPがツール呼び出しを標準化したように、AMPは記憶の保存・検索・共有・忘却を標準化する。

## 関連プロジェクト

| プロジェクト | パス | 関係 |
|------------|------|------|
| **M2C** | `~/_project/PJ26c19_M2C` | メディア→コンテキスト変換プロトコル（AMPの入力側） |
| **AKARI Desktop** | `~/_project/PJ26c12_IndieBase` | AMP最初の実装先（MSP → AMP） |

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
