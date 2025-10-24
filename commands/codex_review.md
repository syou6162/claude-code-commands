---
description: Codex MCPを使ってコードの変更を客観的にレビューします。現在作業中のspec workflowがある場合は仕様に沿ったレビューを実施します。
---

# Codex MCPを使ったコードレビュー

コードの変更内容に対してcodex mcpを使って客観的なレビューを実施します。

## 動作内容

spec workflowを使っている場合：

```
「main/masterブランチとの差分を日本語でレビューしてください。

このプロジェクトではspec workflowという仕様駆動開発のワークフローを使っています。

- `.spec-workflow/steering/`にプロジェクトの方向性を示すsteeringドキュメントがあります
- `.spec-workflow/specs/<現在作業中のspec-id>/`に現在開発中の機能の仕様とタスクの進行状況があります

これらを読んで、仕様に沿った実装になっているかレビューしてください。」
```

spec workflowを使っていない場合：

```
「main/masterブランチとの差分を日本語でレビューしてください。」
```
