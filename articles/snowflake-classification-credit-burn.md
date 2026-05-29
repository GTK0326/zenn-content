---
title: "Snowflakeの自動分類を試したら$500消えた話——見落としがちな継続課金の罠"
emoji: "🔥"
type: "tech"
topics: ["snowflake", "dataengineering", "security", "cost", "datagovernance"]
published: false
---

## この記事で分かること

Snowflake の自動データ分類（Auto Classification）を有効にしたまま放置したら、思っていた以上のクレジットが消えていた。

自分の実体験として、何がまずかったのか・なぜそれほどコストが発生したのかを整理した。

同じ目に遭いたくない人向けの「やらかし防止チェックリスト」も末尾に載せているので、分類機能を試してみようとしている方はぜひ読んでほしい。

公式リリースノート: [Snowflake Documentation](https://docs.snowflake.com/en/user-guide/classify-intro)

:::message
内容は記事作成時点のものです。
仕様は変更され得るため、最終的には最新の公式ドキュメントで確認ください。
:::

## Snowflake の記事を書こうとしていただけなのに

ある日、Snowflake の**データ分類（Data Classification）**機能を試していた。

分類ポリシーを作り、テーブルにアタッチして、正しく動くことを確認して——それで作業を終えた、つもりだった。

翌日、コスト通知メールが来た。

前日比で数十倍のクレジット消費。

数字を見て最初は自分のミスだと思わなかったが、Cost Explorer で調べると、原因は明らかだった。

**前日に作った分類ポリシーが、自動スキャンを続けていた。**

最終的な損失は約 $500。

「ちょっと試しただけ」のつもりが、そのままにしておいたせいでこうなった。

## Snowflake の自動分類は「止まらない機能」

まず、自動分類の仕組みを整理しておく。

Snowflake の Data Classification は、テーブルの列をスキャンして PII（個人情報）や機密データを自動的にタグ付けする機能だ。

`SYSTEM$CLASSIFY` 関数を手動で呼ぶ方法と、スケジュールで自動実行させる方法がある。

問題になるのは後者——**アカウントレベルやスキーマレベルで AUTO CLASSIFICATION を有効にした場合**だ。

```sql
-- これをやると、対象オブジェクト全体に自動分類が走り続ける
ALTER SCHEMA my_db.my_schema SET AUTO CLASSIFY = TRUE;
```

一度有効にすると、Snowflake は指定スコープ内のテーブルをバックグラウンドで定期スキャンし続ける。

スキャンにはウェアハウスのコンピュートが使われる。

テーブルが増えるほど・スキャン頻度が高いほど、クレジットは積み上がる。

## 何が重なって $500 になったのか

当日の失敗を振り返ると、3つの要因が重なっていた。

**① 検証用に作ったテーブルが大量にあった**

機能検証のために CREATE TABLE を何度も繰り返していたため、スキャン対象が意図せず膨らんでいた。

テーブルが多いほどスキャンコストは線形以上に増える。

**② スケジュール間隔が短いままだった**

分類のトリガー設定が「変更検知（`TRIGGER_ON_CHANGES`）」になっており、テーブルを更新するたびに分類が走る設定だった。

```sql
-- 検証中に何度もINSERTしていたため、そのたびに分類がトリガーされた
ALTER TABLE my_db.my_schema.test_table
  SET AUTO CLASSIFY = TRUE
  DATA_CLASSIFICATION_REFRESH_INTERVAL = '1 MINUTE';  -- ← これが問題
```

**③ 作業終了時に設定を元に戻さなかった**

オブジェクトを DROP するとき、テーブルやスキーマは消したが、**アカウントレベルで有効にしていた分類ポリシーのアタッチが残っていた**。

スキーマを DROP してもポリシー自体はアカウントに残り、関連するスキャン設定が稼働し続けた。

## 分類機能で課金が発生する仕組み

Snowflake の自動分類は、内部的にコンピュートリソースを使ってテーブルデータをサンプリング・解析する。

この処理はユーザーが意識しない裏側で走るため、**Query History に明示的には出てこない**場合がある。

Cost Explorer の `CLASSIFICATION` カテゴリに積み上がる形で後から気づくことになる。

特にコストが跳ね上がりやすい条件はこれだ。

| 条件 | 影響 |
|---|---|
| テーブル数が多い | スキャン対象がそのまま増える |
| 更新頻度が高いテーブルへの `TRIGGER_ON_CHANGES` | 更新ごとに分類が走る |
| 列数・行数が多いテーブル | サンプリングコストが増大 |
| アカウントレベルの AUTO CLASSIFY | スコープが無制限に広がる |

## 再発防止のチェックリスト

自動分類を試す前後に確認してほしいこと。

**有効化する前**

```sql
-- 現在の分類設定状況を確認してから始める
SHOW DATA METRIC FUNCTIONS IN ACCOUNT;
SHOW CLASSIFICATION POLICIES IN ACCOUNT;

-- Cost Explorerで現在のベースラインコストを記録しておく
SELECT *
FROM SNOWFLAKE.ACCOUNT_USAGE.METERING_DAILY_HISTORY
WHERE DATE >= DATEADD('day', -3, CURRENT_DATE)
ORDER BY DATE DESC;
```

**有効化するときの鉄則**

- スコープは最小限に（アカウント・データベースレベルは避け、特定テーブルのみ）
- スケジュール間隔は `1 DAY` 以上から始める
- 検証専用のウェアハウスを使い、サイズを `XSMALL` に制限する

```sql
-- 悪い例: アカウント全体に適用
ALTER ACCOUNT SET ENABLE_AUTOMATIC_SENSITIVE_DATA_CLASSIFICATION = TRUE;

-- 良い例: 特定テーブルだけ、しかも手動実行
SELECT SYSTEM$CLASSIFY('my_db.my_schema.target_table', {});
```

**作業を終えるとき**

```sql
-- 有効にした設定は必ず元に戻す
ALTER SCHEMA my_db.my_schema UNSET AUTO CLASSIFY;
DROP CLASSIFICATION POLICY IF EXISTS my_db.my_schema.my_classification_policy;

-- アカウントレベルの設定も確認して戻す
ALTER ACCOUNT UNSET ENABLE_AUTOMATIC_SENSITIVE_DATA_CLASSIFICATION;

-- 費用が増えていないかその日のうちに確認
SELECT USAGE_DATE, CREDITS_USED, SERVICE_TYPE
FROM SNOWFLAKE.ACCOUNT_USAGE.METERING_DAILY_HISTORY
WHERE USAGE_DATE = CURRENT_DATE
ORDER BY CREDITS_USED DESC;
```

## まとめ：「試しただけ」が一番危ない

Snowflake の自動分類は便利な機能だが、有効にした瞬間から課金が始まる継続実行型の仕組みだ。

手動実行（`SYSTEM$CLASSIFY`）であれば都度課金で済むが、自動化すると「気づかないうちに積み上がる」リスクがある。

試す前に現在のコストをメモし、試した後は必ず設定を元に戻す——この2ステップだけで今回の失敗は防げた。

Snowflake の Budget 機能や Cost Anomaly Alert も合わせて設定しておくと、異常課金を早期に検知できるので、まだ設定していない方はこちらも参考にしてほしい。

公式ドキュメント: [Snowflake Data Classification](https://docs.snowflake.com/en/user-guide/classify-intro)
