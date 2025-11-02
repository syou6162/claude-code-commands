---
description: "複数の視点から客観的にレビューし、方針の妥当性を検証します。"
allowed-tools: Bash(git diff:*), Bash(git log:*), Bash(git symbolic-ref refs/remotes/origin/HEAD --short), Bash(date:+%Y%m%d_%H%M%S), Bash(mkdir -p .claude/tmp/multi_perspective_review:*), Write(.claude/tmp/**), Read(.claude/tmp/**)
---

# 複数視点レビューコマンド

<important>

このコマンドは以下の場合に使用してください：

- 実装方針や設計の妥当性を確認したい
- 複数の視点から客観的な意見が欲しい
- 早すぎる最適化やnit な指摘を避けたい
- プロジェクトの現状に合った判断をしたい

</important>

## 実行手順

<procedure>

**1. コンテキストの収集**

<context>

まず、レビューに必要なコンテキストを収集します：

- **デフォルトブランチの取得**

   ```bash
   git symbolic-ref refs/remotes/origin/HEAD --short
   ```

   結果は `origin/main` のような形式なので、後続の処理で使用します

- **現在の変更差分を取得（あれば）**

   デフォルトブランチとの差分を取得：

   ```bash
   git diff <デフォルトブランチ名>
   ```

   例: `git diff origin/main`

   ※ 変更がない場合も問題なし。これから実装する方針のレビューなど、コード差分がない場合もある

- **デフォルトブランチからの差分コミット履歴を取得**

   現在のブランチがデフォルトブランチと異なる場合、差分コミットを取得：

   ```bash
   git log <デフォルトブランチ名>..HEAD --oneline
   ```

   例: `git log origin/main..HEAD --oneline`

   ※ 現在のブランチがデフォルトブランチと同じ場合は、このステップはスキップ

- **会話履歴の確認（重要）**

   - 直前のユーザーとのやり取りから、レビュー対象の意図を把握する
   - 「この方針で良いか確認したい」「この設計をレビューして」などの説明
   - git diffがない場合は、会話履歴が主なコンテキストになる

</context>

**1-2. ログ保存用ディレクトリの作成とコンテキスト情報の保存**

タイムスタンプを取得して定義：

<timestamp>

```bash
date +%Y%m%d_%H%M%S
```

</timestamp>

次に、ディレクトリ構造を作成します：

```bash
mkdir -p .claude/tmp/multi_perspective_review/<timestamp>/round1
mkdir -p .claude/tmp/multi_perspective_review/<timestamp>/round2
```

次に、コンテキストファイルのパスを定義します：

<context-file>.claude/tmp/multi_perspective_review/<timestamp>/context.md</context-file>

<context>タグで収集した内容を<context-file>タグのパスに保存します。

**2. 第1ラウンド: 8つの視点からのレビュー**

収集したコンテキストを基に、8つの異なる視点からレビューを実施します。

**並列実行**: 8つのTaskツール（`subagent_type: "general-purpose"`）を同時に呼び出し、効率的にレビューを実施してください。

各general-purpose subagentには以下の情報を含めたプロンプトを渡します：

```
あなたは<name>タグで定義された視点から、以下の内容をレビューしてください。

## 保存先

<round1-filename>タグで定義されたファイルパスに保存してください。

## レビュー対象

<context-file>に保存されたコンテキスト情報を参照してください。

## レビューの指針

- 客観的な視点から意見を述べてください
- 修正案の提示は不要です（分析と意見のみ）
- 具体的なコードや状況に即した指摘をしてください
- 抽象的すぎる指摘は避けてください

## あなたの視点

<perspective-details>タグで定義された内容を参照してください。

詳細に網羅的に報告してください。具体的なコード箇所、懸念される影響、改善の必要性について、十分な情報を含めて分析してください。

## 出力形式

マークダウン形式で保存後、以下の形式で報告：

```
レビュー結果を保存しました: <round1-filename>
```

<examples>

<example>
<name>アーキテクチャ・設計</name>
<round1-filename>.claude/tmp/multi_perspective_review/<timestamp>/round1/architecture.md</round1-filename>

<perspective-details>

- システム全体の構造が適切か
- モジュール分割が適切か
- 依存関係が適切に管理されているか
- 拡張性が考慮されているか
- 設計原則が遵守されているか

</perspective-details>

</example>

<example>
<name>パフォーマンス・効率性</name>
<round1-filename>.claude/tmp/multi_perspective_review/<timestamp>/round1/performance.md</round1-filename>

<perspective-details>

- 実行速度に問題がないか
- メモリ使用量が適切か
- スケーラビリティが考慮されているか
- アルゴリズムの計算量が適切か

</perspective-details>

</example>

<example>
<name>保守性・可読性</name>
<round1-filename>.claude/tmp/multi_perspective_review/<timestamp>/round1/maintainability.md</round1-filename>

<perspective-details>

- コードが理解しやすいか
- 変更がしやすい構造になっているか
- 命名規則が適切か
- 使われていない関数や変数はないか
- コメントが適切に記述されているか（自明なコメントが付与されていないか）
- コメントと関数名、関数名と実際の処理で乖離がないか
- 類似の機能を持つコードが重複していないか
- 複雑度が適切に管理されているか

</perspective-details>

</example>

<example>
<name>テスタビリティ</name>
<round1-filename>.claude/tmp/multi_perspective_review/<timestamp>/round1/testability.md</round1-filename>

<perspective-details>

- テストがしやすい設計になっているか
- 依存性の注入が適切に行われているか
- モック化が容易な構造になっているか
- テストケースの網羅性が確保されているか

</perspective-details>

</example>

<example>
<name>ユーザー体験・利便性</name>
<round1-filename>.claude/tmp/multi_perspective_review/<timestamp>/round1/user_experience.md</round1-filename>

<perspective-details>

- エンドユーザーやAPI利用者の視点で使いやすいか
- UIが使いやすいか
- APIが直感的か

</perspective-details>

</example>

<example>
<name>プロジェクトフェーズ適合性</name>
<round1-filename>.claude/tmp/multi_perspective_review/<timestamp>/round1/project_phase.md</round1-filename>

<perspective-details>

- 早すぎる最適化になっていないか
- MVP として妥当な範囲か
- 技術的負債とのバランスが適切か

</perspective-details>

</example>

<example>
<name>既存コードとの整合性</name>
<round1-filename>.claude/tmp/multi_perspective_review/<timestamp>/round1/consistency.md</round1-filename>

<perspective-details>

- プロジェクト内のパターン・スタイルと一貫性があるか
- コーディング規約が遵守されているか

</perspective-details>

</example>

<example>
<name>ベストプラクティス・標準準拠</name>
<round1-filename>.claude/tmp/multi_perspective_review/<timestamp>/round1/best_practices.md</round1-filename>

<perspective-details>

- 公式ドキュメントに沿っているか
- コミュニティのベストプラクティスに従っているか
- 言語固有のイディオムが適切に使われているか
- 業界標準に準拠しているか

</perspective-details>

</example>

</examples>

**Taskツールの呼び出し例（8つ並列）:**

<example>

```
Task(
  subagent_type: "general-purpose",
  description: "アーキテクチャ・設計の観点からレビュー",
  prompt: "あなたは<name>タグで定義された視点から...",
  model: "sonnet"
)

Task(
  subagent_type: "general-purpose",
  description: "パフォーマンス・効率性の観点からレビュー",
  prompt: "あなたは<name>タグで定義された視点から...",
  model: "sonnet"
)

... (残り6つも同様に並列で呼び出し、全てmodel: "sonnet"を指定)
```

</example>

**3. 第2ラウンド: 妥当性検証**

第1ラウンドのログファイルを読み込み、5つのsubagentに妥当性検証を依頼します。

**並列実行**: 5つのTaskツール（`subagent_type: "general-purpose"`）を同時に呼び出してください。

メタレビュアー番号を定義：

<meta-reviewer-number>固有の番号</meta-reviewer-number>

各メタレビュアーの出力ファイルパスを定義：

<round2-meta-1>.claude/tmp/multi_perspective_review/<timestamp>/round2/meta_reviewer_1.md</round2-meta-1>
<round2-meta-2>.claude/tmp/multi_perspective_review/<timestamp>/round2/meta_reviewer_2.md</round2-meta-2>
<round2-meta-3>.claude/tmp/multi_perspective_review/<timestamp>/round2/meta_reviewer_3.md</round2-meta-3>
<round2-meta-4>.claude/tmp/multi_perspective_review/<timestamp>/round2/meta_reviewer_4.md</round2-meta-4>
<round2-meta-5>.claude/tmp/multi_perspective_review/<timestamp>/round2/meta_reviewer_5.md</round2-meta-5>

まず、8つの視点の<round1-filename>タグで定義されたファイルパスを収集してください。

次に、各general-purpose subagentには以下のプロンプトを渡します（<meta-reviewer-number>タグに1～5を指定し、[ログファイルパス一覧]の部分には収集した8つのパスを列挙）：

```
あなたは**メタレビュアー<meta-reviewer-number>**として、第1ラウンドのレビュー結果を検証してください。

## 保存先

`<round2-meta-<meta-reviewer-number>>`タグで定義されたファイルパスに保存してください。

## 第1ラウンドのレビューログ

以下の8つのファイルに第1ラウンドの視点別レビュー結果が保存されています。すべて読み込んで内容を確認してください：

[ログファイルパス一覧]

## 元のコンテキスト（検証の裏付け用）

<context-file>に保存されたコンテキスト情報を参照してください。

## 検証観点

以下の観点から、レビュー結果の妥当性を評価してください：

- **コンテキストとの整合性**
   - 提供されたコンテキスト（git diff、コミット履歴、会話内容）を踏まえた意見か
   - 抽象的すぎる指摘ではなく、具体的なコードや状況に即しているか

- **プロジェクトフェーズ適合性**
   - 早すぎる最適化や過剰な設計になっていないか
   - MVPとして妥当な範囲か
   - nitな指摘ではなく、本質的な問題か

## 回答形式

- 妥当な指摘: 「✅ [理由]」
- 疑問のある指摘: 「⚠️ [理由]」
- 不適切な指摘: 「❌ [理由]」

全ての観点に対して客観的かつ詳細に評価してください。各指摘について、コンテキストとの整合性、プロジェクトフェーズとの適合性、具体性の程度を十分に検証し、判断の根拠を明確に示してください。

## 出力形式

マークダウン形式で保存後、以下の形式で報告：

```
検証結果を保存しました: <round2-meta-<meta-reviewer-number>>
```

**4. 最終レポートの生成**

最終レポートの保存先を定義します：

<final-report>.claude/tmp/multi_perspective_review/<timestamp>/final_report.md</final-report>

第2ラウンドの5つのsubagentから返ってきたログファイルパスを収集した後、メインエージェントが以下の処理を実行します：

1. **全ログファイルを読み込む**
   - `.claude/tmp/multi_perspective_review/<timestamp>/round1/` 配下の8ファイル
   - `.claude/tmp/multi_perspective_review/<timestamp>/round2/` 配下の5ファイル

2. **統合処理を実行**
   - 第1ラウンドの8つの視点から、共通指摘をマージ
   - 第2ラウンドの5つの検証結果から、妥当性評価を統合
   - 主要な指摘事項を以下に分類：
     - 妥当性が確認された指摘
     - 検討が必要な指摘
     - 不適切と判断された指摘

3. **最終レポートを生成**

最終レポートを以下の形式で生成します：

<template>

```markdown
# 複数視点レビュー結果

## レビューサマリー
- 第1ラウンド: 8つの視点からレビュー
- 第2ラウンド: 5つのメタレビュアーによる妥当性確認

## 主要な指摘事項

### 妥当性が確認された指摘

**[指摘のタイトル]**
- [判定結果] [指摘内容]
- 理由: [妥当性の根拠]

### 検討が必要な指摘

**[指摘のタイトル]**
- [判定結果] [指摘内容]
- 理由: [検討が必要な理由]

### 不適切と判断された指摘

**[指摘のタイトル]**
- [判定結果] [指摘内容]
- 理由: [不適切と判断した理由]

## 詳細レビュー結果

### 第1ラウンド: 多角的レビュー

（※ 前述の整理結果を挿入）

### 第2ラウンド: 妥当性検証

**メタレビュアーの評価:**
[各メタレビュアーの評価結果を箇条書きで列挙]

## 総合評価

**優先的に対処すべき事項:**
[妥当性が高いと判断された指摘を優先度順に列挙]

**今後検討すべき事項:**
[検討が必要と判断された指摘を列挙]

**実施不要と判断された事項:**
[不適切と判断された指摘を列挙]

[総合的な評価コメント]
```

</template>

<example>

```markdown
# 複数視点レビュー結果

## レビューサマリー
- 第1ラウンド: 8つの視点からレビュー
- 第2ラウンド: 5つのメタレビュアーによる妥当性確認

## 主要な指摘事項

### 妥当性が確認された指摘

**エラーハンドリングの不足**
- 外部API呼び出し時のエラーハンドリングが不十分
- 理由: 実際のコードを見ると、try-catchブロックがなく、本番環境で問題が発生する可能性が高い

**テスタビリティの問題**
- 外部依存が直接インポートされており、モック化が困難
- 理由: 具体的なコード箇所を指摘しており、テストの保守性に直結する実質的な問題

**アーキテクチャの重複**
- 新しいvalidateInput関数が既存のvalidationUtilsモジュールと重複
- 理由: コードベース全体を見ると、同じ機能が2箇所に存在し、保守性を低下させる

### 検討が必要な指摘

**パフォーマンス懸念**
- データベースクエリがN+1問題を引き起こす可能性
- 理由: 現時点のデータ量では問題にならない可能性があるが、将来的なスケールを考慮すると対処すべき

**変数名の曖昧さ**
- 変数名が曖昧（data, result など）
- 理由: プロジェクト内で一貫して使われているパターンであれば、nitな指摘の可能性

### 不適切と判断された指摘

**過剰な抽象化**
- インターフェースを追加してDI パターンを徹底すべき
- 理由: プロジェクトのフェーズを考慮すると早すぎる最適化。現時点では不要な複雑性を導入する

## 詳細レビュー結果

### 第1ラウンド: 多角的レビュー

（※ 前述の整理結果を挿入）

### 第2ラウンド: 妥当性検証

**メタレビュアー1の評価:**
- エラーハンドリング不足は妥当な指摘（実装に直接影響）
- N+1問題は現時点では過剰かもしれないが、将来的には対処が必要
- DI パターンの徹底は早すぎる最適化

**メタレビュアー2の評価:**
- テスタビリティの問題は具体的で改善価値が高い
- validationUtils との重複は明確な問題
- 変数名の指摘はプロジェクト規約次第

（以下、メタレビュアー3-5の評価を同様に列挙）

## 総合評価

**優先的に対処すべき事項:**
1. 外部API呼び出し時のエラーハンドリングを追加
2. 外部依存のモック化を容易にするため、依存性注入パターンを部分的に適用
3. validateInput関数と既存のvalidationUtilsモジュールの重複を解消

**今後検討すべき事項:**
- データベースクエリのN+1問題（データ量が増えた際に再評価）
- 変数名の見直し（チーム全体のコーディング規約を整備してから判断）

**実施不要と判断された事項:**
- 全体的なDIパターンの徹底（現フェーズでは過剰）

全体として、実装は概ね妥当な範囲にあります。上記の優先事項を対処すれば、品質とテスタビリティが大きく向上すると考えられます。
```

</example>

</template>

**4. 最終レポートの保存とユーザーへの表示**

生成した最終レポートを<final-report>タグで定義されたファイルパスに保存してください。

保存後、ユーザーには以下の情報を表示：

```markdown
# 複数視点レビュー完了

## レビュー結果

最終レポートを保存しました: <final-report>

## サマリー

### 妥当性が確認された主要な指摘（優先対処）
1. [指摘事項1]
2. [指摘事項2]
3. [指摘事項3]

### 検討が必要な指摘
- [指摘事項A]
- [指摘事項B]

### 不適切と判断された指摘
- [指摘事項X]

詳細については、上記ファイルを参照してください。

## ログファイル一覧

### コンテキスト情報
- `<context-file>`

### 第1ラウンド（8視点のレビュー）

各視点の<round1-filename>タグで定義されたファイル（8ファイル）

### 第2ラウンド（5メタレビュアーの評価）

各メタレビュアーの`<round2-meta-*>`タグで定義されたファイル（5ファイル）
```

</procedure>

## 重要な注意事項

<important>

1. **ログ保存の流れ**
   - タイムスタンプ付きディレクトリを最初に作成
   - 取得したタイムスタンプを `<timestamp>` タグで定義
   - 各subagentへのプロンプトでは `<timestamp>` タグを参照
   - 各subagentは結果をログファイルに保存し、**パスのみ**を返す
   - メインエージェントは全ログファイルを読み込んで統合処理を実行
   - 最終レポートもファイルに保存し、ユーザーにはパスとサマリーを表示

2. **subagentへの指示**
   - 修正を行わない（分析と意見のみ）
   - 客観的な視点を保つ
   - 具体的なコードに言及する
   - 過度に一般化しない
   - **結果はログファイルに保存し、ファイルパスのみを返す**

3. **並列実行**
   - 第1ラウンドの8つのsubagentは必ず並列で起動
   - 第2ラウンドの5つのsubagentも必ず並列で起動
   - 合計13個のsubagentを使用

4. **コンテキストの重要性**
   - git diff, git log, 会話履歴を必ず収集
   - 不足している場合は、その旨を明記
   - コンテキスト情報は `<context-file>` に保存

5. **ファイル命名規則**
   - 第1ラウンド: 各視点の<round1-filename>タグで定義
   - 第2ラウンド: 各メタレビュアーの`<round2-meta-*>`タグで定義
   - 最終レポート: <final-report>タグで定義
   - コンテキスト: <context-file>タグで定義

</important>
