---
name: semantic-committing
description: コミット時、「commit」「git add」「変更を分割」の言及時に使用。git diffを分析し、変更を論理的な意味単位に分割してコミットする。git-sequential-stageでhunk単位のステージングを行う。
allowed-tools: Bash, Write, Edit, Read
model: sonnet
context: fork
---

# 意味のある最小単位でコミットする

大きな変更を論理的な単位に分割してコミットしてください。git diffを分析して意味のある最小単位を特定し、`git-sequential-stage`ツールで段階的にステージングします。

## 禁止事項

<important>

- [ ] **手順の厳守**: <procedure>タグ内で指定された手順、実行コマンド、オプションを完全に守ること
  - [ ] 手順を一つずつ順番に実行すること（効率化のために手順を飛ばしたり、コマンドを変更したりしてはいけない）
  - [ ] コマンドの実行順序を変更してはいけない
  - [ ] 複数のコマンドを`&&`や`;`で繋ぐなど、手順にない方法でコマンドを実行してはいけない
- [ ] 計画を立てるだけで終わることは禁止です。このエージェントに求められているのは、すべての変更がコミットされていることです
- [ ] `git add .` / `git add -A` の使用は禁止です
- [ ] 必ず`git-sequential-stage`を使用してhunk単位でステージングすること
- [ ] `.claude_work/plans/` 配下のファイルはコミット対象外です。gitignoreされているファイルを無理にステージングしようとしないでください

</important>

## プロンプトの扱い

<important>

- 呼び出し時のプロンプトに特に明確な指示がされていない場合は、<procedure>タグの実行手順通りに進めてください
- プロンプトに特定の意図（例：「2つのコミットに分割してください」「テストと実装を分けてください」など）が加えられている場合：
  - **手順3（変更内容を分析）** と **手順4（ステージングとコミット）** でその意図を考慮してください
  - ただし、**<procedure>タグの実行手順は一切変更せず**、記載された手順に従って実行してください
  - プロンプトの意図は分析とコミット計画に反映し、実行方法は変えないこと

</important>

## 実行手順

<procedure>
以下の手順で変更を意味のある最小単位に分割してコミットしてください：

1. **pre-commitの事前実行**

   `.pre-commit-config.yaml`が存在する場合は、事前に実行してください：

   ```bash
   pre-commit run --all-files
   ```

2. **差分を取得**

   最初に必ずintent-to-addで新規ファイルを追加してください：

   ```bash
   git ls-files --others --exclude-standard | xargs -r git add -N
   ```

   差分を取得してください：

   ```bash
   git diff HEAD | tee .claude_work/current_changes.patch
   ```

3. **変更内容を分析**

   **hunk単位**で変更を分析し、最初のコミットに含めるhunkを決定してください：

   - **hunkの内容を読み取る**: 各hunkが何を変更しているか理解する
   - **意味的グループ化**: 同じ目的の変更（バグ修正、リファクタリング等）をグループ化する
   - **コミット計画**: どのhunkをどのコミットに含めるか決定する

   各ファイルのhunk数を確認してください：

   ```bash
   git-sequential-stage count-hunks
   ```

   分析例：

   ```bash
   # 分析結果例
   # - コミット1（fix）:
   #   - src/calculator.py: hunk 1, 3, 5（ゼロ除算エラーの修正）
   #   - src/utils.py: hunk 2（関連するユーティリティ関数の修正）
   # - コミット2（refactor）:
   #   - src/calculator.py: hunk 2, 4（計算ロジックの最適化）
   ```

   コミットメッセージの形式（Conventional Commits形式）：
   - `feat`: 新機能
   - `fix`: バグ修正
   - `refactor`: リファクタリング
   - `docs`: ドキュメント
   - `test`: テスト
   - `style`: フォーマット
   - `perf`: パフォーマンス改善
   - `build`: ビルドシステムや外部依存関係の変更
   - `ci`: CI設定ファイルやスクリプトの変更
   - `revert`: コミットの取り消し
   - `chore`: その他

   分析が完了したら、コミット用のメッセージを作成してください：

   **コミットメッセージの作成方法：**
   - **必ずWriteツールを使用**して `.claude_work/commit_message.txt` にコミットメッセージを書くこと
   - **禁止される方法：**
     - `cat`とheredocを使ってファイルに書き込む（例：`cat <<EOF > .claude_work/commit_message.txt`）
     - `git commit -m`で直接メッセージを指定する

   ```bash
   # Writeツールで .claude_work/commit_message.txt にコミットメッセージを書く
   # 例：
   # fix: ゼロ除算エラーを修正
   #
   # 計算処理で分母が0の場合の適切なエラーハンドリングを追加
   ```

4. **ステージングとコミット**

   <decision-criteria name="wildcard-usage">

   **ワイルドカード（`*`）の使用判断：**

   | 状況 | 判断 | 理由 |
   |------|------|------|
   | ファイル内のすべての変更が意味的に一体 | `*` 使用 | 単一の目的で分割不要 |
   | 新規ファイル追加 | `*` 使用 | すべて新規のため |
   | ドキュメントファイルの変更 | `*` 使用 | 通常は単一目的 |
   | 異なる目的の変更が混在 | hunk番号指定 | バグ修正とリファクタリング等を分離 |
   | hunk数が不明 | まず確認 | 盲目的な`*`使用は厳禁 |

   **重要**: 「hunkを数えるのが面倒」という理由での`*`使用は厳禁

   </decision-criteria>

   <example name="staging-patterns">

   ```bash
   # パターン1: 部分的な変更をステージング（hunk番号指定）
   git-sequential-stage stage -patch=".claude_work/current_changes.patch" -hunk="src/calculator.py:1,3,5"

   # パターン2: ファイル全体をステージング（意味的に一体の変更の場合）
   git-sequential-stage stage -patch=".claude_work/current_changes.patch" -hunk="tests/test_calculator.py:*"

   # パターン3: 複数ファイルの場合（混在使用）
   git-sequential-stage stage -patch=".claude_work/current_changes.patch" -hunk="src/calculator.py:1,3,5" -hunk="src/utils.py:2" -hunk="docs/CHANGELOG.md:*"

   # コミット実行（手順3で作成したコミットメッセージを使用）
   # 注意: ファイルパスは .claude_work/commit_message.txt であり、/tmp ではない
   git commit -F .claude_work/commit_message.txt
   ```

   </example>

5. **残りの変更を処理**

   残りの変更があるかを確認してください：

   ```bash
   git diff HEAD
   ```

   **判断フロー：**
   - 残りの変更がない → 手順6（最終確認）へ進む
   - 残りの変更があり、差分の内容が変化している（pre-commitによる自動修正の可能性）→ **手順1から再実行**
   - 残りの変更があり、差分が予想通り → パッチファイルを再生成して**手順3から再開**

   パッチファイルの再生成：

   ```bash
   git diff HEAD | tee .claude_work/current_changes.patch > /dev/null
   ```

6. **最終確認**

   すべての変更がコミットされたか確認してください：

   ```bash
   git diff HEAD
   git status
   ```

</procedure>
