# claude-code-commands

Yasuhisa Yoshida's personal custom slash commands for Claude Code.

## Overview

このリポジトリは、Claude Code用のカスタムスラッシュコマンドを管理する個人用リポジトリです。Claude Codeのプラグインシステムを使用してコマンドをインストール・管理できます。

## Installation

Claude Code内で以下のコマンドを実行してプラグインをインストールします：

```bash
/plugin install syou6162/claude-code-commands
```

ローカル開発の場合は、リポジトリをクローンして以下のコマンドでインストールできます：

```bash
/plugin install .
```

## Usage

プラグインをインストール後、Claude Code内でコマンドを呼び出すことができます：

```bash
/syou6162-plugin:command-name
```

引数が必要なコマンドの場合：

```bash
/syou6162-plugin:command-name argument
```

## Plugin Structure

このリポジトリは[Claude Codeのプラグイン](https://docs.claude.com/ja/docs/claude-code/plugins)として構成されており、以下のディレクトリ構造を持ちます：

```
claude-code-commands/
├── .claude-plugin/
│   └── plugin.json          # プラグインマニフェスト（メタデータとコマンド定義）
├── commands/
│   ├── semantic_commit.md
│   ├── triage_pr_comments.md
│   ├── self_review_pr.md
│   ├── estimate_pr_size.md
│   ├── update_pr_title_and_description.md
│   ├── optimize_bq_query.md
│   └── validate_bq_query.md
└── README.md
```

- **`.claude-plugin/plugin.json`**: プラグインのメタデータ（名前、バージョン、作者など）とコマンドリストを定義
- **`commands/`**: 各カスタムスラッシュコマンドのマークダウンファイルを格納

Claude Codeは`plugin.json`を読み込んでプラグインを認識し、`commands/`ディレクトリ内のコマンドを自動的に利用可能にします。

## Available Commands

### semantic_commit
意味のある最小の単位でcommitする。大きな変更を論理的な単位に分割してコミットします。LLMがgit diffを分析して意味のある最小単位を提案し、`git-sequential-stage`ツールによる自動化された逐次ステージングでコミットします。

```bash
# 使用方法 (Claude Code内で)
/syou6162-plugin:semantic_commit
```

### triage_pr_comments
Pull Requestのコメントに対する対応要否をコードベース分析に基づいて判断します。

```bash
# 使用方法 (Claude Code内で)
/syou6162-plugin:triage_pr_comments https://github.com/owner/repo/pull/123
```

### self_review_pr
プルリクエストを提出する前に、自分の変更を客観的にレビューします。レビュアーに指摘されそうな問題点や改善案を提示します。

```bash
# 使用方法 (Claude Code内で)
/syou6162-plugin:self_review_pr https://github.com/owner/repo/pull/123
```

### estimate_pr_size
指定されたタスクに対してPull Requestのサイズを見積もり、必要に応じて分割の提案を行います。過去のPull Request履歴を分析し、作業量を予測して適切な実装順序を提案します。

```bash
# 使用方法 (Claude Code内で)
/syou6162-plugin:estimate_pr_size
```

### update_pr_title_and_description
Pull Requestのタイトルと説明文を修正内容とコミットメッセージに基づいて更新します。`.github/PULL_REQUEST_TEMPLATE.md`テンプレートに対応しています。

```bash
# 使用方法 (Claude Code内で)
/syou6162-plugin:update_pr_title_and_description
```

### optimize_bq_query
BigQueryクエリのパフォーマンスを分析し、2倍以上の性能改善を目標とした最適化を提案します。ジョブIDまたはSQLファイルを入力として、ボトルネック分析・最適化・検証を自動実行します。

```bash
# 使用方法 (Claude Code内で)
/syou6162-plugin:optimize_bq_query query.sql
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