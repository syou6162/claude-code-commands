# claude-code-commands

Yasuhisa Yoshida's personal custom slash commands for Claude Code.

## Overview

このリポジトリは、Claude Code用のカスタムスラッシュコマンドを管理する個人用リポジトリです。[cccsc](https://github.com/hiragram/cccsc)を使用してコマンドをインストール・管理できます。

## Installation

### 特定のコマンドをインストール
```bash
npx cccsc add syou6162/claude-code-commands/command-name
```

### すべてのコマンドをインストール
```bash
npx cccsc add syou6162/claude-code-commands
```

### グローバルにインストール
```bash
npx cccsc add syou6162/claude-code-commands/command-name --global
```

## Usage

インストール後、Claude Code内で以下のように使用できます：

- ローカルコマンド: `/project:command-name`
- グローバルコマンド: `/user:command-name`

## Available Commands

### semantic_commit
意味のある最小の単位でcommitする。大きな変更を論理的な単位に分割してコミットします。

```bash
# インストール
npx cccsc add syou6162/claude-code-commands/semantic_commit

# 使用方法 (Claude Code内で)
/project:semantic_commit  # ローカルインストール時
/user:semantic_commit     # グローバルインストール時
```

### triage_pr_comments
Pull Requestのコメントに対する対応要否をコードベース分析に基づいて判断します。

```bash
# インストール
npx cccsc add syou6162/claude-code-commands/triage_pr_comments

# 使用方法 (Claude Code内で)
/project:triage_pr_comments https://github.com/owner/repo/pull/123  # ローカルインストール時
/user:triage_pr_comments https://github.com/owner/repo/pull/123     # グローバルインストール時
```

### self_review_pr
プルリクエストを提出する前に、自分の変更を客観的にレビューします。レビュアーに指摘されそうな問題点や改善案を提示します。

```bash
# インストール
npx cccsc add syou6162/claude-code-commands/self_review_pr

# 使用方法 (Claude Code内で)
/project:self_review_pr https://github.com/owner/repo/pull/123  # ローカルインストール時
/user:self_review_pr https://github.com/owner/repo/pull/123     # グローバルインストール時
```

## Adding New Commands

新しいカスタムコマンドを追加する手順：

1. **コマンドファイルの作成**: マークダウンファイル（例：`command-name.md`）を作成
2. **コマンドの実装**: Claudeへの詳細な指示を記述
3. **READMEの更新**: Available Commandsセクションに新しいコマンドの説明を追加
4. **コミット＆プッシュ**: GitHubにプッシュすることで、cccsc経由でインストール可能になる

### コマンドファイルの例

```markdown
# コマンドの簡潔な説明

詳細な説明文をここに記載します。

## 使い方

\```
/command-name
\```

## 動作内容

Claudeに実行してもらいたい具体的な指示を記載します。
```

## License

MIT License - see [LICENSE](LICENSE) file for details.