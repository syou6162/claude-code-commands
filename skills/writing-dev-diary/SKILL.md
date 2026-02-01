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

- YAMLファイルは必ず`.claude_work/dev_diary.yaml`に作成すること（**ファイル名固定**）
- esa MCPの書き込み系ツール（`create_esa_post`, `update_esa_post`）は使用禁止。このスキルでは`esa-llm-scoped-guard` CLIのみ使用
- YAMLスキーマは必ず`esa-llm-scoped-guard -help`で確認してから生成すること

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

### 手順1: YAMLスキーマを確認

最初に必ず最新のYAMLスキーマを確認してください：

```bash
esa-llm-scoped-guard -help
```

### 手順2: トリガーによる条件分岐

<decision-criteria name="trigger-flow">

| トリガー | 既存記事取得 | 次の手順 |
|----------|-------------|---------|
| 「開発日誌を作って」 | なし | 手順4 |
| 「開発日誌を更新」+ URL | あり（URL指定） | 手順3 |
| 「開発日誌を更新」（URLなし） | あり（検索） | 手順3（記事あり）/ 手順4（記事なし） |

</decision-criteria>

#### パターンA: 「開発日誌を作って」の場合

検索・取得をスキップして、**手順4（YAML更新）へ直行**してください。

#### パターンB: 「開発日誌を更新」+ URLの場合

1. ユーザーが提示したesaのURLから`post_number`を抽出（例: `https://yasuhisa.esa.io/posts/123` → `123`）

2. **手順3（YAML取得）へ進む**

#### パターンC: 「開発日誌を更新」（URLなし）の場合

1. `mcp__esa-mcp-server__search_esa_posts`で関連記事を検索：
   ```
   query: "in:Claude Code/開発日誌"
   sort: "updated"
   order: "desc"
   perPage: 10
   ```

2. 会話コンテキストから現在のタスクを特定し、検索結果から最も関連性の高い記事を判断

3. **記事あり**: `post_number`を取得 → 更新モードで**手順3へ**

4. **記事なし**: 新規作成モードで**手順4へ**

### 手順3: 既存記事のYAML取得（更新時のみ）

**目的**: 既存記事からYAMLを取得し、差分ゼロの状態を作る

#### 3.1 fetchコマンドで埋め込みYAMLを取得

1. `fetch`コマンドで既存記事からYAMLを直接取得：
   ```bash
   esa-llm-scoped-guard fetch -post <post_number> | tee .claude_work/dev_diary.yaml > /dev/null
   ```

2. **fetch成功の場合**: YAMLが取得できたので、そのまま**手順4へ進む**（差分ゼロ確認は不要）

3. **fetch失敗の場合**: 古い形式（YAML埋め込みなし）なので、以下のフォールバックフローへ進む

#### 3.2 フォールバック: MCP経由でYAMLを構築（最大5回リトライ）

1. 既存記事の内容から、現状を再現するYAMLを`.claude_work/dev_diary.yaml`に作成
   - `post_number`: 既存記事の番号を指定
   - `category`: 既存記事のカテゴリをそのまま維持
   - `name`: 既存記事のタイトル
   - `body`: 既存記事の本文を構造化形式で完全に再現

2. `validate`で形式確認：
   ```bash
   esa-llm-scoped-guard validate -yaml .claude_work/dev_diary.yaml
   ```

3. `diff`で既存記事との差分確認：
   ```bash
   esa-llm-scoped-guard diff -yaml .claude_work/dev_diary.yaml
   ```

4. **差分がある場合**: YAMLを修正して手順3.2を繰り返す（最大5回まで）

5. **5回試しても差分が残る場合**: ユーザーに報告して判断を仰ぐ

6. **差分ゼロになった場合**: 手順4へ進む

### 手順4: YAML更新

#### 新規作成の場合

1. `Write`ツールで`.claude_work/dev_diary.yaml`を作成（**ファイル名固定、常に上書き**）

2. YAMLの構成内容：
   - `create_new: true`を指定、`post_number`は含めない
   - `category`: `Claude Code/開発日誌/yyyy/mm/dd`形式（今日の日付）
   - `name`: 日付ベースのタイトル
   - `body`: 会話コンテキストから抽出したタスク情報を構造化形式で作成

#### 更新の場合

1. 手順3で取得したYAMLに変更を加える（`Edit`ツール使用）
   - **重要**: 既存の`category`、`name`、`post_number`は維持すること（`create_new`は含めない）
   - タスクの追加・更新
   - GitHub URL状態の反映（下記参照）

2. **GitHub URL状態の確認と反映**:

   a. `body.tasks`から`github_urls`を抽出（URLがなければスキップ）

   b. 各URLの状態と内容をgh CLIで確認

<example name="gh-cli-status-check">

**PRの場合**:
```bash
gh pr view <URL> --json state,isDraft,title,body
```

**Issueの場合**:
```bash
gh issue view <URL> --json state,title,body
```

</example>

   c. <github-status-mapping>に従ってGitHub状態を判定

   d. <status-mapping>に従ってタスクstatusを更新

   e. タスクdescriptionの更新要否を判定（<description-update>参照）

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

### 手順5: 投稿前確認

1. `validate`で形式確認：
   ```bash
   esa-llm-scoped-guard validate -yaml .claude_work/dev_diary.yaml
   ```

2. `diff`で差分が意図通りか確認：
   ```bash
   esa-llm-scoped-guard diff -yaml .claude_work/dev_diary.yaml
   ```

   - 新規作成: 全行が`+`で表示される（全体の最終確認）
   - 更新: 意図した変更のみか確認（消しすぎていないか、意図しない変更がないか）
   - **問題がある場合のみ**ユーザーにdiff結果を表示して確認を求める

### 手順6: 投稿

ユーザーの承認後、投稿を実行：

```bash
esa-llm-scoped-guard post -yaml .claude_work/dev_diary.yaml
```

### 手順7: 結果報告

- **成功時**: 記事URLをユーザーに報告
- **失敗時**: エラー内容を確認し、YAMLを修正して手順5から再実行

</procedure>

## 参照データ

<context name="github-status-mapping">

| リソース | 条件 | 判定結果 |
|----------|------|----------|
| PR | state=MERGED | マージ済み |
| PR | state=OPEN, isDraft=true | ドラフト（WIP） |
| PR | state=OPEN, isDraft=false | レビュー中 |
| PR | state=CLOSED | クローズ |
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
