GitHub ActionsのCI実行状態を監視し、失敗したジョブの原因分析と修正案を提案します

## 監視対象
Pull RequestのURL、ブランチ名、またはコミットハッシュ: $ARGUMENTS

## 実行手順

### 1. CI実行状態の取得

```bash
# 引数の解析
TARGET="$ARGUMENTS"

# $ARGUMENTSが空の場合、現在のブランチを使用
if [ -z "$TARGET" ]; then
  CURRENT_BRANCH=$(git branch --show-current 2>/dev/null || echo "")
  if [ -z "$CURRENT_BRANCH" ]; then
    echo "エラー: Pull RequestのURL、ブランチ名、またはコミットハッシュが指定されていません。"
    echo "また、現在のブランチも検出できませんでした。"
    exit 1
  fi
  echo "現在のブランチを使用します: $CURRENT_BRANCH"
  TARGET="$CURRENT_BRANCH"
fi

# Pull Request URLかどうかを判定
if [[ "$TARGET" =~ ^https://github.com/ ]]; then
  echo "=== Pull Request情報取得 ==="
  PR_INFO=$(gh pr view "$TARGET" --json number,headRefName,headRepository,headRepositoryOwner)
  BRANCH=$(echo "$PR_INFO" | jq -r '.headRefName')
  OWNER=$(echo "$PR_INFO" | jq -r '.headRepositoryOwner.login')
  REPO=$(echo "$PR_INFO" | jq -r '.headRepository.name')
  PR_NUMBER=$(echo "$PR_INFO" | jq -r '.number')

  echo "Pull Request #$PR_NUMBER"
  echo "ブランチ: $BRANCH"
  echo "リポジトリ: $OWNER/$REPO"
  echo ""
else
  # ブランチ名またはコミットハッシュとして扱う
  BRANCH="$TARGET"
  OWNER=$(gh repo view --json owner --jq '.owner.login')
  REPO=$(gh repo view --json name --jq '.name')

  # Pull Requestが存在するか確認
  PR_NUMBER=$(gh pr list --head "$BRANCH" --json number --jq '.[0].number' 2>/dev/null || echo "")

  echo "=== ブランチ/コミット情報 ==="
  echo "対象: $BRANCH"
  echo "リポジトリ: $OWNER/$REPO"
  if [ -n "$PR_NUMBER" ]; then
    echo "関連Pull Request: #$PR_NUMBER"
  fi
  echo ""
fi

# 最新のコミットハッシュを取得
if [[ "$TARGET" =~ ^https://github.com/ ]]; then
  COMMIT_SHA=$(gh pr view "$TARGET" --json headRefOid --jq '.headRefOid')
else
  # ブランチから最新コミットを取得
  COMMIT_SHA=$(git rev-parse "$BRANCH" 2>/dev/null || echo "$BRANCH")
fi

echo "=== CI実行状態の概要 ==="
echo "対象コミット: $COMMIT_SHA"
echo ""

# GitHub Actionsのワークフロー実行状態を取得
echo "### ワークフロー実行一覧"
gh run list --commit "$COMMIT_SHA" --json databaseId,name,status,conclusion,headBranch,event,createdAt,updatedAt,url --limit 20

echo ""
echo "=== 詳細なジョブステータス ==="

# 各ワークフロー実行の詳細を取得
RUN_IDS=$(gh run list --commit "$COMMIT_SHA" --json databaseId --jq '.[].databaseId' --limit 10)

for RUN_ID in $RUN_IDS; do
  echo ""
  echo "--- ワークフロー実行 ID: $RUN_ID ---"

  # ワークフロー実行の基本情報
  gh run view "$RUN_ID" --json name,status,conclusion,headBranch,event,createdAt,updatedAt,url

  echo ""
  echo "### ジョブ一覧"
  # 各ジョブのステータス
  gh run view "$RUN_ID" --json jobs --jq '.jobs[] | {name: .name, status: .status, conclusion: .conclusion, startedAt: .startedAt, completedAt: .completedAt}'

  # 失敗したジョブのログを取得
  echo ""
  echo "### 失敗したジョブのログ"
  FAILED_JOBS=$(gh run view "$RUN_ID" --json jobs --jq '.jobs[] | select(.conclusion == "failure") | .databaseId')

  for JOB_ID in $FAILED_JOBS; do
    echo ""
    echo "==== ジョブ ID: $JOB_ID ===="
    JOB_NAME=$(gh run view "$RUN_ID" --json jobs --jq ".jobs[] | select(.databaseId == $JOB_ID) | .name")
    echo "ジョブ名: $JOB_NAME"
    echo ""
    echo "--- ログ出力 ---"
    gh run view --job "$JOB_ID" --log 2>&1 || echo "ログの取得に失敗しました"
    echo ""
  done
done

# 進行中のジョブがある場合
echo ""
echo "=== 進行中のジョブ ==="
gh run list --commit "$COMMIT_SHA" --json databaseId,name,status,conclusion --jq '.[] | select(.status == "in_progress" or .status == "queued")'

echo ""
echo "=== CIチェックサマリー ==="
# Pull Requestのチェック状態（PR URLが指定された場合のみ）
if [[ "$TARGET" =~ ^https://github.com/ ]] || [ -n "$PR_NUMBER" ]; then
  if [ -n "$PR_NUMBER" ]; then
    gh pr checks "$PR_NUMBER"
  else
    gh pr checks "$TARGET"
  fi
fi
```

### 2. CI失敗原因の分析指示

**上記のCI実行情報を取得しました。あなたはこのCI実行結果を分析し、失敗原因と修正案を提案する必要があります。以下の観点で分析してください：**

## A. CI実行状態の整理

### 1. 実行状態の概要
- **全体のステータス**: すべて成功 / 一部失敗 / すべて失敗 / 進行中
- **失敗したワークフロー数**: [件数]
- **失敗したジョブ数**: [件数]
- **進行中のジョブ**: あり/なし

### 2. 失敗したジョブの一覧
各失敗ジョブについて：
- **ワークフロー名**: [名前]
- **ジョブ名**: [名前]
- **失敗時刻**: [タイムスタンプ]

## B. エラーログの詳細分析

失敗した各ジョブについて、以下の形式で分析してください：

### ジョブ: [ジョブ名]

#### エラーの要約
- **エラータイプ**: テスト失敗/ビルド失敗/リント失敗/タイムアウト/その他
- **失敗したステップ**: [ステップ名]
- **エラーメッセージ**: [主要なエラーメッセージを抜粋]

#### 根本原因の分析
**原因**: [なぜこのエラーが発生したか]

**関連するコード変更**:
- [このPR/コミットで変更されたファイルのうち、エラーに関連するもの]

**トリガー条件**:
- [どのような条件でこのエラーが発生するか]

#### 影響範囲
- **重要度**: 🔴高（リリースブロッカー） / 🟡中（修正推奨） / 🟢低（軽微）
- **影響するコンポーネント**: [リスト]
- **他のテストへの影響**: あり/なし

#### 修正案

**推奨される修正方法**:
1. [具体的な修正手順1]
2. [具体的な修正手順2]

**修正対象ファイル**:
- `ファイル名:行番号` - [変更内容]

**修正の難易度**: 簡単 / 中程度 / 困難

**所要時間の見積もり**: [おおよその時間]

**代替案** (複数の修正方法がある場合):
- 方法1: [説明]
- 方法2: [説明]

**修正後の確認方法**:
- [ローカルで確認する方法]
- [CIで確認するポイント]

---

## C. パターン分析

### 1. 失敗の共通点
- [複数のジョブが失敗している場合、共通する原因はあるか]
- [特定の変更ファイルに起因するか]

### 2. 環境依存性
- [特定のOS、言語バージョン、依存パッケージに関連するか]
- [タイミング依存のテスト失敗か]

### 3. 過去の類似エラー
- [このようなエラーが過去にもあったか（git logやIssue履歴から推測）]

## D. 優先順位付き対応リスト

修正の優先順位をつけて整理：

### 🔴 最優先（即座に対応すべき）
1. [ジョブ名] - [理由]

### 🟡 優先（できるだけ早く対応）
1. [ジョブ名] - [理由]

### 🟢 低優先（時間があれば対応）
1. [ジョブ名] - [理由]

## E. CI改善の提案（オプション）

失敗の再発防止や、CI自体の改善案があれば提案：
- [CI設定の改善案]
- [テストの安定化方法]
- [ビルドキャッシュの最適化]

## F. 次のアクション

**ユーザーが取るべき次のステップ**:
1. [最初に行うべきこと]
2. [次に行うべきこと]
3. [確認すべきこと]

**すぐに実行できるコマンド** (ローカルでの確認用):
```bash
# [説明]
[コマンド]
```

---

**重要**: この分析は判断材料の提供であり、最終的な対応判断はユーザーが行います。コード修正の実行は行わず、分析と提案のみを提供してください。

## G. 補足情報

### CIの再実行について
- **再実行が有効なケース**: [一時的なネットワークエラー、外部サービスの問題など]
- **再実行が無効なケース**: [コード起因の問題、設定ミスなど]

### 関連ドキュメント
失敗の解決に役立ちそうなドキュメントや参考リンクがあれば提示してください。
