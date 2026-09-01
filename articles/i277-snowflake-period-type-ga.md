---
title: "Snowflakeに期間型PERIODが登場——Teradataの期間列を2カラムに割らずに移せる"
emoji: "📅"
type: "tech"
topics: ["snowflake", "sql", "datamodeling", "teradata", "migration"]
published: true
---

## この記事で分かること

Snowflake に、開始と終了を1つの値として持つ PERIOD 型が追加されました。
予約や稼働時間のように期間そのものを比べるテーブルを扱う方、Teradata から Snowflake への移行を検討している方に向けた記事です。
実機で動かした結果と、そこで気づいた制約をまとめます。

公式リリースノート: [Snowflake Documentation](https://docs.snowflake.com/en/release-notes/2026/10_30)

:::message
内容は記事作成時点のものです。
仕様は変更され得るため、最終的には最新の公式ドキュメントで確認ください。
:::

![](/images/i277-snowflake-period-type-ga/cover.png)

## 期間を2カラムで持つ設計はどこでつまずくか

会議室の予約テーブルを例にします。
予約は開始と終了を持つので、`START_TIME` と `END_TIME` の2カラムで表すのが普通です。

```sql
CREATE TABLE RESERVATION (
  ROOM_ID    STRING,
  BOOKED_BY  STRING,
  START_TIME TIMESTAMP_NTZ,
  END_TIME   TIMESTAMP_NTZ
);
```

この形でつまずくのは、次の2つの場面です。

- **二重予約を防ぐとき**
  これから入れたい時間帯が、既存の予約と重なっていないか調べます。
- **重なった時間を取り出すとき**
  設備点検の時間帯と、各予約がどれだけ重なるかを出します。

どちらも、片方の時点ではなく期間どうしを比べます。

### 二重予約を防ぐ

「9月10日の14時から16時で会議室Aを押さえたい。すでに埋まっていないか確かめたい」という要件です。

2カラム設計では、条件が交差します。

```sql
SELECT BOOKED_BY
FROM RESERVATION
WHERE ROOM_ID    = 'A'
  AND START_TIME < '2026-09-10 16:00:00'
  AND END_TIME   > '2026-09-10 14:00:00';
```

既存予約の開始が新しい予約の終了より前で、かつ既存予約の終了が新しい予約の開始より後。
この組み合わせを毎回書き下します。

不等号を `<=` にすると、14時ちょうどに終わる予約が「重なっている」ことになります。
会議室を連続して使う予約が取れなくなり、しかも境界の時刻に予約が入ったときしか再現しません。

### 重なった時間を取り出す

「設備点検を14時30分から16時30分に入れる。各予約が点検時間とどれだけ重なるかを利用者に知らせたい」という要件です。

重なりの判定に加えて、重なった範囲そのものを切り出します。

```sql
SELECT BOOKED_BY,
       GREATEST(START_TIME, '2026-09-10 14:30:00'::TIMESTAMP_NTZ) AS OVERLAP_FROM,
       LEAST(END_TIME,      '2026-09-10 16:30:00'::TIMESTAMP_NTZ) AS OVERLAP_TO
FROM RESERVATION
WHERE START_TIME < '2026-09-10 16:30:00'
  AND END_TIME   > '2026-09-10 14:30:00';
```

`GREATEST` と `LEAST` を手で組み合わせます。
WHERE 句の重なり判定と、SELECT 句の切り出しで、同じ境界の値を2回書くことになります。

## PERIOD 型がもたらす2つのこと

型が入ることで変わるのは、次の2点です。

- **期間が1つの値になる**
  境界の扱いが型の側で固定されます。
- **移行で型を作り替えなくてよくなる**
  Teradata の Period 列を、そのままの形で持てます。

### 期間が1つの値になる

PERIOD 値は、開始境界を含み終了境界を含まない半開区間 `[begin, end)` を1つの値として格納します。
要素型には DATE、TIME、TIMESTAMP を使えます。

「終了時刻を含むかどうか」という設計判断が、型の側で固定されます。

関数は14個あります。この記事で使う代表的なものは次のとおりです。

| 関数 | 何をするか |
|---|---|
| `PERIOD_CONSTRUCT` | 開始と終了から期間を作る |
| `PERIOD_BEGIN` / `PERIOD_END` | 期間から境界を取り出す |
| `PERIOD_OVERLAPS` | 2つの期間が重なるかを判定する |
| `PERIOD_INTERSECT` | 2つの期間の重なった部分を返す |

残りは `PERIOD_CONTAINS` や `PERIOD_MEETS` のような判定と、`PERIOD_LDIFF` / `PERIOD_RDIFF` のような差分です。
`LDIFF` と `RDIFF` という名前は Teradata の同名関数に対応します。

### 移行で型を作り替えなくてよくなる

Teradata には以前から `PERIOD(DATE)` などの型があります。
これまでは Snowflake 側に対応する型が無く、移行のたびに作り替えが必要でした。

Snowflake の[移行ドキュメント](https://docs.snowflake.com/en/migrations/snowconvert-docs/translation-references/teradata/sql-translation-reference/data-types)には、Teradata の Period は Snowflake では VARCHAR として格納すること、テーブルの Period 型の列宣言は同じ型の2カラムに分割することが書かれています。
値の構築関数も、開始側と終了側の2つに分けられます。

つまりテーブル定義もクエリもアプリケーション側のマッピングも、まとめて書き直すことになっていました。

ここからは、PERIOD の関数が実際にどう動くかを確かめます。

## 実際に動かしてみよう

:::details セットアップ

```sql
CREATE DATABASE IF NOT EXISTS PERIOD_DEMO_DB;
USE DATABASE PERIOD_DEMO_DB;
CREATE SCHEMA IF NOT EXISTS S1;
USE SCHEMA S1;
```

:::

### ステップ1: PERIOD 値を作る

3つの要素型それぞれで値を作ります。

```sql
SELECT
  PERIOD_CONSTRUCT('2026-01-01'::DATE, '2026-02-01'::DATE) AS P_DATE,
  PERIOD_CONSTRUCT('09:00:00'::TIME, '18:00:00'::TIME)     AS P_TIME,
  PERIOD_CONSTRUCT('2026-01-01 00:00:00'::TIMESTAMP_NTZ,
                   '2026-01-02 00:00:00'::TIMESTAMP_NTZ)   AS P_TS;
```

実行結果:
```
P_DATE                   | P_TIME                                   | P_TS
[2026-01-01, 2026-02-01) | [09:00:00.000000000, 18:00:00.000000000) | [2026-01-01 00:00:00.000000000, 2026-01-02 00:00:00.000000000)
```

終了側が丸括弧で表示されます。
含まないことが値を見ただけで分かります。

### ステップ2: 列として持つ

会議室の予約を2件入れます。
13時から15時と、15時から17時の連続した予約です。

```sql
CREATE OR REPLACE TABLE RESERVATION (
  ROOM_ID   STRING,
  BOOKED_BY STRING,
  STAY      PERIOD(TIMESTAMP_NTZ)
);

INSERT INTO RESERVATION
SELECT 'A', 'sato',
       PERIOD_CONSTRUCT('2026-09-10 13:00:00'::TIMESTAMP_NTZ,
                        '2026-09-10 15:00:00'::TIMESTAMP_NTZ);

INSERT INTO RESERVATION
SELECT 'A', 'suzuki',
       PERIOD_CONSTRUCT('2026-09-10 15:00:00'::TIMESTAMP_NTZ,
                        '2026-09-10 17:00:00'::TIMESTAMP_NTZ);

SELECT ROOM_ID, BOOKED_BY, STAY,
       PERIOD_BEGIN(STAY) AS B, PERIOD_END(STAY) AS E
FROM RESERVATION ORDER BY BOOKED_BY;
```

実行結果:
```
ROOM_ID | BOOKED_BY | STAY                                               | B                   | E
A       | sato      | [2026-09-10 13:00:00.000, 2026-09-10 15:00:00.000) | 2026-09-10 13:00:00 | 2026-09-10 15:00:00
A       | suzuki    | [2026-09-10 15:00:00.000, 2026-09-10 17:00:00.000) | 2026-09-10 15:00:00 | 2026-09-10 17:00:00
```

`PERIOD(TIMESTAMP_NTZ)` と書けば列になります。
格納したあとも `PERIOD_BEGIN` / `PERIOD_END` で境界を取り出せるので、開始時刻だけを使う既存のクエリも書き直せます。

### ステップ3: 二重予約を判定する

14時から16時で予約を入れたい、という条件で調べます。

```sql
SELECT BOOKED_BY
FROM RESERVATION
WHERE ROOM_ID = 'A'
  AND PERIOD_OVERLAPS(STAY,
        PERIOD_CONSTRUCT('2026-09-10 14:00:00'::TIMESTAMP_NTZ,
                         '2026-09-10 16:00:00'::TIMESTAMP_NTZ));
```

実行結果:
```
BOOKED_BY
sato
suzuki
```

2件とも重なります。
課題章で書いた交差する2つの不等号が、`PERIOD_OVERLAPS` 1つになりました。

### ステップ4: 境界の扱いを確認する

17時から19時で予約を入れたい、という条件です。
17時は suzuki の予約が終わる時刻そのものです。

```sql
SELECT BOOKED_BY
FROM RESERVATION
WHERE ROOM_ID = 'A'
  AND PERIOD_OVERLAPS(STAY,
        PERIOD_CONSTRUCT('2026-09-10 17:00:00'::TIMESTAMP_NTZ,
                         '2026-09-10 19:00:00'::TIMESTAMP_NTZ));
```

実行結果:
```
（0件）
```

終了境界を含まないため、接している予約は重なりと判定されません。
連続して会議室を使う予約が、そのまま取れます。

### ステップ5: 重なった時間を取り出す

設備点検を14時30分から16時30分に入れる場合に、各予約と重なる範囲を出します。

```sql
SELECT BOOKED_BY,
       PERIOD_INTERSECT(STAY,
         PERIOD_CONSTRUCT('2026-09-10 14:30:00'::TIMESTAMP_NTZ,
                          '2026-09-10 16:30:00'::TIMESTAMP_NTZ)) AS OVERLAP
FROM RESERVATION ORDER BY BOOKED_BY;
```

実行結果:
```
BOOKED_BY | OVERLAP
sato      | [2026-09-10 14:30:00.000, 2026-09-10 15:00:00.000)
suzuki    | [2026-09-10 15:00:00.000, 2026-09-10 16:30:00.000)
```

`GREATEST` と `LEAST` の組み合わせが `PERIOD_INTERSECT` 1つになりました。
境界の値を2回書く必要もありません。

## 実際に動かして気づいたこと

リリースノートに書かれていない挙動が3つありました。

### MIN / MAX が使えない

集約関数の `MIN` と `MAX` は PERIOD を受け付けません。

```sql
SELECT MIN(STAY) FROM RESERVATION;
```

実行結果:
```
002016 (22000): SQL compilation error:
Function MIN does not support PERIOD(TIMESTAMP_NTZ(9)) argument type
```

最も早い予約を取りたい場合は `MIN(PERIOD_BEGIN(STAY))` と書きます。

`ORDER BY STAY`、`GROUP BY STAY`、`COUNT(DISTINCT STAY)`、`JOIN ... ON a.STAY = b.STAY` はいずれも動きます。
使えないのは `MIN` と `MAX` だけです。

### クラスタリングキーに指定できない

PERIOD 列はクラスタリングキーになりません。

```sql
ALTER TABLE RESERVATION CLUSTER BY (STAY);
```

実行結果:
```
003300 (42601): Unsupported type 'PERIOD(TIMESTAMP_NTZ(9))' for clustering keys
```

`PERIOD_BEGIN(STAY)` のように式にすれば `ALTER TABLE` 自体は通ります。
ただし1000万行のテーブルで20分観測した範囲では、自動クラスタリングの再クラスタ行数は0のままでした。

### プルーニングは効く

クラスタリングキーにできないなら、マイクロパーティションのプルーニングも効かないのではないか。
そう考えて確かめました。

開始時刻順に並べた1000万行の PERIOD 列テーブルに、重なり判定を投げます。

```sql
SELECT COUNT(*) FROM T_PERIOD
WHERE PERIOD_OVERLAPS(STAY, PERIOD_CONSTRUCT('2026-06-01'::DATE,'2026-06-08'::DATE));

SELECT OPERATOR_TYPE,
       OPERATOR_STATISTICS:pruning:partitions_scanned::NUMBER AS SCANNED,
       OPERATOR_STATISTICS:pruning:partitions_total::NUMBER   AS TOTAL
FROM TABLE(GET_QUERY_OPERATOR_STATS(LAST_QUERY_ID()))
WHERE OPERATOR_TYPE = 'TableScan';
```

実行結果:
```
OPERATOR_TYPE | SCANNED | TOTAL
TableScan     | 2       | 64
```

64個のうち2個しか読んでいません。
同じデータをランダム順で格納し直すと64個中64個になったので、並び順に応じてプルーニングが働いていることが分かります。

## 検証コード

この記事のハンズオンで使用した SQL を Jupyter Notebook（.ipynb）形式で公開しています。

Snowflake Notebooks にインポートして、そのまま自分の環境で実行できます。

[📓 検証ノートブックを開く（GitHub）](https://github.com/GTK0326/zenn-content/blob/main/notebooks/i277-snowflake-period-type-ga.ipynb)

## まとめ

PERIOD 型は、期間を1つの値として扱えるようにするものです。
半開区間が型に入るため、重なりの判定で境界を選び分ける必要がなくなります。

Teradata からの移行では、Period 列を VARCHAR や2カラムに割り直す作業が不要になります。
比較の関数も一通り揃っているので、移行元で書いていた判定はそのまま置き換えられます。
`MIN` / `MAX` とクラスタリングキーの制約だけは、先に確かめておくと安全です。

## 参考リンク

- [10.30 Release Notes](https://docs.snowflake.com/en/release-notes/2026/10_30)
- [SnowConvert AI - Teradata - Data Types](https://docs.snowflake.com/en/migrations/snowconvert-docs/translation-references/teradata/sql-translation-reference/data-types)
