---
name: writing-dev-diary
description: 「開発日誌更新」「開発日誌作って」の言及時に使用。esa-llm-scoped-guardで開発日誌を新規作成・更新します。
allowed-tools: Bash, Write, Edit, Read
model: sonnet
context: fork
---

# esa開発日誌の作成・更新

esaに開発日誌を投稿・更新するスキルです。`esa-llm-scoped-guard` CLIを使用して、許可されたカテゴリ配下の記事のみを安全に編集します。

## 禁止事項

<important>

- [ ] JSONファイルは必ず`.claude_work/dev_diary.json`に作成すること（**ファイル名固定**）
- [ ] esa MCPの書き込み系ツール（`create_esa_post`, `update_esa_post`）は使用禁止。このスキルでは`esa-llm-scoped-guard` CLIのみ使用
- [ ] JSONスキーマは必ず`esa-llm-scoped-guard -help`で確認してから生成すること

</important>

## 使用タイミング

<trigger>

以下のトリガーワードで発動します：

- **「開発日誌を作って」**: 検索なしで直接新規作成
- **「開発日誌を更新」+ esaのURL**: 指定されたURLの記事を更新
- **「開発日誌を更新」**（URLなし）: 検索して関連記事を更新、なければ新規作成

</trigger>

## 実行手順

<procedure>

### 手順1: JSONスキーマを確認

最初に必ず最新のJSONスキーマを確認してください：

```bash
esa-llm-scoped-guard -help
```

### 手順2: トリガーによる条件分岐

<decision-criteria name="trigger-flow">

| トリガー | 既存記事取得 | 次の手順 |
|----------|-------------|---------|
| 「開発日誌を作って」 | なし | 手順4 |
| 「開発日誌を更新」+ URL | あり（URL指定） | 手順3（共通サブルーチン） |
| 「開発日誌を更新」（URLなし） | あり（検索） | 手順3（記事あり）/ 手順4（記事なし） |

</decision-criteria>

#### パターンA: 「開発日誌を作って」の場合

検索・取得をスキップして、**手順4（JSON生成）へ直行**してください。

#### パターンB: 「開発日誌を更新」+ URLの場合

1. ユーザーが提示したesaのURLから`post_number`を抽出（例: `https://yasuhisa.esa.io/posts/123` → `123`）

2. `mcp__esa-mcp-server__read_esa_post`で既存記事を取得：
   ```
   postNumber: 123
   ```

3. 既存記事の内容を確認し、**手順3（共通サブルーチン）へ進む**

#### パターンC: 「開発日誌を更新」（URLなし）の場合

1. `mcp__esa-mcp-server__search_esa_posts`で関連記事を検索：
   ```
   query: "in:Claude Code/開発日誌"
   sort: "updated"
   order: "desc"
   perPage: 10
   ```

2. 会話コンテキストから現在のタスクを特定し、検索結果から最も関連性の高い記事を判断

3. **記事あり**: `mcp__esa-mcp-server__read_esa_post`で取得 → 更新モードで**手順3（共通サブルーチン）へ**

4. **記事なし**: 新規作成モードで**手順4へ**

### 手順3: 既存記事内のGitHub URL状態を確認（共通サブルーチン）

既存記事を取得した後、記事内のタスクに含まれるGitHub URL（PR/Issue）の現在の状態と内容を確認します。

1. `body.tasks`から`github_urls`を抽出（URLがなければ**手順4へ**）

2. 各URLの状態と内容をgh CLIで確認

<example name="gh-cli-status-check">

**PRの場合**:
```bash
gh pr view <URL> --json state,isDraft,merged,title,body
```

**Issueの場合**:
```bash
gh issue view <URL> --json state,title,body
```

</example>

3. <github-status-mapping>に従ってGitHub状態を判定

4. <status-mapping>に従ってタスクstatusへのマッピングを記録

5. タスクdescriptionの更新要否を判定し、必要なら更新内容を記録

<decision-criteria name="description-update">

以下のいずれかに該当する場合、タスクdescriptionを更新：

| 判定条件 | 説明 |
|---------|------|
| タイトル差分 | GitHubタイトルがタスクdescriptionと実質的に異なる（スコープの追加・削除） |
| 本文の追加スコープ | bodyに、タスクdescriptionにない追加機能・領域が記載されている |
| 前提/実装変更 | bodyに前提や実装アプローチの変更が記載されている |

</decision-criteria>

更新内容の形式：
- 基本: GitHubタイトルをそのまま使用
- スコープ拡大がある場合: タイトル + "（+ 追加スコープ: ...）"のように本文の要点を短く追記

### 手順4: JSON生成

1. `Write`ツールで`.claude_work/dev_diary.json`を作成（**ファイル名固定、常に上書き**）

2. JSONの構成内容：
   - **新規作成の場合**: `create_new: true`を指定、`post_number`は含めない
   - **更新の場合**: `post_number`を指定、`create_new`は含めない
   - **category**: `Claude Code/開発日誌/yyyy/mm/dd`形式（新規作成時は今日の日付、更新時は既存記事の日付を維持）
   - **body**: `esa-llm-scoped-guard -help`で確認したスキーマに従って構造化形式で作成

3. 会話コンテキストからタスク情報を抽出し、適切な内容を生成

4. **タスクstatusの更新**（手順3を実行した場合）：
   - 既存タスクの`github_urls`に含まれるPR/Issueの状態を確認した結果に基づいて、タスクの`status`を更新
   - <status-mapping>で定義したマッピングに従って、GitHub状態からタスクstatusへ変換
   - 例: PRがマージ済みの場合は`status: "completed"`に更新

5. **タスクdescriptionの更新**（手順3のステップ5で更新が必要と判定された場合）：
   - <description-update>の判定基準に基づいて、タスクの`description`を更新
   - GitHubタイトルを軸に、本文の要点を短く統合して記述
   - 例: "ログイン機能のバグ修正（+ 追加スコープ: セッション管理の改善）"

### 手順5: CLI実行

```bash
esa-llm-scoped-guard -json .claude_work/dev_diary.json
```

### 手順6: 結果報告

- **成功時**: 記事URLをユーザーに報告
- **失敗時**: エラー内容を確認し、JSONを修正して再実行

</procedure>

## 参照データ

<context name="github-status-mapping">

| リソース | 条件 | 判定結果 |
|----------|------|----------|
| PR | merged=true | マージ済み |
| PR | state=OPEN, isDraft=true | ドラフト（WIP） |
| PR | state=OPEN, isDraft=false | レビュー中 |
| PR | state=CLOSED, merged=false | クローズ |
| Issue | state=OPEN | オープン |
| Issue | state=CLOSED | クローズ済み |

</context>

<context name="status-mapping">

| GitHub状態 | タスクstatus |
|------------|--------------|
| PRがマージ済み | `completed` |
| PRがドラフト（WIP） | `in_progress` |
| PRがレビュー中 | `in_review` |
| PRがクローズ（マージなし） | （変更なし） |
| Issueがクローズ | `completed` |
| Issueがオープン | （変更なし） |

</context>
