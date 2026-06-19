---
title: "SnowflakeテーブルからIcebergテーブルへの移行ガイド"
emoji: "🧊"
type: "tech"
topics: ["snowflake", "iceberg", "dataengineering", "sql", "datalake"]
published: true
---

## この記事について

本記事ではバッチのパイプライン（日次などで定期更新されるもの）を対象に、既存のネイティブテーブルをIcebergテーブルへ移行する方法を解説します。移行手順において、既存テーブルに付与された権限・属性・依存オブジェクト（ビュー・ストリーム・タスクなど）が移行後にどうなるかを実際に検証し、問題があった項目については対処方法を示します。

:::message
**本記事の対象範囲**
今回の検証はリアルタイムパイプライン（ストリーミング等）を伴わないバッチ処理を対象としています。リアルタイムパイプラインを伴う移行ではダウンタイムなしでの切り替えが必要となるため、別途検証が必要です。
:::

:::message alert
**免責事項**
本記事の内容は筆者個人の環境での検証結果をまとめたものであり、特定の環境・バージョン・設定における動作を保証するものではありません。本番環境へ適用する際は、必ずご自身の環境で十分な検証を行った上でご判断ください。本記事の情報を参考にした結果生じたいかなる損害についても、筆者は責任を負いかねます。
:::


![](/images/i119-snowflake-iceberg-table-migration/cover.png)

## 背景

Apache Icebergはオープンな大規模テーブルフォーマット仕様です。Snowflake Managed Icebergテーブル（`CATALOG = 'SNOWFLAKE'`）を使うことで、データをSnowflake管理ストレージにIceberg形式で保存しつつ、通常のSnowflakeテーブルと同様のSQL操作が可能になります。

Iceberg形式でデータを保存することには以下のメリットがあります。

**① ベンダーロックインからの解放**
データがオープンフォーマット（Iceberg形式）で保存されるため、特定のデータウェアハウスに縛られません。将来的なプラットフォーム移行の際も、データをそのまま活用できます。

**② 他エンジンとの相互運用性**
1つのデータコピーをSnowflake・Apache Spark・Trino・Prestoなど複数のエンジンから直接参照・処理できます。データのコピーや変換が不要になり、データスタック全体のコストと複雑さを削減できます。

## 移行前に確認すべき観点

既存ネイティブテーブルにはさまざまな属性・依存オブジェクトが設定されています。移行後にこれらが引き続き正常に機能するかを、以下の観点で検証します。

| # | 観点 |
|---|------|
| 1 | 権限 |
| 2 | Future Grants |
| 3 | View |
| 4 | Dynamic Table |
| 5 | Stream |
| 6 | Task |
| 7 | Masking Policy / Row Access Policy |
| 8 | Clustering Key |
| 9 | DEFAULT値 / NOT NULL制約 |
| 10 | Tags |
| 11 | Time Travel |
| 12 | VARIANT |
| 13 | GEOGRAPHY |
| 14 | タイムスタンプ型 |
| 15 | COMMENT |

## 移行時に考えること

Icebergへの移行では、大きく2つのケースを考える必要があります。

**① 既存テーブルをIcebergに変換する**
すでに運用中のネイティブテーブルをIceberg化します。データが既に存在するため、データ移行の手順が必要になります。

**② 新規テーブルをIcebergで作成する**
Snowflakeは[2026年6月5日のアップデート](https://docs.snowflake.com/en/release-notes/2026/other/2026-06-05-iceberg-default-metadata-format-ga)で `DEFAULT_METADATA_WRITE_FORMAT` パラメーターをGAしました。このパラメーターをDB/スキーマレベルで設定すると、通常の `CREATE TABLE` を実行するだけでIcebergテーブルが作成されます。`CREATE ICEBERG TABLE` と書く必要はありません。

本記事ではこの2つをそれぞれ検証します。

- **検証A**: パラメーターを有効化した状態で `CREATE TABLE` を実行し、Icebergテーブルが作成されることを確認します
- **検証B**: 既存テーブルの移行手順（DDL+INSERT+RENAME SWAP）で、各観点（権限・ストリーム・タグなど）がどうなるかを確認します

## 検証対象のデータ型

今回の検証では、以下のデータ型を含むソーステーブルを使います。Icebergテーブルでの動作確認対象は次の型です。

| カラム | ネイティブ型 |
|--------|------------|
| `id` | `NUMBER(10,0)` |
| `name` | `VARCHAR(256)` |
| `dept` | `VARCHAR(50)` |
| `score` | `FLOAT` |
| `payload` | `VARIANT` |
| `geo_point` | `GEOGRAPHY` |
| `created_at` | `TIMESTAMP_TZ(9)` |

これらの型がIcebergテーブルでそのまま使えるか、変換が必要かは検証で明らかにします。

## 移行方式の検討

既存ネイティブテーブルをIceberg化する方式として、主に以下の2つが考えられます。

### 方式1: CTAS（CREATE TABLE AS SELECT）

```sql
CREATE TABLE native_src_iceberg AS SELECT * FROM native_src;
```

シンプルで手順が少ない一方、テーブルオブジェクトを新規作成するため、ソーステーブルに設定された**以下の属性が引き継がれません**。

- Clustering Key
- カラムのDEFAULT値
- NOT NULL制約
- テーブル・カラムのCOMMENT

### 方式2: DDL + INSERT + RENAME SWAP

元のDDLからIceberg対応のDDLを作成し、INSERTでデータを移行した後にテーブル名をRENAMEで入れ替える方式です。

DDLを手動で作成するため手順は増えますが、Clustering Key・DEFAULT値・NOT NULL制約・COMMENTをすべて明示的に引き継げます。また、今回確認すべき観点（権限・タグ・ストリーム・タスクなど）の挙動を正確に把握した上で移行できます。

**今回はこの方式を採用します。**

## 検証

### 検証環境

- Snowflake CLI（snow）v3.18.0
- ロール: ACCOUNTADMIN

### 検証A: パラメーター設定でCREATE TABLEがIcebergになることの確認

最初に、`CREATE TABLE` だけでIcebergテーブルが作成されるパラメーターを確認します。

```sql
-- アカウントレベル: v3をデフォルトに設定
ALTER ACCOUNT SET ICEBERG_VERSION_DEFAULT = 3;

-- DBレベル: CREATE TABLEをすべてIcebergに
ALTER DATABASE verify_iceberg_migrate SET DEFAULT_METADATA_WRITE_FORMAT = 'ICEBERG';

-- CREATE TABLE（Icebergとは書かない）
CREATE TABLE new_after_param (id NUMBER(10,0), val STRING);

SELECT table_name, is_iceberg, iceberg_version
FROM information_schema.tables
WHERE table_name = 'NEW_AFTER_PARAM';
```

**出力:**

```
+-----------------+------------+-----------------+
| TABLE_NAME      | IS_ICEBERG | ICEBERG_VERSION |
|-----------------+------------+-----------------|
| NEW_AFTER_PARAM | YES        | 3               |
+-----------------+------------+-----------------+
```

`CREATE TABLE` だけで Iceberg v3 テーブルが作成されることを確認しました。以降の検証はこのパラメーター設定が有効な状態で行います。

---

### 検証B: DDL+INSERT+RENAME SWAP移行

#### Step 0: 検証用ソーステーブルの準備

移行対象となるネイティブテーブルを作成し、各観点（権限・タグ・ストリームなど）を設定します。

```sql
CREATE OR REPLACE TABLE native_src (
  id           NUMBER(10,0)   NOT NULL,
  name         VARCHAR(256)   DEFAULT 'unknown',
  dept         VARCHAR(50)    NOT NULL,
  score        FLOAT          DEFAULT 0.0,
  payload      VARIANT,
  geo_point    GEOGRAPHY,
  created_at   TIMESTAMP_TZ(9)
)
CLUSTER BY (id)
COMMENT = 'Iceberg migration verification source table';

INSERT INTO native_src (id, name, dept, score, payload, geo_point, created_at)
SELECT 1,'Alice','OPEN',9.5,PARSE_JSON('{"role":"admin"}'),
       ST_GEOGRAPHYFROMWKT('POINT(139.6917 35.6895)'),CURRENT_TIMESTAMP
UNION ALL SELECT 2,'Bob','SECRET',7.0,PARSE_JSON('{"role":"viewer"}'),NULL,CURRENT_TIMESTAMP
UNION ALL SELECT 3,'Carol','OPEN',8.2,NULL,
       ST_GEOGRAPHYFROMWKT('POINT(-122.4194 37.7749)'),CURRENT_TIMESTAMP;

-- 権限付与
CREATE ROLE verify_role_a;
GRANT SELECT, INSERT ON TABLE native_src TO ROLE verify_role_a;
GRANT SELECT ON FUTURE TABLES IN SCHEMA test_schema TO ROLE verify_role_a;

-- タグ設定
CREATE TAG tbl_sensitivity ALLOWED_VALUES 'HIGH', 'MEDIUM', 'LOW';
CREATE TAG col_department;
ALTER TABLE native_src SET TAG tbl_sensitivity = 'HIGH';
ALTER TABLE native_src ALTER COLUMN dept SET TAG col_department = 'HR_SYSTEM';

-- 依存オブジェクト
CREATE STREAM native_src_stream ON TABLE native_src;
CREATE VIEW native_src_view AS SELECT id, name, dept FROM native_src;
CREATE DYNAMIC TABLE native_src_dt TARGET_LAG='1 minute' WAREHOUSE=COMPUTE_WH
  AS SELECT id, name, dept, score FROM native_src;
CREATE TASK native_src_task WAREHOUSE=COMPUTE_WH SCHEDULE='USING CRON 0 * * * * UTC'
  AS INSERT INTO native_src (id, name, dept) SELECT 9999,'task_test','OPEN';
```

#### Step 1: Iceberg用DDL作成

ネイティブテーブルのDDLからIceberg対応のDDLを作成します。型変換の詳細は [データ型変換テーブル](#データ型変換テーブル) を参照してください。

```sql
CREATE TABLE native_src_iceberg (
  id           NUMBER(10,0)    NOT NULL,
  name         STRING          DEFAULT 'unknown',   -- VARCHAR(256) → STRING
  dept         STRING          NOT NULL,            -- VARCHAR(50) → STRING
  score        DOUBLE          DEFAULT 0.0,         -- FLOAT → DOUBLE
  payload      VARIANT,                             -- v3必須
  geo_point    GEOGRAPHY,                           -- v3必須
  created_at   TIMESTAMP_LTZ                        -- TIMESTAMP_TZ(9) → TIMESTAMP_LTZ
)
CLUSTER BY (id)
COMMENT = 'Iceberg migration verification source table'
ICEBERG_VERSION = 3;
```

`DEFAULT_METADATA_WRITE_FORMAT = 'ICEBERG'` が設定済みのため `CREATE TABLE` だけでIcebergテーブルになります。

#### Step 2: データINSERT

```sql
INSERT INTO native_src_iceberg
SELECT id, name, dept, score, payload, geo_point, created_at::TIMESTAMP_LTZ
FROM native_src;
```

**出力:**

```
+-------------------------+
| number of rows inserted |
|-------------------------|
| 3                       |
+-------------------------+
```

#### Step 3 & 4: RENAME SWAP（ダウンタイム発生）

**ダウンタイムはStep 3（ソーステーブルへの書き込み停止）からStep 4（RENAME完了）まで発生します。**

```sql
-- Step 3: パイプラインからnative_srcへの書き込みを停止
--         この時点からStep 4完了までがダウンタイム

-- Step 4: RENAME SWAP
--   ネイティブテーブルは ALTER TABLE
--   Icebergテーブルは ALTER ICEBERG TABLE（必須）
ALTER TABLE native_src RENAME TO native_src_old;
ALTER ICEBERG TABLE native_src_iceberg RENAME TO native_src;
```

RENAME後の確認:

```sql
SELECT table_name, is_iceberg FROM information_schema.tables
WHERE table_name IN ('NATIVE_SRC', 'NATIVE_SRC_OLD')
ORDER BY table_name;
```

**出力:**

```
+----------------+------------+
| TABLE_NAME     | IS_ICEBERG |
|----------------+------------|
| NATIVE_SRC     | YES        |
| NATIVE_SRC_OLD | NO         |
+----------------+------------+
```

---

#### Step 5: 各観点の後処理と検証

**[1] 権限（GRANT）**

RENAMEにより旧テーブルオブジェクト（`native_src_old`）には権限が残り、新テーブル（Iceberg）には引き継がれません。

```sql
SHOW GRANTS ON ICEBERG TABLE native_src;
```

**出力（移行直後）:**

```
+-----------+---------------+--------------+
| privilege | granted_on    | grantee_name |
|-----------+---------------+--------------|
| OWNERSHIP | ICEBERG_TABLE | ACCOUNTADMIN |
+-----------+---------------+--------------+
```

SELECT/INSERTがなく、`verify_role_a` は新テーブルにアクセスできません。

**対処:**

```sql
GRANT SELECT, INSERT ON TABLE native_src TO ROLE verify_role_a;
```

**[2] Future Grants**

`GRANT SELECT ON FUTURE TABLES` は、SnowflakeがIcebergテーブルを内部的に `ICEBERG_TABLE` 型として扱うため適用されません。

```sql
-- 新規Icebergテーブルを作成して確認
CREATE TABLE future_grant_test (id NUMBER(10,0), val STRING);
SHOW GRANTS ON ICEBERG TABLE future_grant_test;
```

**出力:**

```
+-----------+------------+--------------+
| privilege | granted_on | grantee_name |
|-----------+------------+--------------|
| OWNERSHIP | TABLE      | ACCOUNTADMIN |
+-----------+------------+--------------+
```

```sql
SHOW FUTURE GRANTS IN SCHEMA test_schema;
```

**出力（設定前）:**

```
+-----------+----------+---------------+
| privilege | grant_on | grantee_name  |
|-----------+----------+---------------|
| SELECT    | TABLE    | VERIFY_ROLE_A |
+-----------+----------+---------------+
```

`grant_on = TABLE` のみで `ICEBERG_TABLE` 向けの設定がありません。

**対処:**

```sql
GRANT SELECT ON FUTURE ICEBERG TABLES IN SCHEMA test_schema TO ROLE verify_role_a;
```

**出力（設定後の `SHOW FUTURE GRANTS`）:**

```
+-----------+---------------+---------------+
| privilege | grant_on      | grantee_name  |
|-----------+---------------+---------------|
| SELECT    | ICEBERG_TABLE | VERIFY_ROLE_A |
| SELECT    | TABLE         | VERIFY_ROLE_A |
+-----------+---------------+---------------+
```

以降に作成されるIcebergテーブルにはSELECTが自動付与されます。

**[3] View（ビュー）**

ViewはSQLテキスト内のテーブル名を参照するため、RENAME後は自動で新テーブル（Iceberg）を参照します。

```sql
SELECT id, name, dept FROM native_src_view;
```

**出力:**

```
+----+-------+--------+
| ID | NAME  | DEPT   |
|----+-------+--------|
| 1  | Alice | OPEN   |
| 2  | Bob   | SECRET |
| 3  | Carol | OPEN   |
+----+-------+--------+
```

**[4] Dynamic Table**

```sql
ALTER DYNAMIC TABLE native_src_dt REFRESH;
SELECT id, name, dept, score FROM native_src_dt;
```

**出力:**

```
+----+-------+--------+-------+
| ID | NAME  | DEPT   | SCORE |
|----+-------+--------+-------|
| 1  | Alice | OPEN   | 9.5   |
| 2  | Bob   | SECRET | 7.0   |
| 3  | Carol | OPEN   | 8.2   |
+----+-------+--------+-------+
```

**[5] Stream（ストリーム）**

StreamはテーブルのオブジェクトIDで追跡します。RENAME後、`native_src_stream` は旧テーブルオブジェクトを追跡し続け、新テーブルへの変更を検知しません。

```sql
INSERT INTO native_src (id, name, dept) SELECT 100, 'TestUser', 'OPEN';
SELECT id, name, METADATA$ACTION FROM native_src_stream;
```

**出力:**

```
(0 rows)
```

**対処: DROP & 再作成**

```sql
DROP STREAM IF EXISTS native_src_stream;
CREATE STREAM native_src_stream ON TABLE native_src;

INSERT INTO native_src (id, name, dept) SELECT 101, 'StreamTest2', 'OPEN';
SELECT id, name, METADATA$ACTION FROM native_src_stream;
```

**出力:**

```
+-----+-------------+-----------------+
| ID  | NAME        | METADATA$ACTION |
|-----+-------------+-----------------|
| 101 | StreamTest2 | INSERT          |
+-----+-------------+-----------------+
```

StreamはテーブルのオブジェクトIDを内部的に保持しており、テーブル名を変えても参照先テーブルオブジェクトは変わりません。移行後は必ずDROPして新テーブルに再作成してください。

**[6] Task（タスク）**

TaskはSQL文のテーブル名を参照するため、RENAME後は新テーブル（Iceberg）にINSERTします。

```sql
EXECUTE TASK native_src_task;
```

数秒後に確認:

```sql
SELECT id, name FROM native_src WHERE id=9999;
SELECT state FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
  TASK_NAME=>'NATIVE_SRC_TASK', RESULT_LIMIT=>1));
```

**出力:**

```
+------+-----------+
| ID   | NAME      |
|------+-----------|
| 9999 | task_test |
+------+-----------+

+-----------+
| STATE     |
|-----------|
| SUCCEEDED |
+-----------+
```

**[7] Masking Policy / Row Access Policy**

Enterprise Edition以上が必要なため今回の検証環境では確認できませんでした。

**[8] Clustering Key**

```sql
SELECT SYSTEM$CLUSTERING_INFORMATION('native_src','(id)');
```

**出力:**

```json
{
  "cluster_by_keys" : "LINEAR(id)",
  "total_partition_count" : 4,
  "average_overlaps" : 0.0,
  "average_depth" : 1.0,
  ...
}
```

DDL作成時に `CLUSTER BY` を指定することでIcebergテーブルでも機能します。

**[9] DEFAULT / NOT NULL制約**

```sql
-- DEFAULT動作確認
INSERT INTO native_src (id, dept) SELECT 300, 'OPEN';
SELECT id, name, score FROM native_src WHERE id=300;
```

**出力:**

```
+-----+---------+-------+
| ID  | NAME    | SCORE |
|-----+---------+-------|
| 300 | unknown | 0.0   |
+-----+---------+-------+
```

```sql
-- NOT NULL確認
INSERT INTO native_src (id, name, dept) SELECT 301, 'x', NULL;
```

**出力（エラー）:**

```
100072 (22000): DML operation to table NATIVE_SRC failed on column DEPT
with error: NULL result in a non-nullable column
```

**[10] Tags**

IcebergテーブルにTagを設定・変更する際は `ALTER ICEBERG TABLE` 構文が必要です。`ALTER TABLE` を使うとエラーになります。

```sql
-- ALTER TABLEを使うとエラー
ALTER TABLE native_src SET TAG tbl_sensitivity = 'HIGH';
-- SQL compilation error: The table is an Iceberg table.
-- Iceberg tables should use ALTER ICEBERG TABLE commands.

-- 正しい構文
ALTER ICEBERG TABLE native_src SET TAG tbl_sensitivity = 'HIGH';
ALTER ICEBERG TABLE native_src ALTER COLUMN dept SET TAG col_department = 'HR_SYSTEM';
```

```sql
SELECT tag_name, tag_value, column_name
FROM TABLE(INFORMATION_SCHEMA.TAG_REFERENCES_ALL_COLUMNS(
  'VERIFY_ICEBERG_MIGRATE.test_schema.native_src', 'table'));
```

**出力:**

```
+-----------------+-----------+-------------+
| TAG_NAME        | TAG_VALUE | COLUMN_NAME |
|-----------------+-----------+-------------|
| COL_DEPARTMENT  | HR_SYSTEM | DEPT        |
| TBL_SENSITIVITY | HIGH      | ID          |
| TBL_SENSITIVITY | HIGH      | NAME        |
| TBL_SENSITIVITY | HIGH      | DEPT        |
| TBL_SENSITIVITY | HIGH      | SCORE       |
| TBL_SENSITIVITY | HIGH      | PAYLOAD     |
| TBL_SENSITIVITY | HIGH      | GEO_POINT   |
| TBL_SENSITIVITY | HIGH      | CREATED_AT  |
+-----------------+-----------+-------------+
```

**[11] Time Travel**

```sql
SELECT COUNT(1) AS cnt FROM native_src AT(OFFSET => -60);
```

**出力:**

```
+-----+
| CNT |
|-----|
| 7   |
+-----+
```

Snowflake Managed IcebergテーブルでもTime Travelが機能します。

**[12] VARIANT（Iceberg v3必須）**

Iceberg v2ではVARIANTはサポートされていません。DDL作成時に `ICEBERG_VERSION = 3` の指定が必要です。

```sql
SELECT id, payload:role::STRING AS role_val FROM native_src WHERE id=1;
```

**出力:**

```
+----+----------+
| ID | ROLE_VAL |
|----+----------|
| 1  | admin    |
+----+----------+
```

**[13] GEOGRAPHY（Iceberg v3必須）**

VARIANTと同様、Iceberg v2では非サポート。

```sql
SELECT id, ST_ASTEXT(geo_point) AS geo_wkt FROM native_src WHERE id IN (1, 3);
```

**出力:**

```
+----+--------------------------+
| ID | GEO_WKT                  |
|----+--------------------------|
| 1  | POINT(139.6917 35.6895)  |
| 3  | POINT(-122.4194 37.7749) |
+----+--------------------------+
```

```sql
SELECT ST_DISTANCE(
  (SELECT geo_point FROM native_src WHERE id=1),
  (SELECT geo_point FROM native_src WHERE id=3)
) AS dist_meters;
```

**出力:**

```
+--------------------+
| DIST_METERS        |
|--------------------|
| 8270727.1225744635 |
+--------------------+
```

**[14] TIMESTAMP_LTZ（型変換必要）**

`TIMESTAMP_TZ` はIcebergでサポートされていないため、DDL作成時に `TIMESTAMP_LTZ` に変換します。マイクロ秒精度も正しく保持されます。

```sql
INSERT INTO native_src (id, name, dept, created_at)
SELECT 400, 'TsTest', 'OPEN', '2024-01-15 12:34:56.789012'::TIMESTAMP_LTZ;

SELECT id, created_at FROM native_src WHERE id=400;
```

**出力:**

```
+-----+----------------------------------+
| ID  | CREATED_AT                       |
|-----+----------------------------------|
| 400 | 2024-01-15 12:34:56.789012+09:00 |
+-----+----------------------------------+
```

**[15] COMMENT**

```sql
SELECT table_name, comment FROM information_schema.tables
WHERE table_name='NATIVE_SRC';
```

**出力:**

```
+------------+---------------------------------------------+
| TABLE_NAME | COMMENT                                     |
|------------+---------------------------------------------|
| NATIVE_SRC | Iceberg migration verification source table |
+------------+---------------------------------------------+
```

---

## 検証結果まとめ

### 属性・機能の引き継ぎ状況

| # | 属性・機能 | 移行後の状態 | 必要な対処 |
|---|-----------|-----------|----------|
| 1 | 権限（GRANT） | ❌ 旧テーブルに残る | 新テーブルに再付与 |
| 2 | Future Grants（TABLE型） | ❌ Iceberg未適用 | `ON FUTURE ICEBERG TABLES` を別途設定 |
| 3 | View | ✅ 自動で新テーブル参照 | なし |
| 4 | Dynamic Table | ✅ 正常動作 | なし |
| 5 | Stream | ❌ 旧テーブルを追跡 | DROP & 再作成 |
| 6 | Task | ✅ 名前ベースで新テーブルに書き込み | なし |
| 7 | Masking Policy / Row Access Policy | — 未確認 | Enterprise Edition で要確認 |
| 8 | Clustering Key | ✅ DDL指定で機能 | DDL作成時に明示 |
| 9 | DEFAULT値 / NOT NULL制約 | ✅ DDL指定で機能 | DDL作成時に明示 |
| 10 | Tags | ✅ `ALTER ICEBERG TABLE` で設定可 | `ALTER ICEBERG TABLE` 構文を使用 |
| 11 | Time Travel | ✅ 機能する | なし |
| 12 | VARIANT | ✅ v3で対応 | `ICEBERG_VERSION=3` 指定 |
| 13 | GEOGRAPHY | ✅ v3で対応 | `ICEBERG_VERSION=3` 指定 |
| 14 | タイムスタンプ型 | ✅ 型変換後に機能 | TIMESTAMP_TZ → TIMESTAMP_LTZ |
| 15 | COMMENT | ✅ DDL指定で機能 | DDL作成時に明示 |

### データ型変換テーブル

今回の検証で明らかになったデータ型の変換ルールです。Snowflakeのドキュメント（[Supported data types for Iceberg tables](https://docs.snowflake.com/en/user-guide/tables-iceberg-data-types)）の記述と照合しています。

| ネイティブ型 | Iceberg用の型 | 変換が必要な理由 |
|------------|-------------|----------------|
| `VARCHAR(N)` (N < 16,777,216) | `STRING` | Icebergは固定長VARCHAR未対応。最大長（約16MiB）のみ有効 |
| `NUMBER`（precision/scale省略） | `NUMBER(P,S)` | Icebergではprecision/scaleの明示が必須 |
| `TIMESTAMP_TZ(N)` | `TIMESTAMP_LTZ` | Iceberg specでTIMESTAMP_TZ型は非サポート |
| `FLOAT` | `DOUBLE` | SnowflakeのIceberg実装ではDOUBLE（64bit）を使用 |
| `VARIANT` | `VARIANT` + `ICEBERG_VERSION=3` | Iceberg v2ではVARIANT未サポート。v3で対応 |
| `GEOGRAPHY` | `GEOGRAPHY` + `ICEBERG_VERSION=3` | Iceberg v2ではGEOGRAPHY未サポート。v3で対応 |

## 検証して気づいたこと

### `CREATE TABLE` はパラメーターで省略できるが、`ALTER` は `ICEBERG TABLE` が必要

`DEFAULT_METADATA_WRITE_FORMAT = 'ICEBERG'` を設定すると `CREATE TABLE` だけでIcebergテーブルが作成されます（`CREATE ICEBERG TABLE` と書く必要はありません）。一方、作成後のIcebergテーブルに対するDDL操作（RENAME・TAGの設定など）は `ALTER ICEBERG TABLE` を使わなければなりません。`ALTER TABLE` を使うとエラーになります。

```
SQL compilation error: The table is an Iceberg table.
Iceberg tables should use ALTER ICEBERG TABLE commands.
```

`ALTER TABLE` を実行してもテーブルがIceberg以外に変換されるわけではなく、コマンドがエラーで失敗するだけです。`CREATE` は通常の `TABLE` 構文でよくなった一方で、`ALTER` は依然として `ICEBERG TABLE` の指定が必要という非対称な動作になっています。

移行スクリプトを自動化する際は、`information_schema.tables` の `is_iceberg` カラムでIcebergかどうかを判定して `ALTER TABLE` と `ALTER ICEBERG TABLE` を使い分けてください。

### `GRANT ... ON FUTURE TABLES` はIcebergテーブルに適用されない

SnowflakeはIcebergテーブルを内部的に `ICEBERG_TABLE` 型として扱います。`GRANT ... ON FUTURE TABLES` で設定したFuture GrantはIcebergテーブルには適用されません。`GRANT ... ON FUTURE ICEBERG TABLES` を別途設定してください。

### Dynamic Table・Materialized ViewのデータはIceberg形式で保存されない

通常の `CREATE DYNAMIC TABLE` で作成したDynamic TableおよびMaterialized Viewは、Snowflakeのネイティブフォーマットで保存されます。Iceberg形式ではないため、Snowflake Open Catalog（旧Polaris）経由でSparkなどの外部エンジンからこれらオブジェクトのデータを直接参照することはできません。

本記事の検証でも `CREATE DYNAMIC TABLE` で作成した `native_src_dt` はRENAME後も正常にREFRESH・参照できますが、そのデータ自体はIceberg形式では保存されていません。外部エンジンから参照できるのは、あくまでIcebergテーブル（`native_src`）のデータのみです。

- **Dynamic Tableを外部エンジンから参照したい場合**: `CREATE DYNAMIC ICEBERG TABLE` を使う必要があります（[Dynamic Iceberg Tableドキュメント](https://docs.snowflake.com/en/user-guide/dynamic-tables-create-iceberg)）
- **Materialized View**: 現時点ではIceberg形式での保存は非対応です

## まとめ

SnowflakeネイティブテーブルをIcebergテーブルへ移行する際は、DDL+INSERT+RENAME SWAPの手順を推奨します。

### 移行後に対処が必要なもの

RENAME SWAPによる移行後、以下の3点は自動では引き継がれないため、個別に対処が必要です。

| 対処が必要な項目 | 理由 | 必要な作業 |
|--------------|------|-----------|
| 権限（GRANT） | 権限はテーブルオブジェクトに紐づくため、RENAMEで旧テーブルに残る | 新テーブルへ再付与 |
| Future Grants | `ON FUTURE TABLES` はIcebergテーブル（ICEBERG_TABLE型）に未適用 | `GRANT ... ON FUTURE ICEBERG TABLES` を追加設定 |
| Stream | StreamはオブジェクトIDで追跡するためRENAME後も旧テーブルを参照し続ける | DROP & 新テーブルに再作成 |

### それ以外は問題なし

対処が不要な属性・機能は以下のとおりです。

- **View / Dynamic Table / Task**: テーブル名で参照するためRENAME後も新テーブルに自動追従
- **Clustering Key / DEFAULT値 / NOT NULL / COMMENT**: DDL作成時に明示すれば引き継ぎ可能
- **Tags**: `ALTER ICEBERG TABLE` 構文で設定可能
- **Time Travel**: Snowflake Managed Icebergテーブルでも機能

なお、VARIANT・GEOGRAPHYを含むテーブルは `ICEBERG_VERSION = 3`（[Iceberg v3 GA](https://docs.snowflake.com/en/release-notes/2024/ui/2024-12-04)）の指定が必要です。型変換が必要なデータ型は [データ型変換テーブル](#データ型変換テーブル) を参照してください。

### 今回未検証の項目

Row Access PolicyおよびMasking Policyは、Enterprise Edition以上が必要なため今回の検証環境では確認できていません。移行後の挙動については別途ご確認ください。
