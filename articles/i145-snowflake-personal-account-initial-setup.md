---
title: "Snowflake 個人検証アカウントをゼロから構築する — 2026年版初期設定まとめ"
emoji: "❄️"
type: "tech"
topics: ["snowflake", "cortex", "dataengineering", "setup", "security"]
published: false
---

## この記事について

Snowflake の個人検証用アカウントを新しく作ったとき、「最初に何を設定しておくべきか」をまとめた記事です。

実際にこのアカウントで行った初期設定を `SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY` から洗い出し、再現できる形の SQL・手順書として整理しました。Cortex / CoCo を使いたい方向けの設定も含みます。

:::message
**対象環境**
- Snowflake Standard Edition（AWS us-west-2 / Oregon リージョン 推奨）
- アカウント作成日: 2026-06-06
- 本記事の設定は個人検証用途を想定しています
:::

:::message alert
**免責事項**
本記事の内容は筆者個人の環境での設定例であり、本番環境への適用を推奨するものではありません。ご自身の環境・ポリシーに合わせてご判断ください。
:::

![](/images/i145-snowflake-personal-account-initial-setup/cover.png)

## アカウント作成時の選択（推奨: AWS Oregon / Standard）

Snowflake の新規アカウント作成時には **クラウドプロバイダー・リージョン・エディション** の 3 つを選択します。個人検証用途では以下の組み合わせがお勧めです。

| 項目 | 推奨 | 理由 |
|------|------|------|
| クラウド | AWS | Snowflake との親和性が高く、機能カバレッジが最も広い |
| リージョン | **US West (Oregon) / us-west-2** | 新機能の先行提供・価格が安い（後述） |
| エディション | **Standard** | 個人検証に必要な機能はすべてカバー、価格が最も安い |

### Oregon を選ぶ理由

**① 新機能が最初に使える**

Snowflake の新機能は `AWS US West (Oregon)` に最初にデプロイされます。Cortex LLM（`CORTEX_COMPLETE` 等）・Cortex Analyst・CoCo（Cortex Code）など、AI 系の機能は Oregon に先行展開されることが多く、他リージョンへの展開は数日〜数週間遅れることがあります。最新機能をいち早く試したい検証用途では Oregon が有利です。

**② クレジット単価が安い**

Snowflake のクレジット単価はリージョンによって異なります。Oregon は AWS リージョンの中でも最も安い水準で、東京（ap-northeast-1）と比較すると同じエディションでも単価が低くなります。個人の検証用途でコストを抑えたい場合は Oregon が最適です。

**③ CORTEX_ENABLED_CROSS_REGION の設定が不要**

東京リージョンでは Cortex 系機能のモデルがホストされていないため、後述の `CORTEX_ENABLED_CROSS_REGION = 'ANY_REGION'` の設定が必要です。Oregon ならこの設定なしで Cortex 機能をそのまま使えます。

### Standard を選ぶ理由

Enterprise・Business Critical との比較で、個人検証用途で Standard では不足するケースはほとんどありません。

| 機能 | Standard | Enterprise |
|------|---------|------------|
| Time Travel | 1日 | 90日 |
| マルチクラスタ WH | × | ○ |
| Dynamic Data Masking | ○ | ○ |
| Cortex / AI 機能 | ○ | ○ |
| Snowflake CLI / CoCo | ○ | ○ |

「1日の Time Travel で十分・クラスタをスケールアウトする規模ではない」という個人検証の前提であれば Standard で問題ありません。

## アカウント作成直後に用意されているもの

アカウントを新規作成すると、Snowflake が自動的に以下を用意します。

**ロール（7種）**

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
| `COMPUTE_WH` | 汎用クエリ実行用（XS） |
| `SYSTEM$STREAMLIT_NOTEBOOK_WH` | Streamlit / Notebook 専用（自動管理） |

**データベース（1つ）**

- `SNOWFLAKE_SAMPLE_DATA`: Snowflake が提供するサンプルデータ（共有から自動マウント）

## Step 1 — タイムゾーンとタイムアウトの設定

最初にアカウントレベルのタイムゾーンを設定します。デフォルトは UTC のため、日本で使う場合は `Asia/Tokyo` に変更します。

```sql
USE ROLE ACCOUNTADMIN;

ALTER ACCOUNT SET TIMEZONE = 'Asia/Tokyo';
```

あわせてステートメントのタイムアウトも設定します。デフォルト値（`0` = 無制限）のままだと長時間クエリがクレジットを消費し続けるリスクがあるため、検証用途では 1 時間（3,600 秒）を上限としました。

```sql
ALTER ACCOUNT SET STATEMENT_TIMEOUT_IN_SECONDS = 3600;
```

## Step 2 — ウェアハウスの Auto Suspend 設定

デフォルトの `COMPUTE_WH` は作成直後は Auto Suspend が長めに設定されています。検証環境では短めに設定してクレジットの無駄遣いを防ぎます。

```sql
USE ROLE SYSADMIN;

ALTER WAREHOUSE COMPUTE_WH SET
    AUTO_SUSPEND = 60        -- 60秒間クエリがなければ自動停止
    AUTO_RESUME  = TRUE;     -- クエリ発行時に自動起動
```

:::message
**`SYSADMIN` を使う理由**

ウェアハウスの設定変更は `SYSADMIN` ロールで行うのが Snowflake の権限設計の原則です。`ACCOUNTADMIN` は「本当に必要なとき以外は使わない」のが推奨です。
:::

また、`SYSADMIN` がデフォルトウェアハウスを使えるように権限を付与します。

```sql
USE ROLE ACCOUNTADMIN;

GRANT USAGE ON WAREHOUSE COMPUTE_WH TO ROLE SYSADMIN;
```

## Step 3 — Cortex / AI 機能の有効化（CROSS_REGION 設定）

Snowflake の東京リージョンでは、Cortex LLM（`COMPLETE`・`SUMMARIZE` 等）・Cortex Analyst・CoCo（Cortex Code）などの AI 機能は、デフォルト状態では **利用不可** です。

これらの機能のモデルは現時点で東京リージョンでは直接ホストされていないため、**他リージョンへのクロスリージョン呼び出しを明示的に許可** する必要があります。

```sql
USE ROLE ACCOUNTADMIN;

ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION = 'ANY_REGION';
```

| 設定値 | 意味 |
|--------|------|
| `DISABLED`（デフォルト） | クロスリージョン呼び出し不可（東京リージョン単体のみ） |
| `'AWS_US_WEST_2'` など | 特定リージョンのみ許可 |
| `'ANY_REGION'` | すべてのリージョンを許可 |

この設定を行うことで、CoCo CLI・Cortex Analyst・Cortex Agent などが東京リージョンのアカウントから利用可能になります。

:::message alert
**データ越境の考慮**

`ANY_REGION` を設定するとクエリの一部が海外リージョンで処理されます。個人検証用途では問題ありませんが、本番環境では自社のデータガバナンスポリシーや法的要件に照らし合わせて判断してください。
:::

## Step 4 — コスト管理（Budget 設定）

検証用アカウントでは意図しないクレジット超過を防ぐため、**アカウントルートバジェット** を設定します。月次のスペンディングリミットに達すると通知が届きます。

```sql
USE ROLE ACCOUNTADMIN;

-- バジェットを有効化
CALL SNOWFLAKE.LOCAL.ACCOUNT_ROOT_BUDGET!ACTIVATE();

-- 月額 $30 のスペンディングリミットを設定
CALL SNOWFLAKE.LOCAL.ACCOUNT_ROOT_BUDGET!SET_SPENDING_LIMIT(30);

-- 通知先メールアドレスを設定
CALL SNOWFLAKE.LOCAL.ACCOUNT_ROOT_BUDGET!SET_EMAIL_NOTIFICATIONS('your-email@example.com');

-- 設定確認
CALL SNOWFLAKE.LOCAL.ACCOUNT_ROOT_BUDGET!GET_CONFIG();
```

:::message
**Budget vs Resource Monitor**

Snowflake には似た機能として `RESOURCE MONITOR` もあります。

| 機能 | 粒度 | 用途 |
|------|------|------|
| Budget | アカウント全体・オブジェクト単位 | 月次の支出上限管理 |
| Resource Monitor | ウェアハウス単位 | クレジット消費量の監視・停止 |

個人検証では Budget だけで十分です。チームやプロジェクトでウェアハウスを分ける場合は Resource Monitor も検討してください。
:::

## Step 5 — 管理用データベース・スキーマの作成

セキュリティポリシー（マスキングポリシー・行アクセスポリシー等）を格納するための専用 DB とスキーマを作成します。Snowflake ではポリシーオブジェクトも通常のテーブルと同様にスキーマの中に置くため、管理の起点となる DB を用意しておくと後々整理しやすくなります。

```sql
USE ROLE SYSADMIN;

CREATE DATABASE DB_MANAGEMENT;
CREATE SCHEMA    DB_MANAGEMENT.SCM_SECURITY;
```

作成後、スキーマの所有権を `SECURITYADMIN` に移譲します。「ポリシーの作成・管理は `SECURITYADMIN` が担う」という Snowflake の権限設計の原則に沿った設定です。

```sql
USE ROLE ACCOUNTADMIN;

GRANT USAGE     ON DATABASE DB_MANAGEMENT                  TO ROLE SECURITYADMIN;
GRANT OWNERSHIP ON SCHEMA   DB_MANAGEMENT.SCM_SECURITY     TO ROLE SECURITYADMIN;
```

## Step 6 — Email 通知インテグレーションの作成

パイプラインの完了・失敗通知やアラートを Email で受け取るための通知インテグレーションを作成します。Snowflake Tasks や Alerts から `SYSTEM$SEND_EMAIL()` を呼び出す際の送信元として使います。

```sql
USE ROLE ACCOUNTADMIN;

CREATE NOTIFICATION INTEGRATION IF NOT EXISTS SNOWFLAKE_PIPELINE_EMAIL
    TYPE              = EMAIL
    ENABLED           = TRUE
    ALLOWED_RECIPIENTS = ('your-email@example.com');
```

動作確認：

```sql
CALL SYSTEM$SEND_EMAIL(
    'SNOWFLAKE_PIPELINE_EMAIL',
    'your-email@example.com',
    'テスト通知',
    'Snowflake からのメール通知テストです。'
);
```

## Step 7 — ロール・ウェアハウスの設計例（Tasty Bytes を使う場合）

検証データとして Snowflake 公式デモの **Tasty Bytes** を使う場合のロール・ウェアハウス設計例です。用途別にウェアハウスとロールを分けることで、最小権限の原則を体験できます。

### ウェアハウスの作成

```sql
USE ROLE SYSADMIN;

-- 用途別に専用ウェアハウスを作成（すべて XS・Auto Suspend 60秒）
CREATE OR REPLACE WAREHOUSE tasty_de_wh
    WAREHOUSE_SIZE = 'xsmall'  AUTO_SUSPEND = 60  AUTO_RESUME = TRUE  INITIALLY_SUSPENDED = TRUE
    COMMENT = 'data engineering warehouse for tasty bytes';

CREATE OR REPLACE WAREHOUSE tasty_ds_wh
    WAREHOUSE_SIZE = 'xsmall'  AUTO_SUSPEND = 60  AUTO_RESUME = TRUE  INITIALLY_SUSPENDED = TRUE
    COMMENT = 'data science warehouse for tasty bytes';

CREATE OR REPLACE WAREHOUSE tasty_bi_wh
    WAREHOUSE_SIZE = 'xsmall'  AUTO_SUSPEND = 60  AUTO_RESUME = TRUE  INITIALLY_SUSPENDED = TRUE
    COMMENT = 'business intelligence warehouse for tasty bytes';

CREATE OR REPLACE WAREHOUSE tasty_dev_wh
    WAREHOUSE_SIZE = 'xsmall'  AUTO_SUSPEND = 60  AUTO_RESUME = TRUE  INITIALLY_SUSPENDED = TRUE
    COMMENT = 'developer warehouse for tasty bytes';

CREATE OR REPLACE WAREHOUSE tasty_data_app_wh
    WAREHOUSE_SIZE = 'xsmall'  AUTO_SUSPEND = 60  AUTO_RESUME = TRUE  INITIALLY_SUSPENDED = TRUE
    COMMENT = 'data app warehouse for tasty bytes';
```

### ロールの作成と階層設計

```sql
USE ROLE SECURITYADMIN;

-- 専用ロールの作成
CREATE ROLE IF NOT EXISTS tasty_admin          COMMENT = 'admin for tasty bytes';
CREATE ROLE IF NOT EXISTS tasty_data_engineer  COMMENT = 'data engineer for tasty bytes';
CREATE ROLE IF NOT EXISTS tasty_data_scientist COMMENT = 'data scientist for tasty bytes';
CREATE ROLE IF NOT EXISTS tasty_bi             COMMENT = 'business intelligence for tasty bytes';
CREATE ROLE IF NOT EXISTS tasty_data_app       COMMENT = 'data application developer for tasty bytes';
CREATE ROLE IF NOT EXISTS tasty_dev            COMMENT = 'developer for tasty bytes';

-- ロール階層の設計（子ロールの権限を親ロールが継承する）
GRANT ROLE tasty_data_engineer  TO ROLE tasty_admin;
GRANT ROLE tasty_data_scientist TO ROLE tasty_admin;
GRANT ROLE tasty_bi             TO ROLE tasty_admin;
GRANT ROLE tasty_data_app       TO ROLE tasty_admin;
GRANT ROLE tasty_dev            TO ROLE tasty_data_engineer;
GRANT ROLE tasty_admin          TO ROLE SYSADMIN;
```

ロール階層のイメージ：

```
SYSADMIN
└── tasty_admin
    ├── tasty_data_engineer
    │   └── tasty_dev
    ├── tasty_data_scientist
    ├── tasty_bi
    └── tasty_data_app
```

### ウェアハウスとマスキングポリシー権限の付与

```sql
USE ROLE ACCOUNTADMIN;

-- 各ロールがマスキングポリシーを適用できるように権限付与
GRANT APPLY MASKING POLICY ON ACCOUNT TO ROLE tasty_admin;
GRANT APPLY MASKING POLICY ON ACCOUNT TO ROLE tasty_data_engineer;

-- Snowflake メタデータDB へのアクセス（ACCOUNT_USAGE 等）
GRANT IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE TO ROLE tasty_data_engineer;
```

## 設定まとめ

実際にこのアカウントで行った初期設定を一覧にまとめます。

| # | 設定項目 | コマンド / 操作 | 理由 |
|---|---------|----------------|------|
| 1 | タイムゾーン設定 | `ALTER ACCOUNT SET TIMEZONE = 'Asia/Tokyo'` | 日本時間で QUERY_HISTORY・ログを確認するため |
| 2 | ステートメントタイムアウト | `ALTER ACCOUNT SET STATEMENT_TIMEOUT_IN_SECONDS = 3600` | 無制限クエリによるクレジット超過を防ぐ |
| 3 | CROSS_REGION 有効化 | `ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION = 'ANY_REGION'` | Cortex LLM・Analyst・CoCo を東京リージョンで使うため |
| 4 | ウェアハウス Auto Suspend | `ALTER WAREHOUSE COMPUTE_WH SET AUTO_SUSPEND = 60` | アイドル時のクレジット消費を抑える |
| 5 | Budget 設定 | `ACCOUNT_ROOT_BUDGET!ACTIVATE()` + `SET_SPENDING_LIMIT(30)` | 月次の支出上限を $30 に設定 |
| 6 | 管理用 DB 作成 | `CREATE DATABASE DB_MANAGEMENT` + スキーマ + 権限移譲 | ポリシーオブジェクトの管理起点 |
| 7 | Email 通知 Integration | `CREATE NOTIFICATION INTEGRATION` | パイプライン通知・アラートのメール送信 |

最低限 **Step 1〜4** を設定しておくだけで、コスト増大のリスクを大幅に下げながら Cortex 系の機能が使えるようになります。

## まとめ

個人の検証環境でも「まず Cortex 機能が使えるようにしたい」「うっかりクレジットを使いすぎたくない」という 2 点を押さえるだけで、かなり快適に使えます。

特に `CORTEX_ENABLED_CROSS_REGION = 'ANY_REGION'` の設定は、東京リージョンで AI 機能を試したい場合にほぼ必須です。この設定なしで `CORTEX_COMPLETE()` や CoCo を使おうとしてもエラーになるため、アカウント作成直後に設定しておくことをお勧めします。
