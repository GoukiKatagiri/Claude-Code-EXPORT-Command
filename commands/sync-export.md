---
description: ローカル版コマンドをエクスポートリポに同期・コミット・プッシュ
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, AskUserQuestion
argument-hint: [command-name...]
---

ローカル版コマンド（`~/.claude/commands/`）をエクスポートリポジトリ（`~/Projects/`）に同期する。frontmatter の `export-repo` フィールドを持つコマンドが対象。

## 定数

- **ローカルベース**: `~/.claude/commands/`
- **リポベース**: `~/Projects/`

## Step 1: 同期対象の特定

### 引数あり（$ARGUMENTS）

引数をコマンド名として解釈し、`~/.claude/commands/{name}.md` を検索する。`.md` 拡張子は省略可。`export-repo` フィールドがなければエラー報告して終了。

### 引数なし

`~/.claude/commands/*.md` をスキャンし、frontmatter に `export-repo` フィールドを持つファイルをすべて検出する。

検出結果をリポジトリ別にグループ化してテーブル表示:

```
| リポ | コマンド | エクスポートパス |
|------|---------|----------------|
| Claude-Code-RETRO-Command | retro.md | retro.md |
| Claude-Code-EXPORT-Command | export.md | commands/export.md |
| Claude-Code-EXPORT-Command | draft.md | commands/draft.md |
| Claude-Code-MIGRATE-Command | migrate.md | commands/migrate.md |
| ... | ... | ... |
```

## Step 2: サニタイズ

各対象ファイルに以下のサニタイズを適用し、エクスポート版コンテンツを生成する。

### 2a. CBN frontmatter フィールドの除去

以下のフィールドを frontmatter から除去する（`description`, `allowed-tools`, `argument-hint` は残す）:

- `id`
- `type`
- `domain`
- `invokes`
- `aliases`
- `purpose`
- `collaborators`
- `safety`
- `export-repo`
- `export-path`
- `export-files`

### 2b. コンテンツのサニタイズ

全ファイルに以下のルールを適用する。

#### パス系（必ず適用）

| 検出パターン | 置換 |
|---|---|
| `$HOME` の絶対パス（例: `/Users/alice/`） | `$HOME/` または動的検出コードに置換 |
| Obsidian Vault 絶対パス | `{VAULT_PATH}` プレースホルダー + README に設定方法を記載 |
| `~/.claude/projects/` 配下のプロジェクト固有パス | 動的導出コメントに置換 |
| Zenn コンテンツの絶対パス | `{ZENN_CONTENT_PATH}` |
| ユーザー名のリテラル | `yourname` または `alice` に置換 |

#### 参照系（文脈に応じて適用）

| 検出パターン | 処理 |
|---|---|
| 未公開コマンドへの参照（`/command-name`） | 汎用表現に置換（例: 「関連コマンドで処理」） |
| 個人的なフォルダ名 | 不要なら除去 |

#### セキュリティ（必ず適用）

以下のパターンをスキャンし、検出されたら除去またはプレースホルダーに置換:
- API キー: `sk-`, `key-`, `token-` で始まる文字列
- 環境変数の値: `export.*=` の右辺
- パスワード・シークレットパターン

### 2c. CBN 参照の除去

本文中の `[[...]]` wikilink 記法を、リンク先の表示名のみに置換する:

- `[[agent.qa-reviewer]]` → `qa-reviewer`
- `[[skill.dev-workflow]]` → `dev-workflow`
- `[[command.handoff]]` → `/handoff`（コマンド参照は `/` 付きに変換）

## Step 3: 差分チェック

各対象について:

1. Step 2 で生成したサニタイズ済みコンテンツ
2. `~/Projects/{export-repo}/{export-path}` のエクスポート版を Read

両者を比較し、差分の有無を判定する。

差分がないファイルは「同期済み」と表示。差分があるファイルのみ同期対象としてリスト表示し、主要な変更点（行数の増減、追加/削除されたセクション等）を要約する。

差分がゼロの場合は「すべて同期済みです」と報告して終了。

## Step 4: ユーザー確認

```
AskUserQuestion:
  question: "以下のファイルをエクスポートリポに同期しますか？"
  options:
    - label: "すべて同期"
      description: "{N} ファイルを同期し、各リポにコミット&プッシュ"
    - label: "同期のみ（push なし）"
      description: "ファイルを書き出しコミットするがプッシュはしない"
    - label: "中断"
      description: "何もしない"
  multiSelect: false
```

## Step 5: 書き出し

各対象ファイルのサニタイズ済みコンテンツを `~/Projects/{export-repo}/{export-path}` に Write する。

## Step 6: Git 操作

リポごとにまとめて実行:

```bash
cd ~/Projects/{export-repo}
git add -A
git diff --cached --stat
```

変更がある場合のみコミット:

```bash
git commit -m "Sync: {変更ファイル名のカンマ区切り}"
```

「すべて同期」選択時は push も実行:

```bash
git push
```

## Step 7: 完了レポート

```
## /sync-export 完了

| リポ | 同期ファイル | コミット | プッシュ |
|------|-----------|--------|--------|
| {repo} | {files} | {✓/−} | {✓/−} |

次回の同期: `/sync-export`
```
