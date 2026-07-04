---
title: "Snowflake 個人検証アカウントをゼロから構築する — 2026年版初期設定まとめ"
emoji: "❄️"
type: "tech"
topics: ["snowflake", "cortex", "dataengineering", "setup", "security"]
published: false
---

## この記事について

Snowflake の個人検証環境を立ち上げたときの自分用備忘録です。アカウント作成の選択肢から、コスト管理・Cortex AI 機能の有効化まで、最初に設定しておいた内容をまとめています。

![](/images/i145-snowflake-personal-account-initial-setup/cover.png)

## アカウント作成

### まずサインアップ

Snowflake のトライアルアカウントはこちらから作成できます。

→ **[Snowflake AI Data Cloud Trial に申し込む](https://signup.snowflake.com/)**

Snowflake には 2 種類のトライアルがあります。

| 種類 | クレジット | 期間 | 特徴 |
|------|----------|------|------|
| **AI Data Cloud Trial** | $400 | 30日 | 全機能利用可。Cortex / CoCo 対応 |
| **Cortex Code CLI Trial** | $40 | 30日 | CoCo CLI の試用に特化した軽量トライアル |

Cortex Analyst・Cortex Agent・Snowflake Notebooks など CoCo 以外の AI 機能も一緒に試したい場合は $400 クレジットの AI Data Cloud Trial が最適です。

AI Data Cloud Trial で作成後、**クレジットカードを登録するとできることが増えます**。Cortex Inference などの AI 機能制限が解除され、トライアル終了後もそのまま継続利用できる状態になります。

:::message
このアカウントは AI Data Cloud Trial で作成し、クレジットカード登録後に Cortex 機能を有効化した環境です。
:::

### 作成時の選択（推奨: Oregon / Standard）

アカウント作成時には **クラウドプロバイダー・リージョン・エディション** の 3 つを選択します。

| 項目 | 推奨 | 理由 |
|------|------|------|
| クラウド | AWS | 機能カバレッジが最も広い |
| リージョン | **US West (Oregon) / us-west-2** | 新機能が早い・単価が安い（後述） |
| エディション | **Standard** | Cortex / CoCo を含む必要機能をカバー |

### Oregon を選ぶ理由

**① 新機能が早く使える**

Snowflake の新機能は Oregon など一部リージョンから先行展開されます。最新機能をいち早く試したい検証用途では有利です。

**② クレジット単価が安い**

以下は **Platform Credit**（ウェアハウス利用時に消費するクレジット）の On Demand 単価比較です（2026年7月4日現在 / 出典: Snowflake Credit Consumption Table）。

| リージョン | Standard | Enterprise | Business Critical |
|----------|---------|------------|------------------|
| **AWS US West (Oregon)（推奨）** | **$2.00** | $3.00 | $4.00 |
| AWS US East (N. Virginia) | $2.00 | $3.00 | $4.00 |
| AWS AP Northeast 1 (Tokyo) | $2.85 | $4.30 | $5.70 |
| AWS AP Singapore | $2.50 | $3.70 | $5.00 |
| AWS EU Dublin | $2.60 | $3.90 | $5.20 |

「とりあえず Enterprise で作るか」と Tokyo で始めると $4.30/クレジット。Oregon の Standard（$2.00）と比べると **2 倍以上**の単価です。

:::message
**Platform Credit と AI Credit は別物です**

後述する `CORTEX_ENABLED_CROSS_REGION` 設定で変わる AI Credit 単価（$2.20/$2.00）は、ここで説明した Platform Credit とは**別の課金体系**です。ウェアハウスのコンピュート消費が Platform Credit、Cortex LLM などの AI 機能呼び出しが AI Credit です。
:::

### Standard を選ぶ理由

各エディションの主な機能差異は以下の通りです。

| 機能 | Standard | Enterprise | Business Critical |
|------|---------|------------|------------------|
| Time Travel（最大） | **1日** | 90日 | 90日 |
| マルチクラスタ WH | × | ○ | ○ |
| マテリアライズドビュー | × | ○ | ○ |
| Dynamic Data Masking | × | ○ | ○ |
| Row Access Policy | ○ | ○ | ○ |
| Tri-Secret Secure（顧客管理 KMS） | × | × | ○ |
| Cortex / AI 機能 | ○ | ○ | ○ |
| Snowflake CLI / CoCo | ○ | ○ | ○ |

詳細は [Snowflake 公式ドキュメント（Editions）](https://docs.snowflake.com/en/user-guide/intro-editions) を確認してください。

:::message
個人検証では Standard で自分は不足したことがありません。Enterprise 以上への移行は実際のユースケースに応じて検討すれば十分です。
:::

## アカウント作成直後に用意されているもの

アカウントを新規作成すると、Snowflake が自動的に以下を用意します。

**ロール（6種）**

| ロール | 用途 |
|--------|------|
| ACCOUNTADMIN | アカウント全体の管理（最高権限） |
| ORGADMIN | 組織・アカウント管理 |
| SYSADMIN | DB・ウェアハウス作成・管理 |
| SECURITYADMIN | ユーザー・ロール・ポリシー管理 |
| USERADMIN | ユーザー・ロール作成 |
| PUBLIC | 全ユーザーに自動付与 |

**ウェアハウス（2つ）**

| ウェアハウス | 用途 |
|------------|------|
| `COMPUTE_WH` | 汎用クエリ実行用（X-Small） |
| `SYSTEM$STREAMLIT_NOTEBOOK_WH` | Streamlit / Notebook 専用（自動管理） |

**データベース（1つ）**

- `SNOWFLAKE_SAMPLE_DATA`: Snowflake が提供するサンプルデータ（共有から自動マウント）

## Step 1 — アカウントパラメーター

アカウント全体に適用するパラメーターを設定します。

| パラメーター | 設定値 | 設計根拠 |
|------------|--------|--------|
| `TIMEZONE` | `Asia/Tokyo` | QUERY_HISTORY・ログを日本時間で確認するため |
| `STATEMENT_TIMEOUT_IN_SECONDS` | `3600` | 無制限クエリによるクレジット超過を防ぐ |

:::details SQL
```sql
USE ROLE ACCOUNTADMIN;

ALTER ACCOUNT SET TIMEZONE = 'Asia/Tokyo';
ALTER ACCOUNT SET STATEMENT_TIMEOUT_IN_SECONDS = 3600;
```
:::

## Step 2 — ウェアハウス

デフォルトの `COMPUTE_WH` は Gen2 で作成されます。個人検証では Gen1 で十分なためコストを抑えた設定に変更し、あわせて Auto Suspend を設定してアイドル時のクレジット消費を防ぎます。

| 設定 | 値 | 理由 |
|------|------|------|
| `GENERATION` | `1`（Gen1） | Gen2 → Gen1 でクレジット消費を削減 |
| `WAREHOUSE_SIZE` | `X-SMALL` | 個人検証では最小サイズで十分 |
| `AUTO_SUSPEND` | `60` 秒 | アイドル 1 分で自動停止 |
| `AUTO_RESUME` | `TRUE` | クエリ発行時に自動起動 |

:::details SQL
```sql
USE ROLE SYSADMIN;

ALTER WAREHOUSE COMPUTE_WH SET
    GENERATION   = 1
    AUTO_SUSPEND = 60
    AUTO_RESUME  = TRUE;

-- SYSADMIN がデフォルト WH を使えるように権限付与
USE ROLE ACCOUNTADMIN;
GRANT USAGE ON WAREHOUSE COMPUTE_WH TO ROLE SYSADMIN;
```
:::

:::message
**`SYSADMIN` を使う理由**

ウェアハウスの設定変更は `SYSADMIN` ロールで行うのが Snowflake の権限設計の原則です。`ACCOUNTADMIN` は「本当に必要なとき以外は使わない」のが推奨です。
:::

## Step 3 — Cortex AI 機能

Cortex AI 機能（`CORTEX_COMPLETE`・Cortex Analyst・CoCo 等）の AI Credit を Global 価格で使うために `CORTEX_ENABLED_CROSS_REGION = 'ANY_REGION'` を設定します。

| 設定値 | AI Credit 単価（On Demand） |
|--------|--------------------------|
| Regional（デフォルト） | $2.20 |
| **Global（`ANY_REGION` 設定時）** | **$2.00** |

`ANY_REGION` を設定することで AI Credit が Regional（$2.20）から Global（$2.00）に切り替わり、約 9% コストを抑えられます（2026年7月4日現在 / 出典: Snowflake Credit Consumption Table）。

| 設定値 | 意味 |
|--------|------|
| `DISABLED`（デフォルト） | Regional 価格（$2.20） |
| `'AWS_US_WEST_2'` など | 特定リージョン経由を許可 |
| `'ANY_REGION'` | すべてのリージョンを許可（Global 価格 $2.00） |

:::details SQL
```sql
USE ROLE ACCOUNTADMIN;

ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION = 'ANY_REGION';
```
:::

:::message alert
**データ越境の考慮**

`ANY_REGION` を設定するとクエリの一部が他リージョンで処理される場合があります。個人検証用途では問題ありませんが、本番環境ではデータガバナンスポリシーに照らして判断してください。
:::

## Step 4 — コスト管理

意図しないクレジット超過を防ぐために Budget を設定します。あわせて Cost Anomaly Detection で通常パターンから外れた支出増を検知できます。

### Budget

月次の支出上限（Spending Limit）を設定します。上限に近づくと通知が届きます。

| 設定 | 値 | 理由 |
|------|------|------|
| 月額上限 | $30 | 個人検証の目安 |
| 通知メール | 自分のアドレス | 超過前に気づくため |

:::details SQL
```sql
USE ROLE ACCOUNTADMIN;

CALL SNOWFLAKE.LOCAL.ACCOUNT_ROOT_BUDGET!ACTIVATE();
CALL SNOWFLAKE.LOCAL.ACCOUNT_ROOT_BUDGET!SET_SPENDING_LIMIT(30);
CALL SNOWFLAKE.LOCAL.ACCOUNT_ROOT_BUDGET!SET_EMAIL_NOTIFICATIONS('your-email@example.com');

-- 設定確認
CALL SNOWFLAKE.LOCAL.ACCOUNT_ROOT_BUDGET!GET_CONFIG();
```
:::

### Cost Anomaly Detection

Snowflake コンソールの **Admin > Cost Management** から有効化できます。支出が通常パターンから大きく外れた場合に自動アラートが届きます。SQL 設定不要で UI から利用可能です。

:::message
**Budget vs Resource Monitor**

| 機能 | 粒度 | 用途 |
|------|------|------|
| Budget | アカウント全体・オブジェクト単位 | 月次の支出上限管理 |
| Resource Monitor | ウェアハウス単位 | クレジット消費量の監視・停止 |

個人検証では Budget + Cost Anomaly Detection で十分です。
:::

## Step 5 — 管理データベース

セキュリティポリシーオブジェクト（マスキングポリシー等）を格納するための専用 DB・スキーマを作成します。ポリシーオブジェクトも通常のテーブルと同様にスキーマに属するため、管理の起点となる DB を分けておくと後々整理しやすくなります。

| オブジェクト | 名前 | 備考 |
|------------|------|------|
| Database | `DB_GOVERNANCE` | ガバナンス・ポリシー管理専用 |
| Schema | `DB_GOVERNANCE.SCM_SECURITY` | セキュリティポリシー格納 |

:::details SQL
```sql
USE ROLE SYSADMIN;

CREATE DATABASE IF NOT EXISTS DB_GOVERNANCE;
CREATE SCHEMA    IF NOT EXISTS DB_GOVERNANCE.SCM_SECURITY;

-- ポリシー管理は SECURITYADMIN が担う
USE ROLE ACCOUNTADMIN;
GRANT USAGE     ON DATABASE DB_GOVERNANCE              TO ROLE SECURITYADMIN;
GRANT OWNERSHIP ON SCHEMA   DB_GOVERNANCE.SCM_SECURITY TO ROLE SECURITYADMIN;
```
:::

## 設定まとめ

| # | 設定項目 | コマンド / 操作 | 理由 |
|---|---------|----------------|------|
| 1 | タイムゾーン設定 | `ALTER ACCOUNT SET TIMEZONE = 'Asia/Tokyo'` | QUERY_HISTORY・ログを日本時間で確認するため |
| 2 | ステートメントタイムアウト | `ALTER ACCOUNT SET STATEMENT_TIMEOUT_IN_SECONDS = 3600` | 無制限クエリによるクレジット超過を防ぐ |
| 3 | ウェアハウス Gen1・Auto Suspend | `ALTER WAREHOUSE COMPUTE_WH SET GENERATION = 1 AUTO_SUSPEND = 60` | Gen2→Gen1 + アイドル停止でクレジット削減 |
| 4 | CROSS_REGION 有効化 | `ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION = 'ANY_REGION'` | AI Credit を Global 価格（$2.00）で使うため |
| 5 | Budget 設定 | `ACCOUNT_ROOT_BUDGET!ACTIVATE()` + `SET_SPENDING_LIMIT(30)` | 月次の支出上限を $30 に設定 |
| 6 | 管理 DB 作成 | `CREATE DATABASE DB_GOVERNANCE` + スキーマ + 権限移譲 | ポリシーオブジェクトの管理起点 |
