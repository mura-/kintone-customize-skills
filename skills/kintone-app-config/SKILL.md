---
name: kintone-app-config
description: >-
  kintone のアプリ定義（フィールド・フォームレイアウト・アプリ設定）を REST API で操作するときに使う。
  アプリの新規構築・フィールド追加/変更・移行など。
  「アプリにフィールドを追加して」「フォームレイアウトを組んで」「アプリ設定を一括で構築・移行して」等で発火。
  レコードの中身（データ）の操作や、ブラウザ実行のカスタマイズコードは対象外（→ kintone-customize）。
---

# kintone アプリ・フォーム設定の API 操作（v0 叩き台）

レコードの中身ではなく、**アプリの定義（メタ）** を API で操作するときの作法。

## 🔴 最重要：preview → deploy の二段階を絶対に忘れない

- アプリ設定変更系 API は **`/k/v1/preview/...`（テスト環境）にしか効かない**。
- **`/k/v1/preview/app/deploy.json`（deployApp）を叩くまで、運用環境に反映されない。**
- 「`fields.json` に POST したのに反映されない」＝ ほぼ deploy 忘れ。
- 新規アプリも preview で作成 → deploy。アプリ管理権限が必要。

## 主なエンドポイント

- アプリ新規作成：`POST /k/v1/preview/app.json`（preview に作られる。最後に deploy で運用化）
- フィールド定義：追加 `POST` / 更新 `PUT` / 削除 `DELETE` `/k/v1/preview/app/form/fields.json`
- フォームレイアウト：`PUT /k/v1/preview/app/form/layout.json`
- 一般設定など：`PUT /k/v1/preview/app/settings.json` ほか
- 反映：`POST /k/v1/preview/app/deploy.json` → ステータスは `GET /k/v1/preview/app/deploy.json` で確認

## 注意

- 変更を積んでから最後に一度 deploy。フィールド単位で deploy しない。
- **deploy は非同期。** 完了前に次の設定変更を投げると失敗する。`GET .../deploy.json` で `SUCCESS` を待ってから次へ。
- **revision（楽観ロック）に注意。** 設定変更系 API は直前に取得した revision とズレると失敗する。
  チェックを省くなら `revision: -1` を渡す。連続変更時の競合に気をつける。
- リネーム・コード変更は依存（計算式・カスタマイズ・他フィールド参照）を壊しやすい。影響範囲を確認してから。
- 破壊的変更の前にアプリ設定をエクスポート/バックアップしておく。

## 参照（kintone 公式ドキュメント）

- フォームのフィールド／レイアウトの取得・変更 API
- アプリの設定変更 API
- アプリ運用環境への反映（deploy）API

---
※ TODO: 実体験（フィールド一括リネーム/クリーンアップの試行錯誤）からハマりパターンを追記する。
