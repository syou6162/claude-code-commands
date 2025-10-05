---
allowed-tools: Bash(bq query:*), Bash(bq wait:*), Bash(tee:*)
description: "BigQueryクエリのパフォーマンスを分析し、2倍以上の性能改善を目標とした最適化を提案します。"
---

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

**ステップ1: 入力の種類を判定**

まず、$ARGUMENTSがどの形式か判定してください：

- `.sql`で終わる → **ファイルパス** (ステップ2-A)
- `bquxjob_`、`job_`、`bq-`で始まる → **ジョブID** (ステップ2-B)
- それ以外 → **SQLクエリ文字列** (ステップ2-C)

**ステップ2-A: ファイルパスの場合**

- ファイルの存在確認: `!test -f "$ARGUMENTS" && echo "ファイル存在" || echo "ファイルなし"`
- ジョブIDを取得（出力を`JOB_ID`として認識）: `!cat "$ARGUMENTS" | bq query --nosync --use_legacy_sql=false --use_cache=false --format=json | jq -r '.jobReference.jobId'`
  - `JOB_ID`をシェル変数として設定する必要はありません
- ジョブの完了を待機: `!bq wait "<JOB_ID>"`

**ステップ2-B: ジョブIDの場合**

`$ARGUMENTS`がそのまま`JOB_ID`です。この場合はジョブIDが既に確定しているため、waitは不要です。

**ステップ2-C: SQLクエリ文字列の場合**

- ジョブIDを取得（出力を`JOB_ID`として認識）: `!echo "$ARGUMENTS" | bq query --nosync --use_legacy_sql=false --use_cache=false --format=json | jq -r '.jobReference.jobId'`
  - `JOB_ID`をシェル変数として設定する必要はありません
- ジョブの完了を待機: `!bq wait "<JOB_ID>"`

### 2. 全体ボトルネックの特定

#### 基本情報の収集
- 以下のクエリで元のクエリと基本メトリクス(スロット時間、スキャン量)を取得

```bash
bq query --use_legacy_sql=false --format=json --parameter="job_id:STRING:<JOB_ID>" "
  SELECT query, total_slot_ms, total_bytes_processed
  FROM region-us.INFORMATION_SCHEMA.JOBS_BY_PROJECT
  WHERE job_id = @job_id AND creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
" | tee /tmp/job_info.json | head -n 10
```

- 元クエリと基本メトリクスを保存
  - `cat /tmp/job_info.json | jq -r '.[0].query' | tee /tmp/original_query.sql`

#### 最大のボトルネックステージを特定
**目的**: 全体のスロット時間の80%以上を占める真のボトルネックを見つける

```bash
bq query --use_legacy_sql=false --format=pretty --parameter="job_id:STRING:<JOB_ID>" "
  SELECT
    stage.name as stage_name,
    CAST(stage.slot_ms AS INT64) as slot_ms,
    ROUND(100.0 * stage.slot_ms / SUM(stage.slot_ms) OVER(), 1) as pct_of_total
  FROM region-us.INFORMATION_SCHEMA.JOBS_BY_PROJECT,
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
特定されたボトルネックステージについて詳細分析を実行：

```bash
# 各ボトルネックステージの詳細分析（TOP1-3のみ）
echo "=== ボトルネックステージの詳細分析 ==="
bq query --use_legacy_sql=false --format=pretty --parameter="job_id:STRING:<JOB_ID>" "
  SELECT
    stage.name,
    CAST(stage.slot_ms AS INT64) as slot_ms,
    ROUND(stage.wait_ratio_max * 100, 1) as wait_pct,
    ROUND(stage.read_ratio_max * 100, 1) as read_pct,
    ROUND(stage.compute_ratio_max * 100, 1) as compute_pct,
    ROUND(stage.write_ratio_max * 100, 1) as write_pct,
    ROUND(CAST(stage.shuffle_output_bytes AS INT64) / 1048576, 1) as shuffle_mb
  FROM region-us.INFORMATION_SCHEMA.JOBS_BY_PROJECT,
       UNNEST(job_stages) AS stage
  WHERE job_id = @job_id
    AND creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
    AND stage.name IN ('特定されたボトルネックステージ名1', '名前2', '名前3')
  ORDER BY stage.slot_ms DESC
"

# ステージの処理内容を特定
echo "=== ステージの処理内容特定 ==="
bq query --use_legacy_sql=false --format=pretty --parameter="job_id:STRING:<JOB_ID>" "
  SELECT
    stage.name,
    CASE
      WHEN EXISTS(SELECT 1 FROM UNNEST(stage.steps) AS step WHERE step.kind = 'READ') THEN 'READ処理'
      WHEN EXISTS(SELECT 1 FROM UNNEST(stage.steps) AS step WHERE step.kind = 'WRITE') THEN 'WRITE処理'
      WHEN EXISTS(SELECT 1 FROM UNNEST(stage.steps) AS step WHERE step.kind = 'COMPUTE') THEN 'COMPUTE処理'
      WHEN EXISTS(SELECT 1 FROM UNNEST(stage.steps) AS step WHERE step.kind = 'FILTER') THEN 'FILTER処理'
      WHEN EXISTS(SELECT 1 FROM UNNEST(stage.steps) AS step WHERE step.kind = 'JOIN') THEN 'JOIN処理'
      WHEN EXISTS(SELECT 1 FROM UNNEST(stage.steps) AS step WHERE step.kind = 'AGGREGATE') THEN 'AGGREGATE処理'
      WHEN EXISTS(SELECT 1 FROM UNNEST(stage.steps) AS step WHERE step.kind = 'ANALYTIC') THEN 'ANALYTIC処理'
      WHEN EXISTS(SELECT 1 FROM UNNEST(stage.steps) AS step WHERE step.kind = 'SORT') THEN 'SORT処理'
      WHEN EXISTS(SELECT 1 FROM UNNEST(stage.steps) AS step WHERE step.kind = 'LIMIT') THEN 'LIMIT処理'
      ELSE 'その他'
    END as primary_operation
  FROM region-us.INFORMATION_SCHEMA.JOBS_BY_PROJECT,
       UNNEST(job_stages) AS stage
  WHERE job_id = @job_id
    AND creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  ORDER BY stage.slot_ms DESC
  LIMIT 5
"
```

#### 元クエリとの対応付けと分析結果記録
```bash
echo "=== 元クエリの確認 ==="
cat /tmp/original_query.sql

# ボトルネック分析結果を記録ファイルに保存
echo "=== ボトルネック分析結果 ===" > /tmp/bottleneck_analysis.txt
echo "分析日時: $(date)" >> /tmp/bottleneck_analysis.txt
echo "" >> /tmp/bottleneck_analysis.txt

echo "## 特定されたボトルネックステージ:" >> /tmp/bottleneck_analysis.txt
# agentがここでボトルネックステージの詳細を追記
# 例: echo "- S02_Join+: 99秒 (全体の45%)" >> /tmp/bottleneck_analysis.txt

echo "" >> /tmp/bottleneck_analysis.txt
echo "## 根本原因分析:" >> /tmp/bottleneck_analysis.txt
# agentがWait/Read/Compute/Write分析結果を追記
# 例: echo "- 支配的要素: Shuffle/Write (151MB)" >> /tmp/bottleneck_analysis.txt

echo "" >> /tmp/bottleneck_analysis.txt
echo "## 対応するSQL箇所:" >> /tmp/bottleneck_analysis.txt
# agentが元クエリの該当部分を特定して追記
# 例: echo "- INNER JOIN users u ON c.user_id = u.id" >> /tmp/bottleneck_analysis.txt
```

**agentの分析指示**:
1. 上記クエリ結果からボトルネックステージの処理内容を特定
2. Wait/Read/Compute/Writeの支配的要素を確認
3. 元クエリの該当箇所を特定
4. 分析結果を`/tmp/bottleneck_analysis.txt`に具体的に記録

### 4. ボトルネック対応の最適化パターン選択

**特定されたボトルネック要因に基づいて、2倍改善を狙える最適化パターンを選択**

#### Input段階のボトルネック → データ読み込み最適化
- **パーティション絞り込み**: WHERE句での日付/地域等の限定
- **クラスタリング活用**: JOIN/GROUP BYキーでの事前ソート
- **列選択最適化**: SELECT *を避けて必要列のみ

#### Join段階のボトルネック → 結合処理最適化
- **JOIN前データ削減**: 事前フィルタリング/集約で行数削減
- **JOIN順序最適化**: 小テーブル→大テーブルの順序
- **EXISTS/IN変換**: 相関サブクエリから効率的な形式へ

#### Aggregate/Sort段階のボトルネック → 集約処理最適化
- **段階的集約**: 複数CTEでの事前集約
- **LIMIT早期適用**: TOP-N処理での不要計算回避
- **ウィンドウ関数最適化**: PARTITION BY句の最適化

```bash
# 選択した最適化パターンを記録
echo "=== 適用する最適化パターン ===" > /tmp/applied_optimizations.txt
echo "分析日時: $(date)" >> /tmp/applied_optimizations.txt
echo "" >> /tmp/applied_optimizations.txt

# agentが該当するパターンを記録
# 例: echo "1. JOIN前データ削減: users テーブルの事前フィルタリング (期待効果: 60%削減)" >> /tmp/applied_optimizations.txt
# 例: echo "2. JOIN順序最適化: 小テーブル first (期待効果: 25%削減)" >> /tmp/applied_optimizations.txt
```

**agent指示**:
1. ボトルネック診断結果（`/tmp/bottleneck_analysis.txt`）を参照
2. 該当するパターンのみ選択し、2倍改善見込みを計算
3. 選択理由と期待効果を`/tmp/applied_optimizations.txt`に記録

### 5. 最適化クエリの実装と検証

#### 最適化SQLの生成と保存
```bash
echo "最適化クエリを生成中..."
cat /tmp/original_query.sql

# agentが最適化したクエリを保存（具体的な実装が必要）
cat > /tmp/optimized_query.sql << 'EOF'
-- agentがここで最適化後のクエリを具体的に記述
-- 例: WITH filtered_users AS (SELECT * FROM users WHERE active = true)
-- 例: SELECT ... FROM comments c INNER JOIN filtered_users u ON c.user_id = u.id
EOF

echo "最適化クエリを /tmp/optimized_query.sql に保存しました"
cat /tmp/optimized_query.sql
```

#### リファクタリング結果の同一性検証
**重要**: 最適化は性能改善のみで、結果は完全に同一である必要がある

**チェックサムによる検証手順**:

1. **元クエリのチェックサムを計算**
  - `/tmp/original_query.sql`の内容を以下のクエリでラップして実行
  - `SELECT BIT_XOR(FARM_FINGERPRINT(TO_JSON_STRING(t))) as checksum FROM (<元クエリ>) AS t`
  - 結果のチェックサム値を記録

2. **最適化クエリのチェックサムを計算**
  - `/tmp/optimized_query.sql`の内容を同様にラップして実行
  - `SELECT BIT_XOR(FARM_FINGERPRINT(TO_JSON_STRING(t))) as checksum FROM (<最適化クエリ>) AS t`
  - 結果のチェックサム値を記録

3. **チェックサム値を比較**
  - 2つのチェックサム値が完全に一致することを確認
  - 不一致の場合は最適化を中止し、原因を調査

**注意**: この手法は行の順序に依存しないため、ORDER BY句の有無に関わらず結果の同一性を検証できます

### 6. 性能改善効果の測定

#### 最適化後のメトリクス取得
```bash
# 最適化クエリの性能測定のための実行
echo "最適化クエリの性能測定中..."
NEW_JOB_ID=$(cat /tmp/optimized_query.sql | bq query --nosync --use_legacy_sql=false --use_cache=false --format=json | jq -r '.jobReference.jobId')
bq wait "$NEW_JOB_ID"

# 元のスロット時間を取得（セクション2で保存したjsonから）
ORIGINAL_SLOT_MS=$(jq -r '.[0].total_slot_ms' /tmp/job_info.json)

# 新しいジョブのメトリクス取得
bq query --use_legacy_sql=false --format=json --parameter="job_id:STRING:<NEW_JOB_ID>" "
  SELECT total_slot_ms
  FROM region-us.INFORMATION_SCHEMA.JOBS_BY_PROJECT
  WHERE job_id = @job_id AND creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
" | tee /tmp/new_job_info.json

NEW_SLOT_MS=$(jq -r '.[0].total_slot_ms' /tmp/new_job_info.json)

# 改善率の計算
IMPROVEMENT_RATIO=$(echo "scale=2; $ORIGINAL_SLOT_MS / $NEW_SLOT_MS" | bc)
echo "改善率: ${IMPROVEMENT_RATIO}x (元: ${ORIGINAL_SLOT_MS}ms → 後: ${NEW_SLOT_MS}ms)"
```

#### 2倍改善達成判定と反復プロセス
```bash
echo "Iteration 1: ${IMPROVEMENT_RATIO}x" >> /tmp/optimization_history.txt

if (( $(echo "$IMPROVEMENT_RATIO >= 2.0" | bc) )); then
  echo "✅ 目標達成: ${IMPROVEMENT_RATIO}x改善"
else
  echo "⚠️  未達成: ${IMPROVEMENT_RATIO}x改善（目標: 2.0x）"
  # セクション2-3を再実行して追加ボトルネックを特定
  # 最大3回まで反復可能
fi
```

### 7. 最終レポート生成

**目的**: 2倍改善達成の根拠と再現可能な手順を記録

```bash
# レポート生成に必要な変数を取得
ORIGINAL_SLOT_MS=$(jq -r '.[0].total_slot_ms' /tmp/job_info.json)
ORIGINAL_ROWS=$(jq '. | length' /tmp/original_results.json)
NEW_SLOT_MS=$(jq -r '.[0].total_slot_ms' /tmp/new_job_info.json)
IMPROVEMENT_RATIO=$(echo "scale=2; $ORIGINAL_SLOT_MS / $NEW_SLOT_MS" | bc)

# レポート生成
cat > /tmp/optimization_report.md << EOF
# BigQuery最適化レポート

## 実行サマリー
- **元ジョブID**: ${JOB_ID}
- **元スロット時間**: ${ORIGINAL_SLOT_MS}ms
- **最終改善率**: ${IMPROVEMENT_RATIO}x
- **目標達成**: \$([ \$(echo "\$IMPROVEMENT_RATIO >= 2.0" | bc) -eq 1 ] && echo "✅ 達成" || echo "❌ 未達成")

## 特定されたボトルネック
\$(cat /tmp/bottleneck_analysis.txt)

## 適用した最適化手法
\$(cat /tmp/applied_optimizations.txt)

## 最適化後のクエリ
\`\`\`sql
\$(cat /tmp/optimized_query.sql)
\`\`\`

## 検証結果
- **結果一致**: ✅ ${ORIGINAL_ROWS}行で完全一致
- **性能改善**: ${ORIGINAL_SLOT_MS}ms → ${NEW_SLOT_MS}ms

EOF

echo "レポートを /tmp/optimization_report.md に保存しました"
```

**完了条件**:
- 2倍以上の改善達成 OR 3回の反復完了
- 結果の同一性確認済み
- 再現可能な最適化手順の記録
