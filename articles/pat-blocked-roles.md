---
title: "SnowflakeのPAT認証でACCOUNTADMINトークンをブロックする——BLOCKED_ROLES_LISTの使い方"
emoji: "🔐"
type: "tech"
topics: ["snowflake", "security", "authentication", "pat", "accesscontrol"]
published: true
---

## この記事について

Snowflake の PAT（プログラマティックアクセストークン）認証ポリシーに、`BLOCKED_ROLES_LIST` と `REQUIRE_ROLE_RESTRICTION_FOR_PERSON_USERS` という2つの新しいプロパティが追加されました。この記事では、それぞれの動作を SQL で確認しながら、特権ロールのトークン発行をポリシーレベルで組織全体に強制する方法を解説します。

![概念図](https://raw.githubusercontent.com/GTK0326/snowflake-article-images/main/images/pat-blocked-roles.png)

## 背景：なぜこの機能が必要か

Snowflake では OAuth 接続（Tableau Desktop・Tableau Online などのパートナーアプリを含む）に対して、セキュリティインテグレーション側の `BLOCKED_ROLES_LIST` パラメーターにより、特定ロールのブロックが以前から可能でした。デフォルトでは ACCOUNTADMIN・SECURITYADMIN・ORGADMIN・GLOBALORGADMIN が自動的にブロック対象となっており、OAuth 経由で特権ロールが使われるリスクをインテグレーション単位で制御できていました。

一方、PAT には同等の仕組みがありませんでした。認証ポリシーの `PAT_POLICY` にはサービスユーザー向けのロール指定必須化（`REQUIRE_ROLE_RESTRICTION_FOR_SERVICE_USERS`）はすでに存在していたものの、person ユーザーへのロール指定強制や、特定の特権ロールをトークンのロール制限として指定できなくする機能はなく、管理者が ACCOUNTADMIN などをブロックしたくても手段がない状態が続いていました。

今回のアップデートにより、`PAT_POLICY` に `BLOCKED_ROLES_LIST` と `REQUIRE_ROLE_RESTRICTION_FOR_PERSON_USERS` が追加され、OAuth と同等水準のロールレベル制御が PAT でも可能になりました。特に `BLOCKED_ROLES_LIST` はポリシー適用後に既存トークンも即座に無効化されるため、組織全体への一括強制ができます。

## 機能の概要

PAT 認証ポリシーの `PAT_POLICY` ブロックに2つのプロパティが追加されました。

| プロパティ | デフォルト | 説明 |
|---|---|---|
| `REQUIRE_ROLE_RESTRICTION_FOR_PERSON_USERS` | `FALSE` | `TRUE` にすると、person ユーザーは PAT 生成時にロールの指定が必須になります |
| `BLOCKED_ROLES_LIST` | `()` | ここに列挙したロールは PAT のロール制限として指定できなくなります |

`BLOCKED_ROLES_LIST` にロールを設定すると、**既に発行済みのトークン**も含めて即座に無効化されます。これはポリシー変更がリアルタイムで既存セッションに影響を与えるという、Snowflake の認証ポリシーの一般的な特性と一致しています。

## ハンズオン

**ステップ1: 認証ポリシーをデフォルト設定で作成する**

```sql
-- PAT_POLICYブロックで両プロパティをデフォルト値で明示的に指定
CREATE AUTHENTICATION POLICY pat_demo_policy
  PAT_POLICY (
    REQUIRE_ROLE_RESTRICTION_FOR_PERSON_USERS = FALSE,
    BLOCKED_ROLES_LIST = ()
  );
```

実行結果:
```
Authentication policy PAT_DEMO_POLICY successfully created.
```

**ステップ2: 作成したポリシーのデフォルト設定を確認する**

```sql
DESCRIBE AUTHENTICATION POLICY pat_demo_policy;
```

実行結果:
```
property    | property_value
------------+-------------------------------------------------------------------------------------
COMMENT     |
PAT_POLICY  | {"REQUIRE_ROLE_RESTRICTION_FOR_PERSON_USERS": false, "BLOCKED_ROLES_LIST": []}
```

`PAT_POLICY` フィールドに両プロパティが JSON 形式でまとめて格納されています。デフォルトはどちらも無効な状態です。

**ステップ3: person ユーザーへのロール指定を必須化する**

```sql
ALTER AUTHENTICATION POLICY pat_demo_policy
  SET PAT_POLICY (
    REQUIRE_ROLE_RESTRICTION_FOR_PERSON_USERS = TRUE
  );
```

実行結果:
```
Statement executed successfully.
```

```sql
DESCRIBE AUTHENTICATION POLICY pat_demo_policy;
```

実行結果:
```
property    | property_value
------------+-------------------------------------------------------------------------------------
COMMENT     |
PAT_POLICY  | {"REQUIRE_ROLE_RESTRICTION_FOR_PERSON_USERS": true, "BLOCKED_ROLES_LIST": []}
```

`REQUIRE_ROLE_RESTRICTION_FOR_PERSON_USERS` が `true` に変わりました。`BLOCKED_ROLES_LIST` は変更せず、個別のプロパティだけを更新できます。

**ステップ4: 特権ロールをブロックリストに追加する（成功ケース）**

```sql
-- ACCOUNTADMINとSYSADMINをブロック対象に指定する
ALTER AUTHENTICATION POLICY pat_demo_policy
  SET PAT_POLICY (
    BLOCKED_ROLES_LIST = (ACCOUNTADMIN, SYSADMIN)
  );
```

実行結果:
```
Statement executed successfully.
```

```sql
DESCRIBE AUTHENTICATION POLICY pat_demo_policy;
```

実行結果:
```
property    | property_value
------------+-------------------------------------------------------------------------------------
COMMENT     |
PAT_POLICY  | {"REQUIRE_ROLE_RESTRICTION_FOR_PERSON_USERS": true, "BLOCKED_ROLES_LIST": ["ACCOUNTADMIN", "SYSADMIN"]}
```

`ACCOUNTADMIN` と `SYSADMIN` の両方がブロックリストに登録されました。この時点で、これらのロールに紐づく既存の PAT はすべて無効化されます。

**ステップ5: 存在しないロールをブロックリストに指定する（失敗ケース）**

```sql
-- 存在しないロールを指定して、バリデーションの動作を確認する
ALTER AUTHENTICATION POLICY pat_demo_policy
  SET PAT_POLICY (
    BLOCKED_ROLES_LIST = (NONEXISTENT_ROLE)
  );
```

実行結果:
```
SQL compilation error: Role 'NONEXISTENT_ROLE' does not exist.
```

存在しないロール名を指定するとその場でエラーになり、ポリシーは変更されません。ロール名のタイポによる設定ミスを防ぐ仕組みです。

**ステップ6: CREATE 文で両プロパティを同時に指定する**

```sql
-- 新環境向けポリシーを最初から厳格な設定で作成する
CREATE AUTHENTICATION POLICY pat_strict_policy
  PAT_POLICY (
    REQUIRE_ROLE_RESTRICTION_FOR_PERSON_USERS = TRUE,
    BLOCKED_ROLES_LIST = (ACCOUNTADMIN, SYSADMIN)
  );
```

実行結果:
```
Authentication policy PAT_STRICT_POLICY successfully created.
```

`CREATE` 文でも両プロパティを同時に指定できます。新環境の構築時は ALTER を重ねるより、最初から宣言的に設定する方が意図が明確です。

**ステップ7: プロパティをデフォルト値にリセットする**

```sql
-- 両プロパティをデフォルト値に戻す
ALTER AUTHENTICATION POLICY pat_demo_policy
  SET PAT_POLICY (
    REQUIRE_ROLE_RESTRICTION_FOR_PERSON_USERS = FALSE,
    BLOCKED_ROLES_LIST = ()
  );
```

実行結果:
```
Statement executed successfully.
```

```sql
DESCRIBE AUTHENTICATION POLICY pat_demo_policy;
```

実行結果:
```
property    | property_value
------------+-------------------------------------------------------------------------------------
COMMENT     |
PAT_POLICY  | {"REQUIRE_ROLE_RESTRICTION_FOR_PERSON_USERS": false, "BLOCKED_ROLES_LIST": []}
```

両プロパティがデフォルト値に戻りました。

## 検証して気づいたこと

- **ロール名の検証はポリシー設定時に即座に行われる。** `BLOCKED_ROLES_LIST` に存在しないロールを指定するとその場でコンパイルエラーになり、設定変更は一切行われません。削除済みロールや命名規則の揺らぎによる意図しないブロックリストへの混入を防ぐ点で、実運用上の安全弁として機能します。

## まとめ

`BLOCKED_ROLES_LIST` と `REQUIRE_ROLE_RESTRICTION_FOR_PERSON_USERS` の追加により、PAT 認証ポリシーで ACCOUNTADMIN などの特権ロールによるトークン発行を組織全体でブロックできるようになりました。既存トークンの即時無効化も含め、ぜひ本番環境のセキュリティ強化に活用してみてください。