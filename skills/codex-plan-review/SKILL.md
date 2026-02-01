---
name: codex-plan-review
description: planファイルのレビューを依頼された際に使用。Codex CLIを使ってplanファイル自体の実現可能性・技術的妥当性・抜け漏れをレビューします。
allowed-tools: Bash, Write, Read
model: sonnet
context: fork
---

# Codex CLIを使ったplanファイルレビュー

planファイルの実現可能性、技術的妥当性、抜け漏れやリスクについて、Codex CLIを使って客観的にレビューします。

## 実行手順

<procedure>

### 1. planファイルの特定

`.claude_work/plans/`配下からplanファイルを取得してください：

```bash
ls .claude_work/plans/*.md
```

git worktree運用のため、単一のplanファイルが存在する前提です。

このファイルパスを以下のように定義します：

<plan-file>

```
<取得したplanファイルのパス>
```

</plan-file>

### 2. 開発日誌の取得・保存（コンテキストにある場合）

会話のコンテキスト内にesa URLの開発日誌が言及されている場合、以下の手順で取得・保存してください：

1. esa URLからpost番号を抽出（例：`https://xxx.esa.io/posts/1234` → `1234`）
2. `mcp__esa-mcp-server__read_esa_post` ツールでpost内容を取得
3. 取得した内容を以下のパスに保存（Writeツール使用）

<dev-diary-file>

```
.claude_work/dev_diary.md
```

</dev-diary-file>

開発日誌がコンテキストにない場合は、このステップをスキップしてください。

### 3. Codex CLIでのレビュー実行とファイル保存

Bash経由で`codex exec --sandbox read-only`を使ってレビューを実行し、`tee`で出力をファイルに保存してください。

**出力先**: `.claude_work/codex_plan_review.md`（固定パス、既存ファイルは上書き）

<example>

**開発日誌がない場合のコマンド例：**

```bash
echo "以下のplanファイルを日本語でレビューしてください。

## レビュー対象
- planファイル: <plan-fileタグで定義されたパス>

## レビューの観点

### 1. 実現可能性・技術的妥当性
- planで述べられている実装方針は技術的に実現可能か
- 使用予定のライブラリ、API、アーキテクチャは適切か
- 既存のコードベース構造と整合性があるか

### 2. 要件との整合性
- planが解決しようとしている課題は明確か
- 提案されている解決策は要件を満たしているか
- 過不足なく要件をカバーしているか

### 3. 抜け漏れ・リスクの指摘
- 考慮されていないエッジケースはないか
- 潜在的なリスク（パフォーマンス、セキュリティ、保守性）はないか
- 依存関係や影響範囲で見落としはないか

### 4. 実装順序・優先度
- 提案されている実装順序は妥当か
- 依存関係を考慮した適切な順序になっているか

出力は日本語で、具体的な指摘と改善提案を含めてください。" | codex exec --sandbox read-only | tee .claude_work/codex_plan_review.md
```

**開発日誌がある場合のコマンド例：**

```bash
echo "以下のplanファイルを日本語でレビューしてください。

## レビュー対象
- planファイル: <plan-fileタグで定義されたパス>
- 開発日誌: <dev-diary-fileタグで定義されたパス>

## レビューの観点

### 1. 実現可能性・技術的妥当性
- planで述べられている実装方針は技術的に実現可能か
- 使用予定のライブラリ、API、アーキテクチャは適切か
- 既存のコードベース構造と整合性があるか

### 2. 要件との整合性
- planが解決しようとしている課題は明確か
- 提案されている解決策は要件を満たしているか
- 過不足なく要件をカバーしているか

### 3. 抜け漏れ・リスクの指摘
- 考慮されていないエッジケースはないか
- 潜在的なリスク（パフォーマンス、セキュリティ、保守性）はないか
- 依存関係や影響範囲で見落としはないか

### 4. 実装順序・優先度
- 提案されている実装順序は妥当か
- 依存関係を考慮した適切な順序になっているか

### 5. 開発日誌との整合性
- 開発日誌に記載された開発方針・過去の決定事項と矛盾していないか
- 開発日誌で言及されている懸念事項がplanで対処されているか

出力は日本語で、具体的な指摘と改善提案を含めてください。" | codex exec --sandbox read-only | tee .claude_work/codex_plan_review.md
```

**連続会話の場合：**

```bash
echo "追加の質問" | codex exec --sandbox read-only resume <thread-id> | tee .claude_work/codex_plan_review.md
```

</example>

<important>

- ファイルの内容ではなく、ファイルパスを渡すことで、Codexが直接ファイルを読み取ります
- `tee`を使うことで、リアルタイムで出力を確認しつつファイルにも保存されます
- codex CLIの出力からthread-id（セッションID）を確認してください

</important>

### 4. レビュー結果の報告

**以下の2つの情報のみ**をユーザーに報告してください：

1. **レビューファイルのパス**: `.claude_work/codex_plan_review.md`
2. **セッションID**: codex CLIが出力したthread-id

**禁止事項**:
- レビュー内容を要約しないこと
- レビュー内容の詳細を報告しないこと
- ファイルを読んで内容を説明しないこと

</procedure>
