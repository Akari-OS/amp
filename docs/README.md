# AMP (Agent Memory Protocol) Documentation

> **このリポの立ち位置**: Agent Memory Protocol の**仕様リポ**。エージェントが長期記憶を扱うためのプロトコル定義。
> **扱う範囲**: spec v0.1（stable）/ v0.2（draft）/ v0.3（draft）、Known Implementations、Provider 規約、サンプル
> **扱わない範囲**: 実装（→ consumer 側の各エージェントリポ）、Pool のスキーマ設計（→ AkariPool）、運用ハブ（→ Hub）
>
> - 🌐 正典: [Akari-OS/.github](https://github.com/Akari-OS)
> - 🏛 Hub（非公開）: `akari-os` — 横断研究・戦略・Master Index
> - 🗺 全リポマップ: `akari-os/MAP.md`

---

## このリポのドキュメント

### spec（プロトコル仕様）

| パス | 内容 |
|---|---|
| [`../spec/v0.1/`](../spec/v0.1/) | v0.1（stable）— 正式リリース版 |
| [`../spec/v0.1/SCHEMAS.md`](../spec/v0.1/SCHEMAS.md) | v0.1 の JSON Schema ファイル一覧（`schema.json` / `error.schema.json` / `mcp-tools.schema.json`）と追記ルール。SDK codegen の入力にも使われる |
| [`../spec/v0.2/`](../spec/v0.2/) | v0.2（draft）— 検討中の拡張 |
| [`../spec/v0.3/protocol.md`](../spec/v0.3/protocol.md) | **AMP v0.3 (delta spec, draft)** — Strategy Router decision tree 正規化（causal / similarity / tag / relations / hybrid の 5 分岐）+ dedupConfig フィールド + Forward Compatibility (v0.4 graphWalk 予告)。ADR-076 normative reference。2026-04-27 draft。**注**: ADR-076 は akari-os Hub リポ内部ドキュメント（本リポからは参照不可）。外部読者は spec §7.2 記載のしきい値のみ参照すれば実装可能。 |

### docs/specs（逆算 wrapper specs）

| ファイル | Spec ID | 内容 |
|---|---|---|
| [`docs/specs/spec-reverse-amp-v0-1.md`](specs/spec-reverse-amp-v0-1.md) | AKARI-AMP-001 | AMP v0.1 逆算 wrapper — Hub (ADR-014 等) からの参照 anchor |

### ルート直下のメタドキュメント

| ファイル | 内容 |
|---|---|
| [`../README.md`](../README.md) / [`../README.ja.md`](../README.ja.md) | プロトコル概要・導入 |
| [`../CONTRIBUTING.md`](../CONTRIBUTING.md) | コントリビューションガイド |
| [`../CHANGELOG.md`](../CHANGELOG.md) | 変更履歴 |
| [`../IMPLEMENTATIONS.md`](../IMPLEMENTATIONS.md) | Known Implementations 一覧 |
| [`../CODE_OF_CONDUCT.md`](../CODE_OF_CONDUCT.md) | 行動規範 |

> **Note**: `examples/` ディレクトリは現在存在しない。サンプル実装は今後追加予定。

---

## 新規ドキュメントの追加

spec 本体の変更は `spec/vX.Y/` 配下で行う。docs/ 配下には横断的な解説・設計判断ログ等を置く。このファイルの index も必ず更新すること。
