---
title: "Snowflake に ABAC が来た——feature policy rules で接続元の端末ごとに権限を変えられるか検証した"
emoji: "🚧"
type: "tech"
topics: ["snowflake", "sql", "security", "governance", "datagovernance"]
published: true
---

## この記事で分かること

- Snowflake に ABAC（属性ベースアクセス制御）を実現する機能として、
  feature policy rules が登場しました。2026年8月16日に一般提供になっています。
- 実際にこれを使い、「アクセスする場所（接続元の端末）」という属性から、
  使えるロールを制御できるかを検証しました。
- どこまでできて、どこからができないのかを実機で確認しました。

公式リリースノート: [Snowflake Documentation](https://docs.snowflake.com/en/release-notes/2026/other/2026-08-16-feature-policy-rules-ga)

:::message
内容は記事作成時点のものです。
仕様は変更され得るため、最終的には最新の公式ドキュメントで確認ください。
:::


![](/images/i268-feature-update-2026-08-16-feature-policy-rule/cover.png)

## ABAC（属性ベースアクセス制御）とは

ABAC（Attribute-Based Access Control、属性ベースアクセス制御）は、
権限を静的な付与関係ではなく、
属性の組み合わせで動的に決めるセキュリティモデルです。
ここでいう属性とは、ユーザーの役職や所属部署といった「人の属性」だけでなく、
アクセスする時間や曜日、接続元のネットワークや端末といった
「状況の属性」も含みます。
これらを条件式として評価し、その場で許可・拒否を決めます。

ロールベースの RBAC と並べると違いがはっきりします。
RBAC は「誰か」だけで権限が決まります。
ある人がロールを持っているなら、その権限は月曜の朝でも金曜の深夜でも、
社内の端末からでも出先のノート PC からでも、まったく同じように行使できます。

一方 ABAC は「誰が・いつ・どこから」で決まります。
同じ人が同じロールを持っていても、条件を満たさない場面では権限が行使できません。

そして最近は、この細かさを実際に求められる案件が出てきています。
「本番データベースへの変更は、指定の作業用端末からのみ」
「リリース凍結期間中はオブジェクトを作らせない」といった要件は、
RBAC の付与・剥奪だけで表現しようとすると、
運用でロールを付け外しし続けるしかありませんでした。

## 兼務者の権限は、行使できる場面を絞れない

データ分析基盤では、管理者と分析担当が同じ人であることはよくあります。
その人はアカウント管理用の強いロールと、日常業務用のロールを両方持っています。
GRANT の世界では、持っている権限はいつでも、どこからでも行使できてしまいます。

問題は「権限を持っていること」ではなく「行使できる場面を絞れないこと」です。
出先のノート PC からダッシュボードを見るつもりで接続し、
そのままセッションの勢いで検証用テーブルを本番データベースに作ってしまう。
悪意はまったくなく、権限的にも正しいので、Snowflake は何も止めてくれません。

:::message
**実現したい状態**

1. 社内の開発用 PC に座っているときだけ管理者としてふるまい、
   本番データベースにテーブルを作れる。
2. 出先のノート PC からは、これまでどおり接続してデータを参照し、
   分析用の加工まではできる。
3. ただし同じノート PC から管理ロールに切り替えても、オブジェクトを作れない。
4. しかも本人が持つアカウントは1つのままで、
   やることに合わせてロールを切り替えるだけにしたい。
:::

GRANT でロールを分けても、ロールは接続元を見ません。
出先から管理ロールに切り替えれば、そのまま作成できてしまいます。

ユーザーを2つに分け、管理用ユーザーの接続元だけを社内に限定すれば
要件そのものは満たせますが、それでは「アカウントは1つのまま」が崩れます。
兼務者に2つのアカウントを持たせ、パスワードも MFA も二重に管理し、
どちらで接続しているかを常に意識させることになります。

この中間的な要求に、これまでの RBAC では応えられませんでした。

## feature policy rules で何ができるようになったか

2026年8月16日のリリースで、feature policy rules が一般提供になりました。
feature policy は YAML ボディを持ち、リクエストの属性に応じて
オブジェクト作成を条件付きでブロックできます
（[Feature policy rules](https://docs.snowflake.com/en/user-guide/feature-policies)）。

RBAC では表現できなかった制御が、この YAML の条件式で書けるようになります。
具体的には次のようなパターンです。

1. **テーブルの作成は許可するが、temporary table だけを止める。**
   GRANT では「CREATE TABLE を許すが一時テーブルは不可」を表現できませんでした。
   feature policy なら、リクエストの `IS_TEMPORARY` プロパティを条件にして、
   一時テーブルだけを弾けます。
2. **task の作成は許可するが、serverless task だけを止める。**
   同じく、リクエストの `WAREHOUSE` プロパティが空かどうかで、
   ウェアハウス指定のない task だけをブロックできます。
3. **曜日で制御する。月火水木のみ変更を許可し、金土日は凍結する。**
   「誰が」ではなく「いつ」で決まるので、
   RBAC ではロールを付け外しする運用でしか実現できませんでした。
   条件式に日付関数を書けば、ポリシー側で表現できます。
4. **接続元の端末で制御する。**
   通常の業務端末では分析者としての操作のみ、
   社内の作業用端末からのみ管理操作を許可する。
   同じユーザーが同じロールを持っていても、
   接続元によって作成の可否が変わります。

1と2は、リクエスト対象オブジェクトの属性を
`SYS_CONTEXT('SNOWFLAKE$REQUEST', 'GET_OBJECT_PROPERTY', ...)` で
参照して判定するパターンです。
3と4は、オブジェクトの属性ではなくセッションのコンテキストで判定するパターンです。

条件式には「Boolean を返し、テーブルを参照せず、副作用のない式」であれば
書けるとされているため、コンテキスト関数もそのまま条件にできます。

### 条件式で使える材料

作成しようとしているオブジェクトの属性は、
`SYS_CONTEXT('SNOWFLAKE$REQUEST', 'GET_OBJECT_PROPERTY', '<名前>')` で取得します。

| 属性名 | 使えるオブジェクト | 説明 |
|---|---|---|
| `IS_TEMPORARY` | TABLE / VIEW / STAGE | TEMPORARY 指定か |
| `IS_TRANSIENT` | TABLE / SCHEMA / DYNAMIC_TABLE | TRANSIENT 指定か |
| `WAREHOUSE` | TASK | ウェアハウス名。未指定は NULL |
| `EXTERNAL_VOLUME` | ICEBERG_TABLE | 外部ボリューム名 |
| `DATABASE` | スキーマレベルの各型 | 作成先のデータベース名 |
| `SCHEMA` | スキーマレベルの各型 | 作成先のスキーマ名 |

`WAREHOUSE` が NULL になるのは serverless task の場合です。

セッションのコンテキスト関数も条件式に使えます。
以下は網羅リストではなく、今回の検証で実機で使えることを確認した主なものです。

| 関数 | 返すもの |
|---|---|
| `CURRENT_ROLE()` | 実行時のプライマリロール |
| `CURRENT_USER()` | 実行ユーザー |
| `IS_ROLE_IN_SESSION()` | セカンダリロールを含む判定 |
| `CURRENT_IP_ADDRESS()` | 接続元 IP |
| `CURRENT_WAREHOUSE()` | 使用中のウェアハウス |
| `DAYOFWEEK()` / `HOUR()` | 曜日・時刻 |

条件式に書けるのは組み込み関数だけで、
サブクエリ・ユーザー定義関数・正規表現関数（`REGEXP_LIKE` / `RLIKE`）は使えません。

今回はこの4つ目、**端末によって操作できる内容が変わるパターン**を検証します。
`CURRENT_ROLE()` と `CURRENT_IP_ADDRESS()` を条件式に書き、
ロールと接続元の組み合わせで CREATE TABLE を止められるかを実機で確かめます。
3の曜日パターンは同じ書き方の応用になりますが、本記事では検証していません。

## 実際に動かしてみよう

### 何を確かめるのか（前提）

いきなり SQL に入る前に、想定と登場人物を整理します。

**登場するロールと役割**

| ロール | 役割 |
|---|---|
| `VERIFY_ADMIN_ROLE` | 管理操作。テーブル作成を担当 |
| `VERIFY_ANALYST_ROLE` | 日常の分析作業。参照と一時的な加工 |

どちらのロールも同じ1人のユーザー（`DATA_USER`）に付与します。
これが冒頭の「兼務者」です。

**想定する組織の要件**

- 管理操作（テーブル作成）は社内の開発用 PC（`203.0.113.10`）からのみ許可したい。
- 出先のノート PC（`198.51.100.20`）からは、分析ロールでの操作だけを許可したい。
- 開発用 PC は管理作業専用にしたいので、そこからの分析ロールでの作成も止める。
  端末と役割を1対1に固定する。

つまり、4つの組み合わせに対して次の結果を狙います。

| 接続元 | ロール | 作成 |
|---|---|---|
| 開発用 PC | 管理ロール | 許可したい |
| 開発用 PC | 分析ロール | ブロックしたい |
| ノート PC | 管理ロール | ブロックしたい |
| ノート PC | 分析ロール | 許可したい |

**要件をどのルールで表現するか**

この要件を feature policy の YAML に落とすと、次の対応になります。

- ブロックしたい2通りを `conditions` に名前付きの条件として定義する。
- `blocked_creation_rules` の `object_type: TABLE` に対して、
  `block_when_any` でその2つの条件名を並べる。
- ポリシーは `ALTER DATABASE ... SET FEATURE POLICY` で
  データベース `VERIFY_TEMP_DB` に紐づける。

検証は ACCOUNTADMIN で下準備を行い、そのあと専用ロールに切り替えて実施しました。

:::details 事前準備（DB・スキーマ・ロール・権限）

```sql
CREATE DATABASE IF NOT EXISTS VERIFY_TEMP_DB;
CREATE SCHEMA IF NOT EXISTS VERIFY_TEMP_DB.POLICY_TEST;

CREATE ROLE IF NOT EXISTS VERIFY_ADMIN_ROLE;
CREATE ROLE IF NOT EXISTS VERIFY_ANALYST_ROLE;

GRANT USAGE ON DATABASE VERIFY_TEMP_DB TO ROLE VERIFY_ADMIN_ROLE;
GRANT USAGE ON DATABASE VERIFY_TEMP_DB TO ROLE VERIFY_ANALYST_ROLE;
GRANT USAGE ON SCHEMA VERIFY_TEMP_DB.POLICY_TEST TO ROLE VERIFY_ADMIN_ROLE;
GRANT USAGE ON SCHEMA VERIFY_TEMP_DB.POLICY_TEST TO ROLE VERIFY_ANALYST_ROLE;

GRANT CREATE TABLE ON SCHEMA VERIFY_TEMP_DB.POLICY_TEST TO ROLE VERIFY_ADMIN_ROLE;
GRANT CREATE TABLE ON SCHEMA VERIFY_TEMP_DB.POLICY_TEST TO ROLE VERIFY_ANALYST_ROLE;

-- feature policy の作成にはこの権限が必要です
GRANT CREATE FEATURE POLICY ON SCHEMA VERIFY_TEMP_DB.POLICY_TEST
  TO ROLE VERIFY_ADMIN_ROLE;

GRANT ROLE VERIFY_ADMIN_ROLE TO USER DATA_USER;
GRANT ROLE VERIFY_ANALYST_ROLE TO USER DATA_USER;
```

:::

### ステップ1: ロール×IP のポリシーを作る

条件式には端末を表す IP アドレスを直接書きます。
本記事では `203.0.113.10` を社内の開発用 PC、
`198.51.100.20` を出先のノート PC として扱います。

検証では、実際の接続元 IP を開発用 PC の IP とみなして実行しています。
記事中の IP はドキュメント用に予約されたアドレス（RFC 5737）に置き換えています。

```sql
-- 管理ロールは開発用 PC 以外からブロック
-- 分析ロールは開発用 PC からブロック
CREATE OR REPLACE FEATURE POLICY VERIFY_TEMP_DB.POLICY_TEST.COMBINED_ROLE_IP_POLICY
  AS $$
    conditions:
      - name: admin_from_wrong_terminal
        expression: "CURRENT_ROLE() = 'VERIFY_ADMIN_ROLE' AND CURRENT_IP_ADDRESS() <> '203.0.113.10'"
      - name: analyst_from_wrong_terminal
        expression: "CURRENT_ROLE() = 'VERIFY_ANALYST_ROLE' AND CURRENT_IP_ADDRESS() = '203.0.113.10'"
    blocked_creation_rules:
      - object_type: TABLE
        block_when_any:
          - admin_from_wrong_terminal
          - analyst_from_wrong_terminal
  $$;
```

注意したいのは、**`block_when_any` は名前のとおり OR 結合**だという点です。
「管理ロール **かつ** 開発用 PC 以外」のような AND 条件は
`conditions` 側の `expression` の中で書き切る必要があり、
`block_when_any` に条件を2つ並べても AND にはなりません。

### ステップ2: DESCRIBE で YAML ボディを確認する

```sql
-- policy_definition プロパティに YAML ボディが入る
DESCRIBE FEATURE POLICY VERIFY_TEMP_DB.POLICY_TEST.COMBINED_ROLE_IP_POLICY;
```

実行結果（`policy_definition` プロパティの値）:
```
conditions:
  - name: admin_from_wrong_terminal
    expression: "CURRENT_ROLE() = 'VERIFY_ADMIN_ROLE' AND CURRENT_IP_ADDRESS() <> '203.0.113.10'"
  - name: analyst_from_wrong_terminal
    expression: "CURRENT_ROLE() = 'VERIFY_ANALYST_ROLE' AND CURRENT_IP_ADDRESS() = '203.0.113.10'"
blocked_creation_rules:
  - object_type: TABLE
    block_when_any:
      - admin_from_wrong_terminal
      - analyst_from_wrong_terminal
```

`DESCRIBE FEATURE POLICY` も今回あわせて一般提供になりました。
登録した YAML をそのまま読み返せるので、
適用中の条件を SQL だけで追跡できます。

### ステップ3: データベースにポリシーを適用する

```sql
-- スキーマ単位ではなくデータベース単位で紐づける
ALTER DATABASE VERIFY_TEMP_DB
  SET FEATURE POLICY VERIFY_TEMP_DB.POLICY_TEST.COMBINED_ROLE_IP_POLICY;
```

### ステップ4: 許可される組み合わせを試す

管理ロール（`VERIFY_ADMIN_ROLE`）で、開発用 PC から作成します。

```sql
USE ROLE VERIFY_ADMIN_ROLE;
CREATE TABLE VERIFY_TEMP_DB.POLICY_TEST.ADMIN_TABLE_FROM_DEV (id INT);
```

実行結果:
```
Table ADMIN_TABLE_FROM_DEV successfully created.
```

### ステップ5: ブロックされる組み合わせを試す

同じ端末のまま、分析ロールに切り替えて作成します。

```sql
USE ROLE VERIFY_ANALYST_ROLE;
CREATE TABLE VERIFY_TEMP_DB.POLICY_TEST.ANALYST_TABLE_FROM_DEV (id INT);
```

実行結果（エラーコード 3001）:
```
SQL access control error: Insufficient privileges to operate on schema 'POLICY_TEST'. Create TABLE denied due to feature policy restriction on DATABASE VERIFY_TEMP_DB. Consult your internal policy administrator to review the effective feature policy by running 'SHOW FEATURE POLICIES ON DATABASE VERI...
```

権限エラーとして返りつつ、"feature policy restriction" と原因が明記されます。
さらに `SHOW FEATURE POLICIES` で確認せよ、という誘導まで入ります。
GRANT を疑って延々調べる時間が発生しないのは、運用上かなり助かります。

### 4パターンの結果まとめ

残る2パターン（ノート PC からの2通り）は、
検証環境では接続元を物理的に変えられないため、
条件式の比較を反転させた同じ手順のポリシーを作り、
`ALTER DATABASE ... SET FEATURE POLICY ... FORCE` で差し替えて確認しました。

| 接続元 | ロール | 作成 |
|---|---|---|
| 開発用 PC | 管理ロール | 許可 |
| 開発用 PC | 分析ロール | ブロック |
| ノート PC | 管理ロール | ブロック |
| ノート PC | 分析ロール | 許可 |

ロール単独でも IP 単独でもなく、組み合わせで判定できていることが確認できます。
狙いどおり、同じユーザーのまま
「出先では分析だけ、社内では管理作業も」という制御が成立しました。

## 検証してわかった2つのハマりどころ

### 1. 作成は絞れても、参照はまったく絞れない

feature policy がブロックするのはオブジェクトの作成だけです。
CREATE TABLE を止められている状態のロールでも、
既存テーブルへの SELECT は一切影響を受けず、中身をそのまま読めました。
参照を絞りたいなら、
行アクセスポリシーやマスキングポリシーを別に用意する必要があります。

接続は network policy、作成は feature policy、
参照は行アクセス・マスキングポリシー。
層が違うものとして設計しないと、
「端末ごとの権限制御」のつもりで穴を残します。

### 2. ロール判定の関数はセッションの状態で結果が変わる

ロールを見る関数は `CURRENT_ROLE()` だけでなく、
セカンダリロールまで含めて判定する `IS_ROLE_IN_SESSION()` もあります。
ただしオブジェクトの作成はプライマリロールの権限で行われるため、
セカンダリロールに ACCOUNTADMIN が載っていても作成の可否そのものは変わりません。

さらにセカンダリロールを `ALL` で運用している環境では、
ユーザーに付与された全ロールがセッションに載るため
`IS_ROLE_IN_SESSION()` の判定が常に真になり、
利用者側で毎回セカンダリロールを切り替える運用が必要になります。
ポリシーの都合をユーザーのセッション操作に押し付ける形になるため、
今回は「いま何のロールとして操作しているか」だけを見る
`CURRENT_ROLE()` を選びました。

コンテキスト関数はセッションの状態で結果が変わるので、
条件式の意味だけでなく運用まで含めて選ぶ必要があります。


## 検証コード

ハンズオンで使用した SQL を Jupyter Notebook 形式で公開しています。

Snowflake Notebooks にインポートして、そのまま自分の環境で実行できます。

[📓 検証ノートブックを開く（GitHub）](https://github.com/GTK0326/zenn-content/blob/main/notebooks/i268-feature-update-2026-08-16-feature-policy-rule.ipynb)
## まとめ

feature policy rules の YAML にコンテキスト関数を書けるおかげで、
ロールと接続元の組み合わせで CREATE を止める制御が SQL だけで完結しました。
「誰が・どこから」という属性で権限が決まる ABAC が、
GRANT の外側で表現できるようになったということです。

一方で ABAC として止められるのは作成だけであり、参照は別の仕組みが必要です。
Snowflake で ABAC を設計するときは、
feature policy が担う範囲と担わない範囲を最初に切り分けておくのが安全です。

## 参考リンク

- [Feature policy rules](https://docs.snowflake.com/en/user-guide/feature-policies)