# Claude Code Export Commands

Claude Code で作ったカスタムコマンド・エージェント・スキルを、サニタイズして GitHub リポジトリとして公開し、さらに Zenn 記事のドラフトまで自動生成する [Claude Code](https://docs.anthropic.com/en/docs/claude-code) のカスタムスラッシュコマンドです。

| コマンド | 用途 |
|---|---|
| `/export` | 成果物をサニタイズして GitHub リポジトリとして公開 |
| `/draft` | エクスポート済みリポジトリから Zenn 記事ドラフトを生成 |

## ワークフロー

```
成果物を作る → /export → サニタイズ → GitHub 公開
                            ↓
                         /draft → Zenn 記事ドラフト生成 → レビュー → 公開
```

## 導入方法

### A. Claude Code で導入（推奨）

Claude Code に以下のプロンプトを貼るだけで導入できます:

```
このリポジトリのコマンドを自分の環境に導入してください:
https://github.com/GoukiKatagiri/Claude-Code-EXPORT-Command

1. commands/ 内の2ファイル（export.md, draft.md）を ~/.claude/commands/ にコピー
2. 各ファイル内の {GITHUB_USER} を私の GitHub ユーザー名に置換
3. Zenn を使う場合は {ZENN_CONTENT_PATH} を Zenn コンテンツリポジトリのパスに置換
4. Obsidian を使う場合は {VAULT_PATH}, {VAULT_PROJECTS_PATH}, {VAULT_ZENN_PATH} を設定
5. 導入結果を報告
```

### B. 手動導入

```bash
# 1. リポジトリをクローン
gh repo clone GoukiKatagiri/Claude-Code-EXPORT-Command /tmp/Claude-Code-EXPORT-Command

# 2. コマンドファイルをコピー
mkdir -p ~/.claude/commands
cp /tmp/Claude-Code-EXPORT-Command/commands/*.md ~/.claude/commands/

# 3. クリーンアップ
rm -rf /tmp/Claude-Code-EXPORT-Command
```

導入後、各コマンドファイル内のプレースホルダーを自分の環境に合わせて設定してください:

| プレースホルダー | 説明 | 例 |
|---|---|---|
| `{GITHUB_USER}` | GitHub ユーザー名 / Organization | `your-username` |
| `{ZENN_CONTENT_PATH}` | Zenn コンテンツリポジトリのパス | `~/Projects/zenn-content` |
| `{VAULT_PATH}` | Obsidian Vault のパス（オプション） | `~/Documents/MyVault` |
| `{VAULT_PROJECTS_PATH}` | Vault 内のプロジェクトフォルダ（オプション） | `Projects/` |
| `{VAULT_ZENN_PATH}` | Vault 内の Zenn 記事フォルダ（オプション） | `Articles/Zenn/` |

## 使い方

### `/export` — 成果物を GitHub に公開

```
/export                              # 対話的に対象を選択
/export ~/.claude/commands/devlog.md  # 指定ファイルをエクスポート
```

`/export` は以下のステップを自動実行します:

1. **対象特定** — ファイルパス指定または対話的に選択
2. **分類** — command / agent / skill / script / bundle を判定し、リポジトリ名を決定
3. **依存解決** — スキル参照・スクリプト参照を検出し、同梱 or 前提条件として記載
4. **サニタイズ** — 個人パス・API キー・ユーザー名を自動検出して置換
5. **リポジトリ生成** — ディレクトリ構成・README・LICENSE・.gitignore を生成
6. **GitHub 公開** — `gh repo create` でリポジトリを作成・プッシュ
7. **プロジェクト登録** — Obsidian にプロジェクトノートを作成（オプション）

### `/draft` — Zenn 記事ドラフトを生成

```
/draft Claude-Code-EXPORT-Command     # リポジトリ名で指定
/draft                                 # 対話的に選択
```

`/draft` は以下のステップを自動実行します:

1. **リポジトリ分析** — README・コマンドファイル・スクリプトを読み込み
2. **Slug 生成** — Zenn ルール準拠のスラグを自動導出
3. **記事ドラフト** — frontmatter + 本文テンプレートを生成（TODO マーカー付き）
4. **デュアル保存** — Zenn コンテンツリポジトリと Obsidian Vault に保存

生成される記事には `<!-- TODO: ... -->` マーカーが含まれ、人間がレビュー・加筆すべき箇所が明示されます。

## 前提条件

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI がインストール済みであること
- [GitHub CLI (`gh`)](https://cli.github.com/) がインストール・認証済みであること（`/export` で使用）
- [Zenn CLI](https://zenn.dev/zenn/articles/install-zenn-cli) がインストール済みであること（`/draft` で使用、オプション）

### オプション

- **Obsidian Vault**: プロジェクト登録・記事の Vault コピーに使用
- **zenn-article-management スキル**: `/draft` の記事品質が向上

## 設計思想

Claude Code で作ったカスタムツールを公開するには、毎回以下の作業が必要でした:

1. 個人パスやシークレットのサニタイズ
2. リポジトリ構成の作成（README, LICENSE, .gitignore）
3. GitHub リポジトリの作成・プッシュ
4. 紹介記事の執筆

この繰り返し作業を **2コマンドのパイプライン** (`/export` → `/draft`) に圧縮し、「作ったものを公開する」までの摩擦を最小化しました。

サニタイズは3レイヤーで構成されています:
- **パス系**（必須）— 個人ディレクトリの絶対パスをプレースホルダーに
- **参照系**（文脈依存）— 未公開コマンドへの参照を汎用化
- **セキュリティ**（必須）— API キー・トークンのスキャンと除去

README は「別の人の Claude Code がそのプロンプトだけで導入を完結できる」品質を目指しています。

## License

MIT
