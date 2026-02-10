---
name: execute-plan
description: planモードの計画に基づいて実装を開始する。planファイルからタスクリスト作成→タスク順次実行→各タスク完了時にコミット→最後にcodex-reviewを実行する。
disable-model-invocation: true
allowed-tools: Bash, Read, Skill, TaskCreate, TaskUpdate, TaskList, TaskGet
model: sonnet
context: fork
---

# planモードから実装開始

planモードで作成した計画に基づいて実装を開始します。planファイルの内容を読み取り、タスクリストを作成し、各タスクを順次実行していきます。

## 禁止事項

- タスクを完了せずに勝手に終了しない
- すべてのタスクが完了するまで必ず作業を続ける
- planファイルが存在しない場合は実装を開始しない

## 実行手順

### 1. planファイルの特定

`.claude_work/plans/` ディレクトリ内のplanファイルを特定します。

```bash
ls -t .claude_work/plans/*.md | head -n 1
```

- 最終更新日が最も新しいファイルを使用（`ls -t` で降順ソート、`head -n 1` で最新の1ファイルを取得）
- ファイルが存在しない場合: ユーザーに報告して終了

### 2. planファイルの読み取り

特定したplanファイルをReadツールで読み取ります。

### 3. タスクリストの作成

planファイルの「実行計画」セクションから各ステップを抽出し、TaskCreateでタスクリストを作成します。

各タスクは以下の形式で作成してください：

- **subject**: ステップのタイトル（簡潔に）
- **description**: ステップの詳細内容
- **activeForm**: 「〜を実行中」形式

最後に「Codex レビューを実行」タスクを追加します。

### 4. タスクの順次実行

タスクリストの順番に従って、各タスクを実行します。

各タスクについて：

1. TaskUpdateでタスクを `in_progress` に更新
2. タスクの内容を実行
3. TaskUpdateでタスクを `completed` に更新
4. `Skill(syou6162-plugin:semantic-committing)` を実行してコミット作成

### 5. 最終レビュー

すべての実装タスクが完了したら、最後に `Skill(syou6162-plugin:codex-review)` を実行します。

## 注意事項

- このスキルはオーケストレーションが役割です。実装そのものはメインエージェントが行います。
- 各タスク完了時に必ずコミットを作成してください。
- すべてのタスクが完了するまで作業を続けてください。
