---
allowed-tools: Bash(git config:*), Bash(git rev-parse:*), Bash(git log:*), Bash(git filter-branch:*), Bash(git push:*), Bash(git branch:*), Bash(git reset:*), Bash(perl:*), Bash(echo:*), Bash(which:*), Bash([:*), Bash(test:*)
description: "Claude Codeが作成したコミットのAuthor情報と署名を本来のユーザー情報に書き換えます。"
---

# Claude Codeが作成したコミットのメタデータを書き換える

ブランチ内のClaude Codeが作成したコミットから、Claude由来のAuthor情報と自動追加されたコミットメッセージの署名部分を削除し、本来のユーザー情報に修正します。

## 使い方

```
/rewrite-claude-commits
```

## 概要

Claude Codeでコミットを作成すると、以下の情報が自動的に追加されます：

- **Author情報**: `Claude <noreply@anthropic.com>`
- **コミットメッセージの署名**:
  - `🤖 Generated with [Claude Code](https://claude.com/claude-code)`
  - `Co-Authored-By: Claude <noreply@anthropic.com>`

このコマンドは、現在のブランチ内のこれらの情報を一括で修正し、ユーザー本人が作成したコミットに書き換えます。

### 主な特徴

- **Author情報の修正**: `git config` で設定されたユーザー情報に置き換え
- **コミットメッセージの保護**: 署名部分のみを削除し、本来のコミットメッセージは一切変更しない
- **ブランチ全体を処理**: ベースブランチから現在のブランチまでのClaude製コミットをすべて処理
- **安全な書き換え**: `git filter-branch` による確実な履歴書き換え

### 重要な注意事項

⚠️ **このコマンドはコミット履歴を書き換えます**

- すでにリモートにプッシュ済みのブランチで実行した場合、force pushが必要になります
- 他の人と共有しているブランチでは実行しないでください
- 実行前にブランチのバックアップを取ることを推奨します

## 動作原理

### 処理フロー

1. **ユーザー情報の取得**: `git config` から本来のユーザー名とメールアドレスを取得
2. **対象コミットの特定**: ベースブランチから現在のブランチまでのClaude製コミットを検索
3. **Author情報の書き換え**: `GIT_AUTHOR_*` および `GIT_COMMITTER_*` 環境変数を修正
4. **コミットメッセージの整形**: 署名部分のみを削除、他の内容は保持

### 技術的詳細

#### Author情報の書き換え

`git filter-branch --env-filter` を使用して、Author情報を置き換えます：

```bash
git filter-branch --env-filter '
if [ "$GIT_AUTHOR_EMAIL" = "noreply@anthropic.com" ]; then
    export GIT_AUTHOR_NAME="User Name"
    export GIT_AUTHOR_EMAIL="user@example.com"
    export GIT_COMMITTER_NAME="User Name"
    export GIT_COMMITTER_EMAIL="user@example.com"
fi
'
```

#### コミットメッセージの整形

`--msg-filter` を使用して、以下のパターンを削除します：

- 空行 + `🤖 Generated with [Claude Code](...)` + 空行 + `Co-Authored-By: Claude <noreply@anthropic.com>`

**重要**: 上記のパターンに一致する部分のみを削除し、それ以外のコミットメッセージ内容は一切変更しません。

## 実行手順

### Step 1: 事前確認

```bash
# 現在のブランチとユーザー情報を確認
echo "現在のブランチ: $(git rev-parse --abbrev-ref HEAD)"
echo "ユーザー名: $(git config user.name)"
echo "メールアドレス: $(git config user.email)"

# Claude製コミットの存在確認
BASE_BRANCH=$(git rev-parse --abbrev-ref @{upstream} 2>/dev/null || echo "origin/main")
CLAUDE_COMMITS=$(git log --author="Claude <noreply@anthropic.com>" --format="%h %s" $BASE_BRANCH..HEAD)

if [ -z "$CLAUDE_COMMITS" ]; then
  echo ""
  echo "Claude Codeが作成したコミットは見つかりませんでした"
  exit 0
else
  echo ""
  echo "以下のコミットを書き換えます:"
  echo "$CLAUDE_COMMITS"
fi
```

### Step 2: ユーザー情報の取得

```bash
# git configからユーザー情報を取得
USER_NAME=$(git config user.name)
USER_EMAIL=$(git config user.email)

# ユーザー情報が設定されているか確認
if [ -z "$USER_NAME" ] || [ -z "$USER_EMAIL" ]; then
  echo "エラー: git configにユーザー情報が設定されていません"
  echo ""
  echo "以下のコマンドで設定してください："
  echo "  git config user.name \"Your Name\""
  echo "  git config user.email \"your.email@example.com\""
  exit 1
fi

echo ""
echo "新しいAuthor情報:"
echo "  Name: $USER_NAME"
echo "  Email: $USER_EMAIL"
```

### Step 3: コミット履歴の書き換え

```bash
# ベースブランチを特定
BASE_BRANCH=$(git rev-parse --abbrev-ref @{upstream} 2>/dev/null || echo "origin/main")

echo ""
echo "コミット履歴を書き換えています..."
echo ""

# git filter-branchで履歴を書き換え
git filter-branch --force \
  --env-filter '
if [ "$GIT_AUTHOR_EMAIL" = "noreply@anthropic.com" ]; then
    export GIT_AUTHOR_NAME="'"$USER_NAME"'"
    export GIT_AUTHOR_EMAIL="'"$USER_EMAIL"'"
    export GIT_COMMITTER_NAME="'"$USER_NAME"'"
    export GIT_COMMITTER_EMAIL="'"$USER_EMAIL"'"
fi
' \
  --msg-filter '
perl -0777 -pe "s/\n*🤖 Generated with \[Claude Code\].*?\n*Co-Authored-By: Claude <noreply\@anthropic\.com>\s*\$//s"
' \
  $BASE_BRANCH..HEAD

echo ""
echo "履歴の書き換えが完了しました"
```

### Step 4: 結果の確認

```bash
echo ""
echo "=== 書き換え後のコミット一覧 ==="
git log --format="%h %an <%ae> %s" $BASE_BRANCH..HEAD

echo ""
echo "=== 最新コミットの詳細 ==="
git log -1 --format="%H%n%an <%ae>%n%at%n%B"
```

### Step 5: リモートへのプッシュ

```bash
echo ""
echo "変更をリモートにプッシュする場合は、force pushが必要です："
echo "  git push --force-with-lease"
echo ""
echo "⚠️  警告: 他の人と共有しているブランチの場合は実行しないでください"
```

## 環境要件

### 必須ツール

```bash
# Perlの確認（コミットメッセージの整形に使用）
which perl

# gitのバージョン確認（filter-branchをサポートするバージョン）
git --version
```

### 推奨環境

- Git 2.0以降
- Perl 5.x以降（ほとんどのシステムで標準インストール済み）

## 使用例

### 基本的な使用例

```bash
# ブランチ上でコミットを作成（Claude Codeが自動的に署名を追加）
$ git log -1 --format="%an <%ae>%n%s%n%n%b"
Claude <noreply@anthropic.com>
feat: add new feature

Implement the new feature for users

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>

# rewrite_claude_commitsを実行
$ /rewrite-claude-commits

# 結果確認
$ git log -1 --format="%an <%ae>%n%s%n%n%b"
Yasuhisa Yoshida <yoshida@example.com>
feat: add new feature

Implement the new feature for users
```

### 複数コミットの書き換え

```bash
# 複数のClaude製コミットが存在する場合
$ git log --author="Claude" --oneline
a1b2c3d docs: update README
b2c3d4e feat: add authentication
c3d4e5f fix: resolve bug

# rewrite_claude_commitsを実行すると、すべてのコミットが一括で書き換えられる
$ /rewrite-claude-commits

$ git log --format="%h %an %s" origin/main..HEAD
a1b2c3d Yasuhisa Yoshida docs: update README
b2c3d4e Yasuhisa Yoshida feat: add authentication
c3d4e5f Yasuhisa Yoshida fix: resolve bug
```

## ベストプラクティス

1. **実行タイミング**: Pull Request作成前に実行することを推奨
2. **バックアップ**: 重要なブランチの場合、事前にバックアップブランチを作成
   ```bash
   git branch backup-$(git rev-parse --abbrev-ref HEAD)
   ```
3. **force pushの確認**: リモートにプッシュ済みの場合、force pushが必要になることを理解する
4. **他の開発者との調整**: 共有ブランチでは実行しない、または事前に調整する

## トラブルシューティング

### ユーザー情報が設定されていない

```bash
# エラーメッセージ
エラー: git configにユーザー情報が設定されていません

# 対処法
git config user.name "Your Name"
git config user.email "your.email@example.com"

# グローバルに設定する場合
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```

### filter-branchの警告メッセージ

```bash
# 警告: git filter-branch has a glut of gotchas...
# これは通常の警告メッセージです。処理は正常に完了しています。

# より新しいgit-filter-repoツールを使いたい場合
# https://github.com/newren/git-filter-repo を参照
```

### Perlが見つからない

```bash
# macOS/Linux
brew install perl  # macOS
apt-get install perl  # Ubuntu/Debian
yum install perl  # CentOS/RHEL

# Perlは多くのシステムで標準インストールされています
```

### 既にforce pushした後にさらに修正が必要な場合

```bash
# バックアップブランチから再スタート
git reset --hard backup-<branch-name>

# 再度rewrite_claude_commitsを実行
/rewrite-claude-commits
```

### 一部のコミットだけを書き換えたい場合

```bash
# 範囲を指定してfilter-branchを実行
# 例: 最新の3コミットだけを対象にする
git filter-branch --force \
  --env-filter '...' \
  --msg-filter '...' \
  HEAD~3..HEAD
```

## コミットメッセージの保護について

このコマンドは、以下のパターンに一致する部分**のみ**を削除します：

```
（本来のコミットメッセージ）

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
```

以下のような内容は**絶対に変更されません**：

- コミットメッセージのタイトル（1行目）
- コミットメッセージの本文
- 他のCo-Authored-By行（Claudeではない共同作成者）
- その他のメタデータやトレーラー

## 参考資料

- [git filter-branch documentation](https://git-scm.com/docs/git-filter-branch)
- [git-filter-repo (推奨される代替ツール)](https://github.com/newren/git-filter-repo)
- [Git での履歴の書き換え](https://git-scm.com/book/ja/v2/Git-%E3%81%AE%E3%81%95%E3%81%BE%E3%81%96%E3%81%BE%E3%81%AA%E3%83%84%E3%83%BC%E3%83%AB-%E6%AD%B4%E5%8F%B2%E3%81%AE%E6%9B%B8%E3%81%8D%E6%8F%9B%E3%81%88)
