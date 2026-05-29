---
title: "Dynamic TableのADAPTIVEモード——複雑な増分処理もINCREMENTALに対応可能に！"
emoji: "🔄"
type: "tech"
topics: ["snowflake", "dynamictable", "dataengineering", "sql", "datapipeline"]
published: true
---

## この記事について

Snowflake Dynamic Tableに新しいrefreshモード`REFRESH_MODE = ADAPTIVE`がPublic Previewで追加されました。このモードは通常incremental refreshを使用し、バルクロード検出時にのみfull refreshへ自動切り替えします。本記事では、INCREMENTAL・FULL・ADAPTIVEの違いを整理し、SQLで実際の挙動を確認します。

公式リリースノート: [Snowflake Documentation](https://docs.snowflake.com/en/release-notes/2026/other/2026-05-26-dynamic-tables-adaptive-refresh-mode)

:::message
内容は記事作成時点のものです。
仕様は変更され得るため、最終的には最新の公式ドキュメントで確認ください。
:::

## 背景：なぜこの機能が必要か

Dynamic TableのINCREMENTAL refreshは差分のみを処理するため効率的です。しかし`INSERT OVERWRITE`や大量バッチインサートが発生すると、全行が変更扱いになりINCREMENTAL refreshがFULL refreshより大幅にコストが高くなる場面があります。従来は`FULL`か`INCREMENTAL`の二択しかなく、通常は小規模な差分更新が主体でバルクロードが不定期に混在するワークロードでは、どちらを選んでも最適化が難しい状況でした。

## 機能の概要

`REFRESH_MODE = ADAPTIVE`は、SnowflakeがDynamic Tableのリフレッシュコストをリフレッシュごとに評価し、最適な方式を自動選択するモードです。

| モード | 動作 |
|---|---|
| `INCREMENTAL` | 常に差分のみを処理する |
| `FULL` | 常に全件を再計算する |
| `ADAPTIVE` | 通常はincremental、バルクロード検出時はfull refreshへ自動切替 |
| `AUTO` | クエリ内容から**作成時に**INCREMENTALかFULLかを一度だけ決定する |

ADAPTIVEモードの動作フローは次のとおりです。

1. デフォルトでincremental refreshを実行する
2. 上流変更量がincremental refreshをfull refreshより大幅にコスト高と内部ヒューリスティクスが判断すると、Dynamic Tableを**再初期化**（full refresh相当）する
3. 再初期化後はincremental refreshを再開する

この切り替えはSnowflakeが自動で行うため、ユーザーがrefreshモードを手動変更する必要はありません。

## ハンズオン

**ステップ1: ソーステーブルを作成してデータを挿入する**

```sql
-- テスト用スキーマとテーブルを準備
CREATE DATABASE IF NOT EXISTS ADAPTIVE_TEST_DB;
CREATE SCHEMA IF NOT EXISTS ADAPTIVE_TEST_DB.DEMO;
USE SCHEMA ADAPTIVE_TEST_DB.DEMO;

CREATE OR REPLACE TABLE SOURCE_DATA (
    id       INT,
    value    NUMBER,
    category STRING
);

INSERT INTO SOURCE_DATA VALUES
    (1, 100, 'A'),
    (2, 200, 'B'),
    (3, 300, 'A');
```

実行結果:
```
3 Row(s) inserted.
```

**ステップ2: ADAPTIVEモードでDynamic Tableを作成する**

```sql
-- REFRESH_MODE = ADAPTIVE を指定してDynamic Tableを作成
-- COMPUTE_WH は実際のウェアハウス名に置き換えてください
CREATE OR REPLACE DYNAMIC TABLE DT_ADAPTIVE
    TARGET_LAG    = '1 minute'
    WAREHOUSE     = COMPUTE_WH
    REFRESH_MODE  = ADAPTIVE
AS
SELECT id, value, category FROM SOURCE_DATA;
```

実行結果:
```
Dynamic table DT_ADAPTIVE successfully created.
```

**ステップ3: SHOW DYNAMIC TABLESでrefresh_modeを確認する**

```sql
-- refresh_mode 列に ADAPTIVE が設定されているか確認
SHOW DYNAMIC TABLES LIKE 'DT_ADAPTIVE';
```

実行結果:
```
name        | refresh_mode | scheduling_state | ...
------------+--------------+------------------+----
DT_ADAPTIVE | ADAPTIVE     | RUNNING          | ...
```

`refresh_mode`列に`ADAPTIVE`が確認できます。

**ステップ4: データが正しく反映されているか確認する**

```sql
SELECT * FROM DT_ADAPTIVE;
```

実行結果:
```
ID | VALUE | CATEGORY
---+-------+---------
 1 |   100 | A
 2 |   200 | B
 3 |   300 | A
```

ソーステーブルの3行がDynamic Tableに正しく反映されています。

**ステップ5: ALTERでREFRESH_MODEを変更しようとする（制約の確認）**

```sql
-- 作成後にREFRESH_MODEをINCREMENTALへ変更しようとする
ALTER DYNAMIC TABLE DT_ADAPTIVE SET REFRESH_MODE = INCREMENTAL;
```

実行結果:
```
SQL compilation error: invalid property 'REFRESH_MODE' for alter dynamic table
```

`REFRESH_MODE`は作成後に`ALTER`で変更できません。モードを変えたい場合は`CREATE OR REPLACE DYNAMIC TABLE`での再作成が必要です。

**ステップ6: AUTOモードとの違いを確認する**

```sql
-- AUTO モードでDynamic Tableを作成してどのモードが選ばれるか確認
CREATE OR REPLACE DYNAMIC TABLE DT_AUTO
    TARGET_LAG    = '1 minute'
    WAREHOUSE     = COMPUTE_WH
    REFRESH_MODE  = AUTO
AS
SELECT id, value, category FROM SOURCE_DATA;

SHOW DYNAMIC TABLES LIKE 'DT_AUTO';
```

実行結果:
```
name    | refresh_mode | ...
--------+--------------+----
DT_AUTO | INCREMENTAL  | ...
```

単純な`SELECT`クエリに対してAUTOはINCREMENTALを選択しました。ADAPTIVEはAUTOの選択肢に含まれておらず、明示的に指定した場合のみ有効になります。

## 検証して気づいたこと

- **`REFRESH_MODE`はALTERでの変更が不可。** `ALTER DYNAMIC TABLE ... SET REFRESH_MODE = ADAPTIVE`を実行すると`invalid property`エラーが返されます。ADAPTIVEモードへの切り替えが必要な場合は`CREATE OR REPLACE DYNAMIC TABLE`での再作成が必要です。運用中のDynamic Tableを移行する際は、再作成による一時的なデータ不整合期間を考慮した移行計画が必要になります。

- **`REFRESH_MODE = AUTO`はADAPTIVEを選択しない。** 単純なSELECTクエリに対してAUTOはINCREMENTALを選択し、ADAPTIVEは選ばれませんでした。バルクロード混在ワークロードでADAPTIVEの恩恵を受けるには、AUTOに頼らず`REFRESH_MODE = ADAPTIVE`と明示的に指定する必要があります。

## まとめ

`REFRESH_MODE = ADAPTIVE`により、Dynamic Tableは通常incremental refreshを維持しながら、INSERT OVERWRITEなどのバルクロード発生時に自動でfull refreshへ切り替え、その後incremental refreshを再開します。不定期なバルクロードが混在するパイプラインを持つ場合は、ぜひADAPTIVEモードを試してみてください。