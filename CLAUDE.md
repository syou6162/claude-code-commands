# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## プロジェクト概要

このリポジトリは、Claude Code用のカスタムスラッシュコマンド集です。実行可能コードではなく、Claude Codeに対する指示書（マークダウンファイル）を管理しています。[cccsc](https://github.com/hiragram/cccsc)を使用してインストール・管理されます。

## アーキテクチャ

### コマンド構成
- `semantic_commit.md` - 大きな変更を論理的単位に分割してコミット
- `triage_pr_comments.md` - Pull Requestコメントの対応要否判断
- `self_review_pr.md` - Pull Request提出前の客観的セルフレビュー
- `estimate_pr_size.md` - Pull Requestサイズ見積もりと分割提案
- `update_pr_title_and_description.md` - Pull Requestのタイトル・説明文自動更新
- `optimize_bq_query.md` - BigQueryクエリの性能分析と2倍以上の最適化提案

### 設定ファイル
- `cccsc.json` - コマンド定義とメタデータ管理
- `cccsc-lock.json` - バージョン管理・ロックファイル

## 開発コマンド

### コマンドの追加
新しいコマンドを追加する際：

1. `command-name.md` ファイルを作成
2. `cccsc.json` の `only` 配列に追加：
```json
{
  "name": "command-name",
  "path": "command-name.md", 
  "alias": null
}
```
3. `README.md` の Available Commands セクションを更新

### テスト・検証
```bash
# 特定コマンドのローカルテスト
npx cccsc add syou6162/claude-code-commands/command-name

# 全コマンドのテスト
npx cccsc add syou6162/claude-code-commands
```

## 重要な設計原則

### コマンド設計
- **分析・提案重視**: 実際のコード修正は行わず、判断材料を提供
- **GitHub CLI活用**: `gh`コマンドを使った効率的なワークフロー
- **引数受け渡し**: Claude Codeから`$ARGUMENTS`変数で引数を受け取る

### 外部依存関係
コマンドごとに以下のツールを使用：
- **GitHub CLI (`gh`)** - Pull Request操作（triage_pr_comments, self_review_pr, update_pr_title_and_description）
- **git-sequential-stage** - semantic_commitで使用する専用ツール
- **patchutils (filterdiff)** - パッチ解析用（semantic_commit）
- **BigQuery CLI (`bq`)** - BigQuery操作（optimize_bq_query）

### ファイル更新時の注意
- `cccsc.json`とREADMEの整合性を保つ
- コマンド名は一貫してアンダースコア区切りを使用
- 各コマンドの説明は実際の機能と正確に一致させる