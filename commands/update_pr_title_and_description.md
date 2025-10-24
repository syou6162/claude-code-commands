# Pull Requestのタイトルと説明文を更新する

Pull Requestのタイトルと説明文を以下の手順で更新してください：

## 実行手順

1. **PR番号の取得**
   
   PR番号を取得（現在のブランチから）：
   ```bash
   PR_NUMBER=$(gh pr view --json number --jq '.number')
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
   cp .github/PULL_REQUEST_TEMPLATE.md .claude/tmp/pr_body_${PR_NUMBER}.md
   ```
   
   存在しない場合は新規ファイル作成：
   ```bash
   touch .claude/tmp/pr_body_${PR_NUMBER}.md
   ```

5. **Pull Requestの説明文を作成**
   - 作業ファイル（`.claude/tmp/pr_body_${PR_NUMBER}.md`）を編集
   - 上記で取得した情報とチャットの会話内容を考慮して説明文を作成

6. **Pull Requestの更新**
   ```bash
   # 修正内容を考慮したタイトルと説明文を更新
   gh pr edit --title "修正内容を考慮したタイトル" --body-file .claude/tmp/pr_body_${PR_NUMBER}.md
   ```

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
  - ファイル（`.claude/tmp/pr_body_${PR_NUMBER}.md`）は削除せず残しておく
  - 理由：説明文を何度か修正する場合があるため、編集可能な状態で保持
- **git pushについて**：
  - Pull Requestのコミットの状態と手元のコミットの状態で差分があるのであれば、ユーザーにgit pushのし忘れがないかを確認しましょう
  - あなた自身がgit pushする必要はないし、そもそも権限的にできなくなっています