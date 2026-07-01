---
title: "Snowflake CoCo × ontology-stack-builder でセマンティックレイヤーを自動構築する"
emoji: "🧅"
type: "tech"
topics: ["snowflake", "coco", "cortex", "semanticlayer", "dataengineering"]
published: true
---

## この記事について

本記事では、Snowflake Cortex Code（CoCo）の Community Skill である [`ontology-stack-builder`](https://github.com/Snowflake-Labs/coco-skills/tree/main/skills/ontology-stack-builder) を使って、セマンティックレイヤーを自動構築する手順を紹介します。

データソースには Snowflake のデモ用データセット **Tasty Bytes**（フードトラックビジネスのトランザクションデータ）を使用し、構築後は Cortex Agent に日本語で質問して回答精度を確認します。

:::message
**検証環境**
- CoCo CLI v1.1.24
- Snowflake 接続: `article-verify`
- デプロイ先: `FROSTBYTE_TASTY_BYTES.RAW_POS`
:::

:::message alert
**免責事項**
本記事の内容は筆者個人の環境での検証結果をまとめたものであり、特定の環境・バージョン・設定における動作を保証するものではありません。本番環境へ適用する際は、必ずご自身の環境で十分な検証を行った上でご判断ください。
:::

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/cover.png)

## ontology-stack-builder とは

Snowflake で AI を活用したデータ分析を実現するには、テーブル間の意味的なつながりを定義した「セマンティックレイヤー」が必要です。しかし、これを手作業で整備するのは複雑で時間がかかります。クラス定義・リレーション設計・抽象ビューの生成・セマンティックビューのYAML記述・Cortex Agent の設定——これらをすべて揃えて初めて、AI が「正しいコンテキストでクエリを生成できる」状態になります。

`ontology-stack-builder` はこのプロセスを自動化する CoCo Community Skill です。

> **You bring your Snowflake tables and business questions. The skill builds the rest — metadata, abstract views, semantic models, and a Cortex Agent — in a single conversational session.**

### 必要なもの

| 項目 | 説明 |
|------|------|
| ソーステーブル | 既存の Snowflake テーブル（スキーマ解析の対象） |
| ビジネス質問 | 「誰が何をどのくらい買ったか？」などの自然言語での問い |
| オントロジー名 | 生成されるオブジェクトのプレフィックス（例: `MY_ONTOLOGY`） |
| パス選択 | *Direct table path*（データ移動なし）または *Knowledge Graph path*（物理グラフテーブル生成） |

### 自動で作られるもの

下表はスキルが生成する 5 層のアーキテクチャです。今回は「Direct table path × 既存セマンティックビューなし」のパターンで実行しました。

| レイヤー | 生成されるアーティファクト |
|---------|--------------------------|
| **L1 Physical Storage** | ソーステーブルへの軽量ラッパービュー（V_{CLASS} エンティティビュー・V_{REL} リレーションビュー） |
| **L2 Ontology Metadata** | クラス定義・プロパティ・関係を管理する約 22 個の ONT_* メタデータテーブル（シードデータ込み） |
| **L3 Abstract Views** | クラスごとの抽象ビュー（VW_ONT_{CLASS}）・全エンティティ統合ビュー・階層ビュー・ビュー生成ストアドプロシージャ |
| **L4 Semantic Views** | Cortex Analyst 用セマンティックビュー 3種（Base / Ontology / Metadata） |
| **L5 Cortex Agent** | 3 ツールを持つインテントルーティング型 Cortex Agent |

L4 の 3 種類のセマンティックビューはそれぞれ目的が異なります。

| セマンティックビュー | 役割 |
|---------------------|------|
| `{NAME}_BASE` | ソーステーブルへの直接クエリ（エンティティ検索・集計） |
| `{NAME}_ONTOLOGY_MODEL` | テーブル間リレーション・エンティティ分布（抽象的な推論） |
| `{NAME}_METADATA_MODEL` | クラス定義・プロパティ・ビジネスメタデータ（ガバナンス・検索） |

L5 の Cortex Agent はこれら 3 つをツールとして持ち、質問の意図に応じて自動ルーティングします。**AI がクエリを生成する際、「どのテーブルに何があるか」だけでなく「テーブル同士がどう意味的につながっているか」を理解した上で回答する**——これが ontology-stack-builder が実現する価値です。

## 検証に使うデータ：Tasty Bytes

Tasty Bytes はフードトラックビジネスをモデル化した Snowflake 公式デモデータセットです。`FROSTBYTE_TASTY_BYTES` データベースに以下の 8 テーブルが含まれます。

| テーブル | スキーマ | 行数 |
|---------|---------|------|
| CUSTOMER_LOYALTY | RAW_CUSTOMER | 222,540 |
| COUNTRY | RAW_POS | 30 |
| FRANCHISE | RAW_POS | 335 |
| TRUCK | RAW_POS | 450 |
| MENU | RAW_POS | 100 |
| LOCATION | RAW_POS | 13,093 |
| ORDER_HEADER | RAW_POS | 248,201,269 |
| ORDER_DETAIL | RAW_POS | 673,655,465 |

トランザクション・マスタ・顧客・ロケーションとデータが役割ごとに分かれており、スキーマとしてはありがちな実運用データの構造に近いサイズ感です。このデータに対して FK 解析からオントロジー設計まで自動化できるか検証します。

## スキルのインストール

CoCo を起動し、`find-skill` スキルで `ontology-stack-builder` を GitHub から取得します。

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/01.png)

インストールが完了すると、`cortex skill list` で全 15 スキルの一覧が確認できます。`ontology-stack-builder` が正しく登録されていることを確認します。

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/02.png)

## Phase 1 — インプット収集・確認

スキルへの入力は、以下のプロンプトで行います。

```
Use ontology-stack-builder skill. Build an ontology stack on FROSTBYTE_TASTY_BYTES using these inputs:

Database: FROSTBYTE_TASTY_BYTES, Schema: RAW_CUSTOMER, RAW_POS
Source tables: CUSTOMER_LOYALTY, COUNTRY, FRANCHISE, LOCATION, MENU, ORDER_DETAIL, ORDER_HEADER, TRUCK
Ontology name: MY_ONTOLOGY
Path: Direct table path
Business questions: What products does each customer buy? How are customers segmented?
Semantic views: Ontology + Metadata
```

Phase 1 では、スキルがこれらの入力内容を収集・検証します。具体的には、指定されたスキーマに **既存のセマンティックビューが存在するか** を確認し、あれば再利用するか新規作成するかを問います（Phase 4.5 をスキップできます）。今回は既存ビュー 0 件だったため、新規作成に進みます。

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/04.png)

## Phase 2 — オントロジー設計

ソーステーブルのカラム定義と外部キーを解析し、**クラス（エンティティ種別）** と **リレーション（クラス間の関係）** を自動設計します。

FK カバレッジが 100% であることを確認してゲートを通過します。このフェーズは、後続フェーズで生成する SQL やセマンティックビューの設計図になるため、ここでレビューして修正できます。

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/05.png)

## Phase 3 — 構造の可視化・確認

設計したオントロジーをビジュアライズして最終確認するフェーズです。Streamlit が利用可能な環境ではインタラクティブなグラフエディターが起動し、クラス・リレーションをGUIで追加・削除・修正できます。

今回の CoCo CLI 環境では Streamlit の自動起動が利用できなかったため、スキルは ASCII テキストでクラス階層とリレーション構造を表示しました。

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/06.png)

スキル内には Streamlit ビジュアライザーのスクリプト（`visualize_ontology.py`）も含まれています。CoCo CLI セッションとは別に手動実行することで、同じオントロジーをグラフ構造で確認できます。

```bash
pip install streamlit streamlit-agraph pyyaml
python -m streamlit run visualize_ontology.py
```

8 クラス（オレンジ）と 6 リレーション（パープル）が正しくグラフ化されていることが確認できます。

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/13.png)

## Phase 4 — SQL 生成・デプロイ

承認済みのオントロジー設計をもとに、L1〜L3 の全アーティファクトに対する SQL を生成します。生成後に完全性チェックを実行し、問題がなければ Snowflake へデプロイします。

生成される SQL ファイルは 4 種類です。

| ファイル名 | 生成されるオブジェクト |
|-----------|---------------------|
| `02_concrete_views.sql` | エンティティ・リレーションの軽量ビュー（L1）8個 |
| `03_metadata_tables.sql` | クラス定義・プロパティ・関係を管理するメタデータテーブル（L2）22個 |
| `04_abstract_views.sql` | 各クラスの抽象ビュー・統合ビュー・階層ビュー（L3）10個 + ユニファイドビュー2個 |
| `05_view_generator_sp.sql` | オントロジービュー生成ストアドプロシージャ（L3）1個 |

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/07.png)

デプロイ完了後、合計 **46 Snowflake オブジェクト** が作成されます。

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/08.png)

## Phase 4.5 — ベースセマンティックビューの確保

Phase 1 で既存のセマンティックビューが見つからなかったため、このフェーズでネイティブスキル `semantic-view` に委譲してベースセマンティックビューを新規作成します。

ベースセマンティックビュー（`MY_ONTOLOGY_BASE`）は、ソーステーブルに対して直接クエリする際の基盤となります。作成後、Phase 1 で入力したビジネス質問 2 件で動作確認を行い、いずれも正しく回答されたことを確認します。

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/09.png)

## Phase 5 — オントロジーセマンティックビューの作成

ネイティブスキル `semantic-view` に委譲して、Phase 4 でデプロイしたオブジェクト群（L2 メタデータテーブル・L3 抽象ビュー）を対象としたオントロジーレイヤーのセマンティックビューを作成します。

最終的に 3 種類のセマンティックビューが揃います。

| セマンティックビュー | 対象 | 用途 |
|--------------------|------|------|
| `MY_ONTOLOGY_BASE` | ソーステーブル（8つ） | エンティティ検索・集計クエリ |
| `MY_ONTOLOGY_ONTOLOGY_MODEL` | 抽象ビュー（3つ） | クラス間リレーション・分布確認 |
| `MY_ONTOLOGY_METADATA_MODEL` | メタデータテーブル（4つ） | クラス定義・プロパティ・ビジネス用語 |

作成後、Phase 1 のビジネス質問でそれぞれのビューをテストします。

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/10.png)

## Phase 6 — Cortex Agent 作成

ネイティブスキル `cortex-agent` に委譲して、3 つのセマンティックビューをツールとして持つ **インテントルーティング型 Cortex Agent** を作成します。

| ツール名 | セマンティックビュー | ルーティング条件 |
|---------|------------------|----------------|
| `base_query_tool` | MY_ONTOLOGY_BASE | エンティティ検索・集計 |
| `ontology_query_tool` | MY_ONTOLOGY_ONTOLOGY_MODEL | リレーション探索・分布 |
| `metadata_query_tool` | MY_ONTOLOGY_METADATA_MODEL | クラス定義・プロパティ参照 |

Agent は質問の意図を解析し、適切なツール（または複数ツールの組み合わせ）に自動ルーティングします。

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/11.png)

## Phase 7 — E2E 検証

L1〜L5 の全レイヤーについて行数確認・サンプルクエリ・セマンティックビューチェック・エージェントの E2E テストを実行します。全レイヤーが ✓ になれば構築完了です。

**最終デプロイ成果物:** 46 Snowflake オブジェクト + 3 セマンティックビュー + 1 Cortex Agent

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/12.png)

## Cortex Agent への質問検証

構築した `MY_ONTOLOGY_AGENT` に 3 段階の難易度で質問します。確認したいのは「具体的な数字が合っているか」ではなく、**Agent が適切な SQL を自動生成し、意図したルーティングができているか** です。

---

### Q1（シンプル）: メニューカテゴリ別の売上集計

> 売上合計と注文件数をメニューカテゴリ別に教えて

この質問に答えるには `ORDER_DETAIL`（明細）と `MENU`（メニューマスタ）を結合する SQL が必要です。Agent は `base_query_tool` を使い、この結合を含む CTE 付きの集計クエリを自動生成して正しく回答しました。

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/ex1.png)

---

### Q2（中程度）: 特定顧客の購買推移

> 顧客である Rodolfo Tucker 氏の月別注文回数と累計購入金額の推移を教えて

`CUSTOMER_LOYALTY`・`ORDER_HEADER`・`ORDER_DETAIL` の 3 テーブル結合に加え、月次集計と累計計算が必要な質問です。Agent は適切な SQL を生成しただけでなく、推移を直感的に把握しやすい**棒グラフ（月別注文回数）＋折れ線グラフ（累計購入金額）の複合グラフ**を自動で選択して表示しました。

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/ex2.png)

---

### Q3（複雑）: フランチャイズ別売上 × 上位メニュー

> フランチャイズオーナーごとの総売上と、配下のトラックで最も売れたメニュー上位 3 品をまとめて教えて

`FRANCHISE` → `TRUCK` → `ORDER_HEADER` → `ORDER_DETAIL` → `MENU` という 5 テーブルを横断するネスト集計が必要な、難易度の高い質問です。Agent はこれを適切に処理し、336 フランチャイズ全員について正しく集計した結果を返しました。

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/ex3.png)

Q3 の Agent Details パネルを確認すると、Key data objects として今回構築した **`MY_ONTOLOGY_METADATA_MODEL`** が参照されていることが分かります。Agent は単に SQL を生成しているのではなく、オントロジーモデルとメタデータモデルを参照して「テーブル間の意味的なつながり」を理解した上でクエリを構築していることが確認できます。

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/ex3-1.png)

## まとめ

### 作られたオブジェクトの整理

今回 `ontology-stack-builder` が作成したセマンティック系の成果物は以下のとおりです。

| 種別 | 作られたもの |
|------|------------|
| Snowflake オブジェクト（L1〜L3） | 46 個（コンクリートビュー・抽象ビュー・メタデータテーブル・SP） |
| セマンティックビュー（L4） | 3 個（Base / Ontology / Metadata） |
| Cortex Agent（L5） | 1 個（3 ツール、インテントルーティング） |

特に L4 のセマンティックビューが「ソーステーブルへの直接クエリ用」「リレーション・分布の抽象的推論用」「クラス定義・ビジネス用語のガバナンス用」と役割ごとに明確に分かれて生成されたのが印象的でした。このように目的別に整理されたセマンティックレイヤーがあることで、Agent のルーティング精度が上がり、かつ人間にとっても理解・保守しやすい構造になっています。

### 構築にかかった時間

実際の作業時間は **約 10 分**程度でした。作業のほとんどは各フェーズのゲートで「内容を確認して承認ボタンを押す」だけです。スキーマ解析・SQL生成・デプロイ・セマンティックビュー作成・Agent 設定まですべてスキルが自動で行います。

### 自然言語での問い合わせ

構築後は Cortex Agent に自然言語で質問するだけで、複数テーブルを結合した集計クエリが自動生成されます。今回の検証では、単純な集計から 5 テーブル横断のネスト集計まで、いずれも適切に回答されることを確認できました。Agent Details からは、今回定義したオントロジーモデル・メタデータモデルが実際に Key data objects として機能していることも確認できています。

データエンジニアリングのコストを下げながら AI の回答精度を上げるための基盤として、`ontology-stack-builder` は非常に実用的なスキルだと感じました。本番環境への適用も十分検討できると思います。ぜひ試してみてください。
