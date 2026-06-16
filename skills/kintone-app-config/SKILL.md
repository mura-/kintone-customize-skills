---
name: kintone-app-config
description: >-
  kintone のアプリ設定（フィールド・レイアウト・一覧・プロセス管理・通知・アクセス権・グラフ・
  JS/CSS カスタマイズ適用・プラグイン適用など）を REST API で操作するときに使う。
  アプリの新規構築・設定変更・移行など。
  「アプリにフィールドを追加して」「一覧やプロセス管理を設定して」「アクセス権を設定して」
  「JS/CSS をアプリに適用して」「アプリ設定を一括で構築・移行して」等で発火。
  レコードの中身（データ）の操作や、ブラウザ実行のカスタマイズコードは対象外（→ kintone-customize）。
---

# kintone アプリ設定の API 操作（v0 叩き台）

レコードの中身ではなく、**アプリの定義（設定・メタ）** を API で操作するときの作法。
フィールドだけでなく、一覧・プロセス管理・通知・アクセス権・カスタマイズ適用まで、
**アプリ設定の変更はすべてこの Skill の守備範囲**。

## 🔴 最重要：設定変更はすべて preview → deploy の二段階

- アプリ設定変更系 API は **`/k/v1/preview/...`（テスト環境）にしか効かない**。種類を問わず全部そう。
- **`POST /k/v1/preview/app/deploy.json`（deployApp）を叩くまで運用環境に反映されない。**
- 「`fields.json` に POST したのに反映されない」＝ ほぼ deploy 忘れ。
- **deploy は非同期。** 完了前に次の設定変更を投げると失敗する。`GET .../deploy.json` で `SUCCESS` を待ってから次へ。
- **revision（楽観ロック）に注意。** 直前に取得した revision とズレると失敗する。チェックを省くなら `revision: -1`。
- アプリ管理権限が必要。新規アプリも preview で作成 → 設定を積む → 最後に一度だけ deploy。

## 設定変更 API の全体像（どれも preview→deploy に乗る）

| カテゴリ | エンドポイント（`/k/v1/preview/...`） |
|---|---|
| アプリ作成 | `app.json` |
| フィールド | `app/form/fields.json`（追加 POST / 更新 PUT / 削除 DELETE） |
| フォームレイアウト | `app/form/layout.json`（PUT） |
| 一覧（ビュー） | `app/views.json`（PUT） |
| 一般設定（名前・アイコン等） | `app/settings.json`（PUT） |
| プロセス管理 | `app/status.json`（PUT） |
| グラフ | `app/reports.json`（PUT） |
| アプリアクション | `app/actions.json`（PUT） |
| カテゴリー | `app/categories.json`（PUT） |
| 管理者メモ | `app/adminNotes.json`（PUT） |
| 通知（条件 / レコード単位 / リマインダー） | `app/notifications/general.json` / `perRecord.json` / `reminder.json`（PUT） |
| アクセス権（アプリ / レコード / フィールド） | `app/acl.json` / `record/acl.json` / `field/acl.json`（PUT） |
| **JS/CSS カスタマイズの適用** | `app/customize.json`（PUT） |
| **プラグインの適用 / 設定** | `app/plugins.json` / `app/plugin/config.json`（POST/PUT） |
| 反映 | `app/deploy.json`（POST） |

> 個別 API の引数・レスポンスは公式ドキュメントを参照（ここに写経しない）。
> このスキルの役割は「**どの設定も preview→deploy で、共通の罠がある**」という地図を示すこと。

## ハマりどころ

- **JS/CSS の適用も `customize.json` という"設定変更"。** ファイル自体のアップロードは別途必要だが、
  「どのファイルを使うか」の指定は customize.json で行い、deploy で反映される。
  （ローカルでのビルドや customize-uploader ツールの使い方は → kintone-build-deploy）
- **⚠️ `customize.json` の更新は API トークンでは実行できない（403 `GAIA_NO01`）＝パスワード認証が必要。**
  一方で `fields.json` 追加・`deploy.json`・`file.json` アップロードは API トークンで通る（実機確認済み）。
  「設定変更 API は全部トークンで通る」ではない。**カスタマイズ適用だけはパスワード認証**と覚える。
- **アクセス権（acl）はアプリ / レコード / フィールドの 3 層**。意図せず広い権限を与えていないか必ず確認。
- リネーム・フィールドコード変更は依存（計算式・カスタマイズ・他フィールド参照・通知条件・ビュー）を壊しやすい。
  影響範囲を確認してから。
- 破壊的変更の前にアプリ設定をエクスポート/バックアップしておく。

## 参照（kintone 公式ドキュメント）

- アプリの設定変更系 REST API（フォーム / 一覧 / 設定 / プロセス管理 / 通知 / アクセス権 / グラフ / アクション）
- アプリのカスタマイズ変更 API（customize.json）/ プラグイン追加 API（plugins.json）
- アプリ運用環境への反映（deploy）API

---
※ TODO: 実体験（フィールド一括リネーム/クリーンアップの試行錯誤）からハマりパターンを追記する。
