# 入金ファクトテーブル——キャッシュフローと延滞を分析する

請求書を発行しただけでは売上は完結しません。お金が実際に届いて初めて取引は完結します。入金ファクトテーブルは「誰が・いつ・いくら払ったか」を記録し、入金状況の確認・未収金の管理・キャッシュフロー分析を可能にします。

SLG・PLG両方のビジネスモデルで使用します。

---

## 粒度（Grain）

**入金 1件 = 1レコード**

1枚の請求書に対して複数回に分けて入金されることがあります（部分入金）。その場合、入金ごとに独立したレコードになります。

```
請求書 INV-2024-0001（請求額：10万円）
  ├── 入金1：6万円（3月1日）→ 1レコード
  └── 入金2：4万円（3月15日）→ 1レコード
```

---

## 設計判断

### なぜ fact_invoice_line と別テーブルか

請求と入金は別のイベントです。請求書を発行した日と、顧客が実際に振り込んだ日は異なります。この2つを1つのテーブルに結合しようとすると、次の問題が起きます。

- **入金がない場合（未収）**：invoice_line 側に入金情報の NULL が並ぶ
- **部分入金**：1枚の請求書に複数の入金が紐づくため、invoice_line を複製しないと表現できない
- **1入金が複数請求書に充当**：逆方向の多対多が発生する

「請求した事実」と「入金した事実」を分離することで、どちらも独立したレコードとして正確に管理でき、差分（未収金）は結合によって計算できます。

### 部分入金を別レコードにする理由

10万円の請求に対して6万円・4万円に分けて入金された場合、これを1行にまとめると「最終入金日」しか記録できません。入金ごとに1行持つことで、「入金から最終入金完了まで何日かかったか」「部分入金が発生した顧客の傾向」などの分析が可能になります。

### `due_date_id` を持つ理由

支払期日（`due_date_id`）と実際の入金日（`payment_date_id`）の差が遅延日数です。この計算のために支払期日をファクトテーブル側に持ちます。請求書テーブルから JOIN して取得することも可能ですが、分析クエリの単純化のために非正規化して保持します。

---

## テーブル定義

```sql
fact_payment (
  -- 主キー
  payment_id            STRING,   -- 入金の一意識別子

  -- 外部キー（ディメンション）
  customer_id           STRING,   -- 顧客ID → dim_customer
  invoice_id            STRING,   -- 紐づく請求書ID（消込先）
  payment_date_id       DATE,     -- 入金日 → dim_date
  due_date_id           DATE,     -- 支払期日 → dim_date（遅延日数の計算に使用）

  -- 指標
  payment_amount        NUMERIC,  -- 入金額
  applied_amount        NUMERIC,  -- 請求書への充当額（消込額）
  unapplied_amount      NUMERIC,  -- 未充当額（複数請求書への充当がある場合）
  refund_amount         NUMERIC,  -- 返金額（返金が発生した場合）

  -- 属性
  payment_method        STRING,   -- 入金方法（card, bank_transfer, direct_debit）
  currency_code         STRING,   -- 通貨コード（例：JPY, USD）
  payment_status        STRING,   -- ステータス（succeeded, pending, failed, refunded）
  days_to_pay           INTEGER,  -- 支払期日からの経過日数（遅延分析用）
  is_late               BOOLEAN,  -- 支払期日超過かどうか

  -- メタデータ
  source_system         STRING,   -- データ元システム（例：stripe, zuora）
  loaded_at             TIMESTAMP -- データ取込日時
)
```

---

## 指標の定義

| 指標 | 定義 | 注意点 |
|------|------|--------|
| payment_amount | 顧客から受け取った金額の総額 | 手数料差し引き前の金額 |
| applied_amount | 特定の請求書に充当した金額 | 複数請求書への分割充当がある場合に使用します |
| unapplied_amount | まだ充当先が決まっていない金額 | 過払いや前払いの場合に発生します |
| refund_amount | 返金した金額 | 返金がない場合は0 |
| days_to_pay | payment_date − due_date | マイナスなら早払い、プラスなら延滞です |

---

## ディメンション

| ディメンション | 結合キー | 参照先 |
|-------------|---------|--------|
| 顧客 | customer_id | [dim_customer](./04_dim_customer.md) |
| 入金日 | payment_date_id | dim_date |
| 支払期日 | due_date_id | dim_date |

---

## 上流システム

| システム | 取得するデータ | 取得方法 |
|---------|-------------|---------|
| Stripe | Payment Intent / Charge | Stripe API または Stripe Data Pipeline |
| Zuora | Payment / Payment Application | Zuora AQuA API |
| 銀行 | 入金明細 | 銀行API・CSV取込・freee経由 |
| freee | 入金・消込データ | freee API |

---

## 更新方式

**基本：追記（Append）**

入金は発生した時点で確定します。新しい入金が発生するたびにレコードが追加されます。

**例外：ステータス変化を追跡する場合**

カード決済では、決済が成功・失敗・返金とステータスが変化することがあります。

```
決済処理中   → payment_status = 'pending'
決済成功    → payment_status = 'succeeded'
チャージバック → payment_status = 'refunded'
```

ステータス変化を追跡する場合は **Upsert** を使います。

---

## 代表的な分析クエリ例

**月次入金額の集計**
```sql
SELECT
  DATE_TRUNC(payment_date_id, MONTH) AS month,
  SUM(payment_amount) AS total_collected
FROM fact_payment
WHERE payment_status = 'succeeded'
GROUP BY 1
ORDER BY 1
```

**延滞分析：平均支払遅延日数**
```sql
SELECT
  DATE_TRUNC(due_date_id, MONTH) AS due_month,
  AVG(days_to_pay) AS avg_days_to_pay,
  COUNTIF(is_late) AS late_payment_count
FROM fact_payment
WHERE payment_status = 'succeeded'
GROUP BY 1
ORDER BY 1
```

**未収金（請求済みだが未入金）の確認**
```sql
-- fact_invoice_line と fact_payment を結合して未消込の請求書を特定
SELECT
  i.invoice_id,
  i.customer_id,
  i.total_amount AS invoiced_amount,
  COALESCE(SUM(p.applied_amount), 0) AS paid_amount,
  i.total_amount - COALESCE(SUM(p.applied_amount), 0) AS outstanding_amount
FROM fact_invoice_line i
LEFT JOIN fact_payment p USING (invoice_id)
WHERE i.invoice_status = 'open'
  AND i.is_credit_note = FALSE
GROUP BY 1, 2, 3
HAVING outstanding_amount > 0
```

---

## 注意点

**入金方法による取得タイミングの違い**
カード決済はリアルタイムで入金データが取得できますが、銀行振込は入金の検知・消込処理に時差が生じます。データパイプラインの更新頻度を入金方法ごとに設計する必要があります。

**手数料の扱い**
StripeなどのPaymentプロバイダーは、決済手数料を差し引いて入金します（グロスではなくネットで振り込まれます）。`payment_amount` をグロス（決済額）で持つかネット（手数料差し引き後）で持つかを統一しておく必要があります。分析上はグロスを推奨します。

**返金の表現**
返金は `refund_amount` を別カラムで持つ方法と、`payment_amount` がマイナスの別レコードとして追加する方法があります。集計時の一貫性のため、どちらかに統一します。

**fact_invoice_lineとの関係**
入金ファクトと請求書明細ファクトは異なる粒度を持ちます。1件の入金が複数の請求書明細に対応することがあるため、単純に結合すると行数が膨らみます（ファンアウト）。結合する場合は請求書レベルで集約してから結合する設計を推奨します。

---

**次回：** [見積明細ファクトテーブル——受注率と割引傾向を分析する](03_fact_quote_line.md)
