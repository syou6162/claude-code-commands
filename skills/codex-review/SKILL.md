---
name: codex-review
description: Review code changes objectively using Codex CLI. Use when reviewing diffs, checking implementation against plans, or when the user says "レビューして" or "diffを確認して".
allowed-tools: Bash, Write, Edit, Read
model: sonnet
context: fork
---

# Codex CLIを使ったコードレビュー

コードの変更内容に対してcodex CLIを使って客観的なレビューを実施します。

## 実行手順

<procedure>

### 1. デフォルトブランチの取得

まず、リポジトリのデフォルトブランチを取得してください：

```bash
git symbolic-ref refs/remotes/origin/HEAD --short | cut -d/ -f2
```

このコマンドでデフォルトブランチ名（main や master など）を取得し、後続のステップで使用します。

### 2. planファイルの特定

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

### 3. 開発日誌の取得・保存（コンテキストにある場合）

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

### 4. planファイルとの整合性確認

Codexレビュー前に、planファイルを読んで現在の実装（diff）との整合性を確認してください：

1. <plan-file>タグで定義されたパスのファイルを読む（Readツール使用）
2. デフォルトブランチとの差分を確認（`git diff <デフォルトブランチ名>`）
3. planの内容と実装にずれがないか確認
4. ずれがある場合はユーザーに報告

### 5. Codex CLIでのレビュー実行

Bash経由で`codex exec --sandbox read-only`を使ってレビューを実行してください。

<example>

**開発日誌がない場合のコマンド例：**

```bash
echo "<デフォルトブランチ名>ブランチとの差分を日本語でレビューしてください。

以下のファイルを参照して、計画に沿った実装になっているか確認してください：
- planファイル: <plan-fileタグで定義されたパス>

レビューの観点：
- planに記載された変更内容との整合性
- コードの品質（可読性、保守性）
- 潜在的な問題やバグ" | codex exec --sandbox read-only
```

**開発日誌がある場合のコマンド例：**

```bash
echo "<デフォルトブランチ名>ブランチとの差分を日本語でレビューしてください。

以下のファイルを参照して、計画に沿った実装になっているか確認してください：
- planファイル: <plan-fileタグで定義されたパス>
- 開発日誌: <dev-diary-fileタグで定義されたパス>

レビューの観点：
- planに記載された変更内容との整合性
- コードの品質（可読性、保守性）
- 潜在的な問題やバグ
- 開発日誌に記載された開発指針との整合性" | codex exec --sandbox read-only
```

</example>

**重要：** ファイルの内容ではなく、ファイルパスを渡すことで、Codexが直接ファイルを読み取ります。これにより、意図の歪みを防ぎます。

**連続会話：** codex CLIがターミナルに出力するthread-idを確認し、追加の質問がある場合は `echo "追加の質問" | codex exec --sandbox read-only resume <thread-id>` で継続できます。

### 6. レビュー結果の報告

Codex CLIからのレビュー結果をユーザーに報告してください。

</procedure>
