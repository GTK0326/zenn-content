---
title: "Dynamic TableでMERGEロジックを自分で書く——CUSTOM_INCREMENTALが拓く複雑変換の新境地"
emoji: "🔄"
type: "tech"
topics: ["snowflake", "dynamictable", "sql", "dataengineering", "etl"]
published: true
---

## この記事について

2026年5月26日、Snowflake に `CUSTOM_INCREMENTAL` リフレッシュモードが Public Preview として追加されました。このモードを利用すると、Dynamic Table のリフレッシュ時に実行される MERGE または INSERT ロジックをユーザー自身が定義できます。この記事では、stream-static join・ソフトデリート・ステートフル集計の代表的なパターンを SQL で実装し、実際の動作を確認します。

公式リリースノート: [Snowflake Documentation](https://docs.snowflake.com/en/release-notes/2026/other/2026-05-26-dynamic-tables-custom-incremental)

:::message
内容は記事作成時点のものです。
仕様は変更され得るため、最終的には最新の公式ドキュメントで確認ください。
:::

![](/images/i47-feature-update-2026-05-26-custom-incremental/cover.png)

## 背景：なぜこの機能が必要か

Snowflake の Dynamic Table は依存データの変化を検知して自動的にリフレッシュされます。リフレッシュには `INCREMENTAL`（差分処理）と `FULL`（全件再計算）の2モードがありましたが、どちらにも制約がありました。

`INCREMENTAL` はシンプルなフィルタリングや射影を差分処理できますが、ステートフルな集計やソフトデリートのような複雑なパターンは処理できず、`FULL` に自動フォールバックします。`FULL` はどんなクエリも処理できる反面、リフレッシュのたびにテーブル全件をスキャンするため、テーブルが大きくなるほどコストと遅延が増大します。

また、標準の Dynamic Table ではストリームをソースとして参照すること自体ができず、stream-static join のような処理は Dynamic Table では表現できませんでした。これらの処理は stream と task の手動パイプラインで補う必要があり、スケジューリング・リトライ・トランザクション保証をすべて自前で管理しなければなりませんでした。

## 機能の概要

`CUSTOM_INCREMENTAL` モードでは、Dynamic Table の定義に `REFRESH USING` 句を追加します。この句の中に `INSERT INTO SELF` または `MERGE INTO SELF` の SQL を記述します。

```sql
CREATE OR REPLACE DYNAMIC TABLE my_dt
  TARGET_LAG = '1 minute'
  WAREHOUSE = my_wh
  REFRESH_MODE = CUSTOM_INCREMENTAL   -- または AUTO でも可
  INITIALIZE = ON_CREATE
AS
  SELECT ...   -- フルロード用クエリ
REFRESH USING (
  MERGE INTO SELF AS target
  USING ( SELECT ... FROM my_stream ... ) AS src
  ON target.id = src.id
  WHEN MATCHED THEN UPDATE SET ...
  WHEN NOT MATCHED THEN INSERT (...) VALUES (...)
);
```

| 句 | 役割 |
|---|---|
| `AS SELECT ...` | 初回フルロードまたはフルリフレッシュ時のクエリ |
| `REFRESH USING (...)` | 増分リフレッシュ時に実行するカスタムロジック |
| `INSERT INTO SELF` | 追記型パターン（stream-static join など） |
| `MERGE INTO SELF` | 更新・削除を伴うパターン（ソフトデリートなど） |
| `BACKFILL FROM <table>` | 既存テーブルのスナップショットを初期データとして利用する |

`INITIALIZE` には `ON_CREATE`（作成時即実行）と `ON_SCHEDULE`（次回スケジュールまで待機）を指定できます。スケジューリング・リトライ・トランザクション保証は Snowflake が自動管理するため、ユーザーはロジックの記述に集中できます。

## ハンズオン

`CUSTOM_INCREMENTAL` を理解する前提として、まず `REFRESH_MODE = AUTO` の挙動を確認します。AUTO モードでは Snowflake がクエリを解析し、`FULL`・`INCREMENTAL`・`CUSTOM_INCREMENTAL` の中から最適なモードを自動で選択します。ハンズオンでは3種類のクエリパターンを持つ Dynamic Table を同じ `AUTO` 設定で作成し、それぞれが異なるモードに解決される様子を `SHOW DYNAMIC TABLES` で確認します。

---

### ステップ1: スキーマとソーステーブルを準備する

```sql
CREATE OR REPLACE DATABASE dt_custom_demo;
CREATE OR REPLACE SCHEMA dt_custom_demo.main;
USE SCHEMA dt_custom_demo.main;

-- 売上ログ（ソーステーブル）
CREATE OR REPLACE TABLE sales_log (
  sale_id  INT,
  user_id  INT,
  amount   NUMBER(10,2),
  sold_at  TIMESTAMP
);

-- sales_log 用ストリーム（CUSTOM_INCREMENTAL で使用）
CREATE OR REPLACE STREAM sales_log_stream ON TABLE sales_log;

INSERT INTO sales_log VALUES
  (1, 1, 1000.00, CURRENT_TIMESTAMP()),
  (2, 2,  500.00, CURRENT_TIMESTAMP()),
  (3, 1, 2500.00, CURRENT_TIMESTAMP()),
  (4, 3,  800.00, CURRENT_TIMESTAMP());
```

---

### ステップ2: 集計クエリの Dynamic Table を作成する（→ FULL に解決）

`SUM` + `GROUP BY` のような集計クエリは、差分行だけを処理して正しい集計値を維持することができません。そのため Snowflake は `FULL` を選択し、リフレッシュのたびに全件を再計算します。

```sql
CREATE OR REPLACE DYNAMIC TABLE user_sales_summary
  TARGET_LAG = '1 minute'
  WAREHOUSE = compute_wh
  REFRESH_MODE = AUTO
AS
  SELECT
    user_id,
    SUM(amount)  AS total_amount,
    COUNT(*)     AS sale_count
  FROM sales_log
  GROUP BY user_id;
```

---

### ステップ3: フィルタークエリの Dynamic Table を作成する（→ INCREMENTAL に解決）

`WHERE` のみのシンプルなフィルタークエリは、ソーステーブルの差分行に同じ条件を適用するだけで結果を維持できます。そのため Snowflake は `INCREMENTAL` を選択し、変更行のみを処理します。

```sql
CREATE OR REPLACE DYNAMIC TABLE large_sales
  TARGET_LAG = '1 minute'
  WAREHOUSE = compute_wh
  REFRESH_MODE = AUTO
AS
  SELECT sale_id, user_id, amount, sold_at
  FROM sales_log
  WHERE amount >= 1000;
```

---

### ステップ4: REFRESH USING を持つ Dynamic Table を作成する（→ CUSTOM_INCREMENTAL に解決）

`REFRESH USING` 句を追加すると、AUTO モードはこれをユーザーが増分ロジックを明示的に定義したと判断し、`CUSTOM_INCREMENTAL` を選択します。

```sql
CREATE OR REPLACE DYNAMIC TABLE enriched_sales
  TARGET_LAG = '1 minute'
  WAREHOUSE = compute_wh
  REFRESH_MODE = AUTO      -- REFRESH USING があれば CUSTOM_INCREMENTAL に自動解決
  INITIALIZE = ON_CREATE
AS
  SELECT sale_id, user_id, amount, sold_at
  FROM sales_log
REFRESH USING (
  INSERT INTO SELF
    SELECT sale_id, user_id, amount, sold_at
    FROM sales_log_stream
    WHERE METADATA$ACTION = 'INSERT'
);
```

---

### ステップ5: SHOW DYNAMIC TABLES で3モードを一覧確認する

```sql
SHOW DYNAMIC TABLES IN SCHEMA dt_custom_demo.main;
```

実行結果：
```
name               | refresh_mode       | scheduling_state
-------------------|--------------------|------------------
USER_SALES_SUMMARY | FULL               | RUNNING
LARGE_SALES        | INCREMENTAL        | RUNNING
ENRICHED_SALES     | CUSTOM_INCREMENTAL | RUNNING
```

いずれも `REFRESH_MODE = AUTO` で作成しましたが、クエリ構造に応じて異なるモードに解決されています。

| テーブル | クエリの特徴 | 解決されたモード |
|---|---|---|
| `USER_SALES_SUMMARY` | SUM + GROUP BY（集計） | `FULL` |
| `LARGE_SALES` | WHERE のみ（フィルター） | `INCREMENTAL` |
| `ENRICHED_SALES` | REFRESH USING（カスタム増分） | `CUSTOM_INCREMENTAL` |

これが `CUSTOM_INCREMENTAL` 追加によるビフォー・アフターの本質です。従来は `FULL` か `INCREMENTAL` の2択しかなく、stream-static join や ソフトデリートのような処理は Dynamic Table で表現できませんでした。`CUSTOM_INCREMENTAL` が加わったことで、Snowflake はクエリの性質に応じて3つのモードから最適解を選べるようになりました。

---

### ステップ6: 新データを追加してインクリメンタルリフレッシュを確認する

```sql
-- 2件追加
INSERT INTO sales_log VALUES
  (5, 2, 1500.00, CURRENT_TIMESTAMP()),
  (6, 3,  300.00, CURRENT_TIMESTAMP());

ALTER DYNAMIC TABLE enriched_sales REFRESH;

-- amount >= 1000 の行のみが存在するか確認
SELECT * FROM large_sales ORDER BY sale_id;
```

実行結果：
```
sale_id | user_id | amount  | sold_at
--------|---------|---------|---------------------
1       | 1       | 1000.00 | 2026-05-27 10:00:00
3       | 1       | 2500.00 | 2026-05-27 10:00:00
5       | 2       | 1500.00 | 2026-05-27 10:05:00
```

`large_sales`（INCREMENTAL）は追加された2件のうち `amount >= 1000` を満たす sale_id=5 のみを差分処理で取り込み、sale_id=6（300.00）は除外されています。

## 検証して気づいたこと

**stream の UPDATE は DELETE + INSERT ペアで展開されるため、MERGE INTO SELF では重複排除が必須**

`MERGE INTO SELF` の USING 句で stream を参照すると、Snowflake は UPDATE を DELETE + INSERT の2行に展開して返します。同一キーに対して複数のソース行が存在するため、重複排除を行わずに実行すると「MERGE statement with multiple source rows matching the same target row」エラーが発生します。

重複排除の代表的な手法は `QUALIFY ROW_NUMBER() OVER (PARTITION BY key ORDER BY METADATA$ROW_ID DESC) = 1` で DELETE+INSERT ペアを最新の1行に絞ることです。ただしこれは複数ある手法の一つであり、`METADATA$ISUPDATE` を条件として INSERT 側・DELETE 側を明示的に振り分ける方法など他のアプローチも存在します。いずれの方法でも、USING 句に同一キーの重複行が含まれないことを保証することが本質です。標準の `INCREMENTAL` モードでは Snowflake がこの deduplication を自動処理しますが、`CUSTOM_INCREMENTAL` ではユーザーが自前で実装する必要があります。

**REFRESH_MODE = AUTO のまま REFRESH USING を指定すると CUSTOM_INCREMENTAL に自動解決される理由**

Snowflake は Dynamic Table の DDL 解析時に `REFRESH USING` 句の有無を確認します。`REFRESH USING` 内に `INSERT INTO SELF` または `MERGE INTO SELF` が存在する場合、Snowflake はそれを「ユーザーが増分ロジックを明示的に定義している」と判定し、自動的に `CUSTOM_INCREMENTAL` を割り当てます。`REFRESH_MODE = CUSTOM_INCREMENTAL` と明示指定した場合と同じ結果になりますが、`AUTO` のまま `REFRESH USING` を追加する方法は既存の stream + task 構成から Dynamic Table へ段階的に移行する際の足がかりとして有効です。

## 検証コード

この記事のハンズオンで使用した SQL を Jupyter Notebook（.ipynb）形式で公開しています。

Snowflake Notebooks にインポートして、そのまま自分の環境で実行できます。

[📓 検証ノートブックを開く（GitHub）](https://github.com/GTK0326/zenn-content/blob/main/notebooks/i47-feature-update-2026-05-26-custom-incremental.ipynb)

## まとめ

`CUSTOM_INCREMENTAL` モードにより、`FULL` へのフォールバックや stream + task 手動パイプラインの2択だった状況に第三の選択肢が生まれました。stream-static join・ソフトデリート・ステートフル集計を、フルスキャンなしで・スケジューリングやリトライを Snowflake に任せながら Dynamic Table として実装できます。stream + task 構成からの移行先としても有力な選択肢ですので、ぜひ Public Preview 中に試してみてください。