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
   │ JS API          │ fields / layout /   │ PluginPacker     │
   │ REST(データ)     │   settings 操作      │ customize-       │
   │ セキュリティ      │ → deployApp 必須     │   uploader       │
   │ REST↔JS 使い分け │ preview 環境 / 権限   │ 検証→本番反映     │
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
                               （getFieldElement で DOM 直叩きは非推奨）
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

## ② kintone-app-config — アプリ・フォーム設定を API で操作 ★新設

- **description / 発火**：「アプリにフィールドを追加して / フォームレイアウトを組んで / アプリ設定を一括構築・移行して」
- **担当**：アプリ定義（メタ）の API 操作。アプリ構築・移行タスク
- **非担当（→他Skill）**：レコードの中身（→①の rest-api）、カスタマイズコード（→①）

```
kintone-app-config/
  SKILL.md                  ← preview/deploy フローを最初に必ず読ませる
  references/
    form-fields.md          ← /preview/app/form/fields.json（フィールド定義）
    form-layout.md          ← /preview/app/form/layout.json（レイアウト）
    app-settings.md         ← /preview/app/settings.json ほか各種設定
```

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
  SKILL.md                  ← @kintone/plugin-packer / customize-uploader の手順
                               本番反映前に検証環境で動作確認
```

---

## Skill 間の境界（混同しやすい例）

| 場面 | 正しい担当 | 間違えやすい誤答 |
|---|---|---|
| フィールドの**値**を画面で変える | ① JS API（`record[code].value`） | ② 設定APIを叩く |
| フィールドの**定義**を追加・変更する | ② app-config（fields.json→deploy） | ① JS API でやろうとする |
| 外部 API を呼ぶ（認証あり） | ① rest-api（`kintone.proxy()`） | 直 `fetch` |
| プラグインを zip にする | ③ build-deploy（PluginPacker） | 手動 zip |

---

## 配布形態（検討中）

- 公開リポジトリ（例：`kintone-customize-skills`）に3 Skill を同梱。`git clone` → 各自の `.claude/skills/` へ。
- Cursor 等向けに `.mdc` / `AGENTS.md` 版も併置すると「どのエージェントでも使える」が売りになる。
- developers-site-jp（ドキュメントサイトのリポジトリ）には**コミットしない**。ここは作業場として借りているだけ。

## 今後の棚卸し（賞味期限チェック継続）

- `getSpaceElement()` は今も正攻法 → 「DOM操作は全部ダメ」と読めないようにする
- `getSpaceElement()` 以外の DOM 系の可否を精査
- `User-Agent` / `kintone.proxy()` / `'use strict'`・IIFE は補正済み
