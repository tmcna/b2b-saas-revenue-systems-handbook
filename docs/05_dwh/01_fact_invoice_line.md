# 請求明細ファクトテーブル——SaaS収益分析の基盤

「先月の売上はいくらですか？」「商品別で内訳を見せてください」——こうした質問に答えるために最初に整備すべきテーブルが、請求明細ファクトです。請求書の明細1行を1レコードとして記録することで、売上の集計・商品別収益分析・顧客別請求金額の確認が可能になります。

SLG・PLG両方のビジネスモデルで使用します。

---

## 粒度（Grain）

**請求書明細 1行 = 1レコード**

1枚の請求書に複数の明細がある場合、それぞれが独立したレコードになります。

```
請求書 INV-2024-0001
  ├── 明細1：プロプラン 月額10万円  → 1レコード
  └── 明細2：追加ストレージ 月額5千円 → 1レコード
```

---

## テーブル定義

```sql
fact_invoice_line (
  -- 主キー
  invoice_line_id       STRING,   -- 明細の一意識別子

  -- 外部キー（ディメンション）
  invoice_id            STRING,   -- 請求書ID
  customer_id           STRING,   -- 顧客ID → dim_customer
  product_id            STRING,   -- 商品ID → dim_product
  subscription_id       STRING,   -- サブスクリプションID（存在する場合）
  invoice_date_id       DATE,     -- 請求書発行日 → dim_date
  due_date_id           DATE,     -- 支払期日 → dim_date
  service_start_date_id DATE,     -- サービス提供開始日 → dim_date
  service_end_date_id   DATE,     -- サービス提供終了日 → dim_date

  -- 指標
  unit_price            NUMERIC,  -- 単価（税抜）
  quantity              NUMERIC,  -- 数量
  subtotal_amount       NUMERIC,  -- 小計（単価 × 数量、税抜）
  discount_amount       NUMERIC,  -- 割引額
  taxable_amount        NUMERIC,  -- 課税対象額（subtotal - discount）
  tax_amount            NUMERIC,  -- 消費税額
  total_amount          NUMERIC,  -- 合計（税込）

  -- 属性
  currency_code         STRING,   -- 通貨コード（例：JPY, USD）
  tax_rate              NUMERIC,  -- 消費税率（例：0.10）
  invoice_status        STRING,   -- 請求書ステータス（paid, open, voided）
  is_credit_note        BOOLEAN,  -- クレジットノート（マイナス請求）かどうか

  -- メタデータ
  source_system         STRING,   -- データ元システム（例：stripe, zuora）
  loaded_at             TIMESTAMP -- データ取込日時
)
```

---

## 指標の定義

| 指標 | 定義 | 注意点 |
|------|------|--------|
| unit_price | 1単位あたりの税抜価格 | 割引前の価格 |
| quantity | 数量 | シートライセンスなら人数、従量なら使用量 |
| subtotal_amount | unit_price × quantity | — |
| discount_amount | 割引額 | 値引きがない場合は0 |
| taxable_amount | subtotal_amount − discount_amount | 消費税の計算基礎 |
| tax_amount | taxable_amount × tax_rate | — |
| total_amount | taxable_amount + tax_amount | 顧客が実際に払う金額 |

---

## ディメンション

| ディメンション | 結合キー | 参照先 |
|-------------|---------|--------|
| 顧客 | customer_id | [dim_customer](./04_dim_customer.md) |
| 商品 | product_id | [dim_product](./05_dim_product.md) |
| 発行日 | invoice_date_id | dim_date |
| 支払期日 | due_date_id | dim_date |
| サービス期間開始 | service_start_date_id | dim_date |
| サービス期間終了 | service_end_date_id | dim_date |

---

## 上流システム

| システム | 取得するデータ | 取得方法 |
|---------|-------------|---------|
| Stripe | Invoice / Invoice Line Item | Stripe API または Stripe Data Pipeline |
| Zuora | Invoice / Invoice Item | Zuora AQuA API または Zuora Data Query |
| freee | 請求書 / 請求書明細 | freee API |

---

## 更新方式

**基本：追記（Append）**

請求書明細は発行後に変更されないのが原則です。新しい請求書が発行されるたびにレコードが追加されます。

**例外：ステータスの変化を追跡する場合**

請求書のステータス（open → paid → voided）が変化することがあります。ステータスの最新状態だけ持てばよい場合は **Upsert（更新 or 挿入）**、変化の履歴も残したい場合は **SCD Type 2** を使います。

```
例：ステータス変化
発行時     → invoice_status = 'open'
入金確認後 → invoice_status = 'paid'
誤発行取消 → invoice_status = 'voided'
```

---

## 代表的な分析クエリ例

**月次売上の集計**
```sql
SELECT
  DATE_TRUNC(invoice_date_id, MONTH) AS month,
  SUM(total_amount) AS total_revenue
FROM fact_invoice_line
WHERE invoice_status != 'voided'
  AND is_credit_note = FALSE
GROUP BY 1
ORDER BY 1
```

**商品別売上**
```sql
SELECT
  p.product_name,
  SUM(f.total_amount) AS revenue
FROM fact_invoice_line f
JOIN dim_product p USING (product_id)
WHERE f.invoice_status != 'voided'
GROUP BY 1
ORDER BY 2 DESC
```

---

## 注意点

**クレジットノートの扱い**
クレジットノート（返金・値引き）は、total_amountがマイナスの明細として記録します。売上集計時に `is_credit_note = FALSE` でフィルタするか、クレジットノートを含めてネットの売上を計算するかを、分析の目的に応じて選択します。

**voided（無効化）した請求書の除外**
誤って発行・無効化した請求書を含めると売上が過大計上されます。`invoice_status != 'voided'` を必ずフィルタ条件に含めます。

**収益認識との違い**
このテーブルは「いつ請求書を発行したか」ベースの集計です。「いつ売上として認識するか（収益認識）」は別のファクトテーブル（Fact Revenue Schedule）で管理します。年間前払い契約では、請求日と収益認識日が大きくずれます。

**通貨の統一**
複数通貨を扱う場合、集計時に為替レートで円換算する処理が必要です。換算レートをどの時点のものを使うかで集計結果が変わります。

---

**次回：** [入金ファクトテーブル——キャッシュフローと延滞を分析する](02_fact_payment.md)
