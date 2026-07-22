---
title: "個人の無料GmailでSnowflakeにログインできるか試した——OIDCフェデレーテッド認証で検証"
emoji: "🔑"
type: "tech"
topics: ["snowflake", "oidc", "sso", "認証", "security"]
published: false
---

## この記事で分かること

「ECサイトでよく見る『Googleでログイン』みたいに、自分がいつも使っている個人のGmailでSnowflakeにログインできるのか？」を実際に試した記録です。

結論から言うと、**個人の無料Gmailでログインできました**。Google Workspace（有料の組織アカウント）は不要です。

ただし、ECサイトの「Googleでログイン」とは仕組みが根本的に違う点だけ注意が必要です。

公式リリースノート: [July 20, 2026: OIDC federated authentication (Public Preview)](https://docs.snowflake.com/en/release-notes/2026/other/2026-07-17-oidc-federated-authentication-preview)

:::message
OIDCフェデレーテッド認証は Public Preview 機能です。内容は記事作成時点のものです。
:::


![](/images/i184-google-oidc-personal-gmail/cover.png)

## そもそもどういう機能か

2026年7月20日、Snowflake の認証に **OIDC（OpenID Connect）フェデレーテッド認証** が Public Preview で追加されました。

これは、外部の IdP（Identity Provider）を使ったシングルサインオン（SSO）でSnowsightにログインできる機能です。

従来のSnowflakeのSSOは SAML2 が中心でしたが、そこに OIDC という選択肢が加わりました。

そして OIDC プロバイダーとして **Google をマネージドプロバイダーとして使える**のがポイントです。

「マネージド」とは、Google Cloud Console でOAuthクライアントを自分で作る必要がなく、**Snowflake 側が裏側の設定を持っていてくれる**という意味です。

## ECサイトの「Googleでログイン」との決定的な違い

ここが一番の注意点です。

ECサイトの「Googleでログイン」は、**Gmailさえ持っていれば誰でも**その場でアカウントが作られてログインできます（セルフ登録）。

一方、Snowflake の OIDC は **組織向けSSO** なので、そうではありません。

- **管理者が事前にSnowflakeに作ったユーザーしかログインできない**
- Google認証のメールアドレスが、Snowflakeユーザーの `EMAIL_ADDRESS` 属性と一致する必要がある

つまり Google は「本人であることの証明」に使うだけで、**任意のGmailで勝手に入れるわけではありません**。

ボタンの見た目はECサイトと同じ「Sign in with Google」ですが、中身は「登録済みユーザーだけがGoogle経由で入れる」というものです。

## やってみる

### ステップ1: OIDCセキュリティ統合を作成する

Google をマネージドプロバイダーとして、OIDCセキュリティ統合を作成します。

```sql
USE ROLE ACCOUNTADMIN;

CREATE OR REPLACE SECURITY INTEGRATION MY_GOOGLE_OIDC
  TYPE = OIDC
  ENABLED = TRUE
  OIDC_PROVIDER = 'GOOGLE'
  OIDC_LOGIN_PAGE_LABEL = 'Sign in with Google';
```

`OIDC_LOGIN_PAGE_LABEL` がログイン画面のボタンの文言になります。

マネージドプロバイダー（Google）では `OIDC_ISSUER` / `OIDC_CLIENT_ID` / `OIDC_CLIENT_SECRET` といった認証情報は**指定できません**。Snowflakeが管理するためです。

作成できたか確認します。

```sql
DESC SECURITY INTEGRATION MY_GOOGLE_OIDC;
```

`ENABLED = true`、`OIDC_PROVIDER = GOOGLE`、そして `OIDC_REDIRECT_URIS` が自動で設定されていれば準備完了です。

:::message
このアカウントは Standard Edition ですが、OIDCセキュリティ統合は作成できました。
:::

### ステップ2: 自分のユーザーにメールアドレスを設定する

Googleマネージドの場合、ユーザーのマッピングは **emailクレーム → `EMAIL_ADDRESS`** に固定されています（変更不可）。

なので、ログインしたい自分のSnowflakeユーザーの `EMAIL_ADDRESS` に、使いたいGmailアドレスを設定します。

```sql
ALTER USER 自分のユーザー名 SET EMAIL = '自分のアドレス@gmail.com';
```

これで、このGmailで認証した人が、このSnowflakeユーザーとしてログインできるようになります。

### ステップ3: 実際にログインする

Snowsight のログイン画面を開くと、「Sign in with Google」ボタンが表示されます。

![Snowflakeのログイン画面にSign in with Googleボタンが表示される](/images/i184-google-oidc-personal-gmail/login-button.png)

ボタンを押すと Google のアカウント選択画面に移り、snowflake.com への情報提供の同意画面が出ます。

![Googleの同意画面。snowflake.comにメールアドレスと名前を提供する](/images/i184-google-oidc-personal-gmail/consent.png)

`EMAIL_ADDRESS` に設定したGmailを選ぶと……

![Snowsightのホーム画面にログインできた](/images/i184-google-oidc-personal-gmail/home.png)

**個人の無料Gmailで、Snowsightにログインできました。**

## ハマったところ: ドメイン制限には別設定が必要

「gmail.comドメインのメールだけ許可する」といったドメイン制限をかけたくて、`ALLOWED_USER_DOMAINS` を付けようとしました。

```sql
CREATE OR REPLACE SECURITY INTEGRATION MY_GOOGLE_OIDC
  TYPE = OIDC
  ENABLED = TRUE
  OIDC_PROVIDER = 'GOOGLE'
  ALLOWED_USER_DOMAINS = ('gmail.com');   -- これがエラーになる
```

しかしエラーになりました。

```
invalid property 'ALLOWED_USER_DOMAINS'; feature 'ENABLE_IDENTIFIER_FIRST_LOGIN' not enabled
```

`ALLOWED_USER_DOMAINS`（および複数のSSO統合の併用）は、**identifier-first login の有効化が前提**でした。

identifier-first login を有効にすると、ログイン画面がまず「ユーザー名・メールを入力してから認証方法を出す」という2段階の挙動に変わります。

今回は個人検証で挙動を変えたくなかったので、ドメイン制限なしで作成しました。

ドメイン制限なしでも、`EMAIL_ADDRESS` が一致するユーザーしか入れないため、本人限定は担保されています。

## セキュリティ上の注意点

「個人Gmailでログインできる」と聞くと緩く感じるかもしれませんが、実際は次のように守られています。

- **事前にユーザーを作り、`EMAIL_ADDRESS` を一致させた人しかログインできない**。任意のGmailでは入れない。
- 統合を追加してもパスワードログインは無効化されない（併存する）。
- 外部認証（OIDC SSO含む）後にMFAを必須化したい場合は、認証ポリシーで `MFA_POLICY = (ENFORCE_MFA_ON_EXTERNAL_AUTHENTICATION = 'ALL')` を設定できる。

## まとめ

個人の無料Gmailで、Snowflakeにログインできました。

- Google Workspace（有料）は不要。個人Gmailで動く。
- ただしECサイトの「Googleでログイン」とは違い、`EMAIL_ADDRESS` が一致するSnowflakeユーザーを事前に用意しておく必要がある。
- ドメイン制限（`ALLOWED_USER_DOMAINS`）を使いたい場合は identifier-first login の有効化が必要。

個人検証アカウントでも、パスワードを覚えずにGmailでサッとログインできるようになるので、地味に便利です。

## 参考リンク

- [July 20, 2026: OIDC federated authentication (Public Preview)](https://docs.snowflake.com/en/release-notes/2026/other/2026-07-17-oidc-federated-authentication-preview)
- [CREATE SECURITY INTEGRATION (OIDC) | Snowflake Documentation](https://docs.snowflake.com/en/sql-reference/sql/create-security-integration-oidc)
- [Overview of federated authentication and SSO | Snowflake Documentation](https://docs.snowflake.com/en/user-guide/admin-security-fed-auth-overview)
