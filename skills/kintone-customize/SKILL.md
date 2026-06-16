---
name: kintone-customize
description: >-
  kintone の JavaScript カスタマイズ・プラグイン・REST API 連携のコードを書く / レビュー /
  リファクタするときに使う。AI が出しがちな古い書き方・非推奨 API・セキュリティ事故を防ぎ、
  kintone 公式のコーディング規約とセキュアコーディングに沿ったコードを書かせる。
  「kintone のカスタマイズを書いて」「プラグインを作って」「このカスタマイズをレビューして」等で発火。
---

# kintone カスタマイズ作法 Skill（v0 叩き台）

kintone のフロントエンド JS カスタマイズ・プラグイン・REST API 連携を実装するときに守るルール集。
AI が生成しがちなコードには、Web 開発としては自然でも **kintone では非推奨・危険** なパターンが多い。
このスキルが有効なときは、以下を必ず守ること。

## 必ず守る作法（DO）

### API 呼び出し
- フロント（ブラウザ実行）からの REST API は **`kintone.api()`** か **`@kintone/rest-api-client`** を使う。
  - rest-api-client を使うなら URL 組み立ては不要。素の `kintone.api()` を使う場合のみ、URL は文字列結合せず **`kintone.api.url()`** で組み立てる。
- 外部 API を呼ぶとき：
  - **認証情報を伴うなら `kintone.proxy()` 一択**（認証情報をフロントのコードに残さない＋クロスドメイン制約を回避）。
  - CORS 許可済みの公開 API なら直 `fetch` でも可。「必ず proxy」ではなく、判断基準は「認証情報を晒すか／CORS を越えるか」。
  - プラグインなら認証情報をサーバー側に保管できる **`kintone.plugin.app.proxy()`**（+ `setProxyConfig()`）を優先。
- **`User-Agent` ヘッダーは設定できる環境でだけ設定する。**
  - ブラウザ実行の `kintone.api()` / `fetch` では `User-Agent` は禁止ヘッダーで、JS から指定しても**無視される**（＝フロントでは設定不可）。
  - **Node.js など外部から `@kintone/rest-api-client` で叩くバッチ/連携のときだけ**、リクエスト元を識別できるよう設定する。
- 一括処理を優先：複数件まとめて処理するエンドポイント・`cursor` API・`upsert` モードを使う。大量の単発リクエストを投げない。
- 同一レコードへの **並列更新をしない**（競合の原因）。直列実行か `upsert` を使う。

### 表示制御・スタイル変更
- フィールドの表示/非表示は `kintone.app.record.setFieldShown()`。
- フィールドの**スタイル設定**は **`kintone.app.record.setFieldStyle()` / `getFieldStyle()`** を使う。
  - ⚠️ **これは 2026年2月追加の新しい API（モバイルは 3月）。** AI の学習データに含まれていない可能性が高く、
    知らずに「`getFieldElement()` で要素を取って CSS を直接当てる」古い書き方をしがち。**明示的に setFieldStyle を使わせる。**
- `getFieldElement()` で取得した要素に **CSS を直接当てる / DOM を書き換えるのは避ける**（setFieldStyle で代替できる範囲では使わない）。
- ただし **`getFieldElement()` / `getSpaceElement()` 自体は要素取得の正当な API で、非推奨ではない。**
  setFieldStyle で扱えない独自 UI の配置・操作では使ってよい。
- `id` / `class` 属性に依存した DOM 操作はしない（kintone の内部 DOM 構造は変わりうる）。

### アプリ設定の取得
- アプリのフィールド・レイアウト情報は **`fields.json` / `layout.json`**（フォーム設定取得 API）を使う。
- **`form.json`（旧フォーム設定取得 API）は非推奨。** 新 API に寄せる。

### 実装スタイル
- 判断の本質は **「コードが ES Module として評価されるか」**（ビルドツールの有無は関係ない）。
  ES Module はデフォルトで strict mode かつモジュールスコープなので、`'use strict';` も IIFE も **不要**。
  - バンドラ（Vite 等）でバンドルした成果物 → ES Module 扱い → 不要。
  - `<script type="module">` で読み込む → 不要。
- **注意：kintone の「JavaScript ファイル」に上げた素の `.js` は、クラシックスクリプトとして読み込まれる**（`type="module"` にならない）。
  この場合だけ sloppy mode・グローバル汚染が起きるので、IIFE でラップする。
- いずれの場合も **グローバルスコープを汚染しない**：
  - 暗黙のグローバル変数（`globalVariable = 1;`）を作らない。
  - `cybozu.foo = 'bar';` のように既存グローバルを書き換えない。
- ユーザー識別には **ログイン名ではなくユーザーID** を使う。

### 命名規約（プロジェクトで揃える）
- CSS クラスは衝突防止のため **プレフィックス**（例: `mk-`）を付ける。
- フィールドコードは命名パターンを統一（例: `task_title`, `task_status`）。

## 絶対にやらないこと（DON'T / アンチパターン）

- **`cybozu.` で始まる非公開 API を使わない。** 公式ドキュメントに載っている API だけを使う。
- **APIトークン・パスワード・秘密鍵をコードにハードコードしない。** ローカルストレージにも保存しない。
- `innerHTML` で外部由来の文字列を挿入しない（XSS / CSSインジェクション）。エスケープする。
- **JavaScript の `if` 文だけで権限制御を完結させない。** ブラウザの開発者ツールで書き換えられる。
  画面制御は「使いやすさ」のためと位置付け、統制の境界は kintone のアクセス権設定 / APIトークン権限で守る。
- **存在しない API をでっち上げない（ハルシネーション）。** API 名・引数・戻り値の型が公式ドキュメントと一致しているか確認する。
  似た別 API に当たると、動いて見えても挙動が違う。

## 認証情報の扱い

- 認証情報は **プレースホルダ**（`const apiKey = 'YOUR_API_KEY';`）で生成し、人間が実値に差し替える前提で書く。
- バンドル（Vite 等）するなら `.env` に切り出し、ビルド時に環境変数として注入（`.env` は Git 管理外）。
- より強固に守るならプラグイン化し、`setProxyConfig()` で認証情報をサーバー側に保管する。

## 提出前チェックリスト（生成後に必ず通す）

- [ ] APIトークン・秘密情報・署名処理をフロントエンドに露出していない
- [ ] `innerHTML` を使っていない / 外部文字列をエスケープしている
- [ ] DOM 操作を `id` / `class` に依存していない（公式 API を使っている）
- [ ] 認証情報を伴う外部 API を `fetch` 直叩きせず `kintone.proxy()` 経由にしている（公開・CORS 許可 API は任意）
- [ ] グローバルスコープを汚染していない（ES Module 評価なら自動。素の .js 直アップ＝クラシック読み込み時のみ IIFE）
- [ ] `console.log` のデバッグ出力・秘密情報のログが残っていない
- [ ] 使っている API がすべて公式ドキュメントに存在する
- [ ] 依存ライブラリに既知の脆弱性がない（`pnpm audit`）
- [ ] 異常系（入力エラー・通信エラー・権限エラー）を考慮している
- [ ] 本番反映前に検証環境で動作確認している

## 開発環境メモ

> パッケージマネージャ・ビルドツール・アップロード手段は **導入先プロジェクトの規約に従う**。
> このスキルは特定のツールチェーンを前提にしない。以下は代表例。

- REST API クライアント（Node / 外部から叩く場合）：`@kintone/rest-api-client`
- ビルド（任意。使う場合の例）：Vite ほか。バンドルすれば成果物は ES Module 扱いになる。
- アップロード（例）：kintone customize-uploader など。
- 本番反映前に必ず検証環境で動作確認する。

## 参照（kintone 公式ドキュメント）

- kintone セキュアコーディングガイドライン
- kintone コーディングガイドライン
- kintone JavaScript API（setFieldShown / setFieldStyle ほか）
- kintone.proxy() / kintone.api.url()
- プラグインから外部APIを実行する（setProxyConfig / plugin.app.proxy）
- OWASP Top 10

---
※ 知識源：kintone 公式のセキュアコーディングガイドライン・コーディングガイドライン、
および生成AIでkintoneカスタマイズを書く際のベストプラクティス（スタイル設定は setFieldStyle を優先 /
フォーム設定は form.json ではなく fields.json・layout.json を使う 等の新しめの作法を含む）。
導入先の事情に依存しない、汎用の作法のみをまとめている。
