Pull Requestのコメントに対する対応要否をコードベース分析に基づいて判断します

## 分析対象
Pull RequestのURL: $ARGUMENTS

## 実行手順

### 1. Pull Request基本情報の取得
```bash
# Pull RequestのURLから情報を取得
PR_URL="$ARGUMENTS"
echo "=== Pull Request基本情報 ==="

# gh pr viewでPR情報を取得
PR_INFO=$(gh pr view "$PR_URL" --json number,headRepositoryOwner,headRepository,title,author,state,headRefName,baseRefName,changedFiles,additions,deletions,createdAt)

echo "$PR_INFO" | jq -r '
"対象: " + .headRepositoryOwner.login + "/" + .headRepository.name + " PR#" + (.number | tostring),
"タイトル: " + .title,
"作成者: @" + .author.login,
"状態: " + .state,
"ブランチ: " + .headRefName + " → " + .baseRefName,
"変更ファイル数: " + (.changedFiles | tostring),
"追加: +" + (.additions | tostring) + " 削除: -" + (.deletions | tostring),
"作成日: " + (.createdAt | split("T")[0])
'

# 変数の設定
OWNER=$(echo "$PR_INFO" | jq -r '.headRepositoryOwner.login')
REPO=$(echo "$PR_INFO" | jq -r '.headRepository.name')
PR_NUMBER=$(echo "$PR_INFO" | jq -r '.number')

echo ""
```

### 2. 未解決コメントの詳細取得
```bash
echo "=== 未解決コメント一覧 ==="

# PR全体へのコメント（issues/comments API使用）
echo "### PR全体へのコメント"
gh api "/repos/$OWNER/$REPO/issues/$PR_NUMBER/comments" | jq -r '.[] | "ID: \(.id) | @\(.user.login) (\(.created_at | split("T")[0]))", .body, "────────────────────────────────────────────────────────────────────────────────"'

echo ""

# コードレビューコメント（pulls/comments API使用）
echo "### コードレビューコメント"
gh api "/repos/$OWNER/$REPO/pulls/$PR_NUMBER/comments" | jq -r '.[] | "ID: \(.id) | ファイル: \(.path) L\(.line // .original_line)", "@\(.user.login) (\(.created_at | split("T")[0])):", .body, (if .diff_hunk then ("diff:\n" + .diff_hunk) else "" end), "────────────────────────────────────────────────────────────────────────────────"'

echo ""
```

### 3. Pull Request変更内容の詳細分析
```bash
echo "=== 変更内容の詳細分析 ==="

# コミットIDとベースブランチを取得
BASE_SHA=$(echo "$PR_INFO" | jq -r '.base.sha // .baseRefName')
HEAD_SHA=$(echo "$PR_INFO" | jq -r '.head.sha // .headRefName')

# より詳細な情報を取得するためにgh pr viewを使用
PR_COMMITS_INFO=$(gh pr view "$PR_URL" --json commits)
FIRST_COMMIT=$(echo "$PR_COMMITS_INFO" | jq -r '.commits[0].oid')
LAST_COMMIT=$(echo "$PR_COMMITS_INFO" | jq -r '.commits[-1].oid')

echo "ベースブランチ: $BASE_SHA"
echo "Pull Requestのコミット範囲: $FIRST_COMMIT..$LAST_COMMIT"
echo ""

# git diffで詳細な変更内容を取得
echo "### 変更されたファイル一覧"
git diff --name-status "$BASE_SHA..$LAST_COMMIT"
echo ""

echo "### 各ファイルの詳細な変更内容"
git diff --stat "$BASE_SHA..$LAST_COMMIT"
echo ""

# 各ファイルの差分を個別に表示
for file in $(git diff --name-only "$BASE_SHA..$LAST_COMMIT"); do
    echo "============================================================"
    echo "ファイル: $file"
    echo "============================================================"
    git diff "$BASE_SHA..$LAST_COMMIT" -- "$file"
    echo ""
done
```

### 4. 分析指示

**このPull Requestの情報を取得しました。あなたはこのPull Requestの作成者として、コメントに対する対応要否を判断する必要があります。このPRのオーナーシップを持つ立場から、各コメントに対応すべきかどうかを以下の観点で分析してください：**

## A. コードベース全体の理解

まず、リポジトリ全体を把握してください：

1. **プロジェクト構造の分析**
   - ディレクトリ構造、命名規則
   - 使用技術スタック、フレームワーク
   - 設定ファイル（package.json、requirements.txt、go.modなど）

2. **既存パターンの特定**
   - テストファイルの配置規則と命名規則
   - エラーハンドリングの統一パターン
   - ログ出力やデバッグの方法
   - ドキュメンテーションの書き方
   - コーディングスタイル、規約

3. **アーキテクチャの理解**
   - レイヤー構造、依存関係
   - デザインパターンの使用状況
   - 拡張性、保守性への配慮

## B. 変更内容の整合性チェック

今回のPull Requestが既存パターンに従っているかチェック：

- 新機能にテストが含まれているか
- エラーハンドリングが適切か
- ログ出力が既存パターンに従っているか
- ドキュメント更新が必要かどうか
- 依存関係の追加が適切か

## C. 各コメントの詳細分析

**各コメントについて、以下の形式で判断してください：**

### コメントID: [ID番号]
**内容**: [コメントの要約]

#### 対応判断: ✅必要 / ⚠️要検討 / ❌不要

**理由**:
- [客観的な根拠]

**コメント者の意図**:
- [なぜこのコメントをしたのか]

**影響範囲**:
- [修正しない場合のリスク]

**修正方針** (対応必要な場合):
- **場所**: [具体的なファイル、関数]
- **方法**: [どのように修正するか]
- **難易度**: 簡単/中程度/困難
- **時間**: [おおよその所要時間]

**返信内容案**:
- **対応する場合**: [修正内容や対応方針を説明する返信文案]
- **対応しない場合**: [理由を丁寧に説明する返信文案]
- **質問・確認が必要な場合**: [意図を確認するための返信文案]

---

## D. 参考情報の整理

### 対応判断の参考情報
- **既存パターンとの整合性**: [コメントが指摘する内容が既存コードベースの規則に合致するか]
- **技術的妥当性**: [指摘内容が技術的に正しいか]
- **影響範囲**: [対応しない場合の潜在的リスク]

### コメント優先度整理 (参考)
1. [コメントID] - [対応を検討すべき理由]
2. [コメントID] - [対応を検討すべき理由]  
3. [コメントID] - [対応を検討すべき理由]

### ユーザー判断のための補足
- **すぐに対応できそうなもの**: [リスト]
- **時間をかけて検討が必要なもの**: [リスト]
- **コメント者に確認が必要なもの**: [リスト]

**重要**: この分析は判断材料の提供であり、最終的な対応判断はユーザーが行います。コード修正は一切行いません。