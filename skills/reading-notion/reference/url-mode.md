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

## ページID抽出

<important>

URLの最後の部分（32文字の英数字）がページIDです。

例：
```
URL: https://www.notion.so/myorg/abc123def456ghi789jkl012mno34567
                               ↓
ページID: abc123def456ghi789jkl012mno34567
```

</important>

## 実行手順

<procedure>

### 1. プロパティ取得

**目的**: ページのメタデータ（タイトル、最終更新日時、作成日時、カスタムプロパティなど）を取得します。この情報は後段の要約（「## 出力フォーマット」）でページの概要説明に使用されます。

**取得すべき情報**: 以下の情報を必ず取得してください。

- タイトル
- URL
- 最終更新日時
- 作成日時

```bash
mcptools call API-retrieve-a-page npx -y @notionhq/notion-mcp-server --params '{"page_id":"abc123def456ghi789jkl012mno34567"}' | jq '{title: (.properties | to_entries[] | select(.value.type == "title") | .value.title[0].plain_text), url: .url, last_edited: .last_edited_time, created: .created_time}' | tee .claude/tmp/notion/{page_id}_properties.json > /dev/null
```

<context>

**API: API-retrieve-a-page**

```json
{
  "properties": {
    "filter_properties": {
      "description": "A list of page property value IDs associated with the page. Use this param to limit the response to a specific page property value or values. To retrieve multiple properties, specify each page property ID. For example: `?filter_properties=iAk8&filter_properties=b7dh`.",
      "type": "string"
    },
    "page_id": {
      "description": "Identifier for a Notion page",
      "type": "string"
    }
  },
  "required": [
    "page_id"
  ],
  "type": "object"
}
```

</context>

### 2. ブロック（本文）取得

<important>

**block_idの取得**: Notion APIでは、ページ自体が最上位レベルのブロックとして扱われます。したがって、ステップ1で取得した`page_id`をそのまま`block_id`として使用できます。

</important>

```bash
mcptools call API-get-block-children npx -y @notionhq/notion-mcp-server --params '{"block_id":"abc123def456ghi789jkl012mno34567"}' | tee .claude/tmp/notion/{page_id}_blocks.json > /dev/null
```

<context>

**API: API-get-block-children**

```json
{
  "properties": {
    "block_id": {
      "description": "Identifier for a [block](ref:block)",
      "type": "string"
    },
    "page_size": {
      "default": 100,
      "description": "The number of items from the full list desired in the response. Maximum: 100",
      "format": "int32",
      "type": "integer"
    },
    "start_cursor": {
      "description": "If supplied, this endpoint will return a page of results starting after the cursor provided. If not supplied, this endpoint will return the first page of results.",
      "type": "string"
    }
  },
  "required": [
    "block_id"
  ],
  "type": "object"
}
```

</context>

### 3. Markdownファイル出力

<important>

**ファイル出力は要約ではなく本文を出力**: ブロック内容をAIが要約するのではなく、取得したブロックをそのままMarkdown形式に変換して出力してください。

</important>

**出力先**: `.claude/tmp/notion/{page_id}.md`

**ファイル構成**:

```markdown
---
title: {タイトル}
url: {URL}
last_edited: {最終更新日時}
created: {作成日時}
---

{ブロック内容をMarkdown形式で出力}
```

**手順**:

1. 出力ディレクトリ作成: `mkdir -p .claude/tmp/notion`
2. ステップ1で保存したJSONファイル (`.claude/tmp/notion/{page_id}_properties.json`) を読み込み、プロパティをYAML front-matterとして出力
3. ステップ2で保存したJSONファイル (`.claude/tmp/notion/{page_id}_blocks.json`) を読み込み、ブロックをMarkdown形式に変換して本文として出力
4. Writeツールで `.claude/tmp/notion/{page_id}.md` に書き込み

**ブロックタイプの変換ルール**:

- `paragraph`: 段落テキスト
- `heading_1`: `# 見出し1`
- `heading_2`: `## 見出し2`
- `heading_3`: `### 見出し3`
- `bulleted_list_item`: `- 箇条書き項目`
- `numbered_list_item`: `1. 番号付きリスト項目`
- `code`: ` ```言語名\nコード\n``` `
- `quote`: `> 引用文`

<example>

**出力例** (`.claude/tmp/notion/abc123def456ghi789jkl012mno34567.md`):

```markdown
---
title: "プロジェクトXYZ概要"
url: "https://www.notion.so/abc123def456ghi789jkl012mno34567"
last_edited: "2025-11-15T10:30:00.000Z"
created: "2025-11-01T09:00:00.000Z"
---

# プロジェクトXYZ概要

## プロジェクトの目的

このプロジェクトは、新しい機能XYZを開発し、ユーザー体験を向上させることを目的としています。

## 主なマイルストーン

- 要件定義の完了
- プロトタイプの作成
- ベータ版リリース
- 本番環境へのデプロイ

## 技術スタック

### フロントエンド

1. React 18
2. TypeScript
3. Tailwind CSS

> 注意: すべての依存関係は最新版を使用してください。
```

</example>

### 4. 完了報告

ファイル出力完了後、以下の情報をユーザーに報告してください：

1. 出力ファイルパス
2. ページのメタデータ（タイトル、URL、最終更新日時）
3. **ページの要約（箇条書き5行以内）**: ページの主な内容を簡潔に要約

```
Notionページを以下のファイルに出力しました：
.claude/tmp/notion/{page_id}.md

【メタデータ】
- タイトル: {タイトル}
- URL: {URL}
- 最終更新: {更新日時}

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
