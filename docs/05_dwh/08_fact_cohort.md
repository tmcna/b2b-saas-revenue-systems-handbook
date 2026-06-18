# コホート分析ファクトテーブル——顧客の継続率を時系列で捉える

「解約率が5%です」という数字は、何も教えてくれません。どの時期に獲得した顧客が解約しているのか、解約は時間が経つほど増えるのか減るのか——これを答えるのがコホート分析です。コホート分析のためのファクトテーブルは、MRR スナップショット（[DWH編 7](07_fact_mrr.md)）とは異なる粒度で設計します。

---

## コホートとは

**コホート（Cohort）** は、同じ時期に特定のイベントを経験した顧客のグループです。SaaS では通常「契約開始月」でグループを定義します。

```
2024年1月コホート：1月に契約を開始した顧客のグループ
2024年2月コホート：2月に契約を開始した顧客のグループ
```

コホートごとに、時間が経つにつれてどれだけの収益・顧客数が維持されているかを追うことで、**製品の改善・顧客セグメントの質・オンボーディングの効果**を評価できます。

---

## コホート分析テーブルの粒度

コホートファクトテーブルの粒度は「**コホート月 × 経過月数**」です。

| コホート月 | 経過月数 (Period) | 内容 |
|---|---|---|
| 2024-01 | 0 | 1月開始顧客の、1月時点の MRR |
| 2024-01 | 1 | 1月開始顧客の、2月時点の MRR |
| 2024-01 | 2 | 1月開始顧客の、3月時点の MRR |
| 2024-02 | 0 | 2月開始顧客の、2月時点の MRR |

Period 0 が「入会時点（Baseline）」であり、以降の MRR を Period 0 で割ることで継続率を計算します。

---

## テーブル設計

```sql
fact_cohort_mrr (
  -- 粒度: コホート月 × 経過月数
  cohort_month          DATE,       -- 顧客の契約開始月（例：2024-01-31）
  period_number         INTEGER,    -- 経過月数（0 = 契約開始月）
  observation_month     DATE,       -- 実際の観測月（例：2024-03-31）

  -- コホートサイズ（Period 0 での基準値）
  cohort_customers      INTEGER,    -- コホートの初期顧客数
  cohort_mrr            DECIMAL,    -- コホートの初期 MRR 合計

  -- 観測時点の値
  active_customers      INTEGER,    -- 観測月時点のアクティブ顧客数
  active_mrr            DECIMAL,    -- 観測月時点の MRR 合計

  -- 継続率
  customer_retention    DECIMAL,    -- active_customers / cohort_customers
  revenue_retention     DECIMAL     -- active_mrr / cohort_mrr
)
```

---

## 典型的なコホートテーブル（三角形）

コホート分析の結果は「三角形テーブル」として表現されます。

```
コホート月  | Period 0 | Period 1 | Period 2 | Period 3 | ...
------------|----------|----------|----------|----------|
2024-01     |  100%    |   92%    |   87%    |   85%    |
2024-02     |  100%    |   90%    |   86%    |          |
2024-03     |  100%    |   91%    |          |          |
2024-04     |  100%    |          |          |          |
```

右下が空白になるのは、まだ観測できない将来の期間です。

---

## コホート分析からわかること

**継続率の形状**

```
急落型：Period 1-2 で大きく落ち、その後安定
  → オンボーディングに課題がある

緩やかな右肩下がり：毎月一定割合で解約が続く
  → プロダクトの粘着性が不足している

安定型（長期でフラット）：初期の解約後は高水準を維持
  → コアユーザーが明確に存在する
```

**コホート間の比較**

新しいコホートが古いコホートより継続率が高ければ、製品改善・オンボーディング改善の効果が出ています。

---

## 顧客数ベース vs 収益ベース

コホート分析は「顧客数（Customer Retention）」と「収益（Revenue Retention）」の両方で計算します。

```
例：1月コホート 10社・MRR 100万円
6ヶ月後：8社が残存・MRR 110万円

顧客継続率（Customer Retention）= 8 / 10 = 80%
収益継続率（Revenue Retention）= 110 / 100 = 110%
```

収益継続率が100%を超えるのは、残存顧客がアップセルで収益を拡大したためです。顧客数は減っても収益が増えるこの状態を「**ネガティブチャーン（Negative Churn）**」と呼びます。

---

## 計算クエリ

```sql
-- MRR スナップショットからコホートファクトを生成
WITH customer_cohorts AS (
  SELECT
    customer_id,
    MIN(snapshot_month) AS cohort_month
  FROM fact_mrr
  WHERE mrr > 0
  GROUP BY customer_id
),
cohort_data AS (
  SELECT
    c.cohort_month,
    DATEDIFF('month', c.cohort_month, m.snapshot_month) AS period_number,
    m.snapshot_month,
    COUNT(DISTINCT m.customer_id) AS active_customers,
    SUM(m.mrr) AS active_mrr
  FROM fact_mrr m
  JOIN customer_cohorts c ON m.customer_id = c.customer_id
  WHERE m.mrr > 0
  GROUP BY 1, 2, 3
)
SELECT
  cohort_month,
  period_number,
  observation_month,
  FIRST_VALUE(active_customers) OVER (
    PARTITION BY cohort_month ORDER BY period_number
  ) AS cohort_customers,
  FIRST_VALUE(active_mrr) OVER (
    PARTITION BY cohort_month ORDER BY period_number
  ) AS cohort_mrr,
  active_customers,
  active_mrr,
  active_customers * 1.0 / FIRST_VALUE(active_customers) OVER (
    PARTITION BY cohort_month ORDER BY period_number
  ) AS customer_retention,
  active_mrr / FIRST_VALUE(active_mrr) OVER (
    PARTITION BY cohort_month ORDER BY period_number
  ) AS revenue_retention
FROM cohort_data
ORDER BY cohort_month, period_number
```

---

## よくある課題

**コホートの定義のぶれ**
「契約開始日」と「初回請求日」と「初回入金日」は異なります。コホート月の定義を統一しておく必要があります。通常は「サービス開始日（サブスクリプション開始日）」が最も実態に合います。

**トライアルの扱い**
無料トライアルからの転換を「コホートの開始」とするか、有料転換をコホートの開始とするかで、継続率の意味が変わります。

**拡張の扱い**
アップセルによって MRR が増加した顧客を「継続」に含めるのは自然ですが、MRR が 0 になった後に再契約した顧客を継続として扱うかどうかを決めておく必要があります。

---

**次回：** [NRR ファクトテーブル——収益の自己増殖力を測る](09_fact_nrr.md)
