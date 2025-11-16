# 検索モード

「notionでxyzを探して」という形で発動し、キーワードでNotionを検索して結果から選択したページの内容を説明します。

## 発動条件

<trigger>

以下のようなリクエストで発動します：

- 「Notionでプロジェクト〇〇を検索して」
- Notionに関する質問であることが明らかな場合

</trigger>

## 実行フロー

<procedure>

### 1. キーワード検索と結果表示

jqを使って検索結果を箇条書き形式で整形します：

```bash
mcptools call API-post-search npx -y @notionhq/notion-mcp-server --params '{"query":"プロジェクトXYZ","page_size":10}' | jq -r '.results[] | "- \(if .title then .title[0].text.content else (.properties | to_entries[] | select(.value.type == "title") | .value.title[0].text.content) end)\n  - \(.url)"'
```

**出力例**:
```
- プロジェクトXYZ概要
  - https://www.notion.so/abc123de...
- XYZ実装ガイド
  - https://www.notion.so/def456gh...
- XYZ関連メモ
  - https://www.notion.so/ghi789jk...
- プロジェクトXYZ
  - https://www.notion.so/jkl012mn...
```

<context>

**API: API-post-search**

```json
{
  "properties": {
    "filter": {
      "additionalProperties": true,
      "description": "A set of criteria, `value` and `property` keys, that limits the results to either only pages or only databases. Possible `value` values are `\"page\"` or `\"database\"`. The only supported `property` value is `\"object\"`.",
      "properties": {
        "property": {
          "description": "The name of the property to filter by. Currently the only property you can filter by is the object type.  Possible values include `object`.   Limitation: Currently the only filter allowed is `object` which will filter by type of object (either `page` or `database`)",
          "type": "string"
        },
        "value": {
          "description": "The value of the property to filter the results by.  Possible values for object type include `page` or `database`.  **Limitation**: Currently the only filter allowed is `object` which will filter by type of object (either `page` or `database`)",
          "type": "string"
        }
      },
      "type": "object"
    },
    "page_size": {
      "default": 100,
      "description": "The number of items from the full list to include in the response. Maximum: `100`.",
      "format": "int32",
      "type": "integer"
    },
    "query": {
      "description": "The text that the API compares page and database titles against.",
      "type": "string"
    },
    "sort": {
      "additionalProperties": true,
      "description": "A set of criteria, `direction` and `timestamp` keys, that orders the results. The **only** supported timestamp value is `\"last_edited_time\"`. Supported `direction` values are `\"ascending\"` and `\"descending\"`. If `sort` is not provided, then the most recently edited results are returned first.",
      "properties": {
        "direction": {
          "description": "The direction to sort. Possible values include `ascending` and `descending`.",
          "type": "string"
        },
        "timestamp": {
          "description": "The name of the timestamp to sort against. Possible values include `last_edited_time`.",
          "type": "string"
        }
      },
      "type": "object"
    },
    "start_cursor": {
      "description": "A `cursor` value returned in a previous response that If supplied, limits the response to results starting after the `cursor`. If not supplied, then the first page of results is returned. Refer to [pagination](https://developers.notion.com/reference/intro#pagination) for more details.",
      "type": "string"
    }
  },
  "type": "object"
}
```

</context>

### 2. ユーザーへの提示

AIは検索結果を以下の形式でユーザーに提示します：

```
Notionで「プロジェクトXYZ」を検索した結果、以下のページが見つかりました：

- **プロジェクトXYZ**
  - https://www.notion.so/jkl012mn...
- **XYZ関連メモ**
  - https://www.notion.so/ghi789jk...
- **XYZ実装ガイド**
  - https://www.notion.so/def456gh...
```

</procedure>
