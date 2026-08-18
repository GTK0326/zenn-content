---
title: "Power BI の接続元 IP を自分で管理しない — Databricks の Partner Platforms 許可"
emoji: "🔌"
type: "tech"
topics: ["databricks", "security", "powerbi", "dbt", "network"]
published: true
---

## この記事で分かること

Databricks に Power BI や dbt Cloud から接続していて、接続元 IP の許可設定に悩んでいる方に向けた記事です。

2026 年 7 月にベータ提供が始まった Partner Platforms を使うと、IP 範囲そのものを書かずに「Power BI の日本リージョンから」という指定で許可できます。

公式リリースノート: [Databricks Documentation](https://docs.databricks.com/aws/en/release-notes/product/2026/july)

:::message
内容は記事作成時点のものです。
仕様は変更され得るため、最終的には最新の公式ドキュメントで確認ください。
:::


![](/images/d11-allowlist-partner-platform-ips-in-context-base/cover.png)

## SaaS の接続元 IP は、追いかけ続けないと壊れる

Databricks へのアクセスを絞りたい、というのはよくある要件です。

社内ネットワークからだけ許可する、といった制御は IP アクセスリストで実現できます。

問題は、BI ツールが SaaS になったときです。

Power BI Service から Databricks に接続する場合、接続元は利用者の PC ではなく Microsoft のデータセンターです。

その IP 範囲は Microsoft 側の都合で変わります。

こちらが気づかないうちに範囲が変われば、ある日突然レポートが更新されなくなります。

かといって全公開 IP を許可すれば、アクセスを絞った意味がありません。

提供元の IP 一覧を定期的に確認して CIDR を追記し続ける運用は、現実的とは言えません。

## Partner Platforms は IP リストの管理先を Databricks に移す

Partner Platforms は、ネットワークソースとしてサービス名そのものを指定できるようにする機能です。

リリースノートにはこう書かれています。

> Select a partner platform to allowlist the IPs that third-party apps use to connect
> to Databricks, including Power BI, Tableau Cloud, and dbt platform.
> **Databricks manages and updates these IP lists automatically.**

管理者がやることは「Power BI を許可する」と宣言することだけです。

実際の IP 範囲を保持し、変更に追従する責任は Databricks 側に移ります。

### 現在対応しているサービス

- Microsoft Power BI
- Tableau Cloud
- dbt Cloud

![パートナー名の選択肢](/images/d11-allowlist-partner-platform-ips/partner-list.png)

ベータ提供の段階なので、今後の拡充に期待したいところです。

### リージョン単位で絞り込める

パートナーを選ぶと、次にリージョンを選べます。

「全世界の Power BI」ではなく「日本リージョンの Power BI だけ」に絞れます。

リージョンの選択肢はパートナーごとに用意されており、提供元のホスティング構成を反映しています。

Power BI であれば `Japan East` や `Japan West` を選べます。

![リージョンの選択肢](/images/d11-allowlist-partner-platform-ips/region-list.png)

選択欄は検索できるため、`Japan` と入力すれば該当するリージョンだけに絞り込めます。


## 許可の指定が「IP 範囲」から「サービスとリージョン」に変わる

この機能の本質は、許可ルールの書き方が変わることです。

**従来（IP アクセスリスト）**

```
20.37.194.0/24
20.190.128.0/18
40.126.0.0/18
... （提供元が公開する範囲をすべて列挙し、変更のたびに更新）
```

**Partner Platforms**

```
Microsoft Power BI / Japan East, Japan West
```

書く内容が「変わり続ける値」から「変わらない名前」になります。

IP 範囲は Databricks が保持するため、提供元が範囲を変えても設定を直す必要がありません。

## 実際に動かしてみよう

「日本の Power BI から Databricks に接続する」という想定で設定します。

いきなり遮断すると自分が締め出される可能性があるため、**ドライランモードで作成します**。

ドライランモードは、ルールに違反したリクエストを記録するだけで、実際には遮断しません。

公式の説明は次のとおりです。

> Dry run mode for all products: Databricks logs violations but does not block requests.
> Use this mode to evaluate policy impact before enforcing.

影響を確認してから強制モードに切り替える、という順序で進めます。

:::details 前提条件

- Enterprise ティアのアカウント
- アカウント管理者権限
- アカウントコンソールのプレビュー「Context-based ingress beta features」が有効

:::

### ステップ1: 許可するリージョンを決める

Power BI のテナントは、組織が最初にサインアップしたときの国によってデータの保存先が決まります。

日本で登録した組織であれば `Japan East` または `Japan West` です。

利用者が任意に選び直すことはできず、変更には Microsoft サポートへの依頼が必要です。

参照: [Power BI テナントのリージョン](https://learn.microsoft.com/ja-jp/power-bi/support/service-admin-where-is-my-tenant-located)

今回は日本の組織を想定し、両方を許可します。

### ステップ2: ポリシーを作成する

アカウントコンソール → セキュリティ → ネットワーキング → コンテキストベースのイングレスとエグレス → 新しいネットワークポリシーを作成

| 項目 | 設定値 |
|---|---|
| ポリシー名 | `verify-powerbi-japan` |
| ポリシー強制モード | **ドライランモード** |
| パブリックネットワークアクセス | 「すべてのパブリック IP からのアクセスを許可」をオフ |


### ステップ3: 許可ルールを追加する

| 項目 | 設定値 |
|---|---|
| ラベル | `allow-powerbi-japan` |
| ID | すべてのユーザーとサービスプリンシパル |
| ソースタイプ | パートナープラットフォーム |
| パートナー名 | Microsoft Power BI |
| リージョン | Japan East / Japan West |
| 配信先 | ワークスペース UI、ワークスペース API |

リージョン欄は検索できるため、`Japan` と入力すれば該当する 2 つに絞り込めます。

![許可ルールの設定内容](/images/d11-allowlist-partner-platform-ips/allow-rule.png)

保存すると、ポリシーの詳細画面に許可ルールが 1 件表示されます。

![保存後のポリシー](/images/d11-allowlist-partner-platform-ips/policy-saved.png)

実行結果:
```
allow-powerbi-japan
  すべてのユーザーとサービスプリンシパルの許可
  元：Microsoft Power BI (Japan East, Japan West)  配信先：ワークスペースUI, API
```

強制モードが「ドライランモード」であること、アタッチ先が「未選択」であることも同じ画面で確認できます。


### ステップ4: API で実体を確認する

UI で作成したポリシーを API から取得します。

```
GET /api/2.0/accounts/<account-id>/network-policies/verify-powerbi-japan
```

実行結果:
```json
{
  "ingress_dry_run": {
    "public_access": {
      "restriction_mode": "RESTRICTED_ACCESS",
      "allow_rules": [
        {
          "label": "allow-powerbi-japan",
          "origin": {
            "managed_ip_range": {
              "name": "powerbi",
              "regions": ["JapanEast", "JapanWest"]
            }
          },
          "destination": {
            "workspace_ui": { "all_destinations": true },
            "workspace_api": {
              "scope_qualifier": "API_SCOPE_QUALIFIER_ALL",
              "scopes": ["all-apis"]
            }
          }
        }
      ]
    }
  }
}
```

パートナーとリージョンの指定が `managed_ip_range` として保存されていることが確認できます。

### ステップ5: ドライランの記録を確認する

ドライランで検出された違反は `system.access.inbound_network` システムテーブルに記録されます。

```sql
SELECT event_time, rule_label, authenticated_as, source.ip, request_path, policy_outcome
FROM system.access.inbound_network
WHERE policy_outcome = 'DRY_RUN_DENIAL'
ORDER BY event_time DESC
LIMIT 20;
```

`policy_outcome` に入る値は `DENIED` と `DRY_RUN_DENIAL` の 2 つです。

`DRY_RUN_DENIAL` は「実際には通したが、強制モードなら拒否していた」記録です。

このポリシーをワークスペースにアタッチした状態で、自分の環境から SQL を実行してみました。

許可したのは Power BI の日本リージョンだけなので、自宅から叩いた API は本来なら拒否対象です。

実行結果:
```
event_time            2026-08-18T13:55:33.000Z
rule_label            NULL
authenticated_as      xxxxx@example.com
source.ip             125.192.xxx.xxx
request_path          /api/2.0/sql/statements
policy_outcome        DRY_RUN_DENIAL
```

（`authenticated_as` と `source.ip` は伏せています）

`policy_outcome` が `DRY_RUN_DENIAL` になっており、強制モードなら拒否されていたことが分かります。

一方でこの SQL 自体は成功しています。ドライランは記録するだけで遮断しない、という挙動をそのまま確認できました。

`rule_label` が `NULL` なのは、拒否ルールに合致したのではなく、どの許可ルールにも当てはまらなかったためです。

拒否ルールを明示的に作った場合は、そのラベルがここに入ります。

なお記録が現れるまでには数分のラグがありました。設定直後にクエリしても 0 件のことがあります。

実際に運用する場合は、ここに想定外のアクセスが出ていないことを確認してから強制モードへ切り替えます。

## 検証コード

この記事のハンズオンで使用した SQL を Notebook（.ipynb）形式で公開しています。

Databricks にインポートすれば、そのまま自分の環境で実行できます。

[📓 検証ノートブックを開く（GitHub）](https://github.com/GTK0326/zenn-content/blob/main/notebooks/d11-allowlist-partner-platform-ips-in-context-base.ipynb)
## まとめ

IP 範囲の管理が現実的でないという理由で、アクセス制限そのものを見送っていた環境もあると思います。

Partner Platforms を使えば、サービス名とリージョンを指定するだけで接続元を絞れます。

制限をあきらめずに済むようになったので、ぜひ使っていきたい機能です。

## 参考リンク

- [Databricks release notes (July 2026)](https://docs.databricks.com/aws/en/release-notes/product/2026/july)
- [Context-based ingress control](https://docs.databricks.com/aws/en/security/network/front-end/context-based-ingress)
- [Context-based network policies](https://docs.databricks.com/aws/en/security/network/context-based-policies)