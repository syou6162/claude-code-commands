# optimize_bq_query

BigQueryクエリのパフォーマンスを分析し、2倍以上の性能改善を目標とした最適化を提案します。

## 前提条件
- **リージョン**: US（region-us）固定
- **プロジェクト**: デフォルトプロジェクト（`gcloud config get-value project`で設定済み）
- **権限**: INFORMATION_SCHEMAへのアクセス権限あり

## 分析対象
入力: $ARGUMENTS（ジョブID、SQLクエリ、またはSQLファイルパス）

## 実行手順

### 1. ジョブ情報の取得（複数手段でフォールバック）

入力を判定してジョブ詳細を取得してください：

- **ジョブIDの場合**（`bquxjob_`や`job_`で始まる）: そのまま使用
- **ファイルパスの場合**（`.sql`で終わる）: ファイルを読み込んでクエリとして実行
- **SQLクエリの場合**（その他）: 直接実行

**入力処理:**
```bash
INPUT="$ARGUMENTS"

# ファイルパスの場合
if [[ "$INPUT" =~ \.sql$ ]]; then
    if [ -f "$INPUT" ]; then
        QUERY=$(cat "$INPUT")
        echo "SQLファイルを読み込みました: $INPUT"
    else
        echo "Error: ファイルが見つかりません: $INPUT"
        exit 1
    fi
    # クエリを実行してジョブIDを取得（--nosyncで即座に戻る）
    echo "クエリを実行中..."
    JOB_ID=$(bq query --nosync --use_legacy_sql=false --format=json "$QUERY" | jq -r '.jobReference.jobId')

    echo "ジョブID: $JOB_ID"
    # ジョブの完了を待つ（--nosyncのため待機が必要）
    bq wait "$JOB_ID"

# ジョブIDの場合
elif [[ "$INPUT" =~ ^(bquxjob_|job_|bq-) ]]; then
    JOB_ID="$INPUT"

# SQLクエリの場合
else
    echo "SQLクエリを実行中..."
    # クエリを実行してジョブIDを取得（--nosyncで即座に戻る）
    JOB_ID=$(bq query --nosync --use_legacy_sql=false --format=json "$INPUT" | jq -r '.jobReference.jobId')

    echo "ジョブID: $JOB_ID"
    # ジョブの完了を待つ
    bq wait "$JOB_ID"
fi
```

### 2. パフォーマンスメトリクスの収集（SQLのみで完結）

#### 元のクエリと基本メトリクスの取得
```bash
# ジョブの基本情報とクエリを取得（パラメータを使用）
bq query --use_legacy_sql=false --format=json \
  --parameter="job_id:STRING:${JOB_ID}" "
  SELECT
    job_id,
    query,
    total_slot_ms,
    total_bytes_processed,
    TIMESTAMP_DIFF(end_time, start_time, MILLISECOND) as elapsed_ms
  FROM \`region-us\`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
  WHERE job_id = @job_id" > /tmp/job_info.json

# 元のクエリを保存
jq -r '.[0].query' /tmp/job_info.json > /tmp/original_query.sql

# 元の行数を記録
bq query --use_legacy_sql=false --format=json "
  SELECT COUNT(*) as row_count
  FROM ($(cat /tmp/original_query.sql)) AS t" > /tmp/original_count.json

# 元のスキーマ（列数とカラム名）を記録
bq query --use_legacy_sql=false --format=json --max_results=0 \
  "$(cat /tmp/original_query.sql)" 2>&1 | \
  jq '.schema.fields | length' > /tmp/original_columns.txt

# カラム名リストも保存（詳細検証用）
bq query --use_legacy_sql=false --format=json --max_results=0 \
  "$(cat /tmp/original_query.sql)" 2>&1 | \
  jq -r '.schema.fields[].name' > /tmp/original_column_names.txt
```

#### Execution Graph分析（INFORMATION_SCHEMAから直接取得）
```bash
# 全ステージのパフォーマンスデータを取得（パラメータを使用）
bq query --use_legacy_sql=false --format=json \
  --parameter="job_id:STRING:${JOB_ID}" "
  SELECT
    stage.name as stage_name,
    CAST(stage.compute_ms_avg AS INT64) as compute_ms_avg,
    CAST(stage.compute_ms_max AS INT64) as compute_ms_max,
    CAST(stage.wait_ms_avg AS INT64) as wait_ms_avg,
    CAST(stage.wait_ms_max AS INT64) as wait_ms_max,
    CAST(stage.shuffle_output_bytes AS INT64) as shuffle_bytes,
    CAST(stage.shuffle_output_bytes_spilled AS INT64) as shuffle_spilled,
    CAST(stage.records_read AS INT64) as records_read,
    CAST(stage.records_written AS INT64) as records_written,
    CAST(stage.slot_ms AS INT64) as slot_ms,
    -- データスキュー率の計算
    SAFE_DIVIDE(CAST(stage.compute_ms_max AS FLOAT64),
                CAST(stage.compute_ms_avg AS FLOAT64)) as skew_ratio
  FROM
    \`region-us\`.INFORMATION_SCHEMA.JOBS_BY_PROJECT,
    UNNEST(job_stages) AS stage
  WHERE
    job_id = @job_id
  ORDER BY
    stage.slot_ms DESC" > /tmp/execution_graph.json
```

**注目メトリクス:**
- `compute_ms_max`: 最大計算時間
- `shuffle_bytes`: シャッフルデータ量
- `skew_ratio`: データスキュー率（2.0以上で問題）
- `stage_name`: "S00: Input", "S06: Sort+", "S0E: Join+"など

#### 時系列スロット使用データ
```bash
bq query --use_legacy_sql=false --format=json \
  --parameter="job_id:STRING:${JOB_ID}" "
  SELECT
    period_start,
    period_slot_ms,
    state,
    period_shuffle_ram_usage_ratio
  FROM \`region-us\`.INFORMATION_SCHEMA.JOBS_TIMELINE_BY_PROJECT
  WHERE job_id = @job_id
  ORDER BY period_start"
```

### 3. ボトルネック診断（ステージとSQL対応付け）

収集したメトリクスから問題を特定し、**元のSQLのどの部分が原因か説明してください**：

```bash
# execution_graph.jsonから問題のあるステージを分析
echo "=== パフォーマンス分析 ==="

# 1. 最も遅いステージTOP5を表示
echo "最も時間のかかるステージ:"
jq -r '.[] | "\(.stage_name): \(.slot_ms)ms (compute: \(.compute_ms_max)ms, wait: \(.wait_ms_max)ms)"' \
  /tmp/execution_graph.json | head -5

# 2. データスキューの検出（スキュー率2.0以上）
echo -e "\nデータスキュー検出:"
jq -r '.[] | select(.skew_ratio >= 2.0) | "\(.stage_name): スキュー率 \(.skew_ratio)"' \
  /tmp/execution_graph.json

# 3. 大量シャッフルステージ（100MB以上）
echo -e "\n大量データシャッフル:"
jq -r '.[] | select(.shuffle_bytes >= 104857600) | "\(.stage_name): \(.shuffle_bytes / 1048576 | floor)MB"' \
  /tmp/execution_graph.json
```

**分析結果の解釈:**
- Stage名が"Join+"の場合 → SQLのJOIN句を確認、大きいテーブル同士のJOINが原因
- Stage名が"Sort+"の場合 → ORDER BY句やウィンドウ関数のソート処理
- Stage名が"Aggregate+"の場合 → GROUP BY句の集計処理
- スキュー率2.0以上 → 特定のキー値にデータが偏っている
- シャッフル量100MB以上 → 大量データ移動が発生、JOIN順序の見直しが必要

### 4. 最適化パターンの選択（優先度付きで複数提案）

**検出された問題ごとに、効果の高い順に複数の最適化案を提示してください：**

#### 優先度1: データ量削減系
- **アグリゲーション前置**: JOIN前にGROUP BY（効果: 大）
- **パーティションフィルタ**: WHERE句で日付限定（効果: 大）

#### 優先度2: 実行順序最適化
- **JOIN順序変更**: 小→大の順でJOIN（効果: 中）
- **サブクエリ最適化**: EXISTS/INへの書き換え（効果: 中）

#### 優先度3: データ分布改善
- **スキュー対策**: 偏りキーの分離処理（効果: 状況依存）

**複数適用可能な場合は組み合わせ効果も説明**

### 5. 最適化クエリの生成と結果検証

#### 最適化SQLの生成
元のクエリに選択したパターンを適用し、新しいクエリを生成：

```sql
-- 最適化後のクエリを/tmp/optimized_query.sqlに保存
```

#### 結果同一性の完全検証（行数＋列数）
```bash
# 最適化後の行数確認
bq query --use_legacy_sql=false --format=json "
  SELECT COUNT(*) as row_count
  FROM ($(cat /tmp/optimized_query.sql)) AS t" > /tmp/optimized_count.json

# 最適化後の列数確認
bq query --use_legacy_sql=false --format=json --max_results=0 \
  "$(cat /tmp/optimized_query.sql)" 2>&1 | \
  jq '.schema.fields | length' > /tmp/optimized_columns.txt

# 最適化後のカラム名リスト
bq query --use_legacy_sql=false --format=json --max_results=0 \
  "$(cat /tmp/optimized_query.sql)" 2>&1 | \
  jq -r '.schema.fields[].name' > /tmp/optimized_column_names.txt

# 行数と列数の比較
ORIGINAL_ROWS=$(jq -r '.[0].row_count' /tmp/original_count.json)
OPTIMIZED_ROWS=$(jq -r '.[0].row_count' /tmp/optimized_count.json)
ORIGINAL_COLS=$(cat /tmp/original_columns.txt)
OPTIMIZED_COLS=$(cat /tmp/optimized_columns.txt)

if [ "$ORIGINAL_ROWS" != "$OPTIMIZED_ROWS" ]; then
  echo "警告: 行数が一致しません（元: $ORIGINAL_ROWS, 最適化後: $OPTIMIZED_ROWS）"
fi

if [ "$ORIGINAL_COLS" != "$OPTIMIZED_COLS" ]; then
  echo "警告: 列数が一致しません（元: $ORIGINAL_COLS, 最適化後: $OPTIMIZED_COLS）"
fi

# カラム名の差分確認（オプション）
diff /tmp/original_column_names.txt /tmp/optimized_column_names.txt
```

### 6. 改善効果の測定と反復改善

#### 最適化クエリの実行と新メトリクス取得
```bash
# 最適化クエリを実行（両形式のジョブIDに対応）
NEW_JOB_OUTPUT=$(bq query --use_legacy_sql=false --format=none \
  "$(cat /tmp/optimized_query.sql)" 2>&1)
NEW_JOB_ID=$(echo "$NEW_JOB_OUTPUT" | \
  grep -oE '(bquxjob_[a-z0-9_]+|job_[a-zA-Z0-9_-]+)' | head -1)

# 新しいジョブのメトリクスを取得
NEW_JOB_INFO=$(bq query --use_legacy_sql=false --format=json \
  --parameter="job_id:STRING:${NEW_JOB_ID}" "
  SELECT
    total_slot_ms,
    total_bytes_processed,
    TIMESTAMP_DIFF(end_time, start_time, MILLISECOND) as elapsed_ms
  FROM \`region-us\`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
  WHERE job_id = @job_id")

# 改善度を計算
ORIGINAL_SLOT_MS=$(jq -r '.[0].total_slot_ms' /tmp/job_info.json)
NEW_SLOT_MS=$(echo "$NEW_JOB_INFO" | jq -r '.[0].total_slot_ms')
IMPROVEMENT_RATIO=$(echo "scale=2; $ORIGINAL_SLOT_MS / $NEW_SLOT_MS" | bc)
```

#### 反復改善プロセス（2倍未達時）
```bash
# 改善履歴を記録
echo "Iteration 1: ${IMPROVEMENT_RATIO}x improvement" >> /tmp/optimization_history.txt
```

**2倍改善に達しない場合:**
1. 新しいexecution graphから追加のボトルネック特定
2. 未適用の最適化パターンを試行
3. 最大3回まで反復
4. 各イテレーションの結果を履歴に記録

### 7. 最終レポート生成

```markdown
# 最適化レポート

## 実行概要
- 元のジョブID: [ID]
- プロジェクト: [PROJECT_ID]
- リージョン: [LOCATION]
- 処理データ: [X GB]
- スロット時間: [Y ms]

## 検出された問題とSQL対応
1. [ステージX: JOINの非効率] - L10-15のcustomers JOIN orders
2. [ステージY: データスキュー] - L20のGROUP BY user_id

## 適用した最適化（優先度順）
1. [パターンA]: 期待効果60%削減
2. [パターンB]: 期待効果30%削減
3. [組み合わせ効果]: 合計75%削減見込み

## 最適化後のクエリ
```sql
[生成したSQL]
```

## 実測された改善効果
| イテレーション | スロット時間 | 改善率 | 適用パターン |
|--------------|------------|--------|------------|
| 元クエリ | 100,000ms | - | - |
| 1回目 | 60,000ms | 1.67x | アグリゲーション前置 |
| 2回目 | 45,000ms | 2.22x | +JOIN順序最適化 |

## 結果の同一性確認
- 元の行数: [X]行 / 列数: [Y]列
- 最適化後: [X]行 / [Y]列 ✓ 完全一致
- カラム名: ✓ 一致

## 実装時の注意
[本番適用時の注意点]
```

## エラーハンドリング

### よくあるエラーと対処法

#### 1. ジョブが見つからない
```bash
if ! bq show -j --format=json "$JOB_ID" > /tmp/job_details.json 2>&1; then
    echo "Error: ジョブID '$JOB_ID' が見つかりません"
    echo "対処法:"
    echo "  1. ジョブIDが正しいか確認してください"
    echo "  2. プロジェクトIDを明示的に指定: bq show -j --project_id=PROJECT_ID"
    echo "  3. 最近のジョブ一覧を確認: bq ls -j -n 10"
    exit 1
fi
```

#### 2. 権限エラー
```bash
if grep -q "Access Denied" /tmp/job_details.json; then
    echo "Error: ジョブへのアクセス権限がありません"
    echo "対処法:"
    echo "  - bigquery.jobs.get権限が必要です"
    echo "  - プロジェクトのIAM設定を確認してください"
    exit 1
fi
```

#### 3. リージョンエラー
```bash
if grep -q "Not found: Dataset" <<< "$QUERY_OUTPUT"; then
    echo "Error: データセットが指定されたリージョンに存在しません"
    echo "検出されたリージョン: $LOCATION"
    echo "対処法:"
    echo "  - データセットのロケーションを確認"
    echo "  - INFORMATION_SCHEMAのリージョンプレフィックスを調整"
fi
```

#### 4. クエリ実行エラー
```bash
if [ -z "$JOB_ID" ]; then
    echo "Error: クエリの実行に失敗しました"
    echo "考えられる原因:"
    echo "  - SQL構文エラー"
    echo "  - テーブルが存在しない"
    echo "  - クォータ制限"
    echo "デバッグ: bq query --dry_run でクエリを検証"
    exit 1
fi
```

#### 5. Execution Graphが存在しない
```bash
if ! jq -e '.statistics.query.queryPlan' /tmp/job_details.json > /dev/null 2>&1; then
    echo "Warning: このクエリにはexecution graphが生成されていません"
    echo "原因:"
    echo "  - クエリが単純すぎる（単一テーブルのSELECT等）"
    echo "  - キャッシュからの結果返却"
    echo "対処法: より複雑なクエリで分析を実行"
    exit 0
fi
```

#### 6. 最適化後の結果不一致
```bash
if [ "$ORIGINAL_ROWS" != "$OPTIMIZED_ROWS" ] || [ "$ORIGINAL_COLS" != "$OPTIMIZED_COLS" ]; then
    echo "Critical Error: 最適化により結果が変更されました！"
    echo "  元: ${ORIGINAL_ROWS}行 x ${ORIGINAL_COLS}列"
    echo "  新: ${OPTIMIZED_ROWS}行 x ${OPTIMIZED_COLS}列"
    echo ""
    echo "最適化クエリを破棄し、別のパターンを試してください"
    echo "デバッグ: diff /tmp/original_column_names.txt /tmp/optimized_column_names.txt"
    exit 1
fi
```

### リカバリー処理

#### フォールバックチェーン
1. `bq show -j` 失敗 → INFORMATION_SCHEMAで取得
2. INFORMATION_SCHEMA失敗 → Jobs APIを試行
3. 全て失敗 → ユーザーに手動確認を依頼

#### 部分的成功の処理
- Execution Graphの一部のみ取得できた場合でも、可能な範囲で分析継続
- 時系列データが取得できない場合は、静的分析のみ実施

## 使用例とベストプラクティス

### 典型的な使用例

#### 例1: 既存ジョブの最適化
```bash
# 遅いクエリのジョブIDを特定
bq ls -j -n 20 | grep "QUERY" | sort -k4 -r

# 最適化実行
/optimize_bq_query bquxjob_1234567_abcdef
```

#### 例2: 新規クエリの最適化
```bash
# SQLファイルから実行
/optimize_bq_query my_complex_query.sql
```

#### 例3: 直接SQLを入力
```bash
/optimize_bq_query "
  SELECT customer_id, SUM(revenue) as total
  FROM sales s
  JOIN customers c ON s.customer_id = c.id
  WHERE s.date >= '2024-01-01'
  GROUP BY customer_id
"
```

### ベストプラクティス

#### 1. 事前準備
- **本番実行前にドライラン**: `bq query --dry_run`で処理データ量を確認
- **小規模データでテスト**: LIMIT句やサンプルテーブルで検証
- **ベースライン測定**: 最適化前の性能を記録

#### 2. 段階的適用
- **一つずつ最適化**: 複数の最適化を同時適用せず、効果を個別測定
- **最も効果的なものから**: 優先度1の最適化から順に適用
- **測定と記録**: 各段階の改善率を記録

#### 3. よくあるパフォーマンス問題と対策

| 問題パターン | 症状 | 推奨対策 |
|------------|------|----------|
| 巨大テーブル同士のJOIN | shuffleOutputBytes > 1GB | アグリゲーション前置 |
| フルテーブルスキャン | totalBytesProcessed大 | パーティションフィルタ追加 |
| データスキュー | computeMsMax/Avg > 5 | スキューキー分離処理 |
| 過剰なサブクエリ | ステージ数 > 30 | WITH句でのCTE化 |

#### 4. 最適化の限界
- **アーキテクチャ的制約**: テーブル設計が根本原因の場合
- **ビジネスロジック**: 複雑な計算が必須の場合
- **データ量**: 絶対的にデータが多い場合

これらの場合は以下を検討：
- マテリアライズドビューの作成
- インクリメンタル処理への変更
- パーティション・クラスタリングの再設計

## 統合テストガイド

### コマンドの動作確認

1. **基本動作テスト**
```bash
# テスト用の簡単なクエリを実行
TEST_QUERY="SELECT 1 as test"
bq query --use_legacy_sql=false "$TEST_QUERY"
# 生成されたジョブIDで最適化を実行
/optimize_bq_query <job_id>
```

2. **エラーハンドリングテスト**
```bash
# 存在しないジョブID
/optimize_bq_query job_nonexistent_123

# 不正なSQL
/optimize_bq_query "SELECT * FROM nonexistent_table"
```

3. **最適化効果の検証**
- 実際の遅いクエリで2倍改善を確認
- 結果の同一性を検証
- レポートの可読性を確認

## 重要な方針

1. **必ず2倍以上を目指す** - 未達なら反復改善
2. **結果の完全同一性を保証** - 行数・列数・カラム名を検証
3. **ステージとSQLを対応付け** - どこが問題か明確に説明
4. **複数案を優先度順に提示** - 効果の高いものから
5. **実測値で判断** - 推定ではなく実行して確認
6. **エラーは分かりやすく** - 原因と対処法を明示