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


## Step 3 — Cortex AI 機能

Cortex AI 機能（`CORTEX_COMPLETE`・Cortex Analyst・CoCo 等）の AI Credit を Global 価格で使うために `CORTEX_ENABLED_CROSS_REGION = 'ANY_REGION'` を設定します。

`ANY_REGION` を設定することで AI Credit が Regional（$2.20）から Global（$2.00）に切り替わり、約 9% コストを抑えられます（2026年7月4日現在 / 出典: Snowflake Credit Consumption Table）。

| 設定値 | 対象リージョン | AI Credit 単価 |
|--------|--------------|---------------|
| `DISABLED`（デフォルト） | クロスリージョン無効 | $2.20（Regional） |
| `'AWS_JP'`・`'AWS_EU'` など地域グループ | 指定クラウド・地域内 | $2.20（Regional） |
| **`'ANY_REGION'`（推奨）** | **全リージョン** | **$2.00（Global）** |

指定できる値の一覧（`AWS_JP` / `AWS_APJ` / `AWS_AU` / `AWS_EU` / `AWS_US` / `AWS_GLOBAL` / `AZURE_EU` / `AZURE_US` / `AZURE_GLOBAL` / `GCP_US` / `GCP_GLOBAL` / `ANY_REGION`）は [Snowflake 公式ドキュメント（CORTEX_ENABLED_CROSS_REGION）](https://docs.snowflake.com/en/sql-reference/parameters#cortex-enabled-cross-region) を参照してください。

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

個人検証では Budget + Cost Anomaly Detection を組み合わせて使っています。
:::

## Step 5 — 管理データベース

セキュリティポリシーオブジェクト（マスキングポリシー等）を格納するための専用 DB・スキーマを作成します。ポリシーオブジェクトも通常のテーブルと同様にスキーマに属するため、管理の起点となる DB を分けておくと後々整理しやすくなります。

| オブジェクト | 名前 | 備考 |
|------------|------|------|
| Database | `DB_GOVERNANCE` | ガバナンス・ポリシー管理専用 |
| Schema | `DB_GOVERNANCE.SCM_SECURITY` | セキュリティポリシー格納 |
| Authentication Policy | `AUTH_POLICY_DEFAULT` | 認証方式・MFA を定義 |

:::details SQL: DB・スキーマ作成
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

### 認証ポリシー（Authentication Policy）

Snowflake CLI や CoCo CLI から PAT（Programmatic Access Token）で接続するために認証ポリシーを作成し、ユーザーに適用します。

| 設定 | 値 | 理由 |
|------|------|------|
| `AUTHENTICATION_METHODS` | `PROGRAMMATIC_ACCESS_TOKEN`, `PASSWORD` | CLI は PAT、UI はパスワードを使い分ける |
| `MFA_ENROLLMENT` | `REQUIRED` | パスワード認証時は MFA を強制 |

:::message
**PAT 認証と MFA の関係**

Authentication Policy で `MFA_ENROLLMENT = 'REQUIRED'` を設定しても、PAT 認証（CLI からの接続）では MFA はバイパスされます。UI からのパスワード認証にのみ MFA が適用されます。
:::

:::details SQL: 認証ポリシー作成・適用
```sql
USE ROLE SECURITYADMIN;

CREATE AUTHENTICATION POLICY IF NOT EXISTS DB_GOVERNANCE.SCM_SECURITY.AUTH_POLICY_DEFAULT
    AUTHENTICATION_METHODS = ('PROGRAMMATIC_ACCESS_TOKEN', 'PASSWORD')
    MFA_ENROLLMENT         = 'REQUIRED';

-- ユーザーへの適用（ユーザー名を変更して実行）
ALTER USER MY_USERNAME SET AUTHENTICATION POLICY DB_GOVERNANCE.SCM_SECURITY.AUTH_POLICY_DEFAULT;
```
:::

## Step 6 — Snowflake CLI

Snowflake CLI（`snow`）をインストールし、PAT 認証で接続できる状態にします。

### インストール

```bash
# uv（推奨）
uv tool install snowflake-cli

# pip
pip install snowflake-cli

# バージョン確認
snow --version
```

### 接続設定ファイル（config.toml）

設定ファイルの場所:

| OS | パス |
|----|------|
| macOS | `~/Library/Application Support/snowflake/config.toml` |
| Linux | `~/.config/snowflake/config.toml` |
| Windows | `%USERPROFILE%\AppData\Local\snowflake\config.toml` |

`config.toml` の設定例（PAT 認証）:

```toml
default_connection_name = "myaccount"

[connections.myaccount]
account           = "ACCOUNT_IDENTIFIER"    # 例: xy12345.us-west-2
user              = "MY_USERNAME"
authenticator     = "PROGRAMMATIC_ACCESS_TOKEN"
token_file_path   = "/path/to/pat-token.txt"
warehouse         = "COMPUTE_WH"
```

`token_file_path` に指定したファイルに PAT トークン文字列のみを書き込んでおきます。

接続確認:

```bash
snow connection test --connection myaccount
```

### PAT（Programmatic Access Token）の発行

**Snowsight から発行する場合:**
1. **Admin > Users & Roles** でユーザーを選択
2. **Generate new token** をクリック
3. トークン名・有効期限・ロール制限を設定して生成
4. 表示されたトークン文字列を即コピー（再表示不可）
5. `token_file_path` に指定したファイルに書き込む

**SQL から発行する場合:**

:::details SQL
```sql
ALTER USER MY_USERNAME ADD PROGRAMMATIC ACCESS TOKEN MY_PAT_TOKEN;
```
:::

:::message alert
**PAT 認証を使うには認証ポリシーが必要です**

Step 5 で作成した `AUTH_POLICY_DEFAULT` を事前にユーザーに適用しておいてください。ポリシーで `PROGRAMMATIC_ACCESS_TOKEN` が許可されていない場合、PAT 認証は失敗します。
:::

## Step 7 — CoCo CLI

CoCo CLI（`coco`）は Snowflake CLI の接続設定をそのまま利用します。Step 6 で設定した接続が使えます。

### インストール

```bash
# uv（推奨）
uv tool install snowflake-coco

# pip
pip install snowflake-coco

# バージョン確認
coco --version
```

### 起動・接続確認

```bash
# デフォルト接続で起動
coco

# 接続名を明示して起動
coco --connection myaccount
```

起動すると対話型セッションが開始されます。接続先アカウント・ユーザー・ウェアハウスが正しく表示されれば接続成功です。

## 設定まとめ

| # | 設定項目 | コマンド / 操作 | 理由 |
|---|---------|----------------|------|
| 1 | タイムゾーン設定 | `ALTER ACCOUNT SET TIMEZONE = 'Asia/Tokyo'` | QUERY_HISTORY・ログを日本時間で確認するため |
| 2 | ステートメントタイムアウト | `ALTER ACCOUNT SET STATEMENT_TIMEOUT_IN_SECONDS = 3600` | 無制限クエリによるクレジット超過を防ぐ |
| 3 | ウェアハウス Gen1・Auto Suspend | `ALTER WAREHOUSE COMPUTE_WH SET GENERATION = 1 AUTO_SUSPEND = 60` | Gen2→Gen1 + アイドル停止でクレジット削減 |
| 4 | CROSS_REGION 有効化 | `ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION = 'ANY_REGION'` | AI Credit を Global 価格（$2.00）で使うため |
| 5 | Budget 設定 | `ACCOUNT_ROOT_BUDGET!ACTIVATE()` + `SET_SPENDING_LIMIT(30)` | 月次の支出上限を $30 に設定 |
| 6 | 管理 DB・認証ポリシー作成 | `CREATE DATABASE DB_GOVERNANCE` + `CREATE AUTHENTICATION POLICY` + ユーザー適用 | ポリシー管理起点 + PAT 認証の前提 |
| 7 | Snowflake CLI 接続設定 | `config.toml` に PAT 認証設定 + `snow connection test` | CLI からの接続を PAT で行う |
| 8 | CoCo CLI インストール・接続確認 | `uv tool install snowflake-coco` + `coco --connection myaccount` | Cortex Code CLI の利用 |
