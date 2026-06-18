# 商品ディメンションテーブル——価格を持たせてはいけない理由

「プラン別の売上構成はどうか」「従量課金の商品と固定料金の商品で収益の伸び方に差はあるか」——こうした分析には、商品の属性情報が必要です。商品ディメンションは「何を売ったか」の属性を管理します。そして重要な設計原則が1つあります——**価格はここに持たせてはいけない**ということです。

SLG・PLG両方のビジネスモデルで使用します。

---

## 粒度（Grain）

**商品 1件 = 1レコード**

価格は管理しません。価格はファクトテーブルの `unit_price` で追跡します。ディメンションは「何の商品か」という属性を持ちます。

---

## テーブル定義

```sql
dim_product (
  -- サロゲートキー（DWH内部キー）
  product_sk            STRING,   -- DWH内部の一意識別子

  -- ナチュラルキー（元システムのID）
  product_id            STRING,   -- 元システムの商品ID
  stripe_product_id     STRING,   -- StripeのProduct ID（存在する場合）
  erp_product_code      STRING,   -- ERPの商品コード（存在する場合）

  -- 商品属性
  product_name          STRING,   -- 商品名
  product_family        STRING,   -- 商品ファミリー（例：基本プラン, アドオン, 初期費用）
  product_type          STRING,   -- 商品種別（plan, addon, one_time, usage）
  billing_model         STRING,   -- 課金モデル（flat, per_seat, tiered, usage_based）
  billing_frequency     STRING,   -- 請求サイクル（monthly, annual, one_time）
  is_recurring          BOOLEAN,  -- 経常収益（繰り返し課金）かどうか
  is_active             BOOLEAN,  -- 現在販売中かどうか

  -- 分類・タグ
  product_category      STRING,   -- 分析用カテゴリ（例：Core, Analytics, Security）
  plan_tier             STRING,   -- プランのランク（例：Starter, Pro, Enterprise）

  -- SCD管理用
  valid_from            TIMESTAMP, -- このレコードの有効開始日時
  valid_to              TIMESTAMP, -- このレコードの有効終了日時
  is_current            BOOLEAN,   -- 現在有効なレコードかどうか

  -- メタデータ
  source_system         STRING,   -- データ元システム
  loaded_at             TIMESTAMP -- データ取込日時
)
```

---

## 主要属性の定義

### product_type（商品種別）

| 値 | 意味 |
|----|------|
| `plan` | 基本プラン（スタータープラン、プロプランなど） |
| `addon` | 追加オプション（追加ストレージ、追加機能など） |
| `one_time` | 一時費用（初期費用、研修費用など） |
| `usage` | 従量課金商品（APIコール、メール送信数など） |

### billing_model（課金モデル）

| 値 | 意味 | 例 |
|----|------|-----|
| `flat` | 定額 | 月額10万円（数量に関係なく） |
| `per_seat` | シートあたり | 1ユーザー月額1万円 |
| `tiered` | 階段価格 | 〜10名：1万円、11名〜：8千円 |
| `usage_based` | 従量課金 | 1万APIコールあたり100円 |

### is_recurring（経常収益フラグ）

ARRやMRRの計算では、経常収益（繰り返し発生する収益）のみを対象とします。初期費用などの一時費用を誤って含めないために使います。

```sql
-- ARR計算の例（is_recurring = TRUE のみ集計）
SELECT SUM(arr_amount)
FROM fact_invoice_line f
JOIN dim_product p USING (product_id)
WHERE p.is_recurring = TRUE
  AND f.invoice_status != 'voided'
```

---

## ディメンションの更新方式（SCD）

商品の属性変化は顧客ディメンションほど頻繁ではありませんが、重要な変化が起きることがあります。

| 属性 | 変化の例 | 推奨SCDタイプ |
|------|---------|-------------|
| product_name | 商品名の変更・リブランディング | Type 2（履歴保持を推奨） |
| product_family | カテゴリ再編 | Type 1（上書き） |
| is_active | 販売終了 | Type 1（上書き） |
| billing_model | 課金モデルの変更 | Type 2（履歴保持を推奨） |
| plan_tier | プラン体系の再編 | Type 2（履歴保持を推奨） |

商品名や課金モデルが変わった場合に Type 2 を使わないと、「変更前にどの商品が売れていたか」が分析できなくなります。

---

## 上流システム

| システム | 取得するデータ | 取得方法 |
|---------|-------------|---------|
| Salesforce CPQ | Product / Product Family | Salesforce API |
| HubSpot | Product | HubSpot API |
| Stripe | Product / Price | Stripe API |
| Zuora | Product / Rate Plan | Zuora API |

複数システムに商品マスターが分散している場合、どのシステムを正本とするかを決め、他システムのIDは追加カラムとして持ちます（`stripe_product_id` など）。

---

## 代表的な分析クエリ例

**プラン別・課金モデル別の売上構成**
```sql
SELECT
  p.plan_tier,
  p.billing_model,
  COUNT(DISTINCT f.customer_id) AS customer_count,
  SUM(f.total_amount) AS revenue
FROM fact_invoice_line f
JOIN dim_product p USING (product_id)
WHERE p.is_current = TRUE
  AND f.invoice_status != 'voided'
GROUP BY 1, 2
ORDER BY 4 DESC
```

**経常収益 vs 一時費用の内訳**
```sql
SELECT
  p.is_recurring,
  p.product_type,
  SUM(f.total_amount) AS revenue,
  SAFE_DIVIDE(
    SUM(f.total_amount),
    SUM(SUM(f.total_amount)) OVER ()
  ) AS revenue_share
FROM fact_invoice_line f
JOIN dim_product p USING (product_id)
WHERE f.invoice_status != 'voided'
GROUP BY 1, 2
ORDER BY 3 DESC
```

**廃止商品の影響確認（現在も請求が発生していないか）**
```sql
SELECT
  p.product_name,
  p.is_active,
  COUNT(*) AS invoice_line_count,
  SUM(f.total_amount) AS revenue
FROM fact_invoice_line f
JOIN dim_product p USING (product_id)
WHERE p.is_active = FALSE
  AND f.invoice_date_id >= DATE_SUB(CURRENT_DATE(), INTERVAL 3 MONTH)
GROUP BY 1, 2
```

---

## 注意点

**廃止商品のレコードを削除しない**
販売終了した商品でも、過去の請求書・見積がその商品を参照しています。`is_active = FALSE` に更新するだけにして、レコードは残します。

**価格をディメンションに持たない**
商品ディメンションには価格を持たせません。価格は顧客・時期・割引によって変わるため、ファクトテーブルの `unit_price` が正本です。ディメンションに「標準価格」を持たせると、価格改定のたびに履歴管理が必要になり複雑化します。

**商品の粒度とSKUの違い**
物理商品を扱う場合、「商品」と「SKU（色・サイズなどの組み合わせ）」を分けて管理することがあります。B2B SaaSでは通常この区別は不要ですが、ECや製造業のデータを扱う場合は粒度の設計が必要です。

**複数システムの商品マッピング**
CRMの商品IDとStripeの商品IDが1対1で対応しているとは限りません。CRMの1商品がStripeでは複数のPriceに分かれているケースがあります。ファクトテーブルとの結合時にどのIDを使うかを統一しておく必要があります。

---

**次回：** [売掛金エイジングファクトテーブル——未収金をどこまで放置していいのか](06_fact_ar_aging.md)
