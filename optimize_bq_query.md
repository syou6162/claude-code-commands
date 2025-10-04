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

### 2. 全体ボトルネックの特定

#### 基本情報の収集
```bash
# 元のクエリとメトリクスを取得
echo "基本情報を収集中..."
bq query --use_legacy_sql=false --format=json --parameter="job_id:STRING:${JOB_ID}" "
  SELECT query, total_slot_ms, total_bytes_processed
  FROM \`region-us\`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
  WHERE job_id = @job_id AND creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
" > /tmp/job_info.json

# 元クエリと基本メトリクスを保存
jq -r '.[0].query' /tmp/job_info.json > /tmp/original_query.sql
ORIGINAL_SLOT_MS=$(jq -r '.[0].total_slot_ms' /tmp/job_info.json)
echo "元のスロット時間: ${ORIGINAL_SLOT_MS}ms"
```

#### 最大のボトルネックステージを特定
**目的**: 全体のスロット時間の80%以上を占める真のボトルネックを見つける

```bash
echo "=== ボトルネックステージの特定 ==="
bq query --use_legacy_sql=false --format=pretty --parameter="job_id:STRING:${JOB_ID}" "
  SELECT
    stage.name as stage_name,
    CAST(stage.slot_ms AS INT64) as slot_ms,
    ROUND(100.0 * stage.slot_ms / SUM(stage.slot_ms) OVER(), 1) as pct_of_total
  FROM \`region-us\`.INFORMATION_SCHEMA.JOBS_BY_PROJECT,
       UNNEST(job_stages) AS stage
  WHERE job_id = @job_id
    AND creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  ORDER BY stage.slot_ms DESC
  LIMIT 5
"
```

**この結果から上位1-3ステージ（合計80%以上）を特定し、以降はそれらのみを分析対象とする**

### 3. ボトルネック根本原因の分析

**前提**: セクション2で特定したボトルネックステージ（TOP1-3）のみを詳細分析する

#### ボトルネックステージの処理内容を特定
**目的**: 各ボトルネックステージが何の処理をしているかをSQL要素と対応付ける

```bash
echo "=== ボトルネックステージの処理内容 ==="
# ステージ名からSQL操作を特定するクエリを実行
# 結果を見て: Input=テーブル読み込み、Join+=結合処理、Aggregate+=集約、Sort+=ソート
```

#### 根本原因の診断
各ボトルネックステージに対して以下を実行：

**Wait主体の場合**: スケジューリング遅延
```bash
echo "=== Wait時間分析 ==="
# wait_ratio_maxが高いステージのリソース競合状況を確認
```

**Read主体の場合**: データ読み込み効率
```bash
echo "=== Read時間分析 ==="
# 大量テーブルスキャンまたはパーティション設計の問題
```

**Compute主体の場合**: CPU処理負荷
```bash
echo "=== Compute時間分析 ==="
# データスキュー（compute_ms_max/compute_ms_avg >= 2.0）の確認
```

**Write主体の場合**: データ出力負荷
```bash
echo "=== Shuffle/Write分析 ==="
# 大量シャッフル（>100MB）やメモリスピルの確認
```

#### 元クエリとの対応付け
```bash
echo "=== 元クエリの確認 ==="
cat /tmp/original_query.sql
```

**agentの分析指示**:
1. ボトルネックステージの処理内容（JOIN/GROUP BY/ORDER BY等）を特定
2. Wait/Read/Compute/Writeの支配的要素を確認
3. 元クエリの該当箇所を特定
4. 根本原因（シャッフル量/スキュー/パーティション等）を診断

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
  WHERE job_id = @job_id
    AND creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)")

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
