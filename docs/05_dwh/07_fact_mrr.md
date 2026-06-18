# MRR スナップショットファクトテーブル——SaaS の収益変動を月次で捉える

SaaS の収益分析で最も重要な指標は MRR（Monthly Recurring Revenue）です。しかし MRR は単一のテーブルから直接集計できません。請求書（[DWH編 1](01_fact_invoice_line.md)）は「発行した事実」の記録であり、MRR は「今この瞬間に継続している収益の水準」を表すからです。この違いが、MRR 専用のファクトテーブルを必要とする理由です。

---

## MRR とは何か

**MRR（Monthly Recurring Revenue）** は、ある時点において月次で繰り返し発生することが確定している収益の合計です。

```
年間契約 120万円 → MRR = 10万円
月次契約  10万円 → MRR = 10万円
```

MRR はキャッシュの動き（入金）とは独立しています。年間前払いで120万円を受け取っても、MRR は毎月10万円として計上されます。

**ARR（Annual Recurring Revenue）** は MRR × 12 です。期中の値でも単純に12倍することで年換算します。

---

## スナップショット型ファクトテーブルとは

MRR ファクトテーブルは**スナップショット型（Period Snapshot）**です。請求書や入金のような「取引が起きたときに1行追加する」設計ではなく、「ある期間の終わり（月末）に全顧客の状態を記録する」設計です。

| ファクトの種類 | 例 | 粒度 |
|---|---|---|
| トランザクション型 | 請求書発行・入金 | イベントが発生するたびに1行 |
| スナップショット型 | MRR | 月末に全顧客分を1行ずつ |

スナップショット型では、解約した顧客も「MRR = 0」として記録されます。これにより、任意の月の MRR 合計・顧客数が集計できます。

---

## MRR の変動カテゴリ

MRR の前月比変動を5つに分類します。この分類が経営上の意思決定の起点になります。

| カテゴリ | 定義 | 例 |
|---|---|---|
| New | 前月 MRR = 0 → 今月 MRR > 0 | 新規契約 |
| Expansion | 前月 MRR < 今月 MRR（両月 > 0） | アップセル・シート追加 |
| Contraction | 前月 MRR > 今月 MRR（両月 > 0） | ダウングレード・シート削減 |
| Churn | 前月 MRR > 0 → 今月 MRR = 0 | 解約 |
| Reactivation | 前月 MRR = 0 → 今月 MRR > 0 かつ 過去に MRR 実績あり | 再契約 |

---

## テーブル設計

```sql
fact_mrr (
  -- 粒度: 顧客 × 月
  snapshot_month      DATE,          -- 月末日（例：2024-01-31）
  customer_id         VARCHAR,
  subscription_id     VARCHAR,

  -- MRR
  mrr                 DECIMAL,       -- 当月末時点の MRR（0を含む）
  mrr_prior_month     DECIMAL,       -- 前月末時点の MRR

  -- MRR 変動カテゴリ
  mrr_movement_type   VARCHAR,       -- 'new' / 'expansion' / 'contraction' / 'churn' / 'reactivation' / 'flat'
  mrr_movement_amount DECIMAL,       -- 変動額（負値あり）

  -- ディメンションキー
  dim_customer_key    INTEGER,
  dim_date_key        INTEGER,

  -- 参照情報
  plan_name           VARCHAR,
  billing_currency    VARCHAR,
  mrr_usd             DECIMAL        -- 統一通貨での MRR（多通貨対応の場合）
)
```

---

## 主要な集計クエリ

**月次 MRR ブリッジ（滝グラフ用）**

```sql
SELECT
  snapshot_month,
  mrr_movement_type,
  SUM(mrr_movement_amount) AS movement_mrr,
  COUNT(DISTINCT customer_id) AS customer_count
FROM fact_mrr
WHERE snapshot_month BETWEEN '2024-01-31' AND '2024-12-31'
GROUP BY 1, 2
ORDER BY 1, 2
```

**特定月の MRR 合計**

```sql
SELECT
  snapshot_month,
  SUM(mrr) AS total_mrr,
  COUNT(DISTINCT CASE WHEN mrr > 0 THEN customer_id END) AS active_customers
FROM fact_mrr
WHERE snapshot_month = '2024-12-31'
GROUP BY 1
```

---

## MRR の計算方法

MRR は請求書データから計算されることが多いですが、計算方法の選択が重要です。

**方法A：サブスクリプションから計算（推奨）**
サブスクリプションの月額換算値を使います。年間120万円なら MRR = 10万円です。契約が有効な期間、毎月同じ MRR を記録します。

**方法B：実際の請求書から計算**
月次請求なら請求書金額 = MRR ですが、年間前払いの場合は請求月のみに120万円が計上され、他の月は0になります。これは MRR の定義（継続的な収益の水準）を反映しません。

方法Aがサブスクリプションビジネスの実態を正確に反映します。

---

## よくある課題

**無料トライアル・フリープランの扱い**
MRR = 0 の顧客をスナップショットに含めるかどうかを決めておく必要があります。含める場合は「Reactivation」の識別が容易になりますが、テーブルサイズが増えます。

**Pro-rata 請求の月割り**
期中に契約が開始・終了した場合の日割り計算を MRR にどう反映するかの設計が必要です。多くの場合、「月末時点で有効なサブスクリプションの月額換算値」として計算します。

**過去データの再計算**
コミッションや税の遡及修正・契約の遡及修正があった場合、過去のスナップショットを再計算するか・修正を当月に反映するかのポリシーを決めておく必要があります。

---

**次回：** [コホート分析ファクトテーブル——顧客の継続率を時系列で捉える](08_fact_cohort.md)
