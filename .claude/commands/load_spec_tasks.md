# load-spec-tasks

spec workflowのtasks.mdから次にやるべきタスクを読み込み、Claude CodeのToDoリストに自己再生産的なタスクサイクルを設定します。

## 概要

このコマンドは最初に1回実行するだけで、以降は自動的にタスクサイクルが回り続けます。
tasks.mdをシングルソースオブトゥルース（信頼できる唯一の情報源）として扱います。

## 実行手順

### Step 1: 現在のspec-idを判定

`detect-spec-workflow`サブエージェントを呼び出して、現在の作業コンテキストから該当するspec-idを判定してください：

```
Taskツールを使用:
- subagent_type: syou6162-plugin:detect-spec-workflow
- prompt: "現在のコンテキストから該当するspec workflowのspec-idを判定してください"
```

**エラーハンドリング**:
- spec-idが見つからない場合: 「該当するspec workflowが見つかりませんでした。spec workflowのセットアップを確認してください。」と表示して終了

### Step 2: tasks.mdの読み取りと次のタスク取得

判定されたspec-idを使って、tasks.mdを読み取ります：

```bash
# Readツールを使用
.spec-workflow/specs/{spec-id}/tasks.md
```

**tasks.mdのパース処理**:

1. マークダウンチェックボックス形式を検索: `- [ ]`, `- [-]`, `- [x]`
2. `- [x]` (完了済み) をスキップ
3. 最初に見つかった `- [ ]` (pending) または `- [-]` (in-progress) を次のタスクとして取得
4. タスク名を抽出（チェックボックス以降のテキスト）

**パース例**:
```markdown
## Tasks

- [x] ユーザー認証機能を実装  ← スキップ
- [x] パスワードリセット機能  ← スキップ
- [ ] メール通知機能を実装    ← これが次のタスク！
- [ ] ログ機能の追加
```
→ 次のタスク: 「メール通知機能を実装」

**エラーハンドリング**:
- tasks.mdが存在しない: 「tasks.mdが見つかりません。spec workflowのPhase 3 (Tasks)を完了させてください。」
- すべてのタスクが完了済み: 「🎉 すべてのタスクが完了しました！spec workflowの実装フェーズは完了です。」と表示して終了

### Step 3: ToDoリストに5ステップのタスクサイクルを追加

`TodoWrite`ツールを使って、以下の5ステップをToDoリストに追加します：

```javascript
TodoWrite({
  todos: [
    {
      content: "tasks.mdで [タスク名] を [-] に変更（in-progressマーク）",
      activeForm: "tasks.mdで [タスク名] を [-] に変更中",
      status: "pending"
    },
    {
      content: "タスク名",  // Step 2で取得した実際のタスク名
      activeForm: "タスク名を実行中",
      status: "pending"
    },
    {
      content: "tasks.mdで [タスク名] を [x] に変更（完了マーク）",
      activeForm: "tasks.mdで [タスク名] を [x] に変更中",
      status: "pending"
    },
    {
      content: "tasks.mdから次のタスクを確認",
      activeForm: "tasks.mdから次のタスクを確認中",
      status: "pending"
    },
    {
      content: "TodoWriteで次のタスクサイクル（5件）をToDoリストに追加",
      activeForm: "TodoWriteで次のタスクサイクルをToDoリストに追加中",
      status: "pending"
    }
  ]
})
```

**重要な実装ガイダンス**:

#### タスク1: tasks.mdでタスクを[-]に変更
このタスクを実行する際は、`Edit`ツールを使って以下のように編集します：
```
old_string: "- [ ] タスク名"
new_string: "- [-] タスク名"
```

#### タスク2: 実際の実装タスク
tasks.mdの`_Prompt`フィールドに記載されている指示に従って実装します。

#### タスク3: tasks.mdでタスクを[x]に変更
`Edit`ツールを使って以下のように編集します：
```
old_string: "- [-] タスク名"
new_string: "- [x] タスク名"
```

#### タスク4: 次のタスクを確認
`Read`ツールでtasks.mdを読み、次の`[ ]`または`[-]`タスクを確認します。

#### タスク5: 次のサイクルを追加
このタスクを実行すると、新しい5ステップのサイクルが自動的に追加されます。

`TodoWrite`で以下を実行：
1. 完了済み（completed）タスクをToDoリストから削除
2. tasks.mdから次のタスクを取得（Step 2と同じ処理）
3. 新しい5ステップのタスクサイクルを生成
4. ToDoリストに追加
