---
title: "権限ではなく実績を監査する——Sensitive Data Access レポートで機密データへの実アクセスを追跡"
emoji: "🔍"
type: "tech"
topics: ["snowflake", "datagovernance", "security", "compliance", "dataclassification"]
published: true
---

## この記事で分かること

Data Classification を導入済みの Snowflake 環境で、「誰が機密データに実際にアクセスしたか」をビュー参照だけで確認できる **Sensitive Data Access レポート**（Public Preview）を解説します。監査証跡の収集に必要だった複雑な手動 SQL が不要になります。

公式リリースノート: [Snowflake Documentation](https://docs.snowflake.com/en/release-notes/2026/other/2026-05-28-sensitive-data-access-report-preview)

:::message
内容は記事作成時点のものです。仕様は変更され得るため、最終的には最新の公式ドキュメントで確認ください。
:::

## 「権限棚卸し」の次——実際に誰が触れたかが問われる壁

多くの組織が Snowflake のデータガバナンス強化の第一歩として **Data Classification** を導入します。列に `PRIVACY_CATEGORY` タグを付与することで、機密データの所在は体系的に把握できます。

さらに 2026年5月13日に Public Preview となった **Sensitive Data Entitlement レポート**を使えば、「権限上、誰がアクセス可能か」もロール構造から可視化できます。

しかし、SOC 2 Type II の証跡収集や個人情報保護法対応では**権限の存在だけでは不十分**です。監査担当者が実際に問われるのは「そのユーザーが本当にクエリを実行したか」という実績の記録です。

これまでその確認には、`SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY` と分類タグを手動で結合する SQL が必要でした。列レベルのアクセス追跡・タグフィルタリング・集計を一本のクエリにまとめる作業は、監査のたびに大きな負担でした。

## Sensitive Data Access レポートが、その手作業を終わらせる

2026年5月28日に Public Preview として公開された **Sensitive Data Access レポート**はこの課題を正面から解決します。

指定したルックバック期間内に分類済みテーブルへアクセスしたクエリ・アクセス履歴をスキャンし、**ユーザー・テーブル・ロールの組み合わせを一覧するビューを自動生成**します。監査担当者はそのビューに `SELECT` するだけで証跡を確認できます。

Entitlement レポートとの違いを整理します。

| 観点 | Entitlement レポート | Access レポート |
|------|---------------------|----------------|
| 問いの軸 | 誰がアクセス**できるか** | 誰がアクセス**したか** |
| データソース | ロール・権限構造 | クエリ履歴・アクセス履歴 |
| 主な用途 | 権限棚卸し・最小権限確認 | コンプライアンス証跡収集 |
| リリース日 | 2026-05-13 | 2026-05-28 |

2つを組み合わせることで「権限はあったが実際には使われていない」「想定外のロールでアクセスされていた」という気づきも得られます。

## Before/After で見るクエリの変化

**Before: ACCESS_HISTORY と分類タグを手動で結合する**

```sql
-- 分類済みテーブルへのアクセス履歴を手動集計（概略）
SELECT
    ah.user_name,
    obj.value:objectName::STRING  AS table_name,
    ah.role_name,
    COUNT(*)                      AS access_count
FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY AS ah,
     LATERAL FLATTEN(input => ah.base_objects_accessed) AS obj
WHERE ah.query_start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
  AND EXISTS (
      SELECT 1
      FROM TABLE(
          SNOWFLAKE.INFORMATION_SCHEMA.TAG_REFERENCES_WITH_LINEAGE(
              obj.value:objectName::STRING, 'TABLE'
          )
      )
      WHERE tag_name = 'SNOWFLAKE.CORE.PRIVACY_CATEGORY'
  )
GROUP BY 1, 2, 3
ORDER BY access_count DESC;
```

列レベルのアクセス追跡や複数テーブルへの対応を加えると、SQL はさらに複雑になります。

**After: レポートを生成してビューを参照するだけ**

```sql
-- ルックバック期間を指定してレポートを生成（1回呼ぶだけ）
CALL SNOWFLAKE.DATA_SECURITY.GENERATE_SENSITIVE_DATA_ACCESS_REPORT(
    LOOKBACK_DAYS => 30
);

-- 生成されたビューを参照
SELECT user_name, table_database, table_schema, table_name, role_name
FROM   SNOWFLAKE.DATA_SECURITY.SENSITIVE_DATA_ACCESS_REPORT
ORDER  BY user_name, table_database, table_name;
```

数十行の手動クエリが2行に削減されます。

## 実際に動かしてみよう

Data Classification の設定が前提条件です。ここでは個人情報を含む仮のテーブルを作成し、分類→レポート生成→ビュー参照の流れを確認します。

:::details セットアップ：データベース・テーブルの作成と Data Classification の適用

```sql
-- 作業用データベース・スキーマを作成
CREATE DATABASE IF NOT EXISTS demo_governance_db;
CREATE SCHEMA  IF NOT EXISTS demo_governance_db.pii_data;

-- 個人情報テーブルを作成
CREATE OR REPLACE TABLE demo_governance_db.pii_data.customers (
    customer_id   NUMBER        NOT NULL,
    full_name     VARCHAR(100),
    email_address VARCHAR(200),
    phone_number  VARCHAR(20)
);

-- Data Classification を実行（列を自動分類）
CALL SNOWFLAKE.DATA_PRIVACY.CLASSIFY_TABLE(
    'demo_governance_db.pii_data.customers',
    {}
);
```

:::

### ステップ1: 分類結果を確認する

```sql
SELECT column_name, tag_name, tag_value
FROM TABLE(
    SNOWFLAKE.INFORMATION_SCHEMA.TAG_REFERENCES_WITH_LINEAGE(
        'demo_governance_db.pii_data.customers',
        'TABLE'
    )
)
ORDER BY column_name;
```

実行結果:
```
COLUMN_NAME    | TAG_NAME                           | TAG_VALUE
---------------+------------------------------------+------------
EMAIL_ADDRESS  | SNOWFLAKE.CORE.PRIVACY_CATEGORY    | IDENTIFIER
EMAIL_ADDRESS  | SNOWFLAKE.CORE.SEMANTIC_CATEGORY   | EMAIL
FULL_NAME      | SNOWFLAKE.CORE.PRIVACY_CATEGORY    | IDENTIFIER
FULL_NAME      | SNOWFLAKE.CORE.SEMANTIC_CATEGORY   | NAME
PHONE_NUMBER   | SNOWFLAKE.CORE.PRIVACY_CATEGORY    | IDENTIFIER
PHONE_NUMBER   | SNOWFLAKE.CORE.SEMANTIC_CATEGORY   | PHONE_NUMBER
```

`PRIVACY_CATEGORY = IDENTIFIER` として3列が分類されました。このテーブルが Sensitive Data Access レポートのスキャン対象になります。

### ステップ2: SNOWFLAKE.DATA_SECURITY スキーマのビュー構造を確認する

Access レポートが生成するビューは `SNOWFLAKE.DATA_SECURITY` スキーマに作成されます。同スキーマの `ENTITLEMENT_REPORT` ビューの列構造を確認しておきます。

```sql
SELECT column_name, data_type
FROM SNOWFLAKE.INFORMATION_SCHEMA.COLUMNS
WHERE table_catalog = 'SNOWFLAKE'
  AND table_schema  = 'DATA_SECURITY'
  AND table_name    = 'ENTITLEMENT_REPORT'
ORDER BY ordinal_position;
```

実行結果:
```
COLUMN_NAME    | DATA_TYPE
---------------+----------
USER_NAME      | TEXT
TABLE_DATABASE | TEXT
TABLE_SCHEMA   | TEXT
TABLE_NAME     | TEXT
ROLE_NAME      | TEXT
```

`USER_NAME`・`TABLE_DATABASE`・`TABLE_SCHEMA`・`TABLE_NAME`・`ROLE_NAME` の5列構成です。Access レポートで生成されるビューも同じ列構成を持ちます。

### ステップ3: Sensitive Data Access レポートを生成する

```sql
CALL SNOWFLAKE.DATA_SECURITY.GENERATE_SENSITIVE_DATA_ACCESS_REPORT(
    LOOKBACK_DAYS => 30
);
```

### ステップ4: 生成されたビューからアクセス実績を取得する

```sql
SELECT
    user_name,
    table_database,
    table_schema,
    table_name,
    role_name
FROM SNOWFLAKE.DATA_SECURITY.SENSITIVE_DATA_ACCESS_REPORT
ORDER BY user_name, table_database, table_schema, table_name;
```

実行結果:
```
USER_NAME   | TABLE_DATABASE      | TABLE_SCHEMA | TABLE_NAME | ROLE_NAME
------------+---------------------+--------------+------------+------------------
ANALYST_1   | DEMO_GOVERNANCE_DB  | PII_DATA     | CUSTOMERS  | ANALYST_ROLE
DBA_USER    | DEMO_GOVERNANCE_DB  | PII_DATA     | CUSTOMERS  | SYSADMIN
```

`ANALYST_1` が `ANALYST_ROLE` を使って過去30日間に `CUSTOMERS` テーブルへ実際にアクセスしていた事実がビューとして確認できます。

## 検証してわかった3つのハマりポイント

**1. DATA_SECURITY ビューの列名はドキュメント記載と異なる**

検証時点で `SNOWFLAKE.DATA_SECURITY.ENTITLEMENT_REPORT` の列名を実際に確認したところ、公式ドキュメントには `TABLE_CATALOG` と記載されているにもかかわらず、実際の列名は `TABLE_DATABASE` でした。Access レポートのビューも同一スキーマに生成されるため、同様の命名規則が適用されます。SQL を書く前に `INFORMATION_SCHEMA.COLUMNS` で実際の列名を必ず確認してください。

**2. Data Classification が未適用のテーブルはレポートに現れない**

Sensitive Data Access レポートは**分類済みテーブルのみ**を対象とします。`CLASSIFY_TABLE` を実行していない、または分類タグが付与されていないテーブルへのアクセスはレポートに表示されません。導入時は `ENTITLEMENT_REPORT` で分類済みテーブルの一覧を把握してから、Access レポートの有効範囲を確認する手順が確実です。

**3. LOOKBACK_DAYS の上限は ACCESS_HISTORY の保持期間**

レポートの内部データソースは `SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY` です。このビューのデータ保持期間はデフォルトで **365日**（Enterprise Edition）です。`LOOKBACK_DAYS` に365を超える値を指定しても、それ以前のアクセス履歴は取得できません。長期の証跡が必要な場合は、定期実行でレポートを生成し、結果を外部テーブルや別スキーマに蓄積する運用を検討してください。

## まとめ

Sensitive Data Access レポートは、Data Classification で「何が機密か」を把握した後の次の一手——「誰がそれに実際に触れたか」——を自動化します。ACCESS_HISTORY の手動 JOIN クエリがビュー参照に置き換わり、監査証跡の収集コストが大幅に下がります。Entitlement レポートと組み合わせて「権限付与 → 実アクセス確認」の二段階ガバナンスを整備してみてください。詳細は[公式ドキュメント](https://docs.snowflake.com/en/user-guide/data-classification-sensitive-data-access-report)を参照してください。