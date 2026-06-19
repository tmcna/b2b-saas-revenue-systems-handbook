# 顧客ディメンションテーブル——緩やかに変化する属性をどう管理するか

「業種別の売上はどうか」「エンタープライズ顧客のチャーン率は」——ファクトテーブルだけでは答えられない質問です。顧客の業種・規模・ランクなどの属性情報を付与するのが、顧客ディメンションの役割です。そして顧客属性は時間とともに変化します——これをどう扱うかが、このテーブル設計の核心です。

SLG・PLG両方のビジネスモデルで使用します。

---

## 粒度（Grain）

**顧客（アカウント）1社 = 1レコード**（SCD Type 2 の場合は、変化のたびに新レコードが追加されます）

コンタクト（担当者）は管理しません。会社単位での分析を対象とします。

---

## 設計判断

### なぜサロゲートキー（customer_sk）を使うか

CRM・Stripe・ERP でそれぞれ別のIDが顧客に割り振られています。また CRM の顧客IDが統廃合やシステム移行によって変わることもあります。DWH 内部でサロゲートキー（`customer_sk`）を発行し、ファクトテーブルはこれで結合することで、元システムのIDが変わっても DWH 内の参照関係を壊さずに済みます。

`customer_id`・`stripe_customer_id`・`erp_customer_id` は「元システムでの照合」に使うためのナチュラルキーとして保持します。

### なぜ SCD Type 2 か

顧客の業種・規模・ランクは変化します。スタートアップが成長してエンタープライズになることも、M&A で業種が変わることもあります。単純に `UPDATE` してしまうと、「6ヶ月前の時点でスタートアップだった顧客の解約率」を遡及分析できなくなります。

SCD Type 2 では、属性が変化するたびに古いレコードの `valid_to` を埋め、新しいレコードを追加します。これにより、ファクトテーブルの`invoice_date`時点での顧客属性を正確に結合できます。

```
変更前: customer_sk=001, tier='SMB',        valid_from='2024-01-01', valid_to='2024-06-30'
変更後: customer_sk=002, tier='Enterprise',  valid_from='2024-07-01', valid_to=NULL
```

### コンタクト（担当者）を管理しない理由

B2B SaaS の収益分析は「会社単位」で行います。担当者の異動・退職があっても、会社との契約は継続します。コンタクトを取り込むと粒度が分析目的と合わなくなり、売上の重複集計が発生します。担当者情報が必要な分析（例：AE 別の受注率）は CRM 側で行うべきです。

---

## テーブル定義

```sql
dim_customer (
  -- サロゲートキー（DWH内部キー）
  customer_sk           STRING,   -- DWH内部の一意識別子（UUIDなど）

  -- ナチュラルキー（元システムのID）
  customer_id           STRING,   -- 元システムの顧客ID（CRMのAccount ID）
  stripe_customer_id    STRING,   -- StripeのCustomer ID（存在する場合）
  erp_customer_id       STRING,   -- ERPの顧客コード（存在する場合）

  -- 顧客属性
  customer_name         STRING,   -- 会社名
  customer_name_kana    STRING,   -- 会社名（カナ）
  industry              STRING,   -- 業種
  employee_count_range  STRING,   -- 従業員数規模（例：1-50, 51-200, 201-1000, 1001+）
  annual_revenue_range  STRING,   -- 年商規模
  country_code          STRING,   -- 国コード（例：JP, US）
  prefecture            STRING,   -- 都道府県（日本の場合）
  customer_tier         STRING,   -- 顧客ランク（例：SMB, Mid-Market, Enterprise）
  customer_segment      STRING,   -- セグメント（例：スタートアップ, 製造業, 金融）

  -- 取引属性
  first_contract_date   DATE,     -- 初回契約日
  latest_contract_date  DATE,     -- 直近契約日
  churned_date          DATE,     -- 解約日（解約済みの場合）
  customer_status       STRING,   -- ステータス（active, churned, prospect）
  current_plan          STRING,   -- 現在契約中のプラン名
  current_arr           NUMERIC,  -- 現在のARR（年次経常収益）

  -- SCD管理用
  valid_from            TIMESTAMP, -- このレコードの有効開始日時
  valid_to              TIMESTAMP, -- このレコードの有効終了日時（現在有効なレコードはNULL）
  is_current            BOOLEAN,   -- 現在有効なレコードかどうか

  -- メタデータ
  source_system         STRING,   -- データ元システム（例：salesforce, hubspot）
  loaded_at             TIMESTAMP -- データ取込日時
)
```

---

## 主要属性の定義

| 属性 | 定義 | 値の例 |
|------|------|--------|
| customer_tier | 顧客規模による分類 | SMB / Mid-Market / Enterprise |
| employee_count_range | 従業員数の範囲 | 1-50 / 51-200 / 201-1000 / 1001+ |
| customer_status | 現在の取引状況 | active / churned / prospect |
| current_arr | 現在有効なサブスクリプションの年次換算金額合計 | 1,200,000（円） |

---

## ディメンションの更新方式（SCD）

顧客属性は時間とともに変化します。この変化をどう扱うかを「緩やかに変化するディメンション（SCD：Slowly Changing Dimension）」として設計します。

### 変化の種類と推奨するSCDタイプ

| 属性 | 変化の例 | 推奨SCDタイプ |
|------|---------|-------------|
| customer_name | 社名変更 | Type 1（上書き）または Type 2（履歴保持） |
| customer_tier | SMB → Enterprise への成長 | Type 2（履歴保持を推奨） |
| customer_status | active → churned | Type 2（履歴保持を推奨） |
| current_plan | プランアップグレード | Type 1（上書き）※詳細はfactで追跡 |
| industry | 変更はまれ | Type 1（上書き） |

**Type 1（上書き）：** 過去の値を捨て、最新の値に更新します。シンプルですが「以前はどうだったか」がわからなくなります。

**Type 2（履歴保持）：** 変更前のレコードを残し、新しいレコードを追加します。`valid_from`・`valid_to`・`is_current` で管理します。分析の再現性が高いです。

---

## SCD Type 2 の実装アプローチ

SCD Type 2 の前提として、「変更前の値を何らかの方法で捕捉する」必要があります。ここに実装上の難しさがあります。CRM は基本的に CRUD で動いており、顧客の `customer_tier` が変更されると `UPDATE` で上書きされます。DWH が次のバッチで取得するときには、すでに変更前の値は消えています。

この問題への対処アプローチは主に3つあります。

### ① CDC（Change Data Capture）

DBのトランザクションログ（PostgreSQL の WAL など）を読み取り、UPDATE 前後の値をストリームとして取得します。変更が失われる前に捕捉するため、最も精度が高い方法です。

- ツール例：Debezium（OSS）、Fivetran・Airbyte（CDC モード）
- DWH 側で CDC イベントを受け取り、SCD Type 2 に変換する
- 実装・運用コストは高いが、時刻単位の精度が必要な場合はこれが現実的な選択肢

### ② dbt snapshot（定期ポーリング）

CRM の現在の全件データを定期取得し、前回取得分と比較して差分を検出します。dbt の `snapshot` 機能がこの処理をそのまま担います。

```sql
{% snapshot dim_customer_snapshot %}
  {{
    config(
      unique_key='id',
      strategy='check',
      check_cols=['customer_tier', 'industry', 'employee_count_range']
    )
  }}
  select * from {{ source('crm', 'customers') }}
{% endsnapshot %}
```

1日1回実行する場合、**日中に発生した変更は取りこぼします**。ただし、B2B SaaS の顧客属性変化（ティア変更・業種変更）は通常ゆっくり起きるため、日次精度で十分なケースがほとんどです。実装がシンプルで、中小規模では最もよく使われる方法です。

### ③ フルスナップショット（日次保存）

ソーステーブルを毎日まるごと `snapshot_date` 付きでパーティション保存します。SCD Type 2 への変換は取得時ではなくクエリ時に行います。実装は最も単純ですが、ストレージコストがかかります。BigQuery のような列指向 DWH では現実的なコストで運用できます。

### 選択の目安

| 精度の要件 | 推奨アプローチ |
|---|---|
| 時刻単位で変更を正確に追いたい | CDC |
| 日次精度で十分 | dbt snapshot |
| 手軽に始めたい・小規模 | フルスナップショット |

実務では「dbt snapshot で始めて、精度が問題になったら CDC に移行する」という順序が現実的です。

---

## 上流システム

| システム | 取得するデータ | 取得方法 |
|---------|-------------|---------|
| Salesforce / HubSpot | Account | CRM API / Bulk API |
| Stripe | Customer | Stripe API |
| freee / NetSuite | 取引先 | ERP API |

複数システムに同じ顧客が存在する場合、名寄せ（同一顧客の統合）処理が必要です。`customer_id`（CRM）を主キーとして、他システムのIDは `stripe_customer_id`・`erp_customer_id` として管理します。

---

## 代表的な分析クエリ例

**業種別のARR分布**
```sql
SELECT
  c.industry,
  COUNT(DISTINCT c.customer_id) AS customer_count,
  SUM(c.current_arr) AS total_arr
FROM dim_customer c
WHERE c.is_current = TRUE
  AND c.customer_status = 'active'
GROUP BY 1
ORDER BY 3 DESC
```

**チャーン顧客の属性分析**
```sql
SELECT
  c.customer_tier,
  c.industry,
  COUNT(*) AS churned_count,
  AVG(DATE_DIFF(c.churned_date, c.first_contract_date, MONTH)) AS avg_lifetime_months
FROM dim_customer c
WHERE c.is_current = TRUE
  AND c.customer_status = 'churned'
GROUP BY 1, 2
ORDER BY 3 DESC
```

---

## 注意点

**名寄せの難しさ**
複数のシステムに同一顧客が別々に登録されていることがあります（CRMとStripeで別々に作成されたなど）。名寄せのロジックを設計し、`customer_sk`（サロゲートキー）で統一した識別子を付与することが重要です。

**`current_arr` の鮮度**
`current_arr` はサブスクリプションの状態に依存します。毎日更新されるべき値ですが、更新頻度が低いと実態と乖離します。高精度のARR分析が必要な場合は、ファクトテーブルから動的に計算する方が正確です。

**SCD Type 2 の複雑さ**
Type 2を採用すると、同一顧客に複数レコードが存在します。ファクトテーブルとの結合時に `is_current = TRUE` または `valid_from`・`valid_to` を使った期間指定が必要です。これを忘れると行数が意図せず増えます（ファンアウト）。

**グループ企業の扱い**
親会社・子会社の関係を持つ顧客グループでは、グループ全体のARR集計のために `parent_customer_id` のような属性が必要になることがあります。

---

**次回：** [商品ディメンションテーブル——価格を持たせてはいけない理由](05_dim_product.md)
