---
description: Codex MCPを使ってコードの変更を客観的にレビューします。現在作業中のspec workflowがある場合は仕様に沿ったレビューを実施します。
---

# Codex MCPを使ったコードレビュー

コードの変更内容に対してcodex mcpを使って客観的なレビューを実施します。

## 実行手順

### 1. spec-idの判定

まず、`syou6162-plugin:detect-spec-workflow`サブエージェントを使って、現在の作業に該当するspec-idを判定してください：

```
Taskツールを使って`syou6162-plugin:detect-spec-workflow`サブエージェントを呼び出す。
プロンプトには現在のブランチ名やコミットメッセージから推測されるタスク概要を渡す。
```

### 2. Codex MCPでのレビュー実行

`syou6162-plugin:detect-spec-workflow`サブエージェントの結果に応じて、以下のようにCodex MCPを呼び出してください：

**spec-idが取得できた場合：**

```
mcp__codex__codex ツールを使って以下のプロンプトでレビューを実行：

「main/masterブランチとの差分を日本語でレビューしてください。

このプロジェクトではspec workflowという仕様駆動開発のワークフローを使っています。

- `.spec-workflow/steering/`にプロジェクトの方向性を示すsteeringドキュメントがあります
- `.spec-workflow/specs/<取得したspec-id>/`に現在開発中の機能の仕様とタスクの進行状況があります

これらを読んで、仕様に沿った実装になっているかレビューしてください。」
```

**spec-idが取得できなかった場合：**

```
mcp__codex__codex ツールを使って以下のプロンプトでレビューを実行：

「main/masterブランチとの差分を日本語でレビューしてください。」
```

### 3. レビュー結果の報告

Codex MCPからのレビュー結果をユーザーに報告してください。
