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

この仕組みは会計基準（企業会計基準第29号・IFRS 15）が要求するものですが、業務上も重要です。**繰延収益残高は「まだ提供していないサービスの前受け分」**であり、将来の解約リスクと直結しています。

---

## なぜ専用のファクトテーブルが必要か

`fact_invoice_line` は「請求書を発行した事実」を記録します。1月1日に年間120万円の請求書を発行すると、1月に1レコードが生まれます。このテーブルから「2月の売上」を計算することはできません。

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

## 主要な集計クエリ

**月次認識売上（MRRとの比較用）**

```sql
SELECT
  recognition_month,
  SUM(scheduled_amount)                                  AS recognized_revenue,
  COUNT(DISTINCT customer_id)                            AS recognizing_customers
FROM fact_revenue_schedule
WHERE recognition_status IN ('recognized', 'scheduled')
GROUP BY 1
ORDER BY 1
```

**月末時点の繰延収益残高**

```sql
SELECT
  recognition_month,
  SUM(deferred_balance)                                  AS deferred_revenue_balance,
  COUNT(DISTINCT customer_id)                            AS customers_with_deferred
FROM fact_revenue_schedule
WHERE recognition_month = '2024-12-31'
  AND recognition_status IN ('recognized', 'scheduled')
GROUP BY 1
```

**繰延収益の推移（ウォーターフォール分析用）**

```sql
SELECT
  recognition_month,
  SUM(CASE WHEN recognition_status = 'recognized' THEN scheduled_amount  ELSE 0 END) AS recognized,
  SUM(CASE WHEN recognition_status = 'reversed'   THEN scheduled_amount  ELSE 0 END) AS reversed,
  SUM(deferred_balance)                                                                AS ending_balance
FROM fact_revenue_schedule
WHERE recognition_month BETWEEN '2024-01-31' AND '2024-12-31'
GROUP BY 1
ORDER BY 1
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

  fact_invoice_line  → 1月：120万円、2月以降：0円（請求ベース）
  fact_mrr           → 毎月：10万円（契約継続ベース）
  fact_revenue_schedule → 毎月：10万円（収益認識ベース）
```

MRR と認識売上は通常一致しますが、Pro-rata 調整・返金・早期解約があると乖離します。

---

## 総勘定元帳との連携

収益認識スケジュールの月次処理は、ERPで仕訳を生成します。

```
毎月末に認識処理が実行されると：

  借方：繰延収益（負債）   10万円
  貸方：売上収益（収益）   10万円
```

この仕訳が`fact_revenue_schedule`の認識行に対応します。ERPの総勘定元帳（[第2章参照](../02_domains/10_general_ledger.md)）の残高と`deferred_balance`が一致していることが、月次クローズの確認ポイントになります。

---

## よくある課題

**即時解約時のスケジュール取り消し**
解約が確定した時点で、将来の認識予定行を `reversed` に更新し、残余期間分の繰延収益を取り崩す仕訳が必要です。`deferred_balance` がマイナスにならないよう、ERPとの整合性を確認します。

**Pro-rata 変更後のスケジュール再計算**
期中のアップグレード・ダウングレードでは、差分の認識スケジュールが追加・修正されます。元の請求明細を修正するのか、差分の新明細を追加するのかで、スケジュールの構造が変わります。

**複数通貨の換算タイミング**
外貨建て契約では、スケジュール生成時の換算レートを固定するか毎月再換算するかで、認識売上の金額が変わります。会計方針と合わせて決定します。

---

**次：** [複雑さとの向き合い方——知識を実践に変える](../06_conclusion/00_how_to_cope.md)
