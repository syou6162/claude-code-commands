---
description: Codex MCPを使ってコードの変更を客観的にレビューします。現在作業中のspec workflowがある場合は仕様に沿ったレビューを実施します。
---

# Codex MCPを使ったコードレビュー

コードの変更内容に対してcodex mcpを使って客観的なレビューを実施します。

## 実行手順

### 1. デフォルトブランチの取得

まず、リポジトリのデフォルトブランチを取得してください：

```bash
git symbolic-ref refs/remotes/origin/HEAD --short | cut -d/ -f2
```

このコマンドでデフォルトブランチ名（main や master など）を取得し、後続のステップで使用します。

### 2. spec-idの判定

Taskツールで `syou6162-plugin:detect-spec-workflow` サブエージェントを呼び出し、現在の作業に該当するspec-idを判定してください：

```
Taskツールで以下のパラメータを指定：
- subagent_type: "syou6162-plugin:detect-spec-workflow"
- description: "spec-idの判定"
- prompt: 現在のブランチ名やコミットメッセージから推測されるタスク概要
  （例：「detect-spec-workflowサブエージェントの追加とドキュメント更新」）
```

### 3. Codex MCPでのレビュー実行

前述のステップで得たspec-idの有無に応じて、以下のようにCodex MCPを呼び出してください：

**spec-idが取得できた場合：**

```
mcp__codex__codex ツールを使って以下のプロンプトでレビューを実行：

「<デフォルトブランチ名>ブランチとの差分を日本語でレビューしてください。

このプロジェクトではspec workflowという仕様駆動開発のワークフローを使っています。

- `.spec-workflow/steering/`にプロジェクトの方向性を示すsteeringドキュメントがあります
- `.spec-workflow/specs/<取得したspec-id>/`に現在開発中の機能の仕様とタスクの進行状況があります

これらを読んで、仕様に沿った実装になっているかレビューしてください。」
```

**spec-idが取得できなかった場合：**

```
mcp__codex__codex ツールを使って以下のプロンプトでレビューを実行：

「<デフォルトブランチ名>ブランチとの差分を日本語でレビューしてください。」
```

### 4. レビュー結果の報告

Codex MCPからのレビュー結果をユーザーに報告してください。
