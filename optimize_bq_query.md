---
allowed-tools: Bash(bq query:*), Bash(bq wait:*), Bash(tee:*), Write(/tmp/**), Edit(/tmp/**), Read(/tmp/**)
description: "BigQueryクエリのパフォーマンスを分析し、2倍以上の性能改善を目標とした最適化を提案します。"
---

## 前提条件
- **リージョン**: US（region-us）固定
- **プロジェクト**: デフォルトプロジェクト（`gcloud config get-value project`で設定済み）
- **権限**: INFORMATION_SCHEMAへのアクセス権限あり
- **スキャン量削減**: INFORMATION_SCHEMAは直近7日間のみ検索（creation_timeで絞り込み）
- クエリの`FROM`句にバッククオートの付与は禁止
  - バッククオートを付与すると、Bash toolがコマンドと勘違いをする
  - そのため、毎回ユーザーの許可が必要になってしまうため

## 分析対象
- 入力: $ARGUMENTS(SQLファイルパス `.sql`)
- SQLのファイルのみで最適化内容を判断するのではなく、実際にクエリを実行してジョブ情報を取得し、ボトルネック分析を行う

## 実行手順

### 1. クエリ実行とジョブIDの取得

**前提**: `$ARGUMENTS`は最適化対象のSQLクエリが記述された`.sql`ファイルのパスです

1. クエリを実行してジョブIDを取得
  - `cat "$ARGUMENTS" | bq query --nosync --use_legacy_sql=false --use_cache=false --format=json | jq -r '.jobReference.jobId'`
  - 取得したジョブIDを`JOB_ID`として以降の分析で使用
  - `JOB_ID`をシェル変数として設定する必要はありません

2. ジョブの完了を待機
  - `bq wait "<JOB_ID>"`でジョブが完了するまで待機

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

#### 根本原因の診断

1. **ボトルネックステージの詳細メトリクスを取得**
  - セクション2で特定したボトルネックステージ（TOP1-3）に対して以下のクエリを実行:

```bash
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
```

2. **ステージの処理内容を特定**
  - 各ステージがどの種類の処理を行っているかを以下のクエリで確認:

```bash
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

1. **元クエリを確認**
  - `cat "$ARGUMENTS"`で元クエリの内容を表示

2. **ボトルネック分析結果ファイルの初期化**
  - Writeツールを使って`/tmp/bottleneck_analysis.md`を以下の内容で作成:

```markdown
## 特定されたボトルネックステージ:
<!-- 例: - S02_Join+: 99000ms (全体の45%) -->
<!-- 例: - S03_Aggregate: 55000ms (全体の25%) -->

## 根本原因分析:
<!-- 例: - S02_Join+の支配的要素: Write (shuffle_mb: 151MB), wait_pct: 5%, compute_pct: 35% -->
<!-- 例: - S03_Aggregateの支配的要素: Compute (compute_pct: 80%), read_pct: 15% -->

## 対応するSQL箇所:
<!-- 例: - S02_Join+に対応: INNER JOIN users u ON c.user_id = u.id (行11-12) -->
<!-- 例: - S03_Aggregateに対応: GROUP BY category, date (行18) -->
```

3. **分析結果の記録**
  - 上記bqクエリ結果を参照し、Editツールで各セクションに情報を追記

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

**最適化パターンの選択と記録**:

1. **ボトルネック診断結果を参照**
  - `/tmp/bottleneck_analysis.md`の内容を確認

2. **最適化パターンファイルの初期化**
  - Writeツールを使って`/tmp/applied_optimizations.md`を以下の内容で作成:

```markdown
## 適用する最適化パターン
<!-- 例: 1. JOIN前データ削減: users テーブルの事前フィルタリング (期待効果: 60%削減) -->
<!-- 例: 2. JOIN順序最適化: 小テーブル first (期待効果: 25%削減) -->
```

3. **最適化パターンの記録**
  - ボトルネック要因に対応するパターンを選択
  - 各パターンの期待効果を計算し、2倍改善の見込みを確認
  - Editツールで具体的な最適化内容を追記

### 5. 最適化クエリの実装
1. **元クエリを確認**
  - `cat "$ARGUMENTS"`で元クエリの内容を確認
2. **最適化クエリの生成と保存**
  - セクション4で選択した最適化パターンを適用
  - Writeツールを使って`/tmp/optimized_query.sql`に最適化後のクエリを保存
  - 最適化例:

```sql
-- 例: JOIN前データ削減
WITH filtered_users AS (
  SELECT * FROM users WHERE active = true
)
SELECT ...
FROM comments c
INNER JOIN filtered_users u ON c.user_id = u.id
```

3. **最適化内容の確認**
  - `cat /tmp/optimized_query.sql`で保存した最適化クエリを表示

### 6. リファクタリング結果の同一性検証
- **重要**: 最適化は性能改善のみで、出力結果は完全に同一である必要がある
- **注意**: `BIT_XOR`と`FARM_FINGERPRINT`は行の順序に依存しないため、結果の同一性を検証できる

**BigQueryチェックサムによる検証手順**:

1. **元クエリの結果テーブル名を取得してチェックサムを計算**
  - `JOB_ID`(セクション1で取得済み)から以下のコマンドを実行し、`DESTINATION_TABLE`として設定
    - `bq query --use_legacy_sql=false --format=json --parameter="job_id:STRING:<JOB_ID>" "SELECT destination_table FROM region-us.INFORMATION_SCHEMA.JOBS_BY_PROJECT WHERE job_id = @job_id AND creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)" | jq -r '.[0].destination_table | [.project_id, .dataset_id, .table_id] | join(".")'`
  - `DESTINATION_TABLE`に対して、チェックサムを計算
    - `bq query --use_legacy_sql=false --format=json "SELECT BIT_XOR(FARM_FINGERPRINT(TO_JSON_STRING(t))) as checksum FROM <DESTINATION_TABLE> AS t" | jq -r '.[0].checksum'`
  - チェックサム値（整数文字列）を記録

2. **最適化クエリの結果テーブル名を取得してチェックサムを計算**
  - `NEW_JOB_ID`(セクション6で取得)から以下のコマンドを実行し、`DESTINATION_TABLE`として設定
    - `bq query --use_legacy_sql=false --format=json --parameter="job_id:STRING:<NEW_JOB_ID>" "SELECT destination_table FROM region-us.INFORMATION_SCHEMA.JOBS_BY_PROJECT WHERE job_id = @job_id AND creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)" | jq -r '.[0].destination_table | [.project_id, .dataset_id, .table_id] | join(".")'`
  - `DESTINATION_TABLE`に対して、チェックサムを計算
    - `bq query --use_legacy_sql=false --format=json "SELECT BIT_XOR(FARM_FINGERPRINT(TO_JSON_STRING(t))) as checksum FROM <DESTINATION_TABLE> AS t" | jq -r '.[0].checksum'`
  - チェックサム値（整数文字列）を記録

3. **チェックサム値を比較**
  - 2つのチェックサム値が完全に一致することを確認
  - 不一致の場合は最適化を中止し、原因を調査

### 7. 性能改善効果の測定
- 最適化後のクエリを実行し、同様にジョブIDを取得してメトリクスを収集
  - `cat /tmp/optimized_query.sql | bq query --nosync --use_legacy_sql=false --use_cache=false --format=json | jq -r '.jobReference.jobId'`
  - 取得したジョブIDを`NEW_JOB_ID`として以降の分析で使用
  - `NEW_JOB_ID`をシェル変数として設定する必要はありません
- `bq wait "<NEW_JOB_ID>"`でジョブが完了するまで待機
- 元のスロット時間を取得（セクション2で保存したjsonから）
  - !`cat /tmp/job_info.json | jq -r '.[0].total_slot_ms'`
- 以下のコマンドで、最適化後のスロット時間を取得

```bash
bq query --use_legacy_sql=false --format=json --parameter="job_id:STRING:<NEW_JOB_ID>" "
  SELECT total_slot_ms
  FROM region-us.INFORMATION_SCHEMA.JOBS_BY_PROJECT
  WHERE job_id = @job_id AND creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
" | jq -r '.[0].total_slot_ms'
```

#### 改善率の計算と判定:
- 元のスロット時間と最適化後のスロット時間を比較し、改善率を計算
  - `bc`コマンドなどは使用せず、Agent自身が計算を行う
- 2倍以上の改善が達成されたかを判定
  - 改善率が2倍以上であれば成功とし、最適化プロセスを終了
  - 達成されていない場合は、セクション3で特定したボトルネックに戻り、セクション4-6を繰り返す

### 8. 最終レポート生成

**目的**: 2倍改善達成の根拠と再現可能な手順を記録

1. **必要な情報を収集**
  - 元のスロット時間: `cat /tmp/job_info.json | jq -r '.[0].total_slot_ms'`で取得
  - 最適化後のスロット時間: セクション7で取得済み
  - 改善率: Agent自身が計算
  - 目標達成状況: 改善率が2.0倍以上かを判定

2. **最終レポートの生成**
  - Writeツールを使って`/tmp/optimization_report.md`を以下の構造で作成:

```markdown
# BigQuery最適化レポート

## 実行サマリー
- **元クエリファイル**: <$ARGUMENTSの値>
- **元ジョブID**: <JOB_ID>
- **元スロット時間**: <ORIGINAL_SLOT_MS>ms
- **最終改善率**: <IMPROVEMENT_RATIO>x
- **目標達成**: <達成状況（✅ 達成 or ❌ 未達成）>

## 特定されたボトルネック
<bottleneck_analysis.mdの内容をReadツールで読み込んで転記>

## 適用した最適化手法
<applied_optimizations.mdの内容をReadツールで読み込んで転記>

## 最適化後のクエリ
```sql
<optimized_query.sqlの内容をReadツールで読み込んで転記>
```

## 検証結果
- **結果一致**: ✅ チェックサム一致で完全同一
- **性能改善**: <ORIGINAL_SLOT_MS>ms → <NEW_SLOT_MS>ms
```

3. **レポートの確認**
  - `cat /tmp/optimization_report.md`でレポート内容を表示

**完了条件**:
- 2倍以上の改善達成 OR 5回の反復完了
- 結果の同一性確認済み
- 再現可能な最適化手順の記録
