# Project Structure

## Directory Organization

```
claude-code-commands/
├── .claude-plugin/
│   └── plugin.json              # プラグインマニフェスト
├── agents/                       # サブエージェント定義
│   └── <agent-name>.md          # 各サブエージェント（YAMLフロントマター付き）
├── commands/                     # スラッシュコマンド定義
│   └── <command_name>.md        # 各コマンド
├── .spec-workflow/               # spec workflow統合（ランタイム管理）
│   ├── steering/                 # ステアリングドキュメント
│   ├── specs/                    # 機能仕様（spec workflow）
│   ├── templates/                # spec/steeringテンプレート（デフォルト）
│   ├── user-templates/           # ユーザーカスタムテンプレート（優先）
│   ├── approvals/                # 承認リクエスト（ランタイム）
│   ├── archive/                  # アーカイブ済みspec
│   └── config.example.toml       # spec workflow設定例
├── .claude_work/                 # Claude Code作業ディレクトリ（ランタイム、一時ファイル）
├── README.md                     # プロジェクト概要とインストール
└── CLAUDE.md                     # Claude Code向け開発ガイド
```

**組織化の原則**:
- **機能別グループ化**: コマンドとサブエージェントを明確に分離
- **バージョン管理対象とランタイムの分離**:
  - バージョン管理: `agents/`, `commands/`, `.claude-plugin/`, ドキュメント、テンプレート
  - ランタイム: `.claude_work/`, `.spec-workflow/approvals/`, `.spec-workflow/specs/`（一部）
- **agents/とcommands/はフラット**: 各ディレクトリ内は1階層のみ（ネストなし）
- **spec-workflowは階層構造**: 機能ごとにサブディレクトリで整理
- **自己完結性**: 各マークダウンファイルは独立して動作

## Naming Conventions

### ファイル名
- **サブエージェント**: `kebab-case.md`（例: `semantic-commit.md`）
- **コマンド**: `snake_case.md`（例: `load_spec_tasks.md`）
- **設定ファイル**: `plugin.json`, `CLAUDE.md`, `README.md`

### コマンド名（実行時）
- **名前空間付き**: `/syou6162-plugin:command-name`
- **区切り文字**: ハイフン（例: `/syou6162-plugin:optimize-bq-query`）
- **内部参照**: ケバブケース（例: `optimize-bq-query`）

### サブエージェント名（実行時）
- **名前空間付き**: `syou6162-plugin:agent-name`
- **区切り文字**: ハイフン（例: `syou6162-plugin:semantic-commit`）

## File Structure Patterns

### サブエージェント（agents/*.md）

```markdown
---
name: エージェント名
description: いつ呼び出すか + 何をするか
tools: [ツールリスト]
model: モデル指定
---

# エージェント名

## 概要
[エージェントの目的と役割]

## 実行手順
[具体的な実行ステップ]

## 重要な制約
[禁止事項や必須ルール]

## 実行例
[具体的な使用例]
```

**構造の原則**:
- YAMLフロントマターでメタデータを定義
- descriptionは「when + what」形式（howは隠蔽）
- 実装詳細は本文に記述

### コマンド（commands/*.md）

```markdown
# コマンド名

## 目的
[コマンドの目的]

## 実行方法
[引数の受け取り方、実行フロー]

## 出力
[何を出力するか]

## 外部依存
[使用する外部ツール]
```

**構造の原則**:
- YAMLフロントマターはオプション（`description`, `allowed-tools`, `model`等のメタデータ指定に使用可能）
- `$ARGUMENTS`変数で引数受け取り
- このプロジェクトでは分析・提案重視の設計（コマンドによる修正も可能だが、判断材料提供に注力）

## Code Organization Principles

1. **単一責任**: 各ファイルは1つの明確な目的を持つ
   - 1コマンド = 1ファイル
   - 1サブエージェント = 1ファイル

2. **独立性**: コマンド・サブエージェント間の相互依存を避ける
   - 各ファイルは独立して動作
   - 状態の共有は`.claude_work/`経由のみ

3. **テンプレート一貫性**: 同種のファイルは同じ構造を維持
   - サブエージェント同士で構造を統一
   - コマンド同士で構造を統一

4. **発見可能性**: README/CLAUDE.mdで全機能を一覧化
   - 新規追加時は必ずドキュメント更新
   - ステアリングドキュメントは抽象レベルを保つ

## Module Boundaries

### コマンド vs サブエージェント

**コマンド（commands/）**:
- スラッシュコマンドとして呼び出し
- メインコンテキストで実行
- 分析・判断材料の提供が主目的
- 実際の修正はユーザーに委ねる

**サブエージェント（agents/）**:
- Task toolで独立コンテキストで実行
- 専門的なタスクを自律的に処理
- メイン会話を汚染しない
- 実際の操作（git commit等）を実行可能

### プラグインシステムとの境界

**Claude Code側の責任**:
- plugin.jsonの読み込み
- commands/とagents/の自動検出
- マークダウンファイルの解釈と実行
- ツールアクセス制御

**プラグイン側の責任**:
- マークダウン形式の指示書記述
- 外部ツールの正しい使用
- エラーハンドリングと例外処理
- ドキュメントの保守

### 外部ツールとの境界

**統合ツール**: `gh`, `bq`, `git-sequential-stage`, Codex MCP, spec-workflow MCP
**責任範囲**:
- プラグインはツールを呼び出すのみ
- 認証・設定は各ツールが管理
- エラーは各ツール固有のメカニズムで処理

## File Size Guidelines

### マークダウンファイル
- **推奨サイズ**: 200-500行
- **最大サイズ**: 1000行（超える場合は分割を検討）
- **理由**: Claude Codeの読解性と保守性のバランス

### セクション構成
- **概要**: 50-100行
- **手順**: 100-300行
- **例**: 50-100行
- **制約**: 20-50行

### 複雑性の管理
- XMLタグによる構造化（semantic-commit.mdでパイロット中）
- ネストは1-2階層まで
- 表やコードブロックで視認性向上

## Documentation Standards

### プロジェクトレベル
- **README.md**: インストール、使用方法、コマンド一覧
- **CLAUDE.md**: Claude Code向け開発ガイド、設計原則
- **.spec-workflow/steering/**: プロジェクト全体の方向性

### 機能レベル
- 各コマンド/サブエージェントファイル内にドキュメント記述
- YAMLフロントマターでメタデータを明示（サブエージェントのみ）
- 実行例を含める

### メンテナンス原則
- 新機能追加時は README, CLAUDE.md を更新
- ステアリングドキュメントは具体的なコマンド名を列挙しない
- 抽象レベルを保ち、メンテナンス負担を最小化
