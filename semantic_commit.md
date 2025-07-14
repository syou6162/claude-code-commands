# 意味のある最小の単位でcommitする

大きな変更を論理的な単位に分割してコミットします。LLMがgit diffを分析して意味のある最小単位を提案し、`git-sequential-stage`ツールによる自動化された逐次ステージングでコミットします。

## 使い方

```
/semantic-commit
```

## 概要

このコマンドは、LLM Agentが大規模な変更を意味のある最小単位に分割してコミットするためのツールです。**専用ツール`git-sequential-stage`**により、1つのファイル内の異なる意味的変更も安全かつ確実に分割できます。

### 主な特徴

- **完璧な意味的分割**: ファイル内の異なる変更を確実に分離
- **複数ファイルのサポート**: 複数ファイルにまたがる変更を一度に処理
- **シンプルな実行**: 複雑なループロジックは`git-sequential-stage`が内部で処理
- **堅牢性**: Go言語で実装された専用ツールによる安定動作

## 動作原理

### 基本フロー

1. **リポジトリルートに移動**: `git rev-parse --show-toplevel`で確認後、移動
2. **差分取得**: `git diff HEAD`でコンテキスト付きのhunkリストを取得
3. **LLM分析**: hunk単位で意味的グループを決定
4. **自動ステージング**: `git-sequential-stage`が選択したhunkを安全にステージング
5. **コミット実行**: ステージング完了後、`git commit`を実行
6. **繰り返し**: 残りの変更で継続

### 技術的詳細

#### git-sequential-stageツール

`git-sequential-stage`は、hunk単位の部分的なステージングを自動化するためのGoで実装された専用ツールです（[GitHub: syou6162/git-sequential-stage](https://github.com/syou6162/git-sequential-stage)）。

以下の複雑な処理を内部でカプセル化しています：
- `git patch-id`によるhunkの一意識別
- 逐次ステージングによる行番号ズレの回避
- パッチIDベースの照合による確実なhunk特定
- `filterdiff`を使った複数ファイルパッチからの正確なhunk抽出

使用方法：
```bash
# 基本的な使い方
git-sequential-stage -patch="path/to/changes.patch" -hunk="src/main.go:1,3,5"

# 複数ファイルの場合（複数の-hunkフラグを指定）
git-sequential-stage -patch="path/to/changes.patch" \
  -hunk="src/main.go:1,3" \
  -hunk="src/utils.go:2,4"

# 引数の説明：
# -patch: パッチファイルのパス
# -hunk: file:numbers形式で指定（例：src/main.go:1,3）
#        ファイル名とカンマ区切りのhunk番号をコロンで連結
```

## 実行手順

### Step 0: リポジトリルートに移動

```bash
# リポジトリルートを確認
REPO_ROOT=$(git rev-parse --show-toplevel)
echo "リポジトリルート: $REPO_ROOT"

# リポジトリルートに移動
cd "$REPO_ROOT"
```

### Step 1: 差分を取得

```bash
# .claude/tmpディレクトリは既に存在するため、直接ファイルを作成可能

# 新規ファイル（untracked files）をintent-to-addで追加
git ls-files --others --exclude-standard | xargs git add -N

# コンテキスト付きの差分を取得（より安定した位置特定のため）
# 新規ファイルも含めて取得される
git diff HEAD > .claude/tmp/current_changes.patch
```

### Step 2: LLM分析

LLMが**hunk単位**で変更を分析し、最初のコミットに含めるhunkを決定：

- **hunkの内容を読み取る**: 各hunkが何を変更しているか理解
- **意味的グループ化**: 同じ目的の変更（バグ修正、リファクタリング等）をグループ化
- **コミット計画**: どのhunkをどのコミットに含めるか決定

必要に応じて、総hunk数を確認：
```bash
# 全体のhunk数を確認
grep -c "^@@" .claude/tmp/current_changes.patch

# 各ファイルごとのhunk数を確認
git diff HEAD --name-only | xargs -I {} sh -c 'printf "%s: " "{}"; git diff HEAD {} | grep -c "^@@"'
```

例：
```bash
# LLMの分析結果
# - コミット1（fix）: 
#   - src/calculator.py: hunk 1, 3, 5（ゼロ除算エラーの修正）
#   - src/utils.py: hunk 2（関連するユーティリティ関数の修正）
# - コミット2（refactor）: 
#   - src/calculator.py: hunk 2, 4（計算ロジックの最適化）

# 最初のコミット用の設定
COMMIT_MSG="fix: ゼロ除算エラーを修正

計算処理で分母が0の場合の適切なエラーハンドリングを追加"
```

### Step 3: 自動ステージング

選択したhunkを`git-sequential-stage`で自動的にステージング：

```bash
# git-sequential-stageを実行（内部で逐次ステージングを安全に処理）
# 単一ファイルの場合
git-sequential-stage -patch=".claude/tmp/current_changes.patch" -hunk="src/calculator.py:1,3,5"

# 複数ファイルの場合
git-sequential-stage -patch=".claude/tmp/current_changes.patch" \
  -hunk="src/calculator.py:1,3,5" \
  -hunk="src/utils.py:2"

# コミット実行
git commit -m "$COMMIT_MSG"
```

### Step 4: 繰り返し

残りの変更に対して同じプロセスを繰り返し：

```bash
# 残りの差分を確認
if [ $(git diff HEAD | wc -l) -gt 0 ]; then
  echo "残りの変更を処理します..."
  # Step 1に戻る
fi
```

### Step 5: 最終確認

```bash
# すべての変更がコミットされたか確認
if [ $(git diff HEAD | wc -l) -eq 0 ]; then
  echo "すべての変更がコミットされました"
else
  echo "警告: まだコミットされていない変更があります"
  git status
fi
```

## 環境要件

```bash
# 必要なツールの確認
which git-sequential-stage  # 専用ツール
which filterdiff           # patchutilsパッケージ（差分解析用）
```

インストールが必要な場合：
- `git-sequential-stage`: [GitHub: syou6162/git-sequential-stage](https://github.com/syou6162/git-sequential-stage)のREADMEを参照
- patchutils: `brew install patchutils` (macOS) / `apt-get install patchutils` (Ubuntu/Debian)

## 使用例

### ファイル内の意味的分割（最重要例）

```
変更内容: src/calculator.py
- hunk 1: 10行目のゼロ除算チェック追加
- hunk 2: 25-30行目の計算アルゴリズム最適化
- hunk 3: 45行目の別のゼロ除算エラー修正
- hunk 4: 60-80行目の内部構造リファクタリング
- hunk 5: 95行目のゼロ除算時のログ出力追加

↓ 分割結果

コミット1: fix: ゼロ除算エラーを修正
git-sequential-stage -patch=".claude/tmp/current_changes.patch" -hunk="src/calculator.py:1,3,5"

コミット2: refactor: 計算ロジックの最適化
git-sequential-stage -patch=".claude/tmp/current_changes.patch" -hunk="src/calculator.py:2,4"
```

### 複雑な変更パターン

```
変更内容:
- src/auth.py: 認証ロジックの修正（hunk 1,3,5）とリファクタリング（hunk 2,4）
- src/models.py: ユーザーモデルの拡張（hunk 1,2）
- tests/test_auth.py: 新規テスト（hunk 1,2,3）

↓ 分割結果

コミット1: fix: 既存認証のセキュリティ脆弱性修正
git-sequential-stage -patch=".claude/tmp/current_changes.patch" -hunk="src/auth.py:1,3,5"

コミット2: feat: JWT認証機能の実装
git-sequential-stage -patch=".claude/tmp/current_changes.patch" \
  -hunk="src/auth.py:2,4" \
  -hunk="src/models.py:1,2"

コミット3: test: 認証機能のテスト追加
git-sequential-stage -patch=".claude/tmp/current_changes.patch" -hunk="tests/test_auth.py:1,2,3"
```

## ベストプラクティス

1. **事前確認**: `git status`で現在の状態を確認
2. **hunk番号の正確な指定**: ファイル名:hunk番号の形式で指定（例: "file.go:1,3,5"）
3. **意味的一貫性**: 同じ目的の変更は同じコミットに
4. **Conventional Commits**: 適切なプレフィックスを使用
   - `feat:` 新機能
   - `fix:` バグ修正
   - `refactor:` リファクタリング
   - `docs:` ドキュメント
   - `test:` テスト
   - `style:` フォーマット
   - `chore:` その他

## トラブルシューティング

### 新規ファイルがgit diffで表示されない場合

新規作成したファイルが`git diff`で表示されない場合、以下の設定が原因の可能性があります：

```bash
# .gitignoreファイルで除外されていないか確認
git check-ignore -v path/to/new_file.ext

# グローバルのgitignore設定を確認
git config --get core.excludesfile

# グローバル除外ファイルの内容を確認（存在する場合）
cat "$(git config --get core.excludesfile)"

# リポジトリの.git/info/excludeファイルを確認
cat .git/info/exclude
```

対処法：
1. `.gitignore`から該当パターンを削除または修正
2. グローバル設定を確認・修正（`~/.config/git/ignore`など）
3. `.git/info/exclude`の設定を確認・修正

### git-sequential-stageが失敗する場合

```bash
# エラーメッセージを確認
git-sequential-stage -patch=".claude/tmp/current_changes.patch" -hunk="file.go:1,2,3" 2>&1

# パッチファイルの内容を確認
cat .claude/tmp/current_changes.patch | head -50

# 特定ファイルのhunk数を確認
filterdiff -i "path/to/file.go" .claude/tmp/current_changes.patch | grep -c '^@@'
```

### 大きすぎるhunkの扱い

大きなhunkは意味的に分割できない可能性があります。この場合：
1. 一度全体をコミット
2. `git reset HEAD~1`で取り消し
3. より小さな変更単位で再実装

## 注意事項

- **使用しないコマンド**: `git add`、`git checkout`、`git restore`、`git reset`、`git stash`
- **対話的操作の回避**: `git add -p`などのインタラクティブなコマンドは使用しない
- **hunk番号の確認**: 必ず最新の差分で各ファイルのhunk番号を確認してから指定

## 背景と設計思想

[Claude Code](https://docs.anthropic.com/ja/docs/claude-code/overview)などのLLM Agentは、複数の論理的な変更を1つのコミットにまとめてしまう傾向があります。

`git-sequential-stage`ツールは、以下の複雑な処理を内部で自動化します：

1. **パッチIDによる一意識別**: `git patch-id`を使用したhunkの確実な特定
2. **逐次ステージング**: 各hunk適用時の行番号変化に対応
3. **複数ファイルのサポート**: `filterdiff`を使った特定ファイルのhunk抽出
4. **エラーハンドリング**: 様々なエッジケースに対する堅牢な処理

これにより、人間が`git add -p`で行う作業を、LLM Agentが**シンプルなコマンド実行だけで**実現できるようになりました。