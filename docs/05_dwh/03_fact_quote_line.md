# 見積明細ファクトテーブル——受注率と割引傾向を分析する

「どの商品が一番提案されているか」「割引率が高い案件ほど受注しやすいのか」——こうした営業データの分析には、見積明細ファクトが必要です。見積の明細1行を1レコードとして記録することで、受注率の分析・割引の傾向把握・商品別の提案頻度が可能になります。

SLG（営業主導）モデルで使用します。PLGのセルフサービスでは見積が発生しないため、このテーブルは対象外です。

---

## 粒度（Grain）

**見積明細 1行 = 1レコード**

1枚の見積に複数の明細がある場合、それぞれが独立したレコードになります。

```
見積 Q-2024-0001
  ├── 明細1：プロプラン 年額100万円  → 1レコード
  └── 明細2：導入支援サービス 30万円 → 1レコード
```

---

## テーブル定義

```sql
fact_quote_line (
  -- 主キー
  quote_line_id         STRING,   -- 明細の一意識別子

  -- 外部キー（ディメンション）
  quote_id              STRING,   -- 見積ID
  opportunity_id        STRING,   -- 商談ID
  customer_id           STRING,   -- 顧客ID → dim_customer
  product_id            STRING,   -- 商品ID → dim_product
  quote_date_id         DATE,     -- 見積作成日 → dim_date
  expiry_date_id        DATE,     -- 見積有効期限 → dim_date
  close_date_id         DATE,     -- 受注・失注日 → dim_date

  -- 指標
  list_price            NUMERIC,  -- 定価（価格表の単価）
  unit_price            NUMERIC,  -- 見積単価（割引適用後）
  quantity              NUMERIC,  -- 数量
  discount_rate         NUMERIC,  -- 割引率（例：0.10 = 10%割引）
  discount_amount       NUMERIC,  -- 割引額
  subtotal_amount       NUMERIC,  -- 小計（unit_price × quantity）
  total_amount          NUMERIC,  -- 合計金額
  term_months           INTEGER,  -- 契約期間（月数）
  arr_amount            NUMERIC,  -- 年次換算金額（ARR換算）

  -- 属性
  quote_status          STRING,   -- 見積ステータス（draft, presented, accepted, rejected, expired）
  quote_version         INTEGER,  -- 見積バージョン番号
  is_primary_quote      BOOLEAN,  -- 商談に複数見積がある場合の主見積フラグ
  currency_code         STRING,   -- 通貨コード

  -- メタデータ
  source_system         STRING,   -- データ元システム（例：salesforce, hubspot）
  loaded_at             TIMESTAMP -- データ取込日時
)
```

---

## 指標の定義

| 指標 | 定義 | 注意点 |
|------|------|--------|
| list_price | 価格表上の定価 | 割引前の基準価格 |
| unit_price | 実際に見積に記載した単価 | 割引適用後 |
| discount_rate | (list_price − unit_price) / list_price | 何%引いたかを計算できます |
| discount_amount | (list_price − unit_price) × quantity | 値引き総額 |
| subtotal_amount | unit_price × quantity | — |
| term_months | 契約期間を月数で表現 | 年契約なら12、月契約なら1 |
| arr_amount | subtotal_amount / term_months × 12 | 異なる契約期間の見積を比較するための標準化指標 |

---

## ディメンション

| ディメンション | 結合キー | 参照先 |
|-------------|---------|--------|
| 顧客 | customer_id | [dim_customer](./04_dim_customer.md) |
| 商品 | product_id | [dim_product](./05_dim_product.md) |
| 見積作成日 | quote_date_id | dim_date |
| 有効期限 | expiry_date_id | dim_date |
| クローズ日 | close_date_id | dim_date |

---

## 上流システム

| システム | 取得するデータ | 取得方法 |
|---------|-------------|---------|
| Salesforce CPQ | Quote / Quote Line Item | Salesforce API / Bulk API |
| HubSpot | Deal / Line Item | HubSpot API |
| スプレッドシート | 見積台帳 | CSV取込 |

---

## 更新方式

**Upsert（更新 or 挿入）**

見積は作成後にステータスが変化します（draft → presented → accepted / rejected）。最新のステータスを常に保持するためにUpsertを使います。

**バージョン管理**

同一の商談に対して見積が複数バージョン作られることがあります。

```
Q-2024-0001 v1（draft）
Q-2024-0001 v2（presented）← 顧客に提示したバージョン
Q-2024-0001 v3（accepted）← 受注したバージョン
```

分析時には `is_primary_quote = TRUE` かつ `quote_status = 'accepted'` でフィルタして受注見積だけを集計するケースが多いです。

---

## 代表的な分析クエリ例

**商品別の見積提案金額（ARR換算）**
```sql
SELECT
  p.product_name,
  COUNT(*) AS quote_line_count,
  SUM(arr_amount) AS total_arr_proposed
FROM fact_quote_line f
JOIN dim_product p USING (product_id)
WHERE is_primary_quote = TRUE
GROUP BY 1
ORDER BY 3 DESC
```

**割引率の分布（割引の傾向分析）**
```sql
SELECT
  CASE
    WHEN discount_rate = 0         THEN '割引なし'
    WHEN discount_rate < 0.10      THEN '10%未満'
    WHEN discount_rate < 0.20      THEN '10〜20%'
    WHEN discount_rate < 0.30      THEN '20〜30%'
    ELSE '30%以上'
  END AS discount_bucket,
  COUNT(*) AS quote_line_count,
  AVG(arr_amount) AS avg_arr
FROM fact_quote_line
WHERE is_primary_quote = TRUE
  AND quote_status IN ('accepted', 'rejected')
GROUP BY 1
ORDER BY 2 DESC
```

**受注率（Win Rate）の計算**
```sql
SELECT
  DATE_TRUNC(close_date_id, MONTH) AS close_month,
  COUNTIF(quote_status = 'accepted') AS won,
  COUNTIF(quote_status IN ('accepted', 'rejected')) AS total_closed,
  SAFE_DIVIDE(
    COUNTIF(quote_status = 'accepted'),
    COUNTIF(quote_status IN ('accepted', 'rejected'))
  ) AS win_rate
FROM fact_quote_line
WHERE is_primary_quote = TRUE
GROUP BY 1
ORDER BY 1
```

---

## 注意点

**バージョンの重複カウント**
1つの商談に複数バージョンの見積が存在する場合、すべてのバージョンを集計すると金額が重複します。`is_primary_quote = TRUE` または最新バージョンのみを分析対象にするルールを徹底します。

**受注した見積と請求の紐づけ**
受注した見積（`quote_status = 'accepted'`）が、実際に発行された請求書と金額が一致しているかを確認することで、見積から請求へのデータ引き継ぎの品質を検証できます。

**ARR換算の注意**
`arr_amount` は分析上の標準化指標です。一時費用（セットアップ費用・研修費用など）はARR換算すると実態と乖離します。商品の種類でフィルタして計算することを推奨します。

**見積が存在しない受注**
セルフサービスや小規模取引では、見積を作らずに受注するケースがあります。このデータはこのファクトテーブルには存在しません。全体の受注額を把握するには、`fact_invoice_line` を正本とする方が正確です。

---

**次回：** [顧客ディメンションテーブル——緩やかに変化する属性をどう管理するか](04_dim_customer.md)
