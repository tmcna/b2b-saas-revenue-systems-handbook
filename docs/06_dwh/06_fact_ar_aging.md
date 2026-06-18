# 売掛金エイジングファクトテーブル——未収金をどこまで放置していいのか

「先月末時点で、30日以上未払いの顧客は何社あるか」「90日超の未収金に対して貸倒引当金をいくら積むべきか」——こうした問いに答えるのが、売掛金エイジングファクトテーブルです。

このテーブルは、これまで紹介した請求明細ファクト・入金ファクト・見積明細ファクトとは異なる種類のファクトテーブルです。Kimball の分類では「**周期スナップショットファクト（Periodic Snapshot Fact）**」と呼ばれます。

---

## 3種類のファクトテーブル

ここで、Kimball が定義するファクトテーブルの3種類を整理します。

| 種類 | 何を記録するか | DWH編の該当テーブル |
|------|-------------|-----------------|
| **取引ファクト**（Transaction Fact） | 個々の業務イベント | fact_invoice_line, fact_payment, fact_quote_line |
| **周期スナップショットファクト**（Periodic Snapshot Fact） | ある時点の状態 | **fact_ar_aging**（このテーブル） |
| **累積スナップショットファクト**（Accumulating Snapshot Fact） | プロセス全体の進捗 | 受注から入金完了までの1行（本シリーズでは扱いません） |

取引ファクトは「何が起きたか」を記録します。周期スナップショットファクトは「その時点でどういう状態か」を定期的に記録します。エイジングリストは後者の典型例です。

---

## 粒度（Grain）

**顧客 × スナップショット日（月末）= 1レコード**

月末時点の顧客ごとの未収金を、経過日数別のバケットに分けて1行に収めます。

```
2024年1月31日 時点
  顧客A → current: 100万円、1-30日: 0円、31-60日: 0円、61-90日: 0円、91日超: 0円
  顧客B → current: 0円、1-30日: 50万円、31-60日: 20万円、61-90日: 0円、91日超: 0円
  顧客C → current: 0円、1-30日: 0円、31-60日: 0円、61-90日: 0円、91日超: 30万円
```

---

## テーブル定義

```sql
fact_ar_aging (
  -- 主キー
  ar_aging_id           STRING,   -- スナップショットの一意識別子

  -- 外部キー（ディメンション）
  customer_id           STRING,   -- 顧客ID → dim_customer
  snapshot_date_id      DATE,     -- スナップショット日（月末）→ dim_date

  -- 指標：支払期日からの経過日数別の未収金額
  current_amount        NUMERIC,  -- 期日未到来（まだ支払期限内）
  days_1_30_amount      NUMERIC,  -- 支払期日から1〜30日超過
  days_31_60_amount     NUMERIC,  -- 31〜60日超過
  days_61_90_amount     NUMERIC,  -- 61〜90日超過
  days_91_plus_amount   NUMERIC,  -- 91日以上超過（回収リスクが高い）
  total_outstanding     NUMERIC,  -- 合計未収金額

  -- 属性
  open_invoice_count    INTEGER,  -- 未収の請求書件数
  oldest_due_date       DATE,     -- 最も古い未払い請求書の支払期日

  -- メタデータ
  source_system         STRING,
  loaded_at             TIMESTAMP
)
```

---

## 指標の定義

| 指標 | 定義 | 注意点 |
|------|------|--------|
| current_amount | スナップショット日時点で期日未到来の合計 | 支払いが遅れているわけではありません |
| days_1_30_amount | 支払期日を1〜30日超過した請求書の合計 | 支払い忘れの可能性が高いです |
| days_31_60_amount | 31〜60日超過分の合計 | 督促が必要な状態です |
| days_61_90_amount | 61〜90日超過分の合計 | エスカレーションを検討します |
| days_91_plus_amount | 91日以上超過分の合計 | 貸倒引当金の計算対象になりやすいです |
| total_outstanding | 全バケットの合計 | 顧客の現在の売掛金残高です |

---

## 上流：fact_invoice_line と fact_payment から生成する

このテーブルは単独のソースシステムから直接取得するのではなく、**請求書ファクトと入金ファクトを結合して計算**します。

```sql
-- 月末時点（:snapshot_date）の売掛金エイジングを計算する例
WITH outstanding AS (
  SELECT
    i.invoice_id,
    i.customer_id,
    i.due_date_id,
    i.total_amount - COALESCE(SUM(p.applied_amount), 0) AS outstanding_amount
  FROM fact_invoice_line i
  LEFT JOIN fact_payment p
    ON i.invoice_id = p.invoice_id
    AND p.payment_status = 'succeeded'
  WHERE i.invoice_status = 'open'
    AND i.is_credit_note = FALSE
  GROUP BY 1, 2, 3, 4
  HAVING outstanding_amount > 0
)
SELECT
  customer_id,
  DATE ':snapshot_date'                                                           AS snapshot_date_id,
  SUM(CASE WHEN due_date_id >= ':snapshot_date'                            THEN outstanding_amount ELSE 0 END) AS current_amount,
  SUM(CASE WHEN DATE_DIFF(':snapshot_date', due_date_id, DAY) BETWEEN  1 AND 30  THEN outstanding_amount ELSE 0 END) AS days_1_30_amount,
  SUM(CASE WHEN DATE_DIFF(':snapshot_date', due_date_id, DAY) BETWEEN 31 AND 60  THEN outstanding_amount ELSE 0 END) AS days_31_60_amount,
  SUM(CASE WHEN DATE_DIFF(':snapshot_date', due_date_id, DAY) BETWEEN 61 AND 90  THEN outstanding_amount ELSE 0 END) AS days_61_90_amount,
  SUM(CASE WHEN DATE_DIFF(':snapshot_date', due_date_id, DAY) > 90               THEN outstanding_amount ELSE 0 END) AS days_91_plus_amount,
  SUM(outstanding_amount)                                                         AS total_outstanding,
  COUNT(DISTINCT invoice_id)                                                      AS open_invoice_count,
  MIN(due_date_id)                                                                AS oldest_due_date
FROM outstanding
GROUP BY 1, 2
```

---

## 更新方式

**周期スナップショット（Periodic Snapshot）**

取引ファクトのように「何かが起きたとき」に追記するのではなく、**定期的（通常は月末）に全顧客分のスナップショットを追記**します。

```
1月31日 → 全顧客分のスナップショット行を追加
2月28日 → 全顧客分のスナップショット行を追加
3月31日 → 全顧客分のスナップショット行を追加
```

未収残高がゼロの顧客もレコードとして含めることを推奨します。「ある月の未収ゼロ顧客数」の把握や、前月との比較が容易になります。

---

## 代表的な分析クエリ例

**月末時点のエイジングサマリー（全顧客集計）**
```sql
SELECT
  snapshot_date_id,
  SUM(current_amount)      AS current,
  SUM(days_1_30_amount)    AS days_1_30,
  SUM(days_31_60_amount)   AS days_31_60,
  SUM(days_61_90_amount)   AS days_61_90,
  SUM(days_91_plus_amount) AS days_91_plus,
  SUM(total_outstanding)   AS total_ar
FROM fact_ar_aging
GROUP BY 1
ORDER BY 1
```

**90日超の未収金が多い顧客（直近月末）**
```sql
SELECT
  c.company_name,
  f.days_91_plus_amount,
  f.total_outstanding,
  f.oldest_due_date
FROM fact_ar_aging f
JOIN dim_customer c USING (customer_id)
WHERE f.snapshot_date_id = (SELECT MAX(snapshot_date_id) FROM fact_ar_aging)
  AND f.days_91_plus_amount > 0
ORDER BY f.days_91_plus_amount DESC
```

**特定顧客のエイジング推移（時系列）**
```sql
SELECT
  snapshot_date_id,
  current_amount,
  days_1_30_amount,
  days_31_60_amount,
  days_61_90_amount,
  days_91_plus_amount,
  total_outstanding
FROM fact_ar_aging
WHERE customer_id = 'CUST-001'
ORDER BY snapshot_date_id
```

---

## 注意点

**ERPのエイジングレポートとの照合**
ERP でも同様のエイジングレポートを出力できます。DWH版と計算結果が一致しているかを確認することが重要です。ERP が独自の消込ルールを持つ場合（部分消込・前払い充当・クレジットノートの扱いなど）、計算結果がずれることがあります。差異が出た場合はERP側の計算ロジックを正本とし、DWH側の計算ロジックを調整します。

**貸倒引当金への接続**
91日超の未収金は、会計上「貸倒引当金」の計上対象になることがあります。どの閾値で引当を積むか・引当率をいくつにするかは会計方針によって異なります。DWH の集計値を引当計上に使う場合は、CFO・会計担当者・監査人との確認が必要です。

**銀行振込の消込タイミング**
銀行振込の場合、入金の検知から消込完了まで数日かかることがあります。月末ぎりぎりに入金されても消込が翌月になると、エイジングレポートに見かけ上の未収が残ります。「消込日」と「入金日」のどちらを基準にするかを統一しておく必要があります。

**ゼロ残高顧客の扱い**
全額入金済みで未収残高がゼロの顧客もレコードとして含めることを推奨します。「ある月に未収がゼロだった顧客数」の追跡や、月次での回収率の計算が容易になります。

---

**次回：** [MRR スナップショットファクトテーブル——SaaS の収益変動を月次で捉える](07_fact_mrr.md)
