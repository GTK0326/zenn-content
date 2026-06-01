---
title: "テーブルにビュー機能が来た——SnowflakeのVirtual Columnsで売上・利益を計算列として直接定義する"
emoji: "🧮"
type: "tech"
topics: ["snowflake", "sql", "データ基盤", "データウェアハウス", "テーブル設計"]
published: true
---

## この記事について

Virtual Columns（仮想列）が Snowflake で一般提供（GA）になりました。この機能はテーブルにデータを物理保存せず、クエリ実行時に式から値を動的に計算する列を定義します。本記事では、CREATE TABLE・ALTER TABLE での基本構文から、連鎖した仮想列・関数サポート・制約の動作まで SQL で検証します。

公式リリースノート: [Snowflake Documentation](https://docs.snowflake.com/en/release-notes/2026/10_19#virtual-columns-general-availability)

:::message
内容は記事作成時点のものです。
仕様は変更され得るため、最終的には最新の公式ドキュメントで確認ください。
:::

## 背景：なぜこの機能が必要か

単価と販売数をロードしたテーブルがあるとします。そこから売上（単価×販売数）や利益を参照したい場合、これまでは選択肢がふたつしかありませんでした。

ひとつは物理列として保存する方法です。ETL 側で計算して書き込む必要があり、元の列が更新されたときに値がずれるリスクがあります。もうひとつはビューを作成する方法です。テーブルの上にビューを重ねれば計算列を定義できますが、管理オブジェクトが増え、下流のパイプラインやクエリがビューを向くよう変更が波及します。

Virtual Columns を使うと、テーブル定義の AS 句に式を書くだけで済みます。単価・販売数のテーブルに `revenue AS (unit_price * quantity)` を追加すれば、ビューなしで売上列が参照できます。ストレージも増えません。

## 機能の概要

Virtual Columns は列定義に `AS ( <expr> )` 句を追加することで定義します。式にはリテラル・演算子・決定論的なシステム定義関数が使えます。同一テーブル内の他の列を参照でき、先に定義された仮想列を参照することも可能です。列型は省略可能で、省略した場合は Snowflake が自動推論します。

CREATE TABLE と ALTER TABLE の両方で定義できます。

```sql
-- テーブル作成時に定義
CREATE OR REPLACE TABLE sales (
  unit_price INT,
  quantity   INT,
  revenue    INT AS (unit_price * quantity)
);

-- 既存テーブルへの追加
ALTER TABLE sales ADD COLUMN profit INT AS ((unit_price - cost_price) * quantity);
```

SELECT 時に式が評価されるため、物理列が変われば仮想列も自動的に最新値になります。

## 検証環境

| 項目 | 内容 |
|------|------|
| エディション | Enterprise |
| 実施ロール | ACCOUNTADMIN |
| 確認日 | 2026-05-30 |
| 参照 | [Virtual Columns — Snowflake Documentation](https://docs.snowflake.com/en/user-guide/virtual-columns) |

## ハンズオン

単価・販売数・原価だけを持つ販売テーブルをサンプルとして使います。売上・利益・利益率をビューなしでテーブルに直接定義します。

### ステップ1: 仮想列を持つテーブルを作成してデータを確認する

```sql
-- 売上・利益を仮想列として定義
CREATE OR REPLACE TABLE sales (
  product_name VARCHAR,
  unit_price   INT,
  quantity     INT,
  cost_price   INT,
  revenue      INT AS (unit_price * quantity),
  profit       INT AS ((unit_price - cost_price) * quantity)
);

-- 利益率を後から追加
ALTER TABLE sales ADD COLUMN profit_margin FLOAT AS (ROUND((unit_price - cost_price) / unit_price * 100, 1));

-- 物理列にのみ値を挿入する
INSERT INTO sales (product_name, unit_price, quantity, cost_price)
VALUES
  ('商品A', 1500, 10,  800),
  ('商品B', 3000,  4, 1200),
  ('商品C',  800, 25,  300);

SELECT product_name, unit_price, quantity, revenue, profit, profit_margin FROM sales;
```

実行結果:
```
+--------------+------------+----------+---------+--------+---------------+
| PRODUCT_NAME | UNIT_PRICE | QUANTITY | REVENUE | PROFIT | PROFIT_MARGIN |
+--------------+------------+----------+---------+--------+---------------+
| 商品A         |       1500 |       10 |   15000 |   7000 |          46.7 |
| 商品B         |       3000 |        4 |   12000 |   7200 |          60.0 |
| 商品C         |        800 |       25 |   20000 |  12500 |          62.5 |
+--------------+------------+----------+---------+--------+---------------+
```

INSERT では物理列にのみ値を渡しています。`revenue`・`profit`・`profit_margin` はすべて式から自動計算されています。

### ステップ2: DESCRIBE TABLE で仮想列のメタデータを確認する

```sql
DESCRIBE TABLE sales;
```

実行結果:
```
+---------------+---------------+---------+-------+------------------------------------------+
| name          | type          | kind    | null? | expression                               |
+---------------+---------------+---------+-------+------------------------------------------+
| PRODUCT_NAME  | VARCHAR       | COLUMN  | Y     |                                          |
| UNIT_PRICE    | NUMBER(38,0)  | COLUMN  | Y     |                                          |
| QUANTITY      | NUMBER(38,0)  | COLUMN  | Y     |                                          |
| COST_PRICE    | NUMBER(38,0)  | COLUMN  | Y     |                                          |
| REVENUE       | NUMBER(38,0)  | VIRTUAL | Y     | UNIT_PRICE * QUANTITY                    |
| PROFIT        | NUMBER(38,0)  | VIRTUAL | Y     | (UNIT_PRICE - COST_PRICE) * QUANTITY     |
| PROFIT_MARGIN | FLOAT         | VIRTUAL | Y     | ROUND((...) / UNIT_PRICE * 100, 1)       |
+---------------+---------------+---------+-------+------------------------------------------+
```

`REVENUE`・`PROFIT`・`PROFIT_MARGIN` の kind が `VIRTUAL`、expression 列に定義した式が表示されています。物理列は `COLUMN` です。

### ステップ3: 仮想列を別の仮想列から連鎖参照する

先に定義した仮想列を後の仮想列で参照できます。税込み金額を小計から算出する例です。

```sql
CREATE OR REPLACE TABLE order_summary (
  unit_price     INT,
  quantity       INT,
  subtotal       INT   AS (unit_price * quantity),
  total_with_tax FLOAT AS (subtotal * 1.1)   -- 仮想列 subtotal を参照
);

INSERT INTO order_summary (unit_price, quantity) VALUES (1000, 3);

SELECT unit_price, quantity, subtotal, total_with_tax FROM order_summary;
```

実行結果:
```
+------------+----------+----------+----------------+
| UNIT_PRICE | QUANTITY | SUBTOTAL | TOTAL_WITH_TAX |
+------------+----------+----------+----------------+
|       1000 |        3 |     3000 |         3300.0 |
+------------+----------+----------+----------------+
```

`subtotal = 1000×3 = 3000`、`total_with_tax = 3000×1.1 = 3300.0` と連鎖した計算が正しく行われています。

### ステップ4: 文字列関数・型変換関数を仮想列で使う

```sql
CREATE OR REPLACE TABLE product_report (
  product_name  VARCHAR,
  unit_price    INT,
  quantity      INT,
  revenue       INT     AS (unit_price * quantity),
  revenue_label VARCHAR AS (CONCAT(product_name, ' の売上: ¥', TO_VARCHAR(unit_price * quantity)))
);

INSERT INTO product_report (product_name, unit_price, quantity)
VALUES ('商品A', 1500, 10);

SELECT product_name, revenue, revenue_label FROM product_report;
```

実行結果:
```
+--------------+---------+-------------------------+
| PRODUCT_NAME | REVENUE | REVENUE_LABEL           |
+--------------+---------+-------------------------+
| 商品A         |   15000 | 商品A の売上: ¥15000     |
+--------------+---------+-------------------------+
```

CONCAT・TO_VARCHAR などの決定論的関数を仮想列の式で使えます。

### ステップ5: 仮想列に対する制約を確認する（失敗ケース）

非決定論的関数（RANDOM など）は使用できません。

```sql
CREATE OR REPLACE TABLE test_nd (
  id       INT,
  rand_val FLOAT AS (RANDOM())
);
```

実行結果:
```
SQL compilation error: Virtual column expression contains a non-deterministic function.
```

仮想列への直接書き込みも拒否されます。

```sql
-- 仮想列 revenue に値を直接挿入しようとする
INSERT INTO sales (product_name, unit_price, quantity, cost_price, revenue)
VALUES ('商品D', 2000, 5, 1000, 99999);
```

実行結果:
```
SQL compilation error: Virtual column 'REVENUE' cannot be set in an insert or update statement.
```

どちらも SQL コンパイル時にエラーとなります。仮想列の値は常に式から自動計算されます。


## 検証して気づいたこと

- **制約違反はすべて SQL コンパイル時に検出される。** 非決定論的関数の使用・直接書き込みのどちらも、実行前の構文解析段階でエラーになります。誤った定義が実データに影響を与える前に止まるため、開発中の誤操作リスクが低くなります。


## 検証コード

この記事のハンズオンで使用した SQL を Jupyter Notebook（.ipynb）形式で公開しています。

Snowflake Notebooks にインポートして、そのまま自分の環境で実行できます。

[📓 検証ノートブックを開く（GitHub）](https://github.com/GTK0326/zenn-content/blob/main/notebooks/i68-virtual-columns-general-availability.ipynb)
## まとめ

Virtual Columns により、売上・利益・利益率といった派生値をビューなしでテーブルに直接持てるようになりました。ロードしたテーブルの上にビューを重ねる手間がなくなり、管理オブジェクトも増えません。AS 句をテーブル定義に加えるだけなので、まず手元のテーブルで試してみてください。