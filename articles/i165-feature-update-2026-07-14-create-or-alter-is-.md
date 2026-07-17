---
title: "SnowflakeのCREATE OR ALTERが24オブジェクトでGA——「無ければ作る、あれば直す」でTerraform不要に"
emoji: "🔁"
type: "tech"
topics: ["snowflake", "terraform", "cicd", "sql", "devops"]
published: true
---

## この記事で分かること

Snowflake のオブジェクト管理を Terraform で IaC 化して CI/CD に組み込んでいる方、またはこれから宣言的デプロイを設計する方向けの記事です。

`CREATE OR ALTER` が 24 種のオブジェクトで一般提供（GA)となりました。

「オブジェクトが無ければ作成、あれば定義に一致するよう変更」を SQL 1 本で行える構文で、Terraform を経由せずに宣言的デプロイを実現する選択肢になります。

この記事では GA 対象オブジェクトの一覧、Terraform 運用と比べたメリット、実機検証で確認した注意点をまとめます。

公式リリースノート: [Jul 14, 2026: CREATE OR ALTER is generally available for 24 object types](https://docs.snowflake.com/en/release-notes/2026/other/2026-07-14-create-or-alter-ga)

:::message
内容は記事作成時点のものです。
仕様は変更され得るため、最終的には最新の公式ドキュメントで確認ください。
:::


![](/images/i165-feature-update-2026-07-14-create-or-alter-is-/cover.png)

## GA 対象の 24 オブジェクト一覧

GA となった 24 オブジェクトタイプは以下のとおりです。

**アカウントレベル（6 種）**

| オブジェクト |
|------------|
| AUTHENTICATION POLICY |
| DATABASE |
| NETWORK POLICY |
| ROLE |
| SHARE |
| WAREHOUSE |

**データベース・スキーマレベル（18 種）**

| オブジェクト | | |
|------------|------------|------------|
| ALERT | MASKING POLICY | SEQUENCE |
| APPLICATION ROLE | NETWORK RULE | STAGE |
| DATABASE ROLE | PROCEDURE | TABLE |
| DATA METRIC FUNCTION | ROW ACCESS POLICY | TAG |
| FILE FORMAT | SCHEMA | TASK |
| FUNCTION | SEMANTIC VIEW | VIEW |

TABLE・VIEW・TASK・WAREHOUSE・ROLE といった CI/CD で日常的に触るオブジェクトが一通り揃っています。

なお、DYNAMIC TABLE・PIPE・STREAM・EXTERNAL FUNCTION・FUNCTION (Snowpark Container Services)・VERSIONED SCHEMA の 6 種は引き続きプレビューです（[CREATE OR ALTER <object>](https://docs.snowflake.com/en/sql-reference/sql/create-or-alter)）。

## Terraform で IaC してきた運用と何が変わるか

Snowflake のオブジェクト管理を IaC 化する場合、これまでの定番は [Terraform Snowflake プロバイダー](https://registry.terraform.io/providers/snowflakedb/snowflake/latest) でした。

HCL でオブジェクトを宣言し、`terraform plan` / `apply` を CI/CD に組み込む構成です。

この運用と比べて、`CREATE OR ALTER` による宣言的デプロイには 3 つのメリットと、把握しておくべき 1 つのデメリットがあります。

### メリット1: プロバイダーの対応を待たなくてよい

Terraform 経由の管理では、Snowflake の新機能をプロバイダーがリソースとして実装するまで使えません。

実際にギャップは存在します。

今回 GA になった 24 オブジェクトのうち、**DATA METRIC FUNCTION は執筆時点でプロバイダー（v2.18.0）にリソースが存在しません**。

またリリースサイクルにも差があります。

Snowflake 本体は週次でリリースされる一方、プロバイダーのリリースは月 1 回程度（v2.15: 4月 → v2.16: 5月 → v2.17: 5月末 → v2.18: 7月）です。

`CREATE OR ALTER` は Snowflake ネイティブの SQL 構文なので、新機能が GA になった瞬間からそのまま宣言的に管理できます。

### メリット2: プロバイダーのバージョン追従という保守作業が消える

Terraform プロバイダーはそれ自体がソフトウェアであり、バージョンアップへの追従が必要です。

Snowflake プロバイダーは v1.0.0（2024年12月）から v2.0.0(2025年4月）までわずか 4 ヶ月でメジャーバージョンが上がり、破壊的変更への対応（リソース名変更・属性の刷新・移行ガイドに沿った書き換え）が発生しました。

Snowflake 側の仕様変更にプロバイダーが追いつくまで新しい挙動を使えない、逆にプロバイダーを上げると HCL の書き換えが必要になる——この二重のバージョン管理が `CREATE OR ALTER` にはありません。

SQL は Snowflake 本体と同時に更新されるため、追従対象が Snowflake 本体だけになります。

### メリット3: state ファイルとリモートバックエンドの運用が不要になる

Terraform 運用には tfstate の管理インフラが付いて回ります。

S3 などのリモートバックエンドの構築、同時実行を防ぐロック機構の用意、そして state ファイル自体の破損・巻き戻しへの備えです。

`CREATE OR ALTER` には state ファイルという概念がありません。

**Snowflake 上の現在のオブジェクトそのものが唯一の状態**であり、スクリプトを再実行すれば定義に収束します。

state の破損・ロック競合といった運用トラブルが構造的に発生せず、tfstate 保管用のインフラも不要になります。

### デメリット: 「意図しない変更」を検知する仕組みを失う

state がないことには明確な裏面があります。

Terraform では `terraform plan` が「HCL の定義」「state」「Snowflake 上の実物」の 3 つを突き合わせるため、誰かが Snowsight から直接オブジェクトを変更すると**適用前に差分として浮かび上がります**。

ドリフト対応は負担であると同時に、意図しない変更や不正な変更の**検知機構**として機能していたわけです。

`CREATE OR ALTER` にはこの仕組みがありません。

手元の定義と Snowflake 上の実物がズレたとき、挙動は 2 パターンに分かれます。

| ズレの種類 | CREATE OR ALTER での挙動 |
|-----------|------------------------|
| 定義ファイルに**書いてある**プロパティが直接変更された | 次回の再実行で**黙って定義側に上書き**される。収束はするが「誰かが変えていた」ことに気づけない |
| 定義ファイルに**書いていない**変更（別途付与された権限、スクリプト外で作られたオブジェクト等） | 検知も修正もされず、**そのまま残り続ける** |

どちらのケースも「差分を見せて止める」タイミングが存在しません。

`ACCOUNT_USAGE.QUERY_HISTORY` などの監査ログで「誰がいつ何を変更したか」を事後的に追うことはできますが、デプロイ前のゲートにはなりません。

意図しない変更の検出をガバナンス要件として持つ環境では、引き続き Terraform を使うか、Snowflake 側で別途検知の仕組みを用意する必要があります。

:::message
このほか Terraform には、依存関係グラフによる適用順序の自動解決、AWS などクラウドリソースとの一元管理という強みがあります。
「Snowflake 内のオブジェクト管理だけなら SQL で完結できるようになった」というのが今回の変化です。
:::

## 実際に動かしてみよう

普段の運用で最も触る TABLE と WAREHOUSE の 2 つで、冪等な動作を確認します。

:::details セットアップ（DB・スキーマ作成）

```sql
USE ROLE SYSADMIN;
CREATE DATABASE IF NOT EXISTS COA_DEMO;
CREATE SCHEMA IF NOT EXISTS COA_DEMO.PUBLIC;
USE SCHEMA COA_DEMO.PUBLIC;
```

:::

### ステップ1: TABLE を作成し、再実行でカラムを追加する

まず 2 カラムのテーブルを作成し、データを入れておきます。

```sql
CREATE OR ALTER TABLE customers (
  id   NUMBER,
  name STRING
);

INSERT INTO customers VALUES (1, 'Alice'), (2, 'Bob');
```

次に、同じ文の末尾へ `email` を 1 カラム足して再実行します。

```sql
CREATE OR ALTER TABLE customers (
  id    NUMBER,
  name  STRING,
  email STRING
);

SELECT * FROM customers ORDER BY id;
```

実行結果:

```
+----+-------+-------+
| ID | NAME  | EMAIL |
|----+-------+-------|
| 1  | Alice | NULL  |
| 2  | Bob   | NULL  |
+----+-------+-------+
```

既存の 2 行はそのままに、`EMAIL` カラムだけが追加されました。

`CREATE OR REPLACE` と違いテーブルを作り直さないため、データを保持したまま定義に追従できています。

### ステップ2: WAREHOUSE のサイズを再実行で変更する

アカウントレベルのオブジェクトも同じパターンで管理できます。

```sql
USE ROLE ACCOUNTADMIN;

CREATE OR ALTER WAREHOUSE COA_WH
  WAREHOUSE_SIZE = 'XSMALL'
  AUTO_SUSPEND   = 60
  AUTO_RESUME    = TRUE
  INITIALLY_SUSPENDED = TRUE;
```

サイズと Auto Suspend を変えて、同じ文を再実行します。

```sql
CREATE OR ALTER WAREHOUSE COA_WH
  WAREHOUSE_SIZE = 'SMALL'
  AUTO_SUSPEND   = 120
  AUTO_RESUME    = TRUE
  INITIALLY_SUSPENDED = TRUE;

SHOW WAREHOUSES LIKE 'COA_WH';
SELECT "name", "size", "auto_suspend" FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));
```

実行結果:

```
+--------+-------+--------------+
| name   | size  | auto_suspend |
|--------+-------+--------------|
| COA_WH | Small | 120          |
+--------+-------+--------------+
```

X-Small / 60秒 だったウェアハウスが Small / 120秒 に変わりました。

「初回作成」と「設定変更」を同じ 1 本の SQL で書けるため、ウェアハウス定義ファイルを Git 管理して再実行するだけの運用が成立します。

## 実機検証でわかった注意ポイント

「定義に一致するよう変更」といっても、すべての変更が可能なわけではありません。

TABLE のカラム操作で実際に確認できた制約を 2 つ挙げます。

### 1. カラム追加は「末尾のみ」。途中への追加はエラーで止まる

カラムリストの途中に新しいカラムを差し込もうとすると、明示的にエラーになります。

先ほどの `customers`（id, name, email）で、`name` を `customer_name` に変えて再実行してみます。

```sql
CREATE OR ALTER TABLE customers (
  id            NUMBER,
  customer_name STRING,
  email         STRING
);
```

実行結果:

```
000002 (0A000): Unsupported feature
'CREATE OR ALTER TABLE column add before end of column list'.
```

Snowflake から見ると「`name` の削除 + 途中位置への `customer_name` 追加」であり、末尾以外へのカラム追加は未サポートのためエラーで停止します。

CI/CD でこの定義を流すとパイプラインがここで失敗します。

### 2. 末尾カラムの「リネーム」はエラーにならず、データが消える

より危険なのはこちらです。

**末尾の**カラム名を変えた場合、エラーにならずに「旧カラムの DROP + 新カラムの ADD」として実行されます。

`email` にデータが入った状態で、`mail_address` へ「リネーム」したつもりの再実行をしてみます。

```sql
-- email にはデータが入っている
SELECT * FROM customers ORDER BY id;
-- 1  Alice  alice@example.com
-- 2  Bob    bob@example.com

CREATE OR ALTER TABLE customers (
  id           NUMBER,
  name         STRING,
  mail_address STRING
);

SELECT * FROM customers ORDER BY id;
```

実行結果:

```
+----+-------+--------------+
| ID | NAME  | MAIL_ADDRESS |
|----+-------+--------------|
| 1  | Alice | NULL         |
| 2  | Bob   | NULL         |
+----+-------+--------------+
```

ステートメントは**正常終了**し、メールアドレスのデータはすべて失われました。

`CREATE OR ALTER` にとってカラムの同一性は「名前」で判定されるため、リネームという操作は存在しないのです。

この挙動は[公式ドキュメント](https://docs.snowflake.com/en/sql-reference/sql/create-table#create-or-alter-table-usage-notes)にも明記されていますが、エラーで止まらない分、レビューで見落とすと本番データの消失に直結します。

カラム名を変えたい場合は `ALTER TABLE ... RENAME COLUMN` を個別に実行してから、定義ファイルを新しい名前に揃えるという運用が必要です。

### そのほかの制約（公式ドキュメントより）

- マスキングポリシー・行アクセスポリシー・タグの設定/解除は `CREATE OR ALTER TABLE` 構文では扱えない（既存の設定は変更されず残る）
- `CREATE TABLE ... AS SELECT`（CTAS）・`LIKE`・`CLONE` バリアントは未サポート
- テーブルの種類によって対応状況が 3 段階に分かれる

| 対応状況 | テーブルの種類 |
|---------|--------------|
| **GA**（`CREATE OR ALTER TABLE`） | permanent / temporary / transient |
| **プレビュー**（別構文の `CREATE OR ALTER DYNAMIC TABLE`） | dynamic |
| **未対応** | 読み取り専用 / 外部 / Iceberg / ハイブリッド |


## 検証コード

この記事のハンズオンで使用した SQL を Jupyter Notebook（.ipynb）形式で公開しています。

Snowflake Notebooks にインポートして、そのまま自分の環境で実行できます。

[📓 検証ノートブックを開く（GitHub）](https://github.com/GTK0326/zenn-content/blob/main/notebooks/i165-feature-update-2026-07-14-create-or-alter-is-.ipynb)
## まとめ

`CREATE OR ALTER` の GA により、TABLE から WAREHOUSE・ROLE まで 24 種のオブジェクトを SQL だけで宣言的に管理できるようになりました。

Terraform 運用と比べると、プロバイダーの対応待ち・バージョン追従・state 管理インフラという 3 つの負担が構造的に消えます。

一方で、Terraform の plan が担っていた「意図しない変更の検知」は失われます。ガバナンス要件としてドリフト検知が必要な環境では、引き続き Terraform を使うか、Snowflake 側で別途意図しない変更を検知する仕組みを導入する必要があります。

また、カラムのリネームがデータ消失として実行される挙動など、「定義に収束する」というモデル特有の落とし穴があるため、テーブル定義の変更はレビューで意図した差分かを確認する運用を推奨します。

## 参考リンク

- [Jul 14, 2026: CREATE OR ALTER is generally available for 24 object types (Release Notes)](https://docs.snowflake.com/en/release-notes/2026/other/2026-07-14-create-or-alter-ga)
- [CREATE OR ALTER <object> | Snowflake Documentation](https://docs.snowflake.com/en/sql-reference/sql/create-or-alter)
- [CREATE OR ALTER TABLE usage notes](https://docs.snowflake.com/en/sql-reference/sql/create-table#create-or-alter-table-usage-notes)
- [Terraform Snowflake Provider](https://registry.terraform.io/providers/snowflakedb/snowflake/latest)