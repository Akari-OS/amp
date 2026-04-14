# Contributing to AMP

Agent Memory Protocol (AMP) への貢献を歓迎します。

## プロジェクト概要

AMP は AI エージェントが記憶を保存・検索・減衰させるための標準プロトコル。

- **v0.1** — stable（`spec/v0.1/protocol.md`）
- **v0.2** — draft, delta spec 形式（`spec/v0.2/protocol.md`）

## 貢献の種類

- spec の改善提案（Issue または PR）
- 実装の追加報告（`IMPLEMENTATIONS.md` への追記 PR）
- 誤字・文法・翻訳改善
- 日本語翻訳（`spec/v0.X/ja/` 配下）

## spec 変更プロセス

### v0.1 (stable) を変更する場合

- 後方互換を保つ軽微な修正のみ（誤字・曖昧さの解消・例の追加）
- 破壊的変更は v0.2+ へ

### v0.2 (draft) を変更する場合

- delta spec として v0.1 からの差分のみ記述
- 完成したら v0.2.0 リリース版として確定

## PR フロー

1. 大きな変更は先に Issue で議論
2. fork & PR
3. review
4. merge

## コミット規約

AKARI エコシステム共通:

- `[仕様]` spec ファイル変更
- `[ドキュメント]` README / CHANGELOG / ガイド
- `[バグ修正]` 誤字・リンク切れ等
- `[リファクタ]` 構成変更

変数・関数名は英語、コメント・メッセージは日本語 OK。

## 関連

- [Code of Conduct](./CODE_OF_CONDUCT.md)
- [CHANGELOG](./CHANGELOG.md)
- [Implementations](./IMPLEMENTATIONS.md)
- [AKARI-OS Vision](https://github.com/Akari-OS)
