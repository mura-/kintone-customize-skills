---
name: kintone-build-deploy
description: >-
  書き終えた kintone カスタマイズ／プラグインを、パッケージング・アップロード・配信するときに使う。
  「プラグインをパッケージングして」「カスタマイズをアップロードして」「配信して」等で発火。
  コードの中身（→ kintone-customize）やアプリ定義（→ kintone-app-config）は対象外。
---

# kintone カスタマイズのビルド・配布（v0 叩き台）

書き終えたコードを kintone に届けるフェーズの手順。

## プラグイン

- パッケージングは **`@kintone/plugin-packer`**（手動 zip しない）。
- 秘密鍵（`.ppk`）はリポジトリに含めない（`.gitignore`）。配布先で再生成すると plugin ID が変わる点に注意。
- `manifest.json` で要求する権限は必要最小限にする。

## JavaScript / CSS カスタマイズ

- アップロードは **customize-uploader**（`@kintone/customize-uploader`）等を利用。
- 認証情報は `.env` などに分離し、リポジトリに含めない。

## 共通

- **本番反映の前に必ず検証環境で動作確認する。**
- パッケージマネージャ・ビルドツールは導入先プロジェクトの規約に従う（このスキルは前提にしない）。

## 参照（kintone 公式ドキュメント）

- プラグインの作成・パッケージング
- customize-uploader / kintone コマンドラインツール

---
※ TODO: 具体的なコマンド例・CI 連携を追記する。
