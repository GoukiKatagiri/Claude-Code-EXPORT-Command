---
description: Claude Code の成果物をサニタイズして GitHub リポジトリとして公開
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, AskUserQuestion
argument-hint: [file-paths...]
---

Claude Code の成果物（コマンド・エージェント・スキル・スクリプト）をサニタイズし、GitHub リポジトリとして公開する。

## 定数

以下の定数はユーザーの環境に合わせて変更してください:

- **ソースベース**: `~/.claude/`
- **出力先ベース**: `~/Projects/`
- **GitHub ユーザー/Org**: `{GITHUB_USER}` ← あなたの GitHub ユーザー名に置換
- **Vault ベースパス**: `{VAULT_PATH}` ← Obsidian Vault を使う場合のみ設定（例: `~/Documents/MyVault/`）

## 処理フロー

### Step 1: エクスポート対象の特定

`$ARGUMENTS` があればファイルパスとして解釈する。

なければ `~/.claude/` 配下をスキャンし、AskUserQuestion（multiSelect）で選択:
- `~/.claude/commands/*.md` をリスト
- `~/.claude/agents/*.md` をリスト
- `~/.claude/skills/*/` をリスト
- `~/.claude/scripts/*` をリスト

各項目に既公開チェックを付与する:
- `~/Projects/Claude-Code-*` 内に同名ファイルが存在するものは `(公開済み)` と表示
- 未公開のものは `(未公開)` と表示

### Step 2: 成果物の分類とリポジトリ名決定

ソースパスからタイプとリポジトリ名候補を導出する:

| ソースパス | タイプ | リポジトリ名候補 |
|---|---|---|
| `~/.claude/commands/{name}.md` | command | `Claude-Code-{NAME}-Command` |
| `~/.claude/agents/{name}.md` | agent | `Claude-Code-{NAME}-Agent` |
| `~/.claude/skills/{name}/` | skill | `Claude-Code-{NAME}-Skill` |
| `~/.claude/scripts/{name}.*` | script | `{name}` |
| 複合（command+scripts 等） | bundle | ユーザー指定 |

`{NAME}` は name をアッパーケースに変換したもの（例: `devlog` → `DEVLOG`）。

リポジトリ名候補をユーザーに提示し、変更があれば受け付ける。

### Step 3: 依存関係の解決

各ファイルの内容をスキャンし、以下のパターンで依存を検出する:

- **commands → skills**: `Skill` ツール呼び出し、`~/.claude/skills/` パス参照
- **commands → scripts**: `~/.claude/scripts/` パス参照
- **agents → skills**: frontmatter の `skills:` フィールド
- **scripts → config**: `~/.config/` パス参照
- **commands → 他コマンド**: `/command-name` パターン

依存が見つかった場合、依存ごとに AskUserQuestion で対処を選択:
1. **含める** — リポジトリに同梱する
2. **前提条件として記載** — README に前提条件セクションとして記載
3. **スキップ** — 無視する

### Step 4: サニタイズ

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
| 個人的なフォルダ名 | Obsidian Vault のパス構造として残す場合は注記を付与、不要なら除去 |
| プラットフォーム固有コード（`osascript` 等） | コードは残し `# macOS only` 等のコメントで注記 |

#### セキュリティ（必ず適用）

以下のパターンをスキャンし、検出されたら除去またはプレースホルダーに置換:
- API キー: `sk-`, `key-`, `token-` で始まる文字列
- 環境変数の値: `export.*=` の右辺
- パスワード・シークレットパターン

サニタイズ結果をユーザーに要約表示し、確認を取る。

### Step 5: リポジトリ構成の生成

`~/Projects/{repo-name}/` に成果物を配置する。

#### 複合ツール（bundle）の場合

```
{repo-name}/
├── commands/           # コマンドファイル（ある場合）
├── agents/             # エージェントファイル（ある場合）
├── skills/{name}/      # スキル + references/（ある場合）
│   ├── SKILL.md
│   └── references/
├── scripts/            # スクリプト（ある場合）
├── config.example      # 設定ファイル（ある場合）
├── install.sh          # インストーラ（scripts/config がある場合）
├── .gitignore
├── README.md
└── LICENSE             # MIT
```

#### command 単体の場合（フラット構成）

```
{repo-name}/
├── commands/
│   └── {name}.md
├── README.md
└── LICENSE
```

#### agent 単体の場合

```
{repo-name}/
├── agents/
│   └── {name}.md
├── README.md
└── LICENSE
```

#### skill 単体の場合

```
{repo-name}/
├── skills/{name}/
│   ├── SKILL.md
│   └── references/     # references がある場合
├── README.md
└── LICENSE
```

#### .gitignore 内容

```
.DS_Store
*.swp
*.swo
```

#### LICENSE 内容

MIT License。年は現在年、名義は `{GITHUB_USER}`。

#### install.sh（scripts/config がある場合のみ生成）

```bash
#!/bin/bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"

# コマンドのインストール
if [ -d "$SCRIPT_DIR/commands" ]; then
  mkdir -p ~/.claude/commands
  cp "$SCRIPT_DIR/commands/"*.md ~/.claude/commands/
  echo "Commands installed to ~/.claude/commands/"
fi

# エージェントのインストール
if [ -d "$SCRIPT_DIR/agents" ]; then
  mkdir -p ~/.claude/agents
  cp "$SCRIPT_DIR/agents/"*.md ~/.claude/agents/
  echo "Agents installed to ~/.claude/agents/"
fi

# スキルのインストール
if [ -d "$SCRIPT_DIR/skills" ]; then
  mkdir -p ~/.claude/skills
  cp -r "$SCRIPT_DIR/skills/"* ~/.claude/skills/
  echo "Skills installed to ~/.claude/skills/"
fi

# スクリプトのインストール
if [ -d "$SCRIPT_DIR/scripts" ]; then
  mkdir -p ~/.claude/scripts
  cp "$SCRIPT_DIR/scripts/"* ~/.claude/scripts/
  chmod +x ~/.claude/scripts/*
  echo "Scripts installed to ~/.claude/scripts/"
fi

echo "Done!"
```

### Step 6: README 生成

既存リポジトリのパターンに沿い README.md を生成する。

#### command 単体の README 構成

```markdown
# /{name} - Claude Code {説明} Command

{1段落の説明文。何をするコマンドかを日本語で簡潔に。}
[Claude Code](https://docs.anthropic.com/en/docs/claude-code) のカスタムスラッシュコマンドです。

## 概要

{コマンドが行う処理のステップを番号付きリストで。}

## 導入方法

### A. Claude Code で導入（推奨）

Claude Code に以下のプロンプトを貼るだけで導入できます:

\```
このリポジトリのコマンドを自分の環境に導入してください:
https://github.com/{GITHUB_USER}/{repo-name}

1. commands/ 内のファイルを ~/.claude/commands/ にコピー
2. 導入結果を報告
\```

### B. 手動導入

\```bash
gh repo clone {GITHUB_USER}/{repo-name} /tmp/{repo-name}
mkdir -p ~/.claude/commands
cp /tmp/{repo-name}/commands/*.md ~/.claude/commands/
rm -rf /tmp/{repo-name}
\```

## 使い方

\```
/{name}
\```

{使い方の補足説明}

## 前提条件

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI がインストール済みであること

## License

MIT
```

#### 複合ツール（bundle）の README 構成

上記に加えて以下のセクションを含める:
- **コマンド表** — 含まれるコマンド/エージェント/スキルの一覧テーブル
- **ワークフロー** — ツール間の連携を図示
- **各コマンドの説明** — 個別の詳細セクション
- **設定**（config がある場合）— 設定ファイルの説明
- **仕組み** — 内部実装の概要
- **設計思想** — なぜこのツールを作ったか

#### README 生成時の注意

- 「Claude Code で導入」セクションのプロンプトは、別の人の Claude Code がそれだけで導入を完結できる品質にする
- 手動導入のコマンドは `/tmp/` にクローンしてコピー後に削除するパターン
- install.sh がある場合は手動導入セクションで `bash install.sh` を案内

### Step 7: Git + GitHub

#### 重複チェック

```bash
gh repo view {GITHUB_USER}/{repo-name} 2>/dev/null
```

このコマンドが成功（リポジトリ存在）した場合、ユーザーに通知し以下を選択:
1. 処理を中断
2. 別のリポジトリ名で続行
3. 既存リポジトリを上書き（force push — ユーザーの明示的な同意が必要）

#### Description 生成

README.md の冒頭から説明文を抽出し、GitHub リポジトリの Description として使用する。
タイトル行（`# ...`）の直後にある最初の段落テキストを取得し、350文字以内に収める。

#### リポジトリ作成・公開

```bash
cd ~/Projects/{repo-name}
git init
git add .
git commit -m "Initial release"
gh repo create {GITHUB_USER}/{repo-name} --public --source=. --push --description "{description}"
```

### Step 8: Obsidian プロジェクト登録（オプション）

> このステップは Obsidian Vault を使用している場合のみ実行する。`{VAULT_PATH}` が未設定ならスキップ。

以下の定数を使用:
- **Vault ベースパス**: `{VAULT_PATH}`
- **Projects パス**: `{VAULT_PROJECTS_PATH}` ← Vault 内のプロジェクトフォルダパス（例: `Projects/`）

#### 重複チェック

`{VAULT_PATH}/{VAULT_PROJECTS_PATH}/{repo-name}/` が既に存在するか確認。存在する場合はスキップ。

#### フォルダ・ノート作成

```bash
mkdir -p "{VAULT_PATH}/{VAULT_PROJECTS_PATH}/{repo-name}/"
```

プロジェクトインデックスノートを作成する（Obsidian のノート形式に従う）。

### Step 9: 完了レポート

以下を表示する:

```
## Export 完了
- リポジトリ: ~/Projects/{repo-name}/
- GitHub: https://github.com/{GITHUB_USER}/{repo-name}
- Obsidian: {VAULT_PROJECTS_PATH}/{repo-name}/（設定時のみ）

`/draft {repo-name}` で Zenn 記事ドラフトを生成できます。
```
