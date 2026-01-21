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
│   └── plugin.json                          # プラグインマニフェスト（メタデータとコマンド定義）
├── .spec-workflow/
│   └── steering/
│       ├── product.md                       # プロダクト方針
│       ├── tech.md                          # 技術標準
│       └── structure.md                     # プロジェクト構造
├── agents/
│   ├── monitor-ci.md                        # サブエージェント
│   ├── detect-spec-workflow.md              # サブエージェント
│   └── record-current-status.md             # サブエージェント
├── commands/
│   ├── triage_pr_comments.md
│   ├── estimate_pr_size.md
│   ├── optimize_bq_query.md
│   ├── validate_bq_query.md
│   └── codex_review.md
├── skills/
│   ├── ask-user-choice/
│   │   └── SKILL.md                         # スキル定義
│   ├── gha-sha-reference/
│   │   └── SKILL.md                         # スキル定義
│   ├── reading-notion/
│   │   └── SKILL.md                         # スキル定義
│   ├── requesting-gcloud-bq-auth/
│   │   └── SKILL.md                         # スキル定義
│   ├── semantic-committing/
│   │   └── SKILL.md                         # スキル定義
│   └── updating-pr-title-and-description/
│       └── SKILL.md                         # スキル定義
└── README.md
```

- **`.claude-plugin/plugin.json`**: プラグインのメタデータ（名前、バージョン、作者など）とコマンドリストを定義
- **`.spec-workflow/steering/`**: プロジェクト全体の方向性を定義するステアリングドキュメント（spec workflow統合）
- **`agents/`**: サブエージェント定義（専門的なタスクを独立したコンテキストで実行）
- **`commands/`**: 各カスタムスラッシュコマンドのマークダウンファイルを格納
- **`skills/`**: スキル定義（Claudeが自動的に判断して発動する拡張機能）

Claude Codeは`plugin.json`を読み込んでプラグインを認識し、`commands/`ディレクトリ内のコマンドと`agents/`内のサブエージェントを自動的に利用可能にします。

## Available Sub-Agents

各サブエージェントは独立したコンテキストで実行されるため、メイン会話を汚染せずに専門的なタスクを処理できます。

### monitor-ci (サブエージェント)
Pull RequestのCI/CDチェックを監視し、失敗したjobのログを分析して原因を特定するエージェント。失敗内容をメインエージェントに報告します。CI失敗時の原因調査と対応方針の提案を自動化します。

### detect-spec-workflow (サブエージェント)
現在のタスクや仕様の概要から、該当するspec workflowのspec-idを判定するエージェント。spec-workflow/specs/配下のspecを分析し、最も関連性の高いspec-idを返します。

### record-current-status (サブエージェント)
作業のキリが良いタイミングやユーザーへの報告時に、現在の作業状況と本音を`.claude_work/current_status`に記録するエージェント。状況（簡単/普通/やや難/難しい/情報不足/無理）と詳細を140字以内で記録し、作業の進捗や困難を可視化します。

## Available Skills

スキルは、Claudeが自動的に判断して発動する拡張機能です。明示的に呼び出す必要はなく、Claudeが状況に応じて適切に使用します。

### ask-user-choice
ユーザーに質問や相談をする際に自動的に発動するスキル。テキスト入力ではなく、明確な選択肢（2-4個）を提示することで、ユーザーの入力負担を軽減し、より迅速な意思決定を支援します。multiSelectオプションを活用することで、複数選択が適切な場面にも対応します。

### gha-sha-reference
ユーザーがGitHub Actionsのタグ参照をSHA参照に変換するよう要求したときに自動的に発動するスキル。セキュリティのベストプラクティスに従い、`uses:`フィールドのタグ参照（例: `@v4`）を不変なSHA参照（例: `@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2`）に変換します。GitHub APIを使用してコミットSHAを自動取得し、サプライチェーン攻撃のリスクを軽減します。

### reading-notion
NotionページやドキュメントをキーワードまたはURLで検索・取得し、プロパティとブロック内容を読み込んで要約・説明するスキル。Notion MCP Serverは直接使用するとコンテキストを圧迫するため、[mcptools](https://github.com/f/mcptools)を利用してNotion APIをラップしています。Notion URLが会話に登場した時に自動的にページを取得して内容を説明し、キーワード検索時には検索結果から選択したページの内容を説明します。

### requesting-gcloud-bq-auth
gcloudやbqコマンド実行時に認証エラー（`Reauthentication required`や`Your browser has been opened to visit: ...accounts.google.com...`）を検出したときに自動的に発動するスキル。エージェントが勝手に認証コマンドを実行せず、ユーザーに認証を依頼します。ブラウザ操作が必要な認証フローのため、ユーザーによる手動認証を促し、認証完了後に作業を再開します。

### semantic-committing
コミット時、「commit」「git add」「変更を分割」の言及時に自動的に発動するスキル。git diffを分析して変更を論理的な意味単位に分割し、git-sequential-stageでhunk単位のステージングを行います。大きな変更を複数の意味のあるコミットに分けたい時に、Claude Codeが自動的にこのスキルを使用します。

### updating-pr-title-and-description
PR作成・更新時に自動的に発動するスキル。Pull Requestのタイトルと説明文を自動生成・更新します。差分やコミットメッセージを分析し、適切な説明文を作成します。説明文は日本語で記載され、`.github/PULL_REQUEST_TEMPLATE.md`がある場合はテンプレートに沿った形で生成されます。

### writing-dev-diary
「開発日誌更新」「開発日誌作って」の言及時に自動的に発動するスキル。esa-llm-scoped-guardを使って開発日誌を新規作成・更新します。トリガーによって動作を分岐し、「作って」の場合は直接新規作成、「更新」+URLの場合は指定記事を更新、「更新」のみの場合は検索して関連記事を更新（なければ新規作成）します。

## Available Commands

### load_spec_tasks
spec workflowのtasks.mdから次にやるべきタスクを読み込み、Claude CodeのToDoリストに自己再生産的なタスクサイクルを設定します。最初に1回実行するだけで、以降は自動的にタスクサイクルが回り続けます。tasks.mdをシングルソースオブトゥルースとして扱い、Claudeがタスクの状態更新を忘れないようにします。

```bash
# 使用方法 (Claude Code内で)
/syou6162-plugin:load_spec_tasks
```

### codex_review
Codex MCPを使ってコードの変更を客観的にレビューします。planファイルと開発日誌（コンテキストにある場合）を参照し、計画に沿った実装になっているかを確認します。

```bash
# 使用方法 (Claude Code内で)
/syou6162-plugin:codex_review
```

### triage_pr_comments
Pull Requestのコメントに対する対応要否をコードベース分析に基づいて判断します。

```bash
# 使用方法 (Claude Code内で)
/syou6162-plugin:triage_pr_comments https://github.com/owner/repo/pull/123
```

### estimate_pr_size
指定されたタスクに対してPull Requestのサイズを見積もり、必要に応じて分割の提案を行います。過去のPull Request履歴を分析し、作業量を予測して適切な実装順序を提案します。

```bash
# 使用方法 (Claude Code内で)
/syou6162-plugin:estimate_pr_size
```

### optimize_bq_query
BigQueryクエリのパフォーマンスを分析し、2倍以上の性能改善を目標とした最適化を提案します。ジョブIDまたはSQLファイルを入力として、ボトルネック分析・最適化・検証を自動実行します。

```bash
# 使用方法 (Claude Code内で)
/syou6162-plugin:optimize_bq_query query.sql
```

### multi_perspective_review
複数の視点（アーキテクチャ・設計、パフォーマンス、保守性、テスタビリティ、UX、プロジェクトフェーズ適合性、既存コードとの整合性、ベストプラクティス・標準準拠）から客観的にレビューし、方針の妥当性を検証します。8つの視点からのレビュー結果を整理し、さらに5名の検証者による妥当性確認を実施します。

```bash
# 使用方法 (Claude Code内で)
/syou6162-plugin:multi_perspective_review
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

\`\`\`
/command-name
\`\`\`

## 動作内容

Claudeに実行してもらいたい具体的な指示を記載します。
```

## License

MIT License - see [LICENSE](LICENSE) file for details.