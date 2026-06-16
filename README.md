# kintone Customize Skills

AI エージェント（Claude Code / Cursor / その他）に、**正しい kintone カスタマイズを書かせる**ための Agent Skill 群。

kintone の公式コーディング規約・セキュアコーディング・新 API 移行の知見を、
人間向けドキュメントではなく **AI が実行時に読む Skill** の形にまとめたもの。

> AI は「動くコード」を書けても、「kintone として安全・最新の書き方か」までは判断しない。
> その判断基準を Skill として共有するのが狙い。

## 収録 Skill（「書く / 設定する / 配る」の3フェーズ）

| Skill | フェーズ | 役割 |
|---|---|---|
| [`kintone-customize`](skills/kintone-customize/) | ① 書く | JS API / REST(データ) / セキュリティ / REST↔JS 使い分け |
| [`kintone-app-config`](skills/kintone-app-config/) | ② 設定する | fields / layout / settings の API 操作 → `deployApp` |
| [`kintone-build-deploy`](skills/kintone-build-deploy/) | ③ 配る | PluginPacker / customize-uploader |

設計の全体像と分割方針は [ARCHITECTURE.md](ARCHITECTURE.md) を参照。

## インストール

### Claude Code

使いたい Skill を、自分のカスタマイズ開発リポジトリの `.claude/skills/` に置く。

```bash
git clone https://github.com/<owner>/kintone-customize-skills.git
cp -r kintone-customize-skills/skills/kintone-customize  /path/to/your-project/.claude/skills/
```

ユーザー全体で使うなら `~/.claude/skills/` に置く。

### Cursor / 他エージェント

`SKILL.md` はプレーンな Markdown。`.cursor/rules/*.mdc` や `AGENTS.md` にそのまま転用できる
（フロントマター部分は各ツールの形式に合わせる）。

## 前提・スコープ

- **配布物として、特定リポジトリ・特定社内ルールに依存しない。** パッケージマネージャ・ビルドツール・
  ディレクトリ構成は導入先プロジェクトの規約に従う想定で書いている。
- AI 支援を前提にしつつ、**最終的なレビュー・採用判断は人間（エンジニア）が行う**ことを前提とする。

## ライセンス

（未定）
