# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## プロジェクト概要

このリポジトリは、Claude Code用のカスタムスラッシュコマンド集です。実行可能コードではなく、Claude Codeに対する指示書（マークダウンファイル）を管理しています。Claude Codeプラグインシステムで管理され、プラグインとしてインストールされます。

## アーキテクチャ

### ステアリングドキュメント構成
- `.spec-workflow/steering/product.md` - プロダクト方針と目的
- `.spec-workflow/steering/tech.md` - 技術標準とツールチェーン
- `.spec-workflow/steering/structure.md` - プロジェクト構造と組織化原則

### サブエージェント構成
- `agents/monitor-ci.md` - CI/CDチェック監視と失敗原因分析（サブエージェント）
- `agents/detect-spec-workflow.md` - spec workflowのspec-id判定（サブエージェント）
- `agents/record-current-status.md` - 現在の作業状況と本音を記録（サブエージェント）

### コマンド構成
- `commands/load_spec_tasks.md` - spec workflowのtasks.mdとToDoリストの同期
- `commands/triage_pr_comments.md` - Pull Requestコメントの対応要否判断
- `commands/estimate_pr_size.md` - Pull Requestサイズ見積もりと分割提案
- `commands/optimize_bq_query.md` - BigQueryクエリの性能分析と2倍以上の最適化提案
- `commands/validate_bq_query.md` - BigQueryクエリの構文と実行可能性の検証
- `commands/codex_review.md` - Codex MCPを使った客観的コードレビュー（planファイルと開発日誌を参照）
- `commands/multi_perspective_review.md` - 複数視点からの客観的レビューと妥当性検証

### スキル構成
- `skills/ask-user-choice/SKILL.md` - ユーザーに質問する際に選択式で答えやすくするスキル（自動発動）
- `skills/gha-sha-reference/SKILL.md` - GitHub ActionsのタグをSHA参照に変換するスキル（自動発動）
- `skills/reading-notion/SKILL.md` - NotionページやドキュメントをキーワードまたはURLで検索・取得し、内容を要約・説明するスキル（自動発動）
- `skills/requesting-gcloud-bq-auth/SKILL.md` - gcloud/bqコマンドの認証エラーを検出してユーザーに認証を依頼するスキル（自動発動）
- `skills/semantic-committing/SKILL.md` - git diffを分析し、変更を論理的単位に分割してコミットするスキル（自動発動）
- `skills/updating-pr-title-and-description/SKILL.md` - PRタイトル・説明文の自動生成・更新スキル（自動発動）
- `skills/writing-dev-diary/SKILL.md` - esa-llm-scoped-guardで開発日誌を新規作成・更新するスキル（自動発動）

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
4. ローカルテスト：`/plugin install .` でプラグインをインストールして動作確認
5. コミット＆プッシュ

注：`commands/` ディレクトリ内のファイルは自動的に検出されるため、`plugin.json` への追加は不要

### スキルの追加
新しいスキルを追加する際：

1. `skills/skill-name/` ディレクトリを作成
2. `skills/skill-name/SKILL.md` ファイルを作成
3. YAMLフロントマターを追加（name, description）
   - `name`: 小文字・ハイフン区切り（最大64文字）
   - `description`: いつ使うか + 何をするか（最大1024文字）
4. `README.md` の Available Skills セクションを更新
5. `CLAUDE.md` のスキル構成セクションを更新
6. コミット＆プッシュ

注：スキルはClaude Codeが自動的に判断して発動するため、明示的な呼び出しは不要

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

### 公式ドキュメント参照
スラッシュコマンド、サブエージェント、スキル、プラグインの実装や修正が必要になった場合は、**必ず該当する公式ドキュメントを参照すること**：

- [スラッシュコマンド](https://docs.claude.com/en/docs/claude-code/slash-commands)
- [サブエージェント](https://docs.claude.com/en/docs/claude-code/sub-agents)
- [スキル](https://docs.claude.com/en/docs/claude-code/skills)
- [プラグインリファレンス](https://docs.claude.com/en/docs/claude-code/plugins-reference)

**特にサブエージェントのdescription記述について**：
- サブエージェントの `description` は **when（いつ呼び出すか）+ what（何をするか）** で構成する
- **how（どうやるか）の実装詳細は隠蔽する** - メインエージェントは内部実装を知る必要がない
- 記述パターン：`[いつ呼び出すか]に呼び出してください。[何をするか]します。`
- 例：
  - ✅ `git addやgit commitを行う際に呼び出してください。変更を適切な粒度に分割してコミットします。`
  - ❌ `変更を意味のある最小単位に分割してコミットするエージェント。git diffを分析してhunk単位で論理的にグループ化し、git-sequential-stageで段階的にコミットします。git addやgit commitを行う際は常に使用してください。`（whenが後回し、howの詳細が多すぎる）

### XMLタグ構造化の実験（パイロット）

**実験対象**:
- `skills/semantic-committing/SKILL.md` - スキル
- `skills/ask-user-choice/SKILL.md` - スキル
- `commands/multi_perspective_review.md` - コマンド

Claude AIのシステムプロンプトおよび公式ドキュメントでは、XMLタグによるプロンプト構造化が推奨されています。このリポジトリでは、上記ファイルでXMLタグ化を試験的に導入しています。

**標準XMLタグ語彙**:
```markdown
<important>          - 禁止事項や絶対に守るべきルール
<trigger>            - スキル/サブエージェントの起動条件（いつ使うか）
<procedure>          - 重要な実行手順
<context>            - コンテキスト情報の定義（後続の手順で参照される）
<example>            - コマンド実行例
<examples>           - 複数の例を含むコンテナ（個別の<example>をネスト）
<decision-criteria>  - 判断基準（表形式など）
```

**タグ定義と参照の使い分け**:
- **定義**: `<tag>...</tag>` (開始タグと終了タグで内容を囲む)
- **参照**: テキスト中で「<tag>タグで定義した...」のように言及（終了タグなし、バッククォートなし）
- 例: `<context>`で定義 → 「以下は<context>タグで定義した手順に従って...」で参照

**標準語彙への追加ルール**:
標準語彙に新しいタグを追加する場合は、以下の条件をすべて満たすこと:
1. プロジェクト全体（複数のファイル）で共通的に使われる
2. 明確な意味を持つタグ名（抽象的すぎない、具体的すぎない）
3. 既存の標準タグでは表現できない構造である
4. レビューで妥当性を確認後、CLAUDE.mdに追加する

**各ファイル内でのタグ使用ルール**:
各ファイル内で独自のタグを使用する場合（標準語彙に含まれないタグ）:
1. そのファイル内で明確な意味を持ち、分かりやすくなる
2. 複数箇所で参照される情報の定義である
3. 標準語彙に追加する必要はない（そのファイル専用で問題ない）
- 例: `<perspective-details>` (multi_perspective_review.md内でのみ使用)
- 単発の構造化には標準タグまたはMarkdown見出しを使用すること

**XMLタグ使用の原則**:
- ネストは1-2階層まで（過度な入れ子を避ける）
- 意味のあるタグ名を使用（目的が明確）
- Markdown見出しとXMLタグを混在させる場合、見出しはタグの外に配置
- **GitHubレンダリング対応**: XMLタグとMarkdownコンテンツ（コードブロック、箇条書き、表など）の間には空行を入れる
  - 開始タグの直後と終了タグの直前に空行が必要
  - 例: `<important>` + 空行 + 箇条書き + 空行 + `</important>`
  - これによりGitHubのMarkdownパーサーが正しくレンダリングする

**ロールバック方法**:
効果が不明確な場合や問題が発生した場合は `git revert` で元に戻すことができます。

**今後の展開**:
- パイロット結果が良好であれば、他のサブエージェント（monitor-ci.md、update-pr-title-and-description.md等）やコマンドにも適用を検討
- タグ語彙は既にCLAUDE.mdに標準化済み

### コマンド設計
- **分析・提案重視**: 実際のコード修正は行わず、判断材料を提供
- **GitHub CLI活用**: `gh`コマンドを使った効率的なワークフロー
- **引数受け渡し**: Claude Codeから`$ARGUMENTS`変数で引数を受け取る

### 外部依存関係
サブエージェント・スキル・コマンドごとに以下のツールを使用：
- **GitHub CLI (`gh`)** - Pull Request操作（triage_pr_comments, updating-pr-title-and-description, monitor-ci）
- **git-sequential-stage** - semantic-committing（スキル）で使用する専用ツール
- **BigQuery CLI (`bq`)** - BigQuery操作（optimize_bq_query）
- **Codex MCP (`mcp__codex__codex`)** - コードレビュー（codex_review コマンド）
- **esa MCP (`mcp__esa-mcp-server__read_esa_post`)** - 開発日誌取得（codex_review コマンド）
- **spec-workflow MCP** - spec workflow連携（load_spec_tasks コマンド）
- **detect-spec-workflow サブエージェント** - spec-id判定（load_spec_tasks コマンド）

### ファイル更新時の注意
- `.claude-plugin/plugin.json`とREADMEの整合性を保つ
- コマンド名は一貫してアンダースコア区切りを使用
- 各コマンドの説明は実際の機能と正確に一致させる