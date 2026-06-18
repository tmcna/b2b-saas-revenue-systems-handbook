# 付録：主要SaaSプロダクト対応表

本書に登場する主なSaaSプロダクトを、担当ドメインごとに整理したものです。各製品の詳細な比較・選定は本書の範囲外です。自社のシステム構成を本書のドメイン設計に照らし合わせるための参照用として使ってください。

---

## オペレーショナルシステム

| ドメイン | 役割 | 代表プロダクト |
|---------|------|-------------|
| 顧客管理（CRM） | 顧客・商談・活動履歴・契約の管理 | Salesforce Sales Cloud, HubSpot CRM |
| 見積・価格管理（CPQ） | 価格表・見積書・承認フロー | Salesforce CPQ（Revenue Cloud）, HubSpot Quotes |
| 請求・サブスクリプション | 請求書発行・サブスクリプション状態管理・入金照合 | Stripe Billing, Zuora, Chargebee, Maxio |
| 税計算 | 消費税・売上税の税率適用・申告補助 | Avalara AvaTax, TaxJar, Stripe Tax |
| 財務・会計（ERP） | 仕訳・総勘定元帳・財務報告・収益認識 | NetSuite, SAP S/4HANA |
| コミッション管理 | 営業報酬の計算・クォータ管理・支払い処理 | Salesforce Commissions, Spiff, CaptivateIQ |

---

## データ基盤

| レイヤー | 役割 | 代表プロダクト |
|---------|------|-------------|
| DWH | 複数システムのデータを集約・蓄積するクエリ基盤 | Snowflake, BigQuery, Redshift |
| データ変換（ELT） | DWH上でのモデリング・メトリクス定義 | dbt |
| パイプライン管理 | データ取得・ロードのスケジューリングと監視 | Airflow, Prefect |
| BIツール | 可視化・ダッシュボード・アドホック分析 | Looker, Tableau, Metabase |

---

## ドメインとプロダクトの関係

各プロダクトが「どのドメインの正本を持つか」を整理すると、システム間の連携設計の出発点になります。

```
CRM（Salesforce / HubSpot）
  └─ 顧客・商談・契約の正本

CPQ（Salesforce CPQ）
  └─ 価格表・見積の正本

Billing（Stripe / Zuora / Chargebee）
  └─ サブスクリプション・請求書・入金の正本

ERP（NetSuite / SAP）
  └─ 総勘定元帳・財務会計の正本

DWH（Snowflake / BigQuery）
  └─ 派生指標（MRR・NRR・コホート）の計算基盤
     ※ オペレーショナルシステムの正本にはならない
```

---

## 連携パターン別の代表構成

| フェーズ | 典型的な構成 |
|---------|------------|
| スタートアップ初期 | HubSpot CRM + Stripe Billing |
| スタートアップ SLG | Salesforce + Salesforce CPQ + Stripe / Chargebee |
| エンタープライズ SLG | Salesforce + Salesforce CPQ + Zuora + NetSuite |
| PLG セルフサーブ | Stripe Billing（CRM は補助的）|
| 収益分析基盤 | 各 Billing API → Snowflake → dbt → Looker |
