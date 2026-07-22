---
title: "個人の無料GmailでSnowflakeにログインできるか試した——OIDCフェデレーテッド認証で検証"
emoji: "🔑"
type: "tech"
topics: ["snowflake", "oidc", "sso", "認証", "security"]
published: false
---

## この記事で分かること

Snowflake は今、パスワードだけでなく **MFA（多要素認証）が必須**の方向に進んでいます。セキュリティ的には正しいのですが、個人検証環境だと地味に困ることがあります。

毎回のログインで MFA コードを入力したり、パスキーは特定の PC に紐づいたりするため、**スマホや別の端末からサッとログインするのが面倒**なのです。

会社なら Entra ID のような IdP で SSO を組めますが、個人でそんなものは持っていません。

そこで、**手持ちの Google アカウントを IdP 代わりにして、Gmail で Snowflake にログインできるか**を試しました。

結論から言うと、**個人の無料 Gmail でログインできました**（Google Workspace は不要）。高いセキュリティを保ったまま、スマホからでも Gmail ログインだけでサッと入れるようになります。

IdP を持たない小さな組織でも、同じやり方で Snowflake の SSO を導入できます。

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

## なぜ個人環境でこれが嬉しいのか

Snowflake はパスワードログイン時に MFA を求める方向に進んでいます。

会社の環境なら Entra ID や Okta のような IdP を使った SSO でこの手間を吸収できますが、個人検証環境ではそうはいきません。

毎回 MFA コードを入力するか、パスキーを登録するしかありません。

しかもパスキーは登録した端末に紐づくため、**別の PC やスマホからログインしようとすると詰まりがち**でした。

ここで Google OIDC を使うと、認証そのものを Google に委ねられます。

**SSO（外部認証）でのログインでは、Snowflake はデフォルトで独自の MFA を追加で求めません。** 認証は Google 側に委ねられ、Google アカウントの 2 段階認証などがそのまま効きます。

つまり、Google 側でしっかりセキュリティをかけておけば、**Snowflake 側では Gmail ログイン 1 回でサッと入れる**ようになります。スマホのブラウザでも同じです。

「高いセキュリティ（Google の認証）は保ったまま、Snowflake へのログイン体験だけ楽になる」というのが狙いです。

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

## IdP を持たない組織でも使える

この仕組みは個人だけでなく、**Entra ID や Okta のような IdP を持っていない小さな組織**にも応用できます。

Snowflake 側に各メンバーのユーザーを作り、`EMAIL_ADDRESS` にそれぞれの Google アカウントを登録しておけば、メンバーは自分の Google アカウントで SSO ログインできます。

Google アカウントの管理さえしっかりしていれば、専用の IdP を立てずに Snowflake の SSO を導入できるわけです。

## まとめ

個人の無料 Gmail で、Snowflake にログインできました。

- Google Workspace（有料）は不要。個人 Gmail で動く。
- MFA コードやパスキーの手間なく、スマホからでも Gmail ログインだけでサッと入れる。認証は Google 側に委ねられる。
- IdP を持たない組織でも、Gmail をユーザーに登録すれば同じように SSO を導入できる。

高いセキュリティを保ったまま、個人検証環境のログインが一気に楽になります。

## 参考リンク

- [July 20, 2026: OIDC federated authentication (Public Preview)](https://docs.snowflake.com/en/release-notes/2026/other/2026-07-17-oidc-federated-authentication-preview)
- [CREATE SECURITY INTEGRATION (OIDC) | Snowflake Documentation](https://docs.snowflake.com/en/sql-reference/sql/create-security-integration-oidc)
- [Overview of federated authentication and SSO | Snowflake Documentation](https://docs.snowflake.com/en/user-guide/admin-security-fed-auth-overview)
