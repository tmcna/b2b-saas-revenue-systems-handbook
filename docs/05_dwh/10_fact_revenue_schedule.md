# 収益認識スケジュールファクトテーブル——いつ売上を計上するかを管理する

収益認識は本書の核心テーマのひとつです。[序章](../00_introduction/03_revenue_recognition.md)で説明したように、年間前払い契約で120万円を受け取っても、売上は毎月10万円ずつ計上されます。この「受け取ったお金」と「認識した売上」のギャップを管理するのが、収益認識スケジュールファクトテーブルです。

---

## 繰延収益とは何か

年間前払い120万円の契約を締結した場合、1月1日に120万円を請求・入金しても、**売上として認識できるのは毎月10万円**です。

```
1月1日：入金 120万円
          ↓
繰延収益（負債）120万円 として計上

1月末：売上認識 10万円
          繰延収益 110万円（残高）

2月末：売上認識 10万円
          繰延収益 100万円（残高）

  ……（毎月繰り返す）……

12月末：売上認識 10万円
          繰延収益 0円（完全消化）
```

この仕組みは会計基準（企業会計基準第29号・IFRS 15）が要求するものです。月次の仕訳生成や貸借対照表の繰延収益残高の管理は **Billing システムや ERP が担います**。DWHがこのデータを持つ意味は、他のドメインのデータと横断して分析するためです。

---

## なぜ専用のファクトテーブルが必要か

`fact_invoice_line` は「請求書を発行した事実」を記録します。1月1日に年間120万円の請求書を発行すると、1月に1レコードが生まれます。このテーブルから「2月の認識売上」を計算することはできません。

収益認識スケジュールは、**1枚の請求書明細から複数の認識行を生成します**。年間12ヶ月の契約なら、1行の請求明細に対して12行の認識スケジュールが作られます。

```
fact_invoice_line の1行
  INV-2024-0001 / 明細1 / 120万円 / 1月1日請求

  ↓ 1:N で展開

fact_revenue_schedule の12行
  INV-2024-0001 / 明細1 / 2024-01-31 / 10万円
  INV-2024-0001 / 明細1 / 2024-02-29 / 10万円
  INV-2024-0001 / 明細1 / 2024-03-31 / 10万円
  …（12ヶ月分）
```

---

## 設計判断

### なぜ月末日（month-end）を粒度にするか

収益認識の時間軸は月次です。日次にするとデータ量が30倍になり、経営の意思決定に必要な粒度を超えます。`fact_mrr` と同じ月次粒度にすることで、MRRと認識売上を同じ時間軸で比較できます。

### `cumulative_recognized` と `deferred_balance` を事前計算する理由

繰延収益残高は「請求総額 − 累計認識額」ですが、月次残高を毎回ウィンドウ関数で集計すると、スキャン範囲が過去全体にわたりクエリが重くなります。事前計算して持つことで、残高の参照が単純な `WHERE recognition_month = ...` で完結します。

### `recognition_status` を持つ理由

スケジュール通りに進まないケースがあります。

- **reversed（取り消し）**：契約解約で将来の認識行が消える
- **adjusted（調整）**：Pro-rata 変更・返金でスケジュールが修正される

ステータスを持つことで、「将来の認識予定額」と「過去の認識確定額」を区別できます。

---

## テーブル設計

```sql
fact_revenue_schedule (
  -- 粒度: 請求明細 × 認識月
  invoice_line_id       VARCHAR,     -- FK → fact_invoice_line
  recognition_month     DATE,        -- 月末日（例：2024-01-31）

  -- 請求の参照情報
  invoice_id            VARCHAR,
  customer_id           VARCHAR,
  subscription_id       VARCHAR,

  -- 認識金額
  scheduled_amount      DECIMAL,     -- この月に認識する金額
  cumulative_recognized DECIMAL,     -- 累計認識額（この月を含む）
  total_invoice_amount  DECIMAL,     -- 請求明細の合計額（全月共通）
  deferred_balance      DECIMAL,     -- 繰延収益残高 = total - cumulative_recognized

  -- ステータス
  recognition_status    VARCHAR,     -- 'scheduled' / 'recognized' / 'reversed' / 'adjusted'

  -- サービス期間（fact_invoice_lineから引き継ぐ）
  service_start_date    DATE,
  service_end_date      DATE,

  -- ディメンションキー
  dim_customer_key      INTEGER,
  dim_date_key          INTEGER,

  -- 参照情報
  billing_currency      VARCHAR,
  amount_usd            DECIMAL      -- 多通貨対応の場合
)
```

---

## 上流システム：どこからデータを取るか

`fact_revenue_schedule` のデータソースは、他のファクトテーブルと異なり一通りではありません。会社のシステム構成によって2つのパターンがあり、どちらを選ぶかは重要な設計判断です。

### パターンA：DWH 内で計算する

`fact_invoice_line` にはすでに `service_start_date`（サービス開始日）と `service_end_date`（サービス終了日）があります。この2つの日付を使えば、DWH 層（dbt 等）で認識スケジュールを計算できます。ERP に収益認識モジュールがない場合や、Stripe Revenue Recognition 等の収益認識機能を契約・利用していない場合はこのアプローチが現実的です。

```sql
-- dbt モデルのイメージ（Snowflake 方言）
WITH invoice_lines AS (
  SELECT
    invoice_line_id,
    invoice_id,
    customer_id,
    subscription_id,
    subtotal_amount                                                AS total_invoice_amount,
    service_start_date,
    service_end_date,
    DATEDIFF('month', service_start_date, service_end_date) + 1   AS service_months
  FROM {{ ref('fact_invoice_line') }}
  WHERE invoice_status  != 'voided'
    AND is_credit_note   = FALSE
    AND service_start_date IS NOT NULL
    AND service_end_date   IS NOT NULL
),
monthly_schedule AS (
  SELECT
    il.invoice_line_id,
    il.invoice_id,
    il.customer_id,
    il.subscription_id,
    il.total_invoice_amount,
    il.service_months,
    LAST_DAY(DATEADD('month', n.offset, il.service_start_date))   AS recognition_month,
    il.total_invoice_amount / il.service_months                   AS scheduled_amount
  FROM invoice_lines il
  -- offset = 0, 1, 2 … service_months-1 の連番を生成
  CROSS JOIN (
    SELECT ROW_NUMBER() OVER (ORDER BY NULL) - 1 AS offset
    FROM TABLE(GENERATOR(ROWCOUNT => 120))        -- 最大10年分
  ) n
  WHERE n.offset < il.service_months
)
SELECT
  invoice_line_id,
  recognition_month,
  invoice_id,
  customer_id,
  subscription_id,
  scheduled_amount,
  SUM(scheduled_amount) OVER (
    PARTITION BY invoice_line_id
    ORDER BY recognition_month
    ROWS UNBOUNDED PRECEDING
  )                                                               AS cumulative_recognized,
  total_invoice_amount,
  total_invoice_amount - SUM(scheduled_amount) OVER (
    PARTITION BY invoice_line_id
    ORDER BY recognition_month
    ROWS UNBOUNDED PRECEDING
  )                                                               AS deferred_balance,
  'scheduled'                                                     AS recognition_status
FROM monthly_schedule
```

### パターンB：ERP または Billing システムから取り込む

Zuora Revenue・NetSuite Revenue Recognition・Salesforce Revenue Cloud 等、収益認識モジュールを持つシステムが正本の場合は、そちらから DWH へ取り込みます。

| システム | 収益認識モジュール | 取得方法 |
|---------|-----------------|---------|
| Zuora Revenue | あり（Zuora Revenue） | Zuora Data Query / API |
| NetSuite | あり（Revenue Recognition） | NetSuite SuiteAnalytics / API |
| Salesforce | あり（Revenue Cloud / Salesforce Billing） | Salesforce Bulk API |
| Stripe | あり（Stripe Revenue Recognition） | Stripe Revenue Recognition API / パターンA も可 |
| Chargebee | 限定的（月次のみ） | パターンA推奨 |

### どちらを選ぶか

| 状況 | 推奨パターン |
|------|------------|
| Stripe Revenue Recognition を使わず、ERP に収益認識モジュールなし | **A**（DWH で計算） |
| Stripe Revenue Recognition を使用 | **B**（Stripe から取込） |
| Zuora Revenue・NetSuite 等が正本 | **B**（ERP/Billing から取込） |
| GAAP/IFRS 準拠の監査対応が必要 | **B**（ERP の認識結果を正本とする） |

パターンAとBを混在させると DWH と ERP の数字がずれます。いずれかに統一するか、パターンAを採用した上で ERP との照合クエリを定期実行する運用が必要です。

---

## DWH に置く理由：横断分析のユースケース

月次の仕訳生成や残高確認は Billing・ERP ツールが担います。DWH でこのテーブルを持つ意味は、他のドメインのデータと組み合わせて初めて答えられる問いのためです。

---

### ユースケース1：受注・請求・売上の三者照合

[序章](../00_introduction/04_why_numbers_diverge.md)で示した「なぜ数字がずれるのか」への直接の答えがここにあります。

```
1月に年間120万円の契約を締結・請求した場合：

  受注（Bookings）= 120万円  ← CRM / fact_quote_line
  請求（Billings）= 120万円  ← Billing / fact_invoice_line
  売上（Revenue）=  10万円  ← ERP→DWH / fact_revenue_schedule
```

3つの数字はそれぞれ異なるシステムに存在するため、DWH 上で結合しない限り一画面で比較できません。CFO がボードに報告する際の「なぜ受注と売上が違うのか」への説明責任が、この照合クエリで果たせます。

```sql
-- 月次 受注 / 請求 / 売上 の三者照合
WITH bookings AS (
  SELECT
    DATE_TRUNC('month', close_date_id)  AS month,
    SUM(arr_amount / 12)                AS monthly_equivalent
  FROM fact_quote_line
  WHERE quote_status = 'accepted'
    AND is_primary_quote = TRUE
  GROUP BY 1
),
billings AS (
  SELECT
    DATE_TRUNC('month', invoice_date_id) AS month,
    SUM(subtotal_amount)                 AS billed_amount
  FROM fact_invoice_line
  WHERE invoice_status != 'voided'
    AND is_credit_note = FALSE
  GROUP BY 1
),
revenue AS (
  SELECT
    DATE_TRUNC('month', recognition_month) AS month,
    SUM(scheduled_amount)                  AS recognized_amount
  FROM fact_revenue_schedule
  WHERE recognition_status IN ('recognized', 'scheduled')
  GROUP BY 1
)
SELECT
  COALESCE(bk.month, bi.month, rv.month)   AS month,
  COALESCE(bk.monthly_equivalent, 0)        AS bookings,
  COALESCE(bi.billed_amount, 0)             AS billings,
  COALESCE(rv.recognized_amount, 0)         AS revenue
FROM bookings bk
FULL OUTER JOIN billings bi USING (month)
FULL OUTER JOIN revenue  rv USING (month)
ORDER BY 1
```

---

### ユースケース2：解約が収益認識に与えた影響の定量化

`fact_mrr` はチャーン MRR（サブスクリプションの損失）を追跡しますが、年間前払い契約で早期解約が発生した場合、残余期間の繰延収益がどれだけ取り消されたかは MRR だけでは見えません。`fact_revenue_schedule` の `reversed` 行と `fact_mrr` のチャーン行を結合することで、解約の財務的影響を正確に測定できます。

```sql
-- 解約によるチャーンMRR と 繰延収益取り消し額の比較
WITH churn AS (
  SELECT
    snapshot_month,
    customer_id,
    ABS(mrr_movement_amount) AS churn_mrr
  FROM fact_mrr
  WHERE mrr_movement_type = 'churn'
),
reversals AS (
  SELECT
    recognition_month,
    customer_id,
    SUM(ABS(scheduled_amount)) AS reversed_revenue
  FROM fact_revenue_schedule
  WHERE recognition_status = 'reversed'
  GROUP BY 1, 2
)
SELECT
  c.snapshot_month,
  SUM(c.churn_mrr)         AS churn_mrr,
  SUM(r.reversed_revenue)  AS reversed_deferred_revenue
FROM churn c
LEFT JOIN reversals r
  ON  c.customer_id       = r.customer_id
  AND c.snapshot_month    = r.recognition_month
GROUP BY 1
ORDER BY 1
```

チャーン MRR（月次損失）と繰延収益取り消し額（一括損失）が異なる場合、早期解約・Pro-rata 返金の影響を可視化できます。

---

### ユースケース3：顧客セグメント別の認識売上分析

ERP の売上勘定はセグメント情報を持たないことが多く、「エンタープライズ顧客の認識売上は SMB より安定しているか」という問いに ERP 単体では答えられません。`dim_customer` と結合することで、セグメント別・プラン別の認識売上を分析できます。

```sql
-- 顧客セグメント別の月次認識売上
SELECT
  rs.recognition_month,
  dc.customer_tier,
  SUM(rs.scheduled_amount)   AS recognized_revenue,
  COUNT(DISTINCT rs.customer_id) AS customer_count
FROM fact_revenue_schedule rs
JOIN dim_customer dc
  ON rs.dim_customer_key = dc.customer_key
WHERE rs.recognition_status IN ('recognized', 'scheduled')
GROUP BY 1, 2
ORDER BY 1, 2
```

---

## 請求明細・MRRとの関係

3つのテーブルはそれぞれ異なる問いに答えます。

| 問い | テーブル |
|------|---------|
| いつ請求書を発行したか | `fact_invoice_line` |
| 今継続している収益の水準はいくらか | `fact_mrr` |
| **いつ売上として計上したか** | **`fact_revenue_schedule`** |

年間前払い契約では、3つの数字がすべて異なります。これが「同じ顧客の数字なのになぜ合わないのか」の構造的な原因です（[序章参照](../00_introduction/04_why_numbers_diverge.md)）。

```
1月1日に年間120万円の契約を締結した場合：

  fact_invoice_line      → 1月：120万円、2月以降：0円（請求ベース）
  fact_mrr               → 毎月：10万円（契約継続ベース）
  fact_revenue_schedule  → 毎月：10万円（収益認識ベース）
```

MRR と認識売上は通常一致しますが、Pro-rata 調整・返金・早期解約があると乖離します。この乖離がユースケース2で検出できます。

---

## よくある課題

**即時解約時のスケジュール取り消し**
解約が確定した時点で、将来の認識予定行を `reversed` に更新する処理が必要です。この更新が遅れると `deferred_balance` が実態より膨らみ続け、ERP の繰延収益残高と乖離します。

**Pro-rata 変更後のスケジュール再計算**
期中のアップグレード・ダウングレードでは、差分の認識スケジュールが追加・修正されます。元の請求明細を修正するのか、差分の新明細を追加するのかで、スケジュールの構造が変わります。

**複数通貨の換算タイミング**
外貨建て契約では、スケジュール生成時の換算レートを固定するか毎月再換算するかで、認識売上の金額が変わります。会計方針と合わせて決定します。

---

**次：** [複雑さとの向き合い方——知識を実践に変える](../06_conclusion/00_how_to_cope.md)
