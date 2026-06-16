---
name: kintone-build-deploy
description: >-
  書き終えた kintone カスタマイズ／プラグインを、ビルド・パッケージング・アップロードするときに使う。
  「プラグインをパッケージングして」「カスタマイズをアップロードして」「配信して」等で発火。
  コードの中身（→ kintone-customize）や、アプリへの適用設定 customize.json/plugins.json
  （→ kintone-app-config）は対象外。このスキルはローカルの成果物づくりと CLI ツールの操作に専念する。
---

# kintone カスタマイズのビルド・配布（v0 叩き台）

書き終えたコードをビルドして kintone に届けるフェーズの手順。

> **境界**：JS/CSS やプラグインを「アプリに**適用**する設定そのもの」は `customize.json` / `plugins.json`
> という設定変更 API で preview→deploy に乗る（→ **kintone-app-config**）。
> このスキルは **ローカルでの成果物づくり（パッケージング・バンドル）** と、
> **配布を代行する CLI ツール** の使い方に専念する。

## 🔴 最重要：開発 CLI は `cli-kintone` に統合された（旧ツールは 2026年8月メンテ終了）

- プラグイン/カスタマイズ開発の操作は **`cli-kintone`** に一本化された。
- 以下の**旧ツールは 2026年8月にメンテナンス終了**（Web版 plugin-packer も終了）：
  `create-kintone-plugin` / `kintone-plugin-packer` / `kintone-plugin-uploader` / `customize-uploader`。
- ⚠️ **AI の学習データには旧ツールのコマンドしか無い可能性が高い。** 知らずに `kintone-plugin-packer ...` 等を
  出力しがちなので、**明示的に `cli-kintone` を使わせる**。コマンド名・オプションは公式ドキュメント
  （https://cli.kintone.dev/ja/）で確認し、うろ覚えで生成しない。

```bash
# Before（旧・2026年8月終了）            →  After（cli-kintone）
create-kintone-plugin plugin_project   →  cli-kintone plugin init --name plugin_project
kintone-plugin-packer src --ppk *.ppk  →  cli-kintone plugin pack --input src/manifest.json --private-key private.ppk
kintone-plugin-uploader ./plugin.zip   →  cli-kintone plugin upload --input plugin.zip
```

## プラグイン

- ひな型作成 → パッケージング → アップロードを `cli-kintone plugin` サブコマンドで行う。
- **秘密鍵（`.ppk`）はリポジトリに含めない**（`.gitignore`）。
  **同じ `.ppk` を使い続ければ plugin ID は不変（＝更新として扱われる）。鍵を変えると別プラグイン扱いになる。**
- `manifest.json` で要求する権限は必要最小限にする。

## JavaScript / CSS カスタマイズ

- アップロードは `cli-kintone`（旧 `customize-uploader` 相当の機能）で行う。
- **⚠️ カスタマイズ適用（`customize.json` の更新）は API トークンでは弾かれる（403 `GAIA_NO01`、実機確認済み）。
  パスワード認証が必要。** cli-kintone / 旧 customize-uploader がパスワード認証を使うのはこのため。
  → 自動化で API トークンしか持たせていない環境では、ここだけログイン名/パスワードか、管理画面での手動アップロードになる。
- 認証情報（接続先 URL・ログイン名/パスワード）は `.env` 等に分離し、**リポジトリに含めない**。
- バンドルする場合は、kintone に登録する**ファイルの読み込み順**に注意（依存するライブラリを先に）。

## 共通

- **本番反映の前に必ず検証環境で動作確認する。**
- `console.log` などのデバッグ出力・ソースマップを本番成果物に残さない。
- **依存パッケージのバージョン・既知脆弱性に注意**（kintone 提供の npm パッケージでも
  axios のサプライチェーン事例があった。`pnpm/npm audit` とバージョン固定で守る）。
- パッケージマネージャ・ビルドツールは導入先プロジェクトの規約に従う（このスキルは前提にしない）。

## 参照（kintone 公式ドキュメント）

- cli-kintone（https://cli.kintone.dev/ja/）— plugin init / pack / upload、カスタマイズの配布、レコード入出力
- プラグインの作成・パッケージング・秘密鍵と plugin ID
- アプリへの適用（customize.json / plugins.json）は → kintone-app-config
