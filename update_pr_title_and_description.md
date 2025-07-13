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