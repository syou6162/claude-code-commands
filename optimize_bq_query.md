# optimize_bq_query

BigQueryクエリのパフォーマンスを分析し、2倍以上の性能改善を目標とした最適化を提案します。

## 前提条件
- **リージョン**: US（region-us）固定
- **プロジェクト**: デフォルトプロジェクト（`gcloud config get-value project`で設定済み）
- **権限**: INFORMATION_SCHEMAへのアクセス権限あり
- **スキャン量削減**: INFORMATION_SCHEMAは直近7日間のみ検索（creation_timeで絞り込み）

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
        echo "SQLファイルを読み込みました: $INPUT"
    else
        echo "Error: ファイルが見つかりません: $INPUT"
        exit 1
    fi
    # ファイルから直接クエリを実行（改行を含むクエリに対応）
    echo "クエリを実行中..."
    JOB_ID=$(cat "$INPUT" | bq query --nosync --use_legacy_sql=false --use_cache=false --format=json | jq -r '.jobReference.jobId')

    echo "ジョブID: $JOB_ID"
    # ジョブの完了を待つ（--nosyncのため待機が必要）
    bq wait "$JOB_ID"

# ジョブIDの場合
elif [[ "$INPUT" =~ ^(bquxjob_|job_|bq-) ]]; then
    JOB_ID="$INPUT"

# SQLクエリの場合
else
    echo "SQLクエリを実行中..."
    # クエリをパイプで渡して実行（改行を含むクエリに対応）
    JOB_ID=$(echo "$INPUT" | bq query --nosync --use_legacy_sql=false --use_cache=false --format=json | jq -r '.jobReference.jobId')

    echo "ジョブID: $JOB_ID"
    # ジョブの完了を待つ
    bq wait "$JOB_ID"
fi
```

### 2. パフォーマンスメトリクスの収集（SQLのみで完結）

#### 元のクエリと基本メトリクスの取得
```bash
# ジョブの基本情報とクエリを取得（クエリはファイルに直接出力）
echo "ジョブ情報を取得中..."
bq query --use_legacy_sql=false --format=json \
  --parameter="job_id:STRING:${JOB_ID}" "
  SELECT
    query,
    total_slot_ms,
    total_bytes_processed,
    TIMESTAMP_DIFF(end_time, start_time, MILLISECOND) as elapsed_ms
  FROM \`region-us\`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
  WHERE job_id = @job_id
    AND creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)" > /tmp/job_info.json

# メトリクスを変数に、クエリをファイルに保存
jq -r '.[0].query' /tmp/job_info.json > /tmp/original_query.sql
ORIGINAL_SLOT_MS=$(jq -r '.[0].total_slot_ms' /tmp/job_info.json)
echo "元のスロット時間: ${ORIGINAL_SLOT_MS}ms"

# 元のクエリの実際の結果を取得して行数を記録（結果は検証時に使うので保存）
echo "元のクエリ結果を取得中..."
cat /tmp/original_query.sql | bq query --use_legacy_sql=false --use_cache=false --format=json > /tmp/original_results.json
echo "元の行数: $(jq '. | length' /tmp/original_results.json)"

# 元のスキーマ情報を取得して変数に保存
echo "スキーマ情報を取得中..."
ORIGINAL_SCHEMA=$(cat /tmp/original_query.sql | bq query --use_legacy_sql=false --format=json --dry_run)
ORIGINAL_COLS=$(echo "$ORIGINAL_SCHEMA" | jq '.statistics.query.schema.fields | length')
ORIGINAL_COL_NAMES=$(echo "$ORIGINAL_SCHEMA" | jq -r '.statistics.query.schema.fields[].name')
```

#### Execution Graph概要とステージ間依存関係
```bash
# ステージ間の依存関係とデータフローを分析
echo "=== ステージ間の依存関係 ==="
bq query --use_legacy_sql=false --format=pretty \
  --parameter="job_id:STRING:${JOB_ID}" "
  SELECT
    stage.name as stage_name,
    ARRAY_TO_STRING(ARRAY(SELECT CAST(s AS STRING) FROM UNNEST(stage.input_stages) AS s), ', ') as depends_on,
    CAST(stage.slot_ms AS INT64) as slot_ms,
    CAST(stage.parallel_inputs AS INT64) as parallel_units,
    CAST(stage.completed_parallel_inputs AS INT64) as completed_units,
    ROUND(100.0 * stage.completed_parallel_inputs / NULLIF(stage.parallel_inputs, 0), 1) as completion_rate
  FROM
    \`region-us\`.INFORMATION_SCHEMA.JOBS_BY_PROJECT,
    UNNEST(job_stages) AS stage
  WHERE
    job_id = @job_id
    AND creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
    AND creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  ORDER BY
    stage.id"

# 全ステージの概要を表示（上位10ステージ）
echo -e "\n=== Execution Graph概要 ==="
bq query --use_legacy_sql=false --format=pretty \
  --parameter="job_id:STRING:${JOB_ID}" "
  SELECT
    stage.name as stage_name,
    CAST(stage.slot_ms AS INT64) as slot_ms,
    CAST(stage.compute_ms_max AS INT64) as compute_ms,
    ROUND(CAST(stage.shuffle_output_bytes AS INT64) / 1048576, 1) as shuffle_mb,
    ROUND(SAFE_DIVIDE(
      CAST(stage.compute_ms_max AS FLOAT64),
      CAST(stage.compute_ms_avg AS FLOAT64)
    ), 2) as skew_ratio
  FROM
    \`region-us\`.INFORMATION_SCHEMA.JOBS_BY_PROJECT,
    UNNEST(job_stages) AS stage
  WHERE
    job_id = @job_id
    AND creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  ORDER BY
    stage.slot_ms DESC
  LIMIT 10"

# 変数の初期定義を確認（Inputステージのみで実カラム名が分かる）
echo -e "\n=== 変数の初期定義（Inputステージ）==="
bq query --use_legacy_sql=false --format=pretty \
  --parameter="job_id:STRING:${JOB_ID}" "
  SELECT
    stage.name as stage_name,
    substep as column_definition
  FROM
    \`region-us\`.INFORMATION_SCHEMA.JOBS_BY_PROJECT,
    UNNEST(job_stages) AS stage,
    UNNEST(stage.steps) AS step,
    UNNEST(step.substeps) AS substep
  WHERE
    job_id = @job_id
    AND stage.name LIKE '%Input%'
    AND step.kind = 'READ'
    AND substep LIKE '$%:%'  -- $10:column_name形式を検出
  LIMIT 20"
```

**注目メトリクス:**
- `depends_on`: このステージが依存する前段ステージ
- `parallel_units`: 並列実行可能な最大ワークユニット数
- `completion_rate`: 並列処理の完了率（低い場合は偏り）
- `column_definition`: Inputステージでの変数とカラム名の対応（例：$10:id）
  - **制限事項**: 後続ステージでは変数番号のみで実カラム名は不明
- `slot_ms`: スロット時間（全体的なコスト）
- `compute_ms`: 最大計算時間
- `shuffle_mb`: シャッフルデータ量（MB）
- `skew_ratio`: データスキュー率（2.0以上で問題）

#### 時系列スロット使用データ
```bash
echo -e "\n=== スロット使用状況の推移 ==="
bq query --use_legacy_sql=false --format=pretty \
  --parameter="job_id:STRING:${JOB_ID}" "
  SELECT
    FORMAT_TIMESTAMP('%H:%M:%S', period_start) as time,
    period_slot_ms as slot_ms,
    state,
    ROUND(period_shuffle_ram_usage_ratio, 2) as shuffle_ram_ratio
  FROM \`region-us\`.INFORMATION_SCHEMA.JOBS_TIMELINE_BY_PROJECT
  WHERE job_id = @job_id
    AND period_start >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  ORDER BY period_start
  LIMIT 20"
```

### 3. ボトルネック診断とSQL対応付け

#### Wait/Read/Compute/Writeメトリクスの詳細分析
```bash
# 各ステージのWait/Read/Compute/Write時間を詳細分析
echo "=== Wait/Read/Compute/Write詳細分析 ==="
bq query --use_legacy_sql=false --format=pretty \
  --parameter="job_id:STRING:${JOB_ID}" "
  SELECT
    stage.name as stage_name,
    CAST(stage.slot_ms AS INT64) as total_slot_ms,
    -- Wait時間分析（スケジューリング遅延）
    CAST(stage.wait_ms_avg AS INT64) as wait_avg,
    CAST(stage.wait_ms_max AS INT64) as wait_max,
    ROUND(stage.wait_ratio_max * 100, 1) as wait_pct_max,
    -- Read時間分析（データ入力性能）
    CAST(stage.read_ms_avg AS INT64) as read_avg,
    CAST(stage.read_ms_max AS INT64) as read_max,
    ROUND(stage.read_ratio_max * 100, 1) as read_pct_max,
    -- Compute時間分析（CPU処理）
    CAST(stage.compute_ms_avg AS INT64) as compute_avg,
    CAST(stage.compute_ms_max AS INT64) as compute_max,
    ROUND(stage.compute_ratio_max * 100, 1) as compute_pct_max,
    -- Write時間分析（出力性能）
    CAST(stage.write_ms_avg AS INT64) as write_avg,
    CAST(stage.write_ms_max AS INT64) as write_max,
    ROUND(stage.write_ratio_max * 100, 1) as write_pct_max
  FROM
    \`region-us\`.INFORMATION_SCHEMA.JOBS_BY_PROJECT,
    UNNEST(job_stages) AS stage
  WHERE
    job_id = @job_id
    AND creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  ORDER BY
    stage.slot_ms DESC
  LIMIT 10"

# 各処理の支配的な要素を特定（判定ではなく実測値で表示）
echo -e "\n=== ステージ別の支配的な処理要素 ==="
bq query --use_legacy_sql=false --format=pretty \
  --parameter="job_id:STRING:${JOB_ID}" "
  SELECT
    stage.name as stage_name,
    CAST(stage.slot_ms AS INT64) as slot_ms,
    -- 最も時間を消費している処理を特定
    CASE
      WHEN stage.wait_ratio_max = GREATEST(
        stage.wait_ratio_max,
        stage.read_ratio_max,
        stage.compute_ratio_max,
        stage.write_ratio_max
      ) THEN CONCAT('WAIT: ', CAST(ROUND(stage.wait_ratio_max * 100, 1) AS STRING), '%')
      WHEN stage.read_ratio_max = GREATEST(
        stage.wait_ratio_max,
        stage.read_ratio_max,
        stage.compute_ratio_max,
        stage.write_ratio_max
      ) THEN CONCAT('READ: ', CAST(ROUND(stage.read_ratio_max * 100, 1) AS STRING), '%')
      WHEN stage.compute_ratio_max = GREATEST(
        stage.wait_ratio_max,
        stage.read_ratio_max,
        stage.compute_ratio_max,
        stage.write_ratio_max
      ) THEN CONCAT('COMPUTE: ', CAST(ROUND(stage.compute_ratio_max * 100, 1) AS STRING), '%')
      ELSE CONCAT('WRITE: ', CAST(ROUND(stage.write_ratio_max * 100, 1) AS STRING), '%')
    END as dominant_operation,
    -- 参考: Googleドキュメントでは具体的な閾値は示されていない
    -- 各環境で実測値を基に判断することを推奨
    ROUND(GREATEST(
      stage.wait_ratio_max,
      stage.read_ratio_max,
      stage.compute_ratio_max,
      stage.write_ratio_max
    ) * 100, 1) as dominant_pct
  FROM
    \`region-us\`.INFORMATION_SCHEMA.JOBS_BY_PROJECT,
    UNNEST(job_stages) AS stage
  WHERE
    job_id = @job_id
    AND creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
    AND stage.slot_ms > 1000  -- 1秒以上のステージのみ
  ORDER BY
    stage.slot_ms DESC"
```

#### ステージとSQL要素の自動対応付け
```bash
# 元のクエリを取得して表示
echo "=== 元のクエリ ==="
bq query --use_legacy_sql=false --format=json \
  --parameter="job_id:STRING:${JOB_ID}" "
SELECT query
FROM \`region-us\`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
WHERE job_id = @job_id
  AND creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)" | jq -r '.[0].query'

# ステージとSQL操作の詳細な対応関係を分析
echo -e "\n=== ステージとSQL操作の対応付け ==="
bq query --use_legacy_sql=false --format=pretty \
  --parameter="job_id:STRING:${JOB_ID}" "
SELECT
  stage.name as stage_name,
  CAST(stage.slot_ms AS INT64) as slot_ms,
  ROUND(CAST(stage.shuffle_output_bytes AS INT64) / 1048576, 1) as shuffle_mb,
  CASE
    -- Input ステージの判定
    WHEN stage.name LIKE '%Input%' THEN
      (SELECT STRING_AGG(
        REGEXP_EXTRACT(substep, r'FROM \`?([^\`\s]+)\`?'), ', ')
       FROM UNNEST(stage.steps) AS step, UNNEST(step.substeps) AS substep
       WHERE substep LIKE 'FROM %' AND substep NOT LIKE 'FROM __stage%')
    -- Join ステージの判定
    WHEN EXISTS(
      SELECT 1 FROM UNNEST(stage.steps) AS step
      WHERE step.kind = 'JOIN'
    ) THEN
      CONCAT('JOIN処理: ',
        (SELECT REGEXP_EXTRACT(substep, r'(INNER|LEFT|RIGHT|FULL|CROSS).*JOIN')
         FROM UNNEST(stage.steps) AS step, UNNEST(step.substeps) AS substep
         WHERE substep LIKE '%JOIN%' LIMIT 1))
    -- Aggregate ステージの判定
    WHEN EXISTS(
      SELECT 1 FROM UNNEST(stage.steps) AS step
      WHERE step.kind = 'AGGREGATE'
    ) THEN 'GROUP BY処理'
    -- Sort ステージの判定
    WHEN EXISTS(
      SELECT 1 FROM UNNEST(stage.steps) AS step
      WHERE step.kind = 'SORT'
    ) THEN 'ORDER BY/SORT処理'
    -- Filter ステージの判定
    WHEN EXISTS(
      SELECT 1 FROM UNNEST(stage.steps) AS step
      WHERE step.kind = 'FILTER'
    ) THEN 'WHERE句フィルタ処理'
    ELSE 'その他の処理'
  END as sql_operation
FROM
  \`region-us\`.INFORMATION_SCHEMA.JOBS_BY_PROJECT,
  UNNEST(job_stages) AS stage
WHERE
  job_id = @job_id
ORDER BY
  stage.slot_ms DESC"

# JOIN操作の詳細情報を取得
echo -e "\n=== JOIN操作の詳細 ==="
bq query --use_legacy_sql=false --format=pretty \
  --parameter="job_id:STRING:${JOB_ID}" "
SELECT
  stage.name as stage_name,
  CAST(stage.slot_ms AS INT64) as slot_ms,
  SUBSTRING(substep, 1, 100) as join_condition
FROM
  \`region-us\`.INFORMATION_SCHEMA.JOBS_BY_PROJECT,
  UNNEST(job_stages) AS stage,
  UNNEST(stage.steps) AS step,
  UNNEST(step.substeps) AS substep
WHERE
  job_id = @job_id
  AND step.kind = 'JOIN'
  AND substep LIKE '%JOIN%'"

# 各ステップ種別の詳細分析
echo -e "\n=== ステップ種別ごとの処理内容 ==="
bq query --use_legacy_sql=false --format=pretty \
  --parameter="job_id:STRING:${JOB_ID}" "
SELECT
  step.kind as step_type,
  COUNT(*) as step_count,
  STRING_AGG(DISTINCT stage.name, ', ' ORDER BY stage.name LIMIT 3) as stages_using,
  CASE step.kind
    WHEN 'READ' THEN 'テーブルまたは中間シャッフルからデータ読み込み'
    WHEN 'WRITE' THEN 'テーブルまたはシャッフルへデータ書き込み'
    WHEN 'COMPUTE' THEN '式評価やSQL関数の実行'
    WHEN 'FILTER' THEN 'WHERE句、HAVING句の条件評価'
    WHEN 'SORT' THEN 'ORDER BY処理'
    WHEN 'AGGREGATE' THEN 'GROUP BYと集計関数の実行'
    WHEN 'JOIN' THEN 'テーブル結合処理'
    WHEN 'LIMIT' THEN '行数制限の適用'
    WHEN 'ANALYTIC_FUNCTION' THEN 'ウィンドウ関数の処理'
    ELSE 'その他の処理'
  END as description
FROM
  \`region-us\`.INFORMATION_SCHEMA.JOBS_BY_PROJECT,
  UNNEST(job_stages) AS stage,
  UNNEST(stage.steps) AS step
WHERE
  job_id = @job_id
GROUP BY
  step.kind
ORDER BY
  step_count DESC"
```

パフォーマンス問題を分析して、結果をテーブル形式で表示します：

```bash
# 最も遅いステージTOP5
echo "=== 最も時間のかかるステージ TOP5 ==="
bq query --use_legacy_sql=false --format=pretty \
  --parameter="job_id:STRING:${JOB_ID}" "
SELECT
  stage.name as stage_name,
  CAST(stage.slot_ms AS INT64) as slot_ms,
  CAST(stage.compute_ms_max AS INT64) as compute_ms_max,
  CAST(stage.wait_ms_max AS INT64) as wait_ms_max
FROM
  \`region-us\`.INFORMATION_SCHEMA.JOBS_BY_PROJECT,
  UNNEST(job_stages) AS stage
WHERE
  job_id = @job_id
ORDER BY
  stage.slot_ms DESC
LIMIT 5"

# データスキューの検出（スキュー率2.0以上）
echo -e "\n=== データスキュー検出（スキュー率 >= 2.0）==="
bq query --use_legacy_sql=false --format=pretty \
  --parameter="job_id:STRING:${JOB_ID}" "
SELECT
  stage.name as stage_name,
  ROUND(SAFE_DIVIDE(
    CAST(stage.compute_ms_max AS FLOAT64),
    CAST(stage.compute_ms_avg AS FLOAT64)
  ), 2) as skew_ratio
FROM
  \`region-us\`.INFORMATION_SCHEMA.JOBS_BY_PROJECT,
  UNNEST(job_stages) AS stage
WHERE
  job_id = @job_id
  AND SAFE_DIVIDE(
    CAST(stage.compute_ms_max AS FLOAT64),
    CAST(stage.compute_ms_avg AS FLOAT64)
  ) >= 2.0
ORDER BY
  skew_ratio DESC"

# 大量データシャッフル（100MB以上）
echo -e "\n=== 大量データシャッフル（>= 100MB）==="
bq query --use_legacy_sql=false --format=pretty \
  --parameter="job_id:STRING:${JOB_ID}" "
SELECT
  stage.name as stage_name,
  ROUND(CAST(stage.shuffle_output_bytes AS INT64) / 1048576, 1) as shuffle_mb
FROM
  \`region-us\`.INFORMATION_SCHEMA.JOBS_BY_PROJECT,
  UNNEST(job_stages) AS stage
WHERE
  job_id = @job_id
  AND CAST(stage.shuffle_output_bytes AS INT64) >= 104857600
ORDER BY
  shuffle_mb DESC"

# シャッフルスピルの分析（メモリ不足の検出）
echo -e "\n=== シャッフルスピル分析（メモリ不足検出）==="
bq query --use_legacy_sql=false --format=pretty \
  --parameter="job_id:STRING:${JOB_ID}" "
SELECT
  stage.name as stage_name,
  CAST(stage.slot_ms AS INT64) as slot_ms,
  ROUND(CAST(stage.shuffle_output_bytes AS INT64) / 1048576, 1) as shuffle_mb,
  ROUND(CAST(stage.shuffle_output_bytes_spilled AS INT64) / 1048576, 1) as spilled_mb,
  ROUND(100.0 * stage.shuffle_output_bytes_spilled / NULLIF(stage.shuffle_output_bytes, 0), 1) as spill_pct,
  CASE
    WHEN stage.shuffle_output_bytes_spilled > 0 THEN 'メモリ不足発生'
    ELSE 'メモリ十分'
  END as memory_status
FROM
  \`region-us\`.INFORMATION_SCHEMA.JOBS_BY_PROJECT,
  UNNEST(job_stages) AS stage
WHERE
  job_id = @job_id
  AND stage.shuffle_output_bytes > 0
ORDER BY
  spilled_mb DESC"
```

**分析結果の解釈:**
- **Stage名が"Join+"の場合**
  - substepsの`JOIN`句を確認して結合条件を特定
  - shuffle_output_bytesが大きい場合は、JOIN前の絞り込み不足
  - 例：`INNER HASH JOIN EACH WITH EACH ON $2 = $10` → user_idでの結合

- **Stage名が"Input"の場合**
  - substepsの`FROM`句で読み込みテーブルを特定
  - slot_msが大きい場合は、パーティション指定やクラスタリングの活用を検討

- **Stage名が"Sort+"の場合**
  - ORDER BY句やウィンドウ関数のソート処理
  - 最終段階のソートならLIMIT句との組み合わせを確認

- **Stage名が"Aggregate+"の場合**
  - substepsの`GROUP BY`句で集計キーを特定
  - 多段階の集約が発生している場合は、CTEでの事前集約を検討

- **スキュー率2.0以上** → 特定のキー値にデータが偏っている
- **シャッフル量100MB以上** → 大量データ移動が発生、JOIN順序の見直しが必要

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

#### 結果同一性の完全検証（行数＋スキーマ）
```bash
# 最適化後のクエリ結果を取得して行数を記録
echo "最適化後のクエリ結果を取得中..."
OPTIMIZED_ROWS=$(cat /tmp/optimized_query.sql | bq query --use_legacy_sql=false --use_cache=false --format=json | jq '. | length')
echo "最適化後の行数: $OPTIMIZED_ROWS"

# 最適化後のスキーマ確認
echo "スキーマを確認中..."
OPTIMIZED_SCHEMA=$(cat /tmp/optimized_query.sql | bq query --use_legacy_sql=false --format=json --dry_run)
OPTIMIZED_COLS=$(echo "$OPTIMIZED_SCHEMA" | jq '.statistics.query.schema.fields | length')
OPTIMIZED_COL_NAMES=$(echo "$OPTIMIZED_SCHEMA" | jq -r '.statistics.query.schema.fields[].name')

# 元の行数を取得（保存済みの結果から）
ORIGINAL_ROWS=$(jq '. | length' /tmp/original_results.json)

# 行数の比較（リファクタリングでは必須）
if [ "$ORIGINAL_ROWS" != "$OPTIMIZED_ROWS" ]; then
  echo "❌ エラー: 行数が一致しません（元: $ORIGINAL_ROWS, 最適化後: $OPTIMIZED_ROWS）"
  echo "これは真のリファクタリングではありません。結果が変わっています。"
  exit 1
else
  echo "✓ 行数が一致しました: $ORIGINAL_ROWS 行"
fi

# スキーマの比較
if [ "$ORIGINAL_COLS" != "$OPTIMIZED_COLS" ]; then
  echo "❌ エラー: 列数が一致しません（元: $ORIGINAL_COLS, 最適化後: $OPTIMIZED_COLS）"
  exit 1
else
  echo "✓ スキーマが一致しました（列数: $ORIGINAL_COLS）"
fi

# カラム名の差分確認
if [ "$ORIGINAL_COL_NAMES" != "$OPTIMIZED_COL_NAMES" ]; then
  echo "警告: カラム名に差分があります"
  diff <(echo "$ORIGINAL_COL_NAMES") <(echo "$OPTIMIZED_COL_NAMES")
fi
```

### 6. 改善効果の測定と反復改善

#### 最適化クエリの実行と新メトリクス取得
```bash
# 最適化クエリを実行してジョブIDを取得
echo "最適化クエリを実行中..."
NEW_JOB_ID=$(cat /tmp/optimized_query.sql | bq query --nosync --use_legacy_sql=false --use_cache=false --format=json | jq -r '.jobReference.jobId')
echo "新しいジョブID: $NEW_JOB_ID"
# ジョブの完了を待つ
bq wait "$NEW_JOB_ID"

# 新しいジョブのメトリクスを取得
NEW_JOB_INFO=$(bq query --use_legacy_sql=false --format=json \
  --parameter="job_id:STRING:${NEW_JOB_ID}" "
  SELECT
    total_slot_ms,
    total_bytes_processed,
    TIMESTAMP_DIFF(end_time, start_time, MILLISECOND) as elapsed_ms
  FROM \`region-us\`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
  WHERE job_id = @job_id")

# 改善度を計算（ORIGINAL_SLOT_MSは既に取得済み）
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
- プロジェクト: デフォルトプロジェクト
- リージョン: [LOCATION]
- 処理データ: [X GB]
- スロット時間: [Y ms]

## 検出された問題とSQL対応
1. [ステージS01: Input - 254秒] - bigquery-public-data.stackoverflow.comments
   - 14GBのシャッフル発生 → パーティション/クラスタリングの活用を推奨
2. [ステージS02: Join+ - 99秒] - INNER HASH JOIN ON c.user_id = u.id
   - 151MBのシャッフル → JOIN前にフィルタリングで絞り込み
3. [ステージS03: Sort+ - 2秒] - GROUP BY user_id, ORDER BY comments_count DESC
   - 最終集約処理 → CTEでの事前集約を検討

## 適用した最適化（優先度順）
1. [パターンA]: 期待効果60%削減
2. [パターンB]: 期待効果30%削減
3. [組み合わせ効果]: 合計75%削減見込み

## 最適化後のクエリ
\`\`\`sql
[生成したSQL]
\`\`\`

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
```
