---
name: reading-notion
description: Notion URLが会話に登場した時、またはNotionのコンテンツを検索・取得する必要がある時に使用してください。
allowed-tools: Bash
---

# Reading Notion

NotionページやドキュメントをmcptoolsとNotion MCP Serverを使って読み取り、内容を要約・説明するスキルです。

## 概要

このスキルは2つのモードで動作します：

1. **URL直接モード**: Notion URLが会話に登場した時、自動的にそのページを取得して内容を説明
2. **検索モード**: キーワードでNotionを検索し、結果から選択したページの内容を説明

どちらのモードでも、ページのプロパティ（メタデータ）とブロック（本文）の両方を取得し、AIが要約して自然言語でユーザーに説明します。

<important>

**実行方法**: どちらのモードも必ずTaskツールでgeneral-purposeサブエージェントを起動して実行してください。メインエージェントで直接実行しないこと。

</important>

## モード詳細

<trigger>

### URL直接モード

Notion URLを検出すると自動的に発動します。詳細は **reference/url-mode.md** を参照してください。

**発動例**:
- ユーザーが `https://www.notion.so/myorg/abc123def456ghi789jkl012mno34567` のようなURLを提示
- 会話中にNotion URLが登場

### 検索モード

キーワードでNotion内を検索したい時に発動します。詳細は **reference/search-mode.md** を参照してください。

**発動例**:
- 「データ基盤チームについて教えて」
- 「Notionでプロジェクト〇〇を検索して」
- 特定のキーワードに関する情報を求められた時

</trigger>
