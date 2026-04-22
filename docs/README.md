# AMP (Agent Memory Protocol) Documentation

> **このリポの立ち位置**: Agent Memory Protocol の**仕様リポ**。エージェントが長期記憶を扱うためのプロトコル定義。
> **扱う範囲**: spec v0.1（stable）/ v0.2（draft）、Known Implementations、Provider 規約、サンプル
> **扱わない範囲**: 実装（→ consumer 側の各エージェントリポ）、Pool のスキーマ設計（→ AkariPool）、運用ハブ（→ Hub）
>
> - 🌐 正典: [Akari-OS](https://github.com/Akari-OS)
> - 🏛 Hub（非公開）: `akari-os` — 横断研究・戦略・Master Index
> - 🗺 全リポマップ: `akari-os/MAP.md`

---

## このリポのドキュメント

### spec（プロトコル仕様）

| パス | 内容 |
|---|---|
| [`../spec/v0.1/`](../spec/v0.1/) | v0.1（stable）— 正式リリース版 |
| [`../spec/v0.2/`](../spec/v0.2/) | v0.2（draft）— 検討中の拡張 |

### ルート直下のメタドキュメント

| ファイル | 内容 |
|---|---|
| [`../README.md`](../README.md) / [`../README.ja.md`](../README.ja.md) | プロトコル概要・導入 |
| [`../CONTRIBUTING.md`](../CONTRIBUTING.md) | コントリビューションガイド |
| [`../CHANGELOG.md`](../CHANGELOG.md) | 変更履歴 |
| [`../IMPLEMENTATIONS.md`](../IMPLEMENTATIONS.md) | Known Implementations 一覧 |
| [`../CODE_OF_CONDUCT.md`](../CODE_OF_CONDUCT.md) | 行動規範 |
| [`../examples/`](../examples/) | サンプル実装 |

---

## 新規ドキュメントの追加

spec 本体の変更は `spec/vX.Y/` 配下で行う。docs/ 配下には横断的な解説・設計判断ログ等を置く。このファイルの index も必ず更新すること。
