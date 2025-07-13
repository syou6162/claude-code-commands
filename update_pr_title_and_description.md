# Pull Requestのタイトルと説明文を更新する

Pull Requestのタイトルと説明文を以下の手順で更新してください：

## 実行手順

1. **修正内容の確認**
   ```bash
   gh pr diff
   ```

2. **コミットメッセージの確認**
   ```bash
   gh pr view --json commits --jq '.commits[] | .messageHeadline'
   ```

3. **Pull Requestの更新**
   - 上記で取得した情報とチャットの会話内容を考慮してPull Requestのタイトルと説明文を更新
   - 実際の更新には`gh pr edit`を使用

## 説明文の生成ルール

- `.github/PULL_REQUEST_TEMPLATE.md`が存在する場合：
  - そのテンプレートに沿った形でPull Requestの説明文を生成
  
- `.github/PULL_REQUEST_TEMPLATE.md`が存在しない場合：
  - 「何をやったか」を記載
  - 「修正が必要になった背景」を記載

## 注意事項

- 修正内容とコミットメッセージを把握した上でPull Requestの内容を決定する
- チャットの会話内容も考慮してPull Requestの説明を作成する
- **説明文にエスケープ文字列やコマンドに影響する特殊文字が含まれている場合**：
  - `gh pr edit`コマンドが失敗することがある
  - その場合は説明文を一時ファイルに保存してから`--body-file`オプションを使用する：
    ```bash
    # 説明文を一時ファイルに保存
    echo "Pull Requestの説明文..." > /tmp/pr_body.md
    
    # ファイルから説明文を読み込んで更新
    gh pr edit --body-file /tmp/pr_body.md
    
    # 一時ファイルを削除
    rm /tmp/pr_body.md
    ```