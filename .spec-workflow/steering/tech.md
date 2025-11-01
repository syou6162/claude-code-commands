# Technology Stack

## Project Type

Claude Code plugin - マークダウンベースの指示書集

このプロジェクトは実行可能コードではなく、Claude Codeに対する指示を記述したマークダウンファイルで構成されています。プラグインシステムを通じて、カスタムスラッシュコマンドとサブエージェントを提供します。

## Core Technologies

### Primary Language(s)
- **Language**: Markdown
- **Runtime**: Claude Code（Anthropic製CLI）
- **特記事項**: プログラミング言語ではなく、自然言語で記述された指示書

### Key Dependencies/Libraries

このプロジェクト自体は外部ライブラリに依存しませんが、以下の外部ツールとの統合を前提としています：

- **GitHub CLI (`gh`)**
- **git-sequential-stage**
- **BigQuery CLI (`bq`)**
- **Codex MCP**
- **spec-workflow MCP**

### Application Architecture

プラグインベースアーキテクチャ：
- **マニフェスト（plugin.json）**: プラグインメタデータとコマンド/エージェント定義。Claude Codeはこのファイルを読み込んでプラグインを認識し、`commands/`と`agents/`ディレクトリ内のマークダウンファイルを自動的に発見する
- **コマンド（commands/*.md）**: スラッシュコマンドの実装。各ファイルはClaude Codeへの自然言語による指示書
- **サブエージェント（agents/*.md）**: 独立したコンテキストで動作する専門エージェント。YAMLフロントマターでメタデータを定義

各コマンドとサブエージェントは独立して動作し、相互依存しません。

### Data Storage

- **Primary storage**: なし（ステートレス）
- **一時データ**: `.claude/tmp/` ディレクトリ（作業状況記録など）。ライフサイクルはClaude Codeが管理し、セッション間で永続化される
- **Data formats**: Markdown、JSON（外部ツールとのインターフェース）

### External Integrations

- **APIs**: GitHub API（`gh` CLI経由）、BigQuery API（`bq` CLI経由）
- **Protocols**: HTTP/REST（外部CLIツール経由）
- **Authentication**: 各ツールの認証機構に依存（`gh auth`、`gcloud auth`など）

## Development Environment

### Build & Development Tools

- **Build System**: 不要（マークダウンファイル）
- **Package Management**: Claude Code plugin system
- **Development workflow**:
  - ローカルテスト: `/plugin install .`
  - リモートインストール: `/plugin marketplace add syou6162/claude-code-commands` → `/plugin install syou6162-plugin@syou6162-marketplace`

### Code Quality Tools

- **Static Analysis**: なし（自然言語）
- **Formatting**: Markdown linter（任意）
- **Testing**: 手動テスト（実際のClaude Code環境で動作確認）
- **Documentation**: README.md、CLAUDE.md

### Version Control & Collaboration

- **VCS**: Git
- **Branching Strategy**: GitHub Flow（feature branchからmainへのPull Request）
- **Code Review Process**: Pull Requestベース、Codex MCPによる自動レビュー支援

## Deployment & Distribution

- **Target Platform(s)**: Claude Code CLI（macOS、Linux、Windows）
- **Distribution Method**: GitHubリポジトリ経由（`/plugin install`コマンド）
- **Installation Requirements**:
  - Claude Code CLI
  - 各機能で使用する外部ツール（`gh`、`bq`など）は個別にインストール必要
- **Update Mechanism**: Gitリポジトリのpull、または再インストール

## Technical Requirements & Constraints

### Performance Requirements

- コマンド実行時のレスポンスタイムは外部ツール（`gh`、`bq`など）の性能に依存
- Claude Codeの会話コンテキストサイズに制約あり（大量のテキスト出力は避ける）

### Compatibility Requirements

- **Platform Support**: Claude Code CLIがサポートするプラットフォーム（macOS、Linux、Windowsでの動作が期待される）
- **Dependency Versions**:
  - Claude Code: 最新版推奨
  - GitHub CLI: v2.0以降
  - BigQuery CLI: Google Cloud SDKに含まれる最新版
- **Standards Compliance**:
  - Markdown: CommonMark仕様
  - JSON: RFC 8259

### Security & Compliance

- **Security Requirements**:
  - 機密情報（APIキー、トークンなど）はプラグイン内に含めない
  - 認証は各外部ツールの機構に委譲
- **Threat Model**:
  - 破壊的操作（force push、データ削除など）は常にユーザー承認を要求
  - 外部APIへのアクセスは読み取り専用を原則とし、書き込みは明示的な確認後のみ

### Scalability & Reliability

- **Expected Load**: 単一ユーザー、1日100回程度のコマンド実行
- **Availability**: Claude Code CLI自体の可用性に依存
- **Growth**: ユーザー数ではなく、機能追加（新コマンド/エージェント）による拡張

## Technical Decisions & Rationale

### Decision Log

1. **Markdownベースの指示書形式**:
   - **理由**: プログラミング言語ではなく自然言語で記述することで、保守性と可読性を向上
   - **代替案**: Python/TypeScriptでのプラグイン実装
   - **トレードオフ**: 実行時の柔軟性は低いが、記述と理解が容易

2. **外部CLIツールとの統合**:
   - **理由**: 既存の優れたツール（`gh`、`bq`）を活用し、車輪の再発明を避ける
   - **代替案**: API直接呼び出し
   - **トレードオフ**: CLIツールのインストールが必要だが、実装がシンプル

3. **サブエージェントパターン**:
   - **理由**: 専門的なタスクを独立したコンテキストで実行し、メイン会話を汚染しない
   - **代替案**: 全てをメインコンテキストで実行
   - **トレードオフ**: コンテキスト分離により効率的だが、エージェント間の情報共有は制限される

4. **spec-workflow MCP統合**:
   - **理由**: 仕様駆動開発を実践し、仕様・設計・タスク・実装の一貫性を保つ。承認ワークフローにより変更管理を効率化
   - **代替案**: 手動でのドキュメント管理、または別のタスク管理ツールとの統合
   - **トレードオフ**: spec-workflow MCPへの依存が生まれるが、仕様とタスクの自動同期により情報の断片化を防止

## Known Limitations

- **Claude Codeプラットフォーム依存**: Claude Code以外の環境では動作しない。これは意図的な設計判断
- **外部ツール依存**: `gh`、`bq`などの外部ツールが正しくインストール・設定されていることが前提。エラーハンドリングは各コマンド/エージェント実装に依存
- **自然言語の曖昧性**: マークダウン形式の指示は、プログラミング言語ほど厳密ではないため、Claude Codeの解釈に依存する部分がある
