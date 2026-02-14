---
description: エクスポート済みリポジトリから Zenn 記事ドラフトを生成
allowed-tools: Read, Write, Bash, Glob, Grep, AskUserQuestion
argument-hint: [repo-name-or-path]
---

エクスポート済みリポジトリを分析し、Zenn 記事のドラフトを生成する。Zenn の記事管理スキルがあれば、そのルールに従う。

## 定数

以下の定数はユーザーの環境に合わせて変更してください:

- **リポジトリベース**: `~/Projects/`
- **Zenn 記事パス**: `{ZENN_CONTENT_PATH}/articles/` ← Zenn コンテンツリポジトリの articles ディレクトリ（例: `~/Projects/zenn-content/articles/`）
- **Vault ベースパス**: `{VAULT_PATH}` ← Obsidian Vault を使う場合のみ設定
- **Obsidian Zenn 記事パス**: `{VAULT_ZENN_PATH}` ← Vault 内の Zenn 記事保存先（例: `Articles/Zenn/`）

## 処理フロー

### Step 1: 対象リポジトリの特定

`$ARGUMENTS` がある場合、以下の順でリポジトリを探索:
1. `~/Projects/$ARGUMENTS/` が存在するか
2. `~/Projects/Claude-Code-$ARGUMENTS/` 系パターンで存在するか（部分一致）
3. 見つからない場合はユーザーに通知

`$ARGUMENTS` がない場合:
- `~/Projects/` 内のディレクトリをリストし、`.git/` を持つものをフィルタ
- AskUserQuestion で対象を選択

リポジトリパスを `$REPO_PATH`、リポジトリ名を `$REPO_NAME` とする。

### Step 2: リポジトリ分析

以下のファイルを順に読み込み、ツールの全体像を把握する:

1. **`$REPO_PATH/README.md`** — 主要な情報源。概要・使い方・ワークフローを抽出
2. **`$REPO_PATH/commands/*.md`** — コマンドの処理フローと allowed-tools を確認
3. **`$REPO_PATH/agents/*.md`** — エージェントの能力と制約を確認
4. **`$REPO_PATH/skills/*/SKILL.md`** — スキルの仕様を確認
5. **`$REPO_PATH/scripts/*`** — スクリプトの技術的詳細を確認
6. **`$REPO_PATH/install.sh`** — インストール手順を確認
7. **`$REPO_PATH/config.example`** — 設定項目を確認

さらに、対応するローカル版ファイル（`~/.claude/` 内）も参照し、サニタイズ前のオリジナルから、より深い文脈（動機・設計判断・利用実績）を得る。

### Step 3: Slug 生成

Zenn の slug ルールに従い生成:
- 使用可能文字: `a-z`, `0-9`, `-` のみ
- 長さ: 12〜50文字
- リポジトリ名からケバブケースで導出

導出ロジック:
1. `$REPO_NAME` からプレフィックス `Claude-Code-` を除去
2. アッパーケース・パスカルケースをケバブケースに変換（小文字化）
3. `_` を `-` に置換
4. 先頭に `claude-code-` を付与
5. 長さが 12〜50文字の範囲に収まるか確認

例: `Claude-Code-RETRO-Command` → `claude-code-retro-command`

slug をユーザーに提示し、変更があれば受け付ける。

### Step 4: 記事ドラフト生成

以下の構成で記事を生成する。

#### Frontmatter

```yaml
---
title: "{ツールが何をするかの一文}"
emoji: "{ツールの性質に合った絵文字}"
type: "tech"
topics: ["ClaudeCode", "{topic2}", "{topic3}", "{topic4}", "{topic5}"]
published: false
---
```

topics の選定基準:
- 1つ目は必ず `ClaudeCode`
- 残りはツールの技術領域から選定（例: `AI`, `GitHub`, `Zenn`, `Obsidian`, `ShellScript`, `Markdown`）
- 最大5個

#### 本文構成

```markdown
{フック: 1行で何ができるかを簡潔に。読者の興味を引く。}

**リポジトリ**: [GitHub]({リポジトリURL})

## はじめに — 動機と完成形

<!-- TODO: レビュー — 個人的な動機を加筆 -->

{README とローカル版から推測できる動機を記述}
{完成形として何ができるようになったかを記述}

## アーキテクチャ

{README のワークフロー図やステップ説明を転用・図示}

## {技術的ポイント — ツール固有の見出し}

<!-- TODO: レビュー — 技術的に興味深いポイントを深掘り -->

{ソースコード分析から特に興味深い実装ポイントを抽出}
{コードブロックで具体的な実装例を示す}

## 使い方

{README の使い方セクションから抽出・再構成}

## 導入方法

詳しい導入手順は [README]({リポジトリURL}) を参照してください。

{README の導入方法セクションから要約}

## 設計思想

<!-- TODO: 加筆 — なぜこの設計にしたか -->

{README に設計思想セクションがあれば転用、なければスケルトン}

## おわりに

<!-- TODO: 執筆 -->

```

#### 生成ルール

- **自動生成**: frontmatter, フック, リポジトリリンク, 使い方, 導入方法
- **ドラフト + TODO マーカー**: はじめに, アーキテクチャ, 技術セクション, 設計思想
- **スケルトンのみ**: おわりに
- 見出しは `##` から開始（`#` は Zenn がタイトルに使用するため）
- Zenn 記法（`:::message`, `:::details` 等）を適切に活用
- TODO マーカーは HTML コメント `<!-- TODO: ... -->` 形式で記述

### Step 5: デュアル保存

#### 重複チェック

以下の2箇所で既存ファイルとの重複を確認:
1. `{ZENN_CONTENT_PATH}/articles/{slug}.md`
2. `{VAULT_PATH}/{VAULT_ZENN_PATH}/{slug}.md`（Vault 設定時のみ）

いずれかが存在する場合、AskUserQuestion で対処を選択:
1. **上書き** — 既存ファイルを置換
2. **別 slug で保存** — 新しい slug を入力
3. **中断** — 処理を停止

#### ファイル保存

1. `{ZENN_CONTENT_PATH}/articles/{slug}.md` に記事を書き出す
2. `{VAULT_PATH}/{VAULT_ZENN_PATH}/{slug}.md` に同じ内容をコピー（Vault 設定時のみ）

### Step 6: 完了レポート

以下を表示する:

```
## ドラフト生成完了
- Zenn: articles/{slug}.md
- Obsidian: {VAULT_ZENN_PATH}/{slug}.md（設定時のみ）
- 状態: published: false（下書き）
- TODO: {TODO マーカーのあるセクション一覧}

`npx zenn preview` でプレビュー。公開時は `published: true` に変更して push。
```
