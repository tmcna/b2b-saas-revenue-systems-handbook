# NRR ファクトテーブル——収益の自己増殖力を測る

NRR（Net Revenue Retention）は、既存顧客だけで収益が成長しているかを示す指標です。新規顧客を1人も獲得しなくても NRR が 100% を超えていれば、既存顧客からの拡張が解約の損失を上回っています。この指標は SaaS ビジネスの健全性を示す最重要指標のひとつです。

---

## NRR の定義

```
NRR = （期末 MRR of 期初の既存顧客）÷ （期初 MRR）× 100%
```

計算に含まれるもの：
- **Expansion**（拡張）：アップセル・シート追加による MRR 増加
- **Contraction**（縮小）：ダウングレードによる MRR 減少
- **Churn**（解約）：解約による MRR 消失

計算に含まれないもの：
- **New**（新規）：期間中に新たに獲得した顧客の MRR

---

## NRR と GRR の違い

| 指標 | 計算式 | 含む変動 | 上限 |
|---|---|---|---|
| NRR（Net Revenue Retention） | （Expansion + Contraction + Churn を加味した期末 MRR）÷ 期初 MRR | Expansion・Contraction・Churn | 上限なし（>100% も可） |
| GRR（Gross Revenue Retention） | （Contraction + Churn のみ加味した期末 MRR）÷ 期初 MRR | Contraction・Churn のみ | 最大 100% |

GRR は「拡張なしにどれだけ収益が維持されているか」を測ります。NRR - GRR の差が Expansion の貢献度を示します。

**業界のベンチマーク**

| NRR | 評価 |
|---|---|
| 120% 以上 | 優秀（Expansion が大きく Churn を上回っている） |
| 100〜120% | 良好 |
| 85〜100% | 改善が必要 |
| 85% 未満 | 危険水域 |

---

## 設計判断

### なぜ毎回クエリで計算せず事前集計するか

NRR は `fact_mrr` を2ヶ月分 JOIN して計算します。クエリ自体は書けますが、BI ツールからのアドホック分析で毎回実行すると次の問題が起きます。

1. **定義の散在**：「Expansion とは何か」「New 顧客をどう除外するか」のロジックが複数のダッシュボードやレポートにコピーされ、実装が微妙に食い違う
2. **計算コスト**：MRR スナップショット全体に対して self-join が走り、クエリが重くなる

`fact_nrr_monthly` として事前集計することで、定義を一箇所に集め、BI ツールは単純な SELECT で NRR を参照できます。

### セグメント別を同じテーブルで持つ理由

`segment = 'all'`（全社）・`segment = 'enterprise'`・`segment = 'smb'` のように、セグメントを行として持つことで、1つのテーブルで全社/セグメント別の NRR をカバーできます。セグメントごとにテーブルを分けると、新しいセグメントが追加されるたびにテーブルが増えます。

### GRR を NRR と同じテーブルで持つ理由

NRR と GRR はセットで見ることで意味が明確になります（GRR は解約耐性、NRR - GRR の差は Expansion の貢献）。別テーブルにすると JOIN が必要になり、分析が複雑になります。

---

## テーブル設計

NRR は MRR スナップショット（[DWH編 7](07_fact_mrr.md)）から計算できますが、定期的に集計してファクトテーブルに保存することで、BI ツールでの参照が容易になります。

```sql
fact_nrr_monthly (
  -- 粒度: 月次（全社 or セグメント別）
  measurement_month     DATE,        -- 測定月（例：2024-03-31）
  segment               VARCHAR,     -- 'all' / 'enterprise' / 'smb' など

  -- 期初・期末 MRR
  beginning_mrr         DECIMAL,     -- 期初（前月末）の既存顧客 MRR
  ending_mrr            DECIMAL,     -- 期末時点での同じ顧客コホートの MRR

  -- MRR 変動内訳
  expansion_mrr         DECIMAL,     -- 拡張による増加
  contraction_mrr       DECIMAL,     -- 縮小による減少（負値）
  churn_mrr             DECIMAL,     -- 解約による消失（負値）

  -- 指標
  nrr                   DECIMAL,     -- ending_mrr / beginning_mrr
  grr                   DECIMAL,     -- (beginning_mrr + contraction + churn) / beginning_mrr

  -- 顧客数
  beginning_customers   INTEGER,
  churned_customers     INTEGER,
  expanded_customers    INTEGER,
  contracted_customers  INTEGER
)
```

---

## NRR の計算クエリ

```sql
WITH mrr_movements AS (
  SELECT
    snapshot_month,
    SUM(CASE WHEN mrr_movement_type = 'churn'       THEN mrr_movement_amount ELSE 0 END) AS churn_mrr,
    SUM(CASE WHEN mrr_movement_type = 'contraction' THEN mrr_movement_amount ELSE 0 END) AS contraction_mrr,
    SUM(CASE WHEN mrr_movement_type = 'expansion'   THEN mrr_movement_amount ELSE 0 END) AS expansion_mrr,
    SUM(CASE WHEN mrr_movement_type IN ('flat', 'expansion', 'contraction', 'churn')
             THEN mrr_prior_month ELSE 0 END)                                             AS beginning_mrr
  FROM fact_mrr
  GROUP BY 1
)
SELECT
  snapshot_month                                    AS measurement_month,
  beginning_mrr,
  beginning_mrr + expansion_mrr + contraction_mrr + churn_mrr AS ending_mrr,
  expansion_mrr,
  contraction_mrr,
  churn_mrr,
  (beginning_mrr + expansion_mrr + contraction_mrr + churn_mrr)
    / NULLIF(beginning_mrr, 0)                     AS nrr,
  (beginning_mrr + contraction_mrr + churn_mrr)
    / NULLIF(beginning_mrr, 0)                     AS grr
FROM mrr_movements
ORDER BY snapshot_month
```

---

## NRR を分解する

NRR の悪化原因を特定するために、以下の軸で分解します。

**セグメント別**

```
全社 NRR = 90%
  エンタープライズ NRR = 115%
  SMB NRR = 72%
```

SMB セグメントが全社 NRR を引き下げていることが可視化されます。

**チャーン vs コントラクション**

```
GRR = 88%  ← Churn + Contraction の影響
NRR = 95%  ← Expansion で7%分を取り戻している
```

GRR が低い場合はチャーン対策が優先課題です。GRR は高いが NRR が低い場合は Expansion 施策が弱いことを示します。

**コホート別 NRR**

[DWH編 8 のコホート分析](08_fact_cohort.md)における「収益継続率（Revenue Retention）」が、コホート単位の NRR に相当します。全社 NRR が改善しているのに特定コホートの継続率が低い場合、その時期に獲得した顧客セグメントに問題がある可能性があります。

---

## NRR を改善する施策との連携

NRR の分析は、施策の優先順位付けに使います。

| 課題 | NRR での見え方 | 施策 |
|---|---|---|
| チャーンが多い | GRR が低い | オンボーディング改善・ヘルススコアモニタリング |
| 拡張が少ない | NRR - GRR の差が小さい | アップセルプログラム・利用量に連動した課金 |
| 特定セグメントのチャーン | セグメント別 NRR に差 | ICP（理想顧客プロフィール）の見直し |

---

## よくある課題

**NRR の分母の定義**
「期初 MRR」は「前月末時点の全既存顧客の MRR」ですが、新規獲得した顧客を分母に含めるかどうかで数値が変わります。一般的には「前月末時点でアクティブだった顧客のみ」を分母にします。

**月次 NRR と年次 NRR**
月次 NRR を 12 倍しても年次 NRR にはなりません。年次 NRR は「1年前に存在していた顧客コホートが、1年後にどれだけの MRR を持っているか」で計算します。

**Reactivation の扱い**
解約後に再契約した顧客の MRR を NRR の分子に含めるかどうかを決めておく必要があります。含める場合は「New」ではなく「Reactivation」として分類することが重要です。

---

---

**次回：** [収益認識スケジュールファクトテーブル——いつ売上を計上するかを管理する](10_fact_revenue_schedule.md)
