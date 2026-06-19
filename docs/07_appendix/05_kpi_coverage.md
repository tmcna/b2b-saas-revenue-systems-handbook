# KPIカバレッジマップ——このモデルで算出できる指標・できない指標

本書で設計したデータモデルを使って、どのSaaS主要KPIが算出できるかを一覧で示します。算出できるものは計算式とソーステーブルを、算出できないものは不足しているデータを記載します。

---

## 算出できるKPI

### 収益指標

| KPI | 計算式 | ソーステーブル |
|-----|--------|-------------|
| **MRR**（月次経常収益） | `SUM(mrr)` WHERE mrr > 0 | `fact_mrr` |
| **ARR**（年次経常収益） | MRR × 12 | `fact_mrr` |
| **New MRR** | `SUM(mrr_movement_amount)` WHERE mrr_movement_type = 'new' | `fact_mrr` |
| **Expansion MRR** | `SUM(expansion_mrr)` | `fact_nrr_monthly` |
| **Contraction MRR** | `SUM(contraction_mrr)` | `fact_nrr_monthly` |
| **Churn MRR** | `SUM(churn_mrr)` | `fact_nrr_monthly` |
| **ARPU**（顧客1社あたり平均MRR） | `SUM(mrr) / COUNT(DISTINCT customer_id)` WHERE mrr > 0 | `fact_mrr` |
| **ACV**（年間契約金額） | `SUM(arr_amount)` WHERE quote_status = 'accepted' | `fact_quote_line` |

```sql
-- 月次MRRブリッジ（New / Expansion / Contraction / Churn の内訳）
SELECT
  snapshot_month,
  SUM(CASE WHEN mrr_movement_type = 'new'         THEN mrr_movement_amount ELSE 0 END) AS new_mrr,
  SUM(CASE WHEN mrr_movement_type = 'expansion'   THEN mrr_movement_amount ELSE 0 END) AS expansion_mrr,
  SUM(CASE WHEN mrr_movement_type = 'contraction' THEN mrr_movement_amount ELSE 0 END) AS contraction_mrr,
  SUM(CASE WHEN mrr_movement_type = 'churn'       THEN mrr_movement_amount ELSE 0 END) AS churn_mrr
FROM fact_mrr
GROUP BY 1
ORDER BY 1
```

---

### 解約・継続指標

| KPI | 計算式 | ソーステーブル |
|-----|--------|-------------|
| **Revenue Churn Rate**（収益解約率） | `-churn_mrr / beginning_mrr` | `fact_nrr_monthly` |
| **Logo Churn Rate**（顧客解約率） | `churned_customers / beginning_customers` | `fact_nrr_monthly` |
| **GRR**（グロス収益維持率） | `(beginning_mrr + contraction_mrr + churn_mrr) / beginning_mrr` | `fact_nrr_monthly` |
| **NRR**（ネット収益維持率） | `ending_mrr / beginning_mrr` | `fact_nrr_monthly` |
| **コホート顧客継続率** | `active_customers / cohort_customers` | `fact_cohort_mrr` |
| **コホート収益継続率** | `active_mrr / cohort_mrr` | `fact_cohort_mrr` |

```sql
-- 月次Revenue Churn RateとLogo Churn Rate
SELECT
  measurement_month,
  -churn_mrr / NULLIF(beginning_mrr, 0)                      AS revenue_churn_rate,
  churned_customers::DECIMAL / NULLIF(beginning_customers, 0) AS logo_churn_rate,
  grr,
  nrr
FROM fact_nrr_monthly
WHERE segment = 'all'
ORDER BY 1
```

---

### 成長効率指標

| KPI | 計算式 | ソーステーブル |
|-----|--------|-------------|
| **Quick Ratio** | `(New MRR + Expansion MRR) / ABS(Churn MRR + Contraction MRR)` | `fact_mrr` + `fact_nrr_monthly` |
| **LTV**（顧客生涯価値・近似値） | `ARPU / Monthly Churn Rate` | `fact_mrr` + `fact_nrr_monthly` |

```sql
-- 月次Quick Ratio
WITH movements AS (
  SELECT
    snapshot_month,
    SUM(CASE WHEN mrr_movement_type = 'new'       THEN mrr_movement_amount ELSE 0 END) AS new_mrr,
    SUM(CASE WHEN mrr_movement_type = 'expansion' THEN mrr_movement_amount ELSE 0 END) AS expansion_mrr
  FROM fact_mrr
  GROUP BY 1
)
SELECT
  n.measurement_month,
  (m.new_mrr + m.expansion_mrr)
    / NULLIF(ABS(n.churn_mrr + n.contraction_mrr), 0) AS quick_ratio
FROM fact_nrr_monthly n
JOIN movements m ON n.measurement_month = m.snapshot_month
WHERE n.segment = 'all'
```

> **LTVの計算上の注意：** `ARPU / Churn Rate` は解約率が一定と仮定した近似式です。コホート分析（`fact_cohort_mrr`）から実測の平均契約期間を求める方が正確な場合があります。

---

### 営業・回収指標

| KPI | 計算式 | ソーステーブル |
|-----|--------|-------------|
| **Win Rate**（受注率） | `COUNT(*) FILTER (WHERE quote_status = 'accepted') / COUNT(*) FILTER (WHERE quote_status IN ('accepted','rejected'))` | `fact_quote_line` |
| **平均割引率** | `AVG(discount_rate)` WHERE quote_status = 'accepted' | `fact_quote_line` |
| **DSO**（売上債権回転日数） | `SUM(outstanding_amount) / SUM(invoiced_amount) × 日数` | `fact_ar_aging` + `fact_invoice_line` |
| **回収率** | `SUM(paid_amount) / SUM(invoiced_amount)` | `fact_payment` + `fact_invoice_line` |

---

## 算出できないKPI

### コストデータが存在しない

本モデルは収益側のみを設計しています。マーケティング費用・セールス人件費・原価のデータがないため、以下のKPIは算出できません。

| KPI | 算出に必要なデータ |
|-----|----------------|
| **CAC**（顧客獲得コスト） | マーケティング費用 + セールス人件費（月次） |
| **Payback Period** | CAC + 粗利率 |
| **LTV:CAC 比率** | CAC |
| **粗利率 / Gross Margin** | COGS（売上原価）——ホスティング費・サポート人件費など |
| **Rule of 40** | 営業利益率 または EBITDA マージン（原価・費用データが必要） |
| **Magic Number** | 直近四半期のS&Mコスト + Net New ARR |

**必要な追加テーブル例：**

```sql
-- 費用データを追加する場合のイメージ
fact_expense_monthly (
  expense_month     DATE,
  expense_category  VARCHAR,   -- 'sales', 'marketing', 'cogs', 'r&d', 'g&a'
  amount            DECIMAL,
  headcount         INTEGER    -- 人件費は人数も持つと分析しやすい
)
```

---

### プロダクト利用データが存在しない

サービスの使われ方を記録するイベントデータが設計されていないため、以下のKPIは算出できません。

| KPI | 算出に必要なデータ |
|-----|----------------|
| **DAU / MAU**（日次・月次アクティブユーザー） | ログイン・操作イベントログ |
| **機能別採用率**（Feature Adoption Rate） | 機能単位の利用イベントログ |
| **Time to Value** | オンボーディング完了イベント・初回コアアクション到達イベント |
| **ヘルススコア** | 利用頻度・深度・サポートチケット数・NPS等の複合データ |
| **トライアル→有料転換率** | トライアル開始イベント（`fact_mrr` ではMRR=0の顧客収録が任意のため） |

**必要な追加テーブル例：**

```sql
-- プロダクト利用データを追加する場合のイメージ
fact_product_usage_daily (
  usage_date        DATE,
  customer_id       VARCHAR,
  user_id           VARCHAR,
  feature_name      VARCHAR,
  event_count       INTEGER,
  session_minutes   DECIMAL
)
```

---

### アンケート・フィードバックデータが存在しない

| KPI | 算出に必要なデータ |
|-----|----------------|
| **NPS**（ネットプロモータースコア） | アンケート回答データ（0〜10点スコア） |
| **CSAT**（顧客満足度スコア） | サポートチケット・オンボーディング後のアンケートデータ |

---

### 設計が未完のテーブル

| KPI | 不足しているテーブル |
|-----|------------------|
| **繰延収益残高**（Deferred Revenue） | `fact_revenue_schedule`（収益認識スケジュール）——`fact_invoice_line` の注意点として言及されているが未設計 |

---

## まとめ

```
算出できる
  ├── 収益   MRR / ARR / ARPU / ACV / MRRブリッジ
  ├── 解約   Revenue Churn / Logo Churn / GRR / NRR / コホート継続率
  ├── 成長   Quick Ratio / LTV（近似）
  └── 営業   Win Rate / 割引率 / DSO / 回収率

算出できない（不足データ別）
  ├── コスト未整備   CAC / Payback / LTV:CAC / 粗利率 / Rule of 40
  ├── 利用データ未整備  DAU/MAU / 機能採用率 / TTV / ヘルススコア / 転換率
  ├── アンケート未整備  NPS / CSAT
  └── テーブル未設計   繰延収益残高
```

収益・解約・成長の主要指標はこのモデルでカバーできます。コスト効率（CAC・Payback）とプロダクト利用（DAU・ヘルススコア）を分析したい場合は、費用ファクトテーブルとプロダクト利用イベントテーブルの追加が必要です。
