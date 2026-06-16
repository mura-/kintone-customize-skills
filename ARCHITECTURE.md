# kintone カスタマイズ Skill 群 — 設計図（v0）

> AI（Claude Code / Cursor / その他エージェント）に、**正しい kintone カスタマイズを書かせる**ための Skill 群。
> 各エバ・開発者が自分の開発リポジトリに入れて使う**配布物**。特定リポジトリ・特定社内ルールに依存しない。

---

## 設計原則

1. **分割は "発火単位" で決める** — 内容のジャンルではなく「同じ場面で同時に必要になるか」で割る。
   同時に要るものは1つの Skill に。場面（トリガー）が違うものは別 Skill に。
2. **肥大化は "ファイル分割" で解く（Progressive Disclosure）** — SKILL.md 本体は薄い索引＋判断フローだけ。
   詳細は `references/*.md` に逃がし、Claude が必要なときだけ読みに行く。Skill を増やさず内容を増やせる。
3. **"使い分け" は割らない** — 複数 API を俯瞰しないと判断できない知識（REST↔JS API の選択など）は、
   分割すると判断ロジックが破綻する。索引（SKILL.md 本体）の最上段に置く。
4. **配布先に依存しない** — パッケージマネージャ・ビルドツール・ディレクトリ構成は導入先の規約に従う。
   Skill 側で前提にしない。

---

## 全体像 — 「書く / 設定する / 配る」の3フェーズ

```
                    kintone 開発ライフサイクル
   ┌─────────────────┬─────────────────────┬──────────────────┐
   │   ① 書く         │   ② 設定する          │   ③ 配る          │
   │  (カスタマイズ)   │  (アプリ定義)         │  (ビルド/配信)    │
   ├─────────────────┼─────────────────────┼──────────────────┤
   │ kintone-        │ kintone-            │ kintone-         │
   │   customize     │   app-config        │   build-deploy   │
   ├─────────────────┼─────────────────────┼──────────────────┤
   │ JS API          │ 設定変更API全般      │ cli-kintone      │
   │ REST(データ)     │ form/一覧/権限/通知   │ (plugin pack/    │
   │ セキュリティ      │ 適用(customize/plugin)│  upload)         │
   │ REST↔JS 使い分け │ → preview→deploy     │ 検証→本番反映     │
   └─────────────────┴─────────────────────┴──────────────────┘
        書くとき同時に効く        書く場面とは別トリガー    書き終えた後のフェーズ
```

---

## ① kintone-customize — 書く・レビューする

- **description / 発火**：「kintone のカスタマイズを書いて / プラグインを作って / このカスタマイズをレビューして」
- **担当**：ブラウザ実行のカスタマイズコードの作法とレビュー
- **非担当（→他Skill）**：アプリ定義の変更（→②）、パッケージング/配信（→③）

```
kintone-customize/
  SKILL.md                  ← 薄い索引。冒頭に「REST or JS APIどっち？」判断フロー（割らない知識）
  references/
    javascript-api.md       ← setFieldShown / setFieldStyle / イベント / getSpaceElement
                               （スタイルは setFieldStyle[2026/2 追加の新API]優先。
                                 getFieldElement/getSpaceElement 自体は非推奨ではない）
    rest-api.md             ← レコード操作。一括処理 / cursor / upsert / 並列更新NG
                               外部API: kintone.proxy()（認証情報を伴うとき一択）
                               User-Agent はフロントでは設定不可（Node外部連携時のみ）
    security.md             ← XSS/CSSインジェクション / 認証情報をコードに残さない
                               権限境界はアクセス権で守る / ハルシネーションAPIの検証
```

### 索引最上段に置く「使い分け」判断フロー
| やりたいこと | API |
|---|---|
| 画面の表示制御・イベント・今開いてるレコードの操作 | **JavaScript API**（`kintone.app.record.*`） |
| データの CRUD・アプリ設定の取得 | **REST API** |
| フロントから REST を叩く | `kintone.api()` |
| 外部 / Node から叩く | `@kintone/rest-api-client` |

---

## ② kintone-app-config — アプリ設定を API で操作

- **description / 発火**：「アプリにフィールドを追加して / 一覧やプロセス管理を設定して / アクセス権を設定して /
  JS・CSS をアプリに適用して / アプリ設定を一括構築・移行して」
- **担当**：**preview→deploy に乗る設定変更 API 全般**。フォームだけでなく、一覧・プロセス管理・通知・
  アクセス権・グラフ・**カスタマイズ適用（customize.json）・プラグイン適用（plugins.json）** まで含む。
- **非担当（→他Skill）**：レコードの中身（→①の rest-api）、カスタマイズコードの中身（→①）、
  ローカルのビルド/パッケージング・customize-uploader ツール（→③）

```
kintone-app-config/
  SKILL.md                  ← preview/deploy フロー＋設定変更API全体マップを最初に読ませる
  references/
    form.md                 ← form/fields.json・layout.json（フィールド/レイアウト）
    views-process.md        ← views.json（一覧）/ status.json（プロセス管理）
    notifications-acl.md    ← notifications/*・acl（アプリ/レコード/フィールドの3層）
    customize-plugins.md    ← customize.json（JS/CSS適用）/ plugins.json（プラグイン適用）
```

> 個別 API の引数は公式ドキュメントへ（写経しない）。Skill は「どの設定も preview→deploy・共通の罠」の地図に徹する。

### 🔴 最重要・最頻ハマりポイント
- 設定変更系 API は **`/preview/`（テスト環境）にしか効かない**。
- **`deployApp()`（`/preview/app/deploy.json`）を叩くまで運用環境に反映されない。** AI はこれをほぼ必ず忘れる。
- 「fields.json に POST したのに反映されない」＝ deploy 忘れ。索引の冒頭で強制的に意識させる。
- アプリ管理権限が必要。新規アプリも preview で作って deploy。

---

## ③ kintone-build-deploy — パッケージング・配布

- **description / 発火**：「プラグインをパッケージングして / カスタマイズをアップロードして / 配信して」
- **担当**：書き終えたコードのビルド・配信（手順書的）
- **非担当（→他Skill）**：コードの中身（→①）、アプリ定義（→②）

```
kintone-build-deploy/
  SKILL.md                  ← cli-kintone（plugin init/pack/upload・カスタマイズ配布）の手順
                               ※旧ツール(plugin-packer/customize-uploader)は2026年8月メンテ終了
                               .ppk と plugin ID / 検証→本番反映 / 依存脆弱性
```

---

## Skill 間の境界（混同しやすい例）

| 場面 | 正しい担当 | 間違えやすい誤答 |
|---|---|---|
| フィールドの**値**を画面で変える | ① JS API（`record[code].value`） | ② 設定APIを叩く |
| フィールドの**定義**を追加・変更する | ② app-config（fields.json→deploy） | ① JS API でやろうとする |
| 外部 API を呼ぶ（認証あり） | ① rest-api（`kintone.proxy()`） | 直 `fetch` |
| JS/CSS を**アプリに適用する設定** | ② app-config（`customize.json`→deploy） | ③ の話だと思い込む |
| JS/CSS を**ビルド/アップロードする操作** | ③ build-deploy（`cli-kintone`） | 旧 customize-uploader / 手で customize.json |
| プラグインを zip にする | ③ build-deploy（`cli-kintone plugin pack`） | 旧 plugin-packer / 手動 zip |

---

## 配布形態（検討中）

- 公開リポジトリ（例：`kintone-customize-skills`）に3 Skill を同梱。`git clone` → 各自の `.claude/skills/` へ。
- Cursor 等向けに `.mdc` / `AGENTS.md` 版も併置すると「どのエージェントでも使える」が売りになる。
- developers-site-jp（ドキュメントサイトのリポジトリ）には**コミットしない**。ここは作業場として借りているだけ。

## 棚卸しログ（賞味期限チェック）

補正済み：
- `'use strict'` / IIFE … ES Module 評価なら不要、素の .js 直アップ時のみ、に補正
- `User-Agent` … フロントでは禁止ヘッダーで設定不可、Node 外部連携時のみ、に補正
- `kintone.proxy()` … 「必ず」ではなく「認証情報を伴う/CORS を越えるとき」に補正
- `getFieldElement()` / `getSpaceElement()` … 非推奨ではない（要素取得の正当な API）。
  スタイル設定だけ `setFieldStyle()`（2026年2月追加の新 API）を優先、に補正
- 開発 CLI … `plugin-packer` / `customize-uploader` 等は 2026年8月メンテ終了。`cli-kintone` に統合、に補正

> 「AI の学習データは 2026年の変化に追いついていない」共通パターン：
> ① `setFieldStyle`（2026/2 追加）、③ `cli-kintone`（2026/8 統合）。新しい API/ツールほど Skill で明示する価値が高い。

実機検証で追加・実証（Sandbox app=128 / setFieldStyle 条件付き書式を E2E で実行）：
- ① イベントモデル（`events.on`・`change.<コード>`・ハンドラは event を返す）を追加
- ① フィールド型ごとの値構造を追加（**NUMBER の value は文字列** 等を実機で確認）
- ② 設定変更の権限差を実証：`fields.json`追加 / `deploy` / `file.json` は **API トークンで通る**
- ②③ `customize.json`（カスタマイズ適用）は **API トークン不可(403)・パスワード認証必須**を実証
- ③ パスワードはコードに書かず**環境変数で渡して実行**するパターンを実践（①の認証情報の作法）

> Skill を使う → 実機で検証 → 制約を Skill に還元、のループが回った。これ自体が「Skill の育て方」。

継続ウォッチ：
- 新しく追加された JS API・CLI・ツール（AI の学習データに無いもの）を随時取り込む
