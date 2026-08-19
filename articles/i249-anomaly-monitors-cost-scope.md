---
title: "Snowflakeのコスト異常検知をスコープ別に設定できるようになった——Anomaly Monitors（Preview）を実機検証"
emoji: "🚨"
type: "tech"
topics: ["snowflake", "cost", "finops", "sql", "monitoring"]
published: true
---

## この記事で分かること

Snowflake でコスト管理をしている方向けの記事です。

10.29 で Preview 公開された anomaly monitors を使うと、これまでアカウント全体・組織全体しか選べなかったコスト異常検知のスコープを、自分で定義できるようになります。

本記事では `ANOMALY_INSIGHTS` クラスと Snowsight の両方から monitor を作り、次の3点を実機ベースで示します。

- 閾値を設定しなくても判定レンジが自動で引かれること
- `service_types` がクレジット種別で分かれていること
- タグでスコープを切るには Snowsight が要ること

公式リリースノート: [Snowflake Documentation](https://docs.snowflake.com/en/release-notes/2026/10_29#anomaly-monitors-for-cost-anomalies-preview)

:::message
内容は記事作成時点のものです。
仕様は変更され得るため、最終的には最新の公式ドキュメントで確認ください。
:::

![](/images/i249-anomaly-monitors-cost-scope/cover.png)

## コストを監視する3つの機能と、それぞれの課題

Snowflake でコストを監視する機能は、Resource Monitor、Budget、コスト異常検知の3つです。この3つは「何を基準に異常とみなすか」がそれぞれ違います。

| 機能 | 異常とみなす基準 | 監視対象に指定できるもの | 設定できるレベル |
| --- | --- | --- | --- |
| Resource Monitor | 自分で決めたクレジット上限の消化率 | ウェアハウス | アカウント / ウェアハウス |
| Budget | 自分で決めた月次の支出上限 | ウェアハウス・DB・テーブルなど | アカウント / カスタムグループ |
| コスト異常検知 | 過去の消費傾向から算出した予測レンジ | 指定できない（消費全体が対象） | アカウント / 組織 |

3つめは [Snowflake のドキュメント](https://docs.snowflake.com/en/user-guide/cost-anomalies)では cost anomalies として説明されています。本記事では「コスト異常検知」と呼びます。

この3つは、機能そのものと使い方の両面で課題を抱えていました。順に見ていきます。

### 課題1: Resource Monitor と Budget は、閾値を運用で更新し続ける

上の2つは、監視対象を自由に選べる代わりに、基準となる数値を人間が決めます。[Snowflake のドキュメント](https://docs.snowflake.com/en/user-guide/budgets)にも、Budget は支出上限を自分で設定すると書かれています。

普通の環境では、利用は少しずつ伸びていきます。そのたびに、去年決めた上限は今年の実態に合わなくなります。上限を上げすぎれば検知しなくなり、上げなさすぎればアラートが鳴り続けます。

### 課題2: コスト異常検知は閾値が要らない代わりに、粒度が粗かった

3つめのコスト異常検知だけは、考え方が違います。過去の消費実績からアルゴリズムがその日の期待レンジを算出し、そこから外れたときを異常とみなします。

利用者が閾値を決める必要はなく、利用が伸びれば期待レンジも一緒に伸びます。運用でメンテナンスするものが無いという点で、課題1を持っていません。

ただし、スコープはアカウント全体か組織全体の2択でした。ここが使いどころを狭めていました。

たとえば月間 3,000 クレジットを消費するアカウントで、あるプロジェクトが 30 クレジット余計に使ったとします。そのプロジェクト単体では消費が 3 倍になっているかもしれません。しかしアカウント全体で見れば 1% の増加なので、期待レンジの中に収まってしまいます。

つまり「閾値の運用が要らない検知」と「チーム単位の粒度」が、これまで両立していませんでした。

## Anomaly Monitors がスコープの制約を外す

10.29 で Preview 公開された anomaly monitor が、この3つめを拡張します。コスト異常検知の検知アルゴリズムをそのまま使い、スコープだけを自分で定義できるようにした機能です。先ほどの2つの課題は、それぞれ次のように解消します。

| これまでの課題 | anomaly monitor では |
| --- | --- |
| 課題1: 閾値を人が決めて更新し続ける | 決めるのはスコープだけ |
| 課題2: スコープがアカウント/組織のみ | サービスタイプとタグで定義できる |

課題1 については、検知の仕組みがコスト異常検知と同じなので、閾値は引き続き決めなくて済みます。決めるのは「どこを見るか」だけです。

課題2 が外れると、先ほどの 30 クレジットの例が変わります。プロジェクト単位のスコープで見れば、同じ消費が 3 倍の逸脱として扱われます。全体に埋もれていたスパイクが、そのまま検知対象になります。

加えて、通知リストが monitor ごとに独立します。アカウント共通の宛先に全チームのアラートが飛ぶ状態は、何度か続くと誰もメールを開かなくなります。

## スコープ定義の決め方

monitor を作るときに決めるのは、次の2つです。どちらも必須で、この組み合わせが監視範囲になります。

| 決めること | 選択肢 | 何を選ぶか |
| --- | --- | --- |
| 追跡するクレジット | `CREDITS` / `AI_CREDITS` | どちらか一方だけ |
| スコープ | `service_types` / `resource_tags` | 片方でも両方でもよい |

### 追跡するクレジットは一方だけ

1つの monitor が追跡するのは `CREDITS` か `AI_CREDITS` のどちらか一方で、両方をまとめた monitor は作れません。

同じチームを両方のクレジットで見張りたい場合は、monitor を2つ作ることになります。この制約の理由はハマりポイントに書きました。

### スコープはサービスタイプとタグで決める

`service_types` と `resource_tags` は、両方を指定するとその積集合が対象になります。どちらを主軸にするかは、監視したい単位が「サービスの種類」なのか「組織の区切り」なのかで決まります。

| 軸 | 向いている場合 | 前提 |
| --- | --- | --- |
| `service_types` | 課金の発生源を見張りたい | なし。その日から使える |
| `resource_tags` | 誰の消費かで切りたい | タグ付与の運用が定着していること |

サービスタイプ軸は、サーバーレスタスクの暴走や AI 機能の使いすぎのように、原因の種類がそのまま対処に直結する場合に向きます。タグ軸は事業部門別・コストセンター別のように、誰に連絡するかで切りたい場合に向きます。

タグ軸は、新しいウェアハウスにタグを付ける運用が回っていないと漏れます。逆に回っていれば、タグを付けた時点で自動的に監視対象に入ります。

なお、この2つは作成手段が分かれます。サービスタイプ軸は SQL からも Snowsight からも作れますが、タグ軸は Snowsight から作る必要がありました。

## 実際に動かしてみよう

まずは、通常のクレジットをサービスタイプ軸で監視する monitor を1つ作り、通知先を設定するところまでを通します。

`ANOMALY_INSIGHTS` のインスタンスは `SNOWFLAKE.LOCAL` に用意されています。自分でインスタンスを作る必要はありません。

:::details 前提条件（クリックで展開）

```sql
-- ACCOUNTADMIN で実行しています
USE ROLE accountadmin;
```

検証は Standard Edition のアカウントで行いました。
`ANOMALY_INSIGHTS` クラスのメソッドはアカウント全体の使用状況を参照するため、
実行ロールにはアカウント使用状況への参照権限が必要です。
:::

### ステップ1: monitor を作成する

`CREATE_MONITOR` に渡すのは、monitor の別名と設定オブジェクトの2つです。設定はすべて第2引数の `OBJECT` にまとめます。

```sql
-- ウェアハウス消費を対象にした monitor を作る
CALL snowflake.local.anomaly_insights!CREATE_MONITOR(
    'ML_WH_MONITOR',
    OBJECT_CONSTRUCT(
        'credit_family', 'CREDITS',
        'service_types', ARRAY_CONSTRUCT('WAREHOUSE_METERING')
    )
);
```

実行結果:
```
+---------------------------------+
| CREATE_MONITOR                  |
|---------------------------------|
| {                               |
|   "credit_family": "CREDITS",   |
|   "resource_tags": {            |
|     "operator": "UNION",        |
|     "tags": []                  |
|   },                            |
|   "service_types": [            |
|     "WAREHOUSE_METERING"        |
|   ]                             |
| }                               |
+---------------------------------+
```

戻り値は正規化された設定オブジェクトです。指定しなかった `resource_tags` に既定値が埋まっており、タグでの絞り込みは無し、という状態になっています。

`credit_family` は省略できず、`service_types` に空配列を渡すこともできません。

### ステップ2: 判定結果を確認する

monitor が実際にどう判定しているかを見ます。

```sql
-- monitor 単位の日次判定結果。開始日と終了日を渡す
CALL snowflake.local.anomaly_insights!GET_MONITOR_ANOMALIES(
    'ML_WH_MONITOR', '2026-08-01', '2026-08-15'
);
```

実行結果（一部を抜粋）:
```
+------------+-------------+---------------+------------------------+-------------+-------------+------------+
| USAGE_DATE | CONSUMPTION | CURRENCY_TYPE | FORECASTED_CONSUMPTION | UPPER_BOUND | LOWER_BOUND | IS_ANOMALY |
|------------+-------------+---------------+------------------------+-------------+-------------+------------|
| 2026-08-01 |       0.184 | CREDITS       |                  0.076 |       0.185 |       0.000 | False      |
| 2026-08-06 |       0.100 | CREDITS       |                  0.072 |       0.185 |       0.000 | False      |
| 2026-08-09 |       0.001 | CREDITS       |                  0.017 |       0.137 |       0.000 | False      |
| 2026-08-11 |       0.000 | CREDITS       |                 -0.015 |       0.105 |       0.000 | False      |
| 2026-08-15 |       0.017 | CREDITS       |                  0.012 |       0.127 |       0.000 | False      |
+------------+-------------+---------------+------------------------+-------------+-------------+------------+
```

`UPPER_BOUND` が 0.185 → 0.137 → 0.105 → 0.127 と、日ごとに動いています。この monitor には閾値を一切設定していません。`FORECASTED_CONSUMPTION` から判定レンジが自動で引き直され、消費が落ち着けばレンジも下がります。

作成直後から過去に遡って結果が返るので、30 日待つ必要はありませんでした。スコープが妥当かどうかを、その場で数字を見ながら判断できます。

### ステップ3: 通知先を設定する

monitor 専用の通知メールを設定します。

```sql
-- この monitor の異常だけを受け取る宛先
CALL snowflake.local.anomaly_insights!SET_MONITOR_NOTIFICATION_EMAILS(
    'ML_WH_MONITOR', 'ml-team@example.com'
);
```

第2引数は配列ではなく文字列です。この宛先はアカウント共通の通知先とは独立して管理されるので、チームごとにアラートを振り分けられます。

ここまでで、1つの monitor が運用に載る状態になりました。

### ステップ4: AI クレジット専用の monitor を作る

`credit_family` を `AI_CREDITS` に変えるだけです。

```sql
-- AI クレジットだけを追跡する monitor
CALL snowflake.local.anomaly_insights!CREATE_MONITOR(
    'AI_MONITOR',
    OBJECT_CONSTRUCT(
        'credit_family', 'AI_CREDITS',
        'service_types', ARRAY_CONSTRUCT('CORTEX_SEARCH', 'CORTEX_AGENTS', 'AI_FUNCTIONS')
    )
);
```

AI クレジット専用の monitor が作成できました。ステップ1 の通常クレジットと合わせて、両方を別々に監視できます。

### 番外: タグスコープは Snowsight から作る

ここまでのステップとは別の流れになります。タグでスコープを切る場合は SQL ではなく Snowsight を使うため、操作をひととおり載せておきます。

画面は Admin > Cost management > Anomalies タブです。

![Anomalies タブ](/images/i249-anomaly-monitors-cost-scope/anomalies-tab.png)

フィルタが並んでいます。

左から、追跡するクレジット種別、`Tags`、`Service types`、そして右端の歯車が monitor の操作メニューです。`Monitors` が `None` になっているのは、まだ monitor を選んでいない状態を表します。

`Tags` を開くと、タグ名から値へと辿れます。

![Tags ドロップダウン](/images/i249-anomaly-monitors-cost-scope/tags-dropdown.png)

ここに出てくるのは、**実際にコストが按分されたタグだけ**です。タグを作ってオブジェクトに付けただけでは出てきません。

検証中、タグを付与した直後は `N/A` しか表示されず、タグ付きのウェアハウスで実際にクエリを流して消費を発生させたあと、ようやく `analytics` / `bi` / `ml` が現れました。

タグ設計をした直後に画面を開いて「出てこない」となった場合は、そのタグの付いたリソースがまだ課金されていないことを疑ってください。

値を選ぶと、フィルタの表示が変わります。

![タグを選択した状態](/images/i249-anomaly-monitors-cost-scope/tags-selected.png)

`Apply` を押すまでは反映されません。ここで一度 `Apply` を押すと、そのスコープでの消費と判定結果がグラフに出ます。

**保存する前にスコープの妥当性を目で確認できる**のが、この画面の良いところです。

想定より広すぎる、狭すぎるといった調整を、保存前に済ませられます。

スコープが決まったら、右端の歯車を開きます。

![monitor 操作メニュー](/images/i249-anomaly-monitors-cost-scope/monitor-actions.png)

`Create new monitor from config` が有効になっています。その上の `Save changes` などが暗いままなのは、既存の monitor を選んでいないためです。

スコープを何も選んでいない状態だと `Create new monitor from config` も暗いままで、「Select at least one tag or service type」というヒントが出ます。つまり「全部を対象にした monitor」は作れません。

クリックすると名前を付ける画面になります。

![New monitor ダイアログ](/images/i249-anomaly-monitors-cost-scope/new-monitor-dialog.png)

通知先はここで設定できますが、任意です。後から SQL の `SET_MONITOR_NOTIFICATION_EMAILS` でも設定できます。

作成すると、`Monitors` が作った monitor の名前に変わります。

![作成後](/images/i249-anomaly-monitors-cost-scope/monitor-created.png)

作成した monitor は SQL 側からも見えます。

```sql
CALL snowflake.local.anomaly_insights!LIST_MONITORS();
```

実行結果:
```
+------------------+-------------------------------------+
| ALIAS            | CONFIG                              |
|------------------+-------------------------------------|
| GUI_TAG_MONITOR2 | {                                   |
|                  |   "resource_tags": {                |
|                  |     "operator": "UNION",            |
|                  |     "tags": [                       |
|                  |       {                             |
|                  |         "tagDatabase": "TAGGUI_DB", |
|                  |         "tagSchema": "TAGS",        |
|                  |         "tagName": "COST_CENTER",   |
|                  |         "tagValues": [              |
|                  |           "analytics"               |
|                  |         ]                           |
|                  |       }                             |
|                  |     ]                               |
|                  |   },                                |
|                  |   "service_types": []               |
|                  | }                                   |
+------------------+-------------------------------------+
```

`GET_MONITOR_ANOMALIES` も同じように使えます。読み取りは SQL、タグスコープの作成は Snowsight、という住み分けになります。

## 実際に検証してわかったハマりポイント

### 1. 通常クレジットと AI クレジットは合算できない

1つの monitor が追跡できるのは `CREDITS` か `AI_CREDITS` の一方だけです。両方をまとめた monitor は作れず、`credit_family` と `service_types` の家系が揃っていないと `INVALID_MONITOR_CONFIG` になります。

これは制約というより、単位の違うものを混ぜないための設計だと思います。[Snowflake のドキュメント](https://docs.snowflake.com/en/user-guide/snowflake-cortex/pricing)によると、Platform Credit はエディションとリージョンで単価が変わるのに対し、AI Credit は一律です。決まり方が違うので、足した数字を見ても打つべき手が決まりません。

ただし運用上は、1チームを見張るのに monitor が2つ必要になります。

### 2. 20 monitor の上限は、全社展開には足りない

monitor はアカウントあたり 20 個までです。1点目と組み合わせると効いてきます。

- 1チームを両方のクレジットで見張ると monitor を2個消費する
- つまり実質的に見られるのは 10 チーム程度

事業部門を数個切るだけなら足りますが、全社の各部署に配るとすぐ埋まります。チーム単位で切れるようになった機能なのに、チームが増えると足りなくなるのは惜しいところです。GA での緩和を期待しています。

### 3. タグスコープは SQL からは作れない

Snowsight を使ったのは、SQL から同じものを作れなかったためです。

タグだけをスコープにした monitor を、素直な書き方で作ろうとしました。

```sql
CALL snowflake.local.anomaly_insights!CREATE_MONITOR(
    'SQL_TAG_TEST',
    OBJECT_CONSTRUCT(
        'credit_family', 'CREDITS',
        'resource_tags', OBJECT_CONSTRUCT(
            'operator', 'UNION',
            'tags', ARRAY_CONSTRUCT(
                ARRAY_CONSTRUCT('TAGGUI_DB.TAGS.COST_CENTER', 'analytics')
            )
        )
    )
);
```

実行結果:
```
002001 (02000): SQL compilation error:
Object 'TAGGUI_DB.TAGS.COST_CENTER' does not exist or not authorized.
```

このタグは実在し、`SYSTEM$GET_TAG` では値が返ります。存在しないタグ名を渡しても同じエラーになるため、渡した名前の問題ではなさそうです。

[公式リリースノート](https://docs.snowflake.com/en/release-notes/2026/10_29#anomaly-monitors-for-cost-anomalies-preview)を読み返すと、タグからスコープを構築する話は Snowsight の段落にしか書かれていません。`ANOMALY_INSIGHTS` クラスの段落は monitor の作成・更新・参照・削除に触れるだけで、タグには触れていません。

Preview 時点では、タグスコープは Snowsight から設定するもの、と考えておくのがよさそうです。IaC でコスト監視まで管理したい場合、タグ軸の monitor だけは手作業が残ります。

ただし、これはリリース直後に触った結果です。SQL 側の実装が追いついていないだけ、という可能性は十分あります。導入を検討する段階では、最新の状況を確認してください。

### 補足: service_types に指定できる値

`service_types` に指定できる値の一覧が、公式ドキュメントに見当たりませんでした。

自分の環境で確認できた範囲では次の値が使えます。`credit_family` ごとに分かれています。

**`CREDITS` 側**

```
WAREHOUSE_METERING, AUTO_CLUSTERING, MATERIALIZED_VIEW, SEARCH_OPTIMIZATION,
PIPE, SNOWPIPE_STREAMING, REPLICATION, SERVERLESS_TASK, SERVERLESS_ALERTS,
QUERY_ACCELERATION, COPY_FILES, LOGGING, TELEMETRY_DATA_INGEST,
HYBRID_TABLE_REQUESTS, DATA_QUALITY_MONITORING, TRUST_CENTER,
SNOWPARK_CONTAINER_SERVICES, AI_SERVICES
```

**`AI_CREDITS` 側**

```
AI_FUNCTIONS, CORTEX_SEARCH, CORTEX_AGENTS, SNOWFLAKE_COCO,
SNOWFLAKE_COCO_CLI, SNOWFLAKE_COCO_SNOWSIGHT, SNOWFLAKE_COWORK
```

ウェアハウス以外にも、サーバーレスタスク・Snowpipe・レプリケーション・クエリアクセラレーションといった「気づいたら増えていた」系の消費を個別に見張れます。

名前から受ける印象と違うものが2つありました。

- `AI_SERVICES` は `AI_CREDITS` 側ではなく `CREDITS` 側の値
- `CORTEX_CODE_CLI` は使えず `SNOWFLAKE_COCO_CLI` を指定する

`METERING_HISTORY` の値をそのまま渡せるとは限らない、と考えておくのがよさそうです。

:::message
上記は自分の環境で確認できた範囲の一覧で、網羅を保証するものではありません。
Preview 中に増減する可能性もあります。
:::

## 検証コード

ハンズオンで使用した SQL を Jupyter Notebook（.ipynb）形式で公開しています。

Snowflake Notebooks にインポートして、そのまま自分の環境で実行できます。

[📓 検証ノートブックを開く（GitHub）](https://github.com/GTK0326/zenn-content/blob/main/notebooks/i249-anomaly-monitors-cost-scope.ipynb)

## まとめ

**この機能の位置づけ**

- コスト監視の機能が3つ、という構図は変わらない
- 変わったのはコスト異常検知のスコープ
- 2択の固定から、自分で定義できるようになった
- 検知アルゴリズムは従来と同じ。閾値の設定は不要

**使ってみての評価**

- サービスタイプ軸なら `CREATE_MONITOR` 1回で始まる
- 作成直後から過去に遡って結果が返る
- スコープの妥当性をその場で確認できる
- タグ軸は Snowsight から作る
- 保存前にグラフで確認できるぶん、画面のほうが分かりやすい
- 20 個の上限は全社展開には足りない。実質 10 チーム程度

## 参考リンク

- [Snowflake 10.29 Release Notes — Anomaly monitors for cost anomalies (Preview)](https://docs.snowflake.com/en/release-notes/2026/10_29#anomaly-monitors-for-cost-anomalies-preview)
- [Introduction to cost anomalies](https://docs.snowflake.com/en/user-guide/cost-anomalies)
- [Programmatically work with cost anomalies](https://docs.snowflake.com/en/user-guide/cost-anomalies-class)
- [Use Snowsight to work with cost anomalies](https://docs.snowflake.com/en/user-guide/cost-anomalies-ui)
- [ANOMALY_INSIGHTS class](https://docs.snowflake.com/en/sql-reference/classes/anomaly_insights)
- [Working with resource monitors](https://docs.snowflake.com/en/user-guide/resource-monitors)
- [Snowflake Budgets](https://docs.snowflake.com/en/user-guide/budgets)
- [Snowflake AI pricing](https://docs.snowflake.com/en/user-guide/snowflake-cortex/pricing)
- [Object tagging](https://docs.snowflake.com/en/user-guide/object-tagging)

