# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## プロジェクト概要

このリポジトリは、Claude Code用のカスタムスラッシュコマンド集です。実行可能コードではなく、Claude Codeに対する指示書（マークダウンファイル）を管理しています。Claude Codeプラグインシステムで管理され、プラグインとしてインストールされます。

## アーキテクチャ

### サブエージェント構成
- `agents/semantic-commit.md` - 大きな変更を論理的単位に分割してコミット（サブエージェント）
- `agents/update-pr-title-and-description.md` - Pull Requestのタイトル・説明文自動更新（サブエージェント）
- `agents/monitor-ci.md` - CI/CDチェック監視と失敗原因分析（サブエージェント）
- `agents/detect-spec-workflow.md` - spec workflowのspec-id判定（サブエージェント）

### コマンド構成
- `commands/triage_pr_comments.md` - Pull Requestコメントの対応要否判断
- `commands/estimate_pr_size.md` - Pull Requestサイズ見積もりと分割提案
- `commands/optimize_bq_query.md` - BigQueryクエリの性能分析と2倍以上の最適化提案
- `commands/validate_bq_query.md` - BigQueryクエリの構文と実行可能性の検証
- `commands/codex_review.md` - Codex MCPを使った客観的コードレビュー

### プラグイン設定
- `.claude-plugin/plugin.json` - プラグインマニフェスト（メタデータとコマンド定義）

## 開発コマンド

### サブエージェントの追加
新しいサブエージェントを追加する際：

1. `agents/` ディレクトリに `agent-name.md` ファイルを作成
2. YAMLフロントマターを追加（name, description, tools, model）
3. `README.md` の Available Sub-Agents セクションを更新
4. `CLAUDE.md` のサブエージェント構成セクションを更新
5. コミット＆プッシュ

注：`agents/` ディレクトリ内のファイルは自動的に検出されるため、`plugin.json` への追加は不要

### コマンドの追加
新しいコマンドを追加する際：

1. `commands/` ディレクトリに `command-name.md` ファイルを作成
2. `README.md` の Available Commands セクションを更新
3. `CLAUDE.md` のコマンド構成セクションを更新
4. コミット＆プッシュ

注：`commands/` ディレクトリ内のファイルは自動的に検出されるため、`plugin.json` への追加は不要

### テスト・検証
```bash
# プラグイン設定のバリデーション
claude plugin validate .

# GitHubリポジトリからインストール
/plugin marketplace add syou6162/claude-code-commands
/plugin install syou6162-plugin@syou6162-marketplace

# コマンドの呼び出しテスト
/syou6162-plugin:command-name
```

## 重要な設計原則

### コマンド設計
- **分析・提案重視**: 実際のコード修正は行わず、判断材料を提供
- **GitHub CLI活用**: `gh`コマンドを使った効率的なワークフロー
- **引数受け渡し**: Claude Codeから`$ARGUMENTS`変数で引数を受け取る

### 外部依存関係
サブエージェント・コマンドごとに以下のツールを使用：
- **GitHub CLI (`gh`)** - Pull Request操作（triage_pr_comments, update-pr-title-and-description, monitor-ci）
- **git-sequential-stage** - semantic-commit（サブエージェント）で使用する専用ツール
- **BigQuery CLI (`bq`)** - BigQuery操作（optimize_bq_query）
- **Codex MCP (`mcp__codex__codex`)** - コードレビュー（codex_review コマンド）

### ファイル更新時の注意
- `.claude-plugin/plugin.json`とREADMEの整合性を保つ
- コマンド名は一貫してアンダースコア区切りを使用
- 各コマンドの説明は実際の機能と正確に一致させる