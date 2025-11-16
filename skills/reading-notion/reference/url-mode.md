# URL直接モード

Notion URLを検出すると自動的に発動し、そのページの内容を取得して要約します。

## URL検出パターン

以下のようなNotion URLを検出します：

```
https://www.notion.so/{workspace}/{page-id}
https://www.notion.so/{page-id}
```

例：
```
https://www.notion.so/myorg/abc123def456ghi789jkl012mno34567
https://www.notion.so/abc123def456ghi789jkl012mno34567
```

## 実行手順

<procedure>

### 1. Markdown変換と出力

`notion-to-md`コマンドを使ってNotion URLを直接Markdownに変換します。

**実行コマンド**:

```bash
mkdir -p .claude/tmp/notion && notion-to-md {notion_url} | tee .claude/tmp/notion/{page_id}.md > /dev/null
```

**説明**:
- `{notion_url}`: ユーザーが提示したNotion URL（例: `https://www.notion.so/workspace/Page-title-abc123...`）
- `{page_id}`: URLの最後の32文字（例: `abc123def456ghi789jkl012mno34567`）
- `tee`を使うことで許可なしでファイル保存が可能

### 2. Markdownファイルの読み込みと要約報告

出力されたMarkdownファイルをReadツールで読み込み、内容を要約してユーザーに報告します。

**手順**:

1. Readツールで `.claude/tmp/notion/{page_id}.md` を読み込む
2. ページの主な内容を要約してユーザーに報告（箇条書き5行以内）

**報告フォーマット**:

```
Notionページを以下のファイルに出力しました：
.claude/tmp/notion/{page_id}.md

【ページの要約】
- {要約ポイント1}
- {要約ポイント2}
- {要約ポイント3}
- {要約ポイント4}
- {要約ポイント5}
```

</procedure>

## エラーハンドリング

### ページが見つからない場合

```
申し訳ございません。指定されたNotion URLからページを取得できませんでした。
以下の可能性があります：
- ページが削除されている
- アクセス権限がない
- URLが正しくない
```

### API呼び出しが失敗した場合

```
Notion APIへの接続に失敗しました。しばらくしてから再度お試しください。
```
