---
name: updating-pr-title-and-description
description: Pull Request作成・更新時に使用。タイトルと説明文を自動生成・更新する。
allowed-tools: Bash, Write, Edit, Read
model: opus
context: fork
---

# Pull Requestのタイトルと説明文を更新する

Pull Requestのタイトルと説明文を以下の手順で更新してください。

## 実行手順

<procedure>

1. **デフォルトブランチの取得**

   デフォルトブランチを取得：
   ```bash
   git symbolic-ref refs/remotes/origin/HEAD --short | cut -d/ -f2
   ```

2. **修正内容の確認**

   デフォルトブランチからの差分を確認：
   ```bash
   # <default-branch> には手順1で取得したブランチ名を使用
   git diff <default-branch>...HEAD
   ```

3. **コミットメッセージの確認**

   デフォルトブランチからのコミット履歴を確認（本文も含む）：
   ```bash
   # <default-branch> には手順1で取得したブランチ名を使用
   git log <default-branch>..HEAD
   ```

4. **説明文ファイルの準備**

   `.github/PULL_REQUEST_TEMPLATE.md`が存在する場合はコピー：
   ```bash
   cp .github/PULL_REQUEST_TEMPLATE.md .claude_work/pr_body_draft.md
   ```

   存在しない場合は新規ファイル作成：
   ```bash
   touch .claude_work/pr_body_draft.md
   ```

5. **Pull Requestの説明文を作成**
   - **重要**: 説明文を書く前に、必ず **reference/description-rules.md** を読み、そのルールに従うこと
   - 作業ファイル（`.claude_work/pr_body_draft.md`）を編集
   - 上記で取得した情報とチャットの会話内容を考慮して説明文を作成
   - **説明文は必ず日本語で記載すること**
   - **重要**：ファイル編集には必ず`Write`ツールまたは`Edit`ツールを使用すること
   - bashコマンド（`cat <<EOF > file`、`echo "..." > file`など）でファイルを書き込んではいけません

6. **Pull Requestの作成または更新**

   PRの存在確認と作成/更新：
   ```bash
   # PRが存在するか確認
   if gh pr view >/dev/null 2>&1; then
     # PRが存在する場合：更新
     gh pr edit --title "修正内容を考慮したタイトル" --body-file .claude_work/pr_body_draft.md
   else
     # PRが存在しない場合：ドラフトPRを作成
     gh pr create --draft --title "修正内容を考慮したタイトル" --body-file .claude_work/pr_body_draft.md
   fi
   ```

7. **更新後の確認と文字化けチェック**
   ```bash
   # PRの内容を確認
   gh pr view
   ```

   - タイトルと説明文が正しく設定されているか確認
   - **文字化けチェック**：日本語が文字化けしていないか確認
   - **文字化けが検出された場合**：
     1. `.claude_work/pr_body_draft.md` を確認し、UTF-8エンコーディングで保存されているか確認
     2. ファイルを修正（必要に応じて文字エンコーディングを修正）
     3. 再度 `gh pr edit --body-file .claude_work/pr_body_draft.md` で更新
     4. もう一度 `gh pr view` で確認

</procedure>
