---
name: requesting-gcloud-bq-auth
description: gcloudやbqコマンド実行時に認証エラー（Reauthentication required等）を検出した場合に使用。エージェントが自動で認証コマンドを実行することを防ぎ、ユーザーに認証を依頼します。
---

# Requesting gcloud/bq Auth Skill

## 目的

このスキルは、gcloudやbqコマンドの実行時に認証エラーを検出し、エージェントが勝手に認証コマンドを実行することを防ぎます。認証はブラウザ操作が必要なため、ユーザーに明示的に依頼します。

## 使用タイミング

<trigger>

以下のエラーパターンを検出した場合にこのスキルを発動してください：

**パターン1**: bqコマンドでの再認証要求
- `Reauthentication required.` と `Please enter your password:` が出力された場合

**パターン2**: gcloud認証でブラウザが開く
- `Your browser has been opened to visit:` と `accounts.google.com` URLが出力された場合

</trigger>

## 実行手順

<procedure>

1. **即座に作業を中断**: 認証エラーを検出したら、関連するgcloud/bqコマンドの実行を停止する

2. **ユーザーに通知**: 以下の内容をユーザーに伝える
   - 認証が切れていること
   - 推奨される認証コマンド（例: `gcloud auth application-default login`）
   - 認証完了後に作業を再開できること

3. **認証完了を待つ**: ユーザーが認証を完了するまで、関連するコマンドを実行しない

</procedure>

## エラーパターン例

<examples>

### パターン1: bqコマンドでの再認証要求

<example>

```
% bq ls
Reauthentication required.
Please enter your password:
```

**対応**:
```
認証が切れています。以下のコマンドで認証を完了してください：

  gcloud auth application-default login

認証完了後、作業を再開します。
```

</example>

### パターン2: gcloud認証でブラウザが開く

<example>

```
% gcloud auth application-default login
Your browser has been opened to visit:

    https://accounts.google.com/o/oauth2/auth?response_type=code&client_id=XXX&redirect_uri=XXX&scope=XXX&state=XXX&access_type=offline&code_challenge=XXX&code_challenge_method=S256
```

**対応**:
```
ブラウザで認証フローが開始されました。
ブラウザで認証を完了してください。

認証完了後、作業を再開します。
```

</example>

</examples>

## 重要な注意事項

<important>

✅ **すべきこと**:
- 認証エラーを検出したら即座に作業を中断する
- ユーザーに明確な指示を提供する（認証コマンド、手順）
- 認証完了を待つ

❌ **してはいけないこと**:
- **絶対に認証コマンドを自動実行しない**（ブラウザ操作が必要なため）
- 認証が完了していない状態でgcloud/bqコマンドを実行しない
- URLに含まれる `client_id`、`redirect_uri`、`state`、`code_challenge` などの機密情報をログに残さない（例示する場合は `XXX` で伏せ字にする）

</important>
