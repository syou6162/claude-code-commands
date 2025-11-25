---
name: update-pr-title-and-description
description: Pull Requestのタイトルと説明文を設定・更新する際に呼び出してください。
tools: Bash, Write, Edit, Read
model: haiku
permissionMode: acceptEdits
---

# Pull Requestのタイトルと説明文を更新する

Pull Requestのタイトルと説明文を以下の手順で更新してください：

## 実行手順

1. **PR番号の取得**

   PR番号を取得（現在のブランチから）：
   ```bash
   gh pr view --json number --jq '.number'
   ```

2. **修正内容の確認**
   ```bash
   gh pr diff
   ```

3. **コミットメッセージの確認**
   ```bash
   gh pr view --json commits --jq '.commits[] | .messageHeadline'
   ```

4. **説明文ファイルの準備**

   `.github/PULL_REQUEST_TEMPLATE.md`が存在する場合はコピー：
   ```bash
   cp .github/PULL_REQUEST_TEMPLATE.md .claude_work/pr_body_<PR番号>.md
   ```

   存在しない場合は新規ファイル作成：
   ```bash
   touch .claude_work/pr_body_<PR番号>.md
   ```

5. **Pull Requestの説明文を作成**
   - 作業ファイル（`.claude_work/pr_body_<PR番号>.md`）を編集
   - 上記で取得した情報とチャットの会話内容を考慮して説明文を作成
   - **説明文は必ず日本語で記載すること**
   - **重要**：ファイル編集には必ず`Write`ツールまたは`Edit`ツールを使用すること
   - bashコマンド（`cat <<EOF > file`、`echo "..." > file`など）でファイルを書き込んではいけません

6. **Pull Requestの更新**
   ```bash
   # 修正内容を考慮したタイトルと説明文を更新
   gh pr edit --title "修正内容を考慮したタイトル" --body-file .claude_work/pr_body_<PR番号>.md
   ```

7. **更新後の確認と文字化けチェック**
   ```bash
   # 更新されたPRの内容を確認
   gh pr view
   ```

   - タイトルと説明文が正しく更新されているか確認
   - **文字化けチェック**：日本語が文字化けしていないか確認
   - **文字化けが検出された場合**：
     1. `.claude_work/pr_body_<PR番号>.md` を確認し、UTF-8エンコーディングで保存されているか確認
     2. ファイルを修正（必要に応じて文字エンコーディングを修正）
     3. 再度 `gh pr edit --body-file .claude_work/pr_body_<PR番号>.md` で更新
     4. もう一度 `gh pr view` で確認

## 説明文の生成ルール

- `.github/PULL_REQUEST_TEMPLATE.md`が存在する場合：
  - テンプレートに沿った形でPull Requestの説明文を生成
  
- `.github/PULL_REQUEST_TEMPLATE.md`が存在しない場合：
  - 「何をやったか」を記載
  - 「修正が必要になった背景」を記載

## 注意事項

- 修正内容とコミットメッセージを把握した上でPull Requestの内容を決定する
- チャットの会話内容も考慮してPull Requestの説明を作成する
- **作業ファイルについて**：
  - 常に`--body-file`オプションを使用して安全に更新
  - ファイル（`.claude_work/pr_body_<PR番号>.md`）は削除せず残しておく
  - 理由：説明文を何度か修正する場合があるため、編集可能な状態で保持
- **ファイル編集について**：
  - ファイルの作成・編集には必ず`Write`ツールまたは`Edit`ツールを使用すること
  - bashコマンド（`cat <<EOF > file`、`cat << 'EOF' > file`、`echo "..." > file`など）でファイルを書き込まないこと
  - 理由：専用ツールの方が安全で確実、かつ文字エンコーディングの問題を回避できる
- **git pushについて**：
  - Pull Requestのコミットの状態と手元のコミットの状態で差分があるのであれば、ユーザーにgit pushのし忘れがないかを確認しましょう
  - あなた自身がgit pushする必要はないし、そもそも権限的にできなくなっています