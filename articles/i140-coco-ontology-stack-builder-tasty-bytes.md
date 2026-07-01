---
title: "CoCo × ontology-stack-builder でセマンティックレイヤーを自動構築する"
emoji: "🧅"
type: "tech"
topics: ["snowflake", "coco", "cortex", "semanticlayer", "dataengineering"]
published: false
---

## この記事について

本記事では、Snowflake Cortex Code（CoCo）の Community Skill である [`ontology-stack-builder`](https://github.com/Snowflake-Labs/coco-skills/tree/main/skills/ontology-stack-builder) を使って、**チャットプロンプト1つ**でセマンティックレイヤーを自動構築する手順を紹介します。

データソースには Snowflake のデモ用データセット **Tasty Bytes**（フードトラックビジネスのトランザクションデータ）を使用し、構築後は Cortex Agent に日本語で質問して回答精度を確認します。

:::message
**検証環境**
- CoCo CLI v1.1.24
- Snowflake 接続: `article-verify`
- デプロイ先: `FROSTBYTE_TASTY_BYTES.ONTOLOGY`
:::

:::message alert
**免責事項**
本記事の内容は筆者個人の環境での検証結果をまとめたものであり、特定の環境・バージョン・設定における動作を保証するものではありません。本番環境へ適用する際は、必ずご自身の環境で十分な検証を行った上でご判断ください。
:::

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/cover.png)

## ontology-stack-builder とは

`ontology-stack-builder` は Snowflake Labs が公開している CoCo 向けの Community Skill です。ソーステーブルを指定するだけで、以下の **5層オントロジースタック** を全自動で設計・デプロイします。

| Layer | 内容 |
|-------|------|
| **L1 Source Tables** | 既存の生テーブル（入力） |
| **L2 Concrete Views** | ソーステーブルのビューラッパー（型変換・命名統一） |
| **L3 Abstract Views** | ビジネスロジックを抽象化したビュー |
| **L4 Semantic Views** | Cortex Analyst 用セマンティックビュー（YAML定義） |
| **L5 Cortex Agent** | 3ツールを持つ会話型AIエージェント |

L4には3種類のセマンティックビューが生成されます。

| セマンティックビュー | 役割 |
|---------------------|------|
| `{NAME}_BASE` | ソーステーブルへの直接クエリ用（エンティティ・集計） |
| `{NAME}_ONTOLOGY_MODEL` | テーブル間リレーション・エンティティ分布情報 |
| `{NAME}_METADATA_MODEL` | クラス定義・プロパティ・ビジネスメタデータ |

L5 の Cortex Agent はこれら3つをツールとして持ち、質問内容に応じてルーティングします。

## 検証に使うデータ：Tasty Bytes

Tasty Bytes はフードトラックビジネスをモデル化した Snowflake 公式デモデータセットです。`FROSTBYTE_TASTY_BYTES` データベースに以下の8テーブルが含まれます。

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

ORDER_DETAIL が6.7億行・ORDER_HEADER が2.5億行という実運用に近いスケールで、集計クエリの妥当性が検証しやすいのが特徴です。

## 1. スキルのインストール

CoCo を起動し、`find-skill` スキルで `ontology-stack-builder` を GitHub から取得します。

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/01.png)

インストールが完了すると、`cortex skill list` で全15スキルの一覧が確認できます。`ontology-stack-builder` が正しく登録されていることを確認します。

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/02.png)

:::message
**スキルのインストール方法**

GitHub リポジトリを直接指定する場合、リポジトリ直下に複数の `SKILL.md` が存在すると `No valid skills found` エラーになります。その場合はキャッシュされたローカルパスを直接指定します。

```
cortex skill add "~/.snowflake/cortex/remote_cache/github_Snowflake-Labs_coco-skills_.../skills/ontology-stack-builder"
```
:::

## 2. スキルの起動

以下のプロンプトでスキルを明示的に呼び出します（`$skill-name` 構文で意図ベースルーティングを回避）。

```
Use ontology-stack-builder skill. Build an ontology stack on FROSTBYTE_TASTY_BYTES using these inputs:

Database: FROSTBYTE_TASTY_BYTES, Schema: RAW_CUSTOMER, RAW_POS
Source tables: CUSTOMER_LOYALTY, COUNTRY, FRANCHISE, LOCATION, MENU, ORDER_DETAIL, ORDER_HEADER, TRUCK
Ontology name: MY_ONTOLOGY
Path: Direct table path
Business questions: What products does each customer buy? How are customers segmented?
Semantic views: Ontology + Metadata
```

スキルはこのプロンプトを解析し、7つのフェーズをゲート付きで順番に実行します。

## 3. Phase 1 — インプット収集・確認

スキルが入力内容を整理し、既存のセマンティックビューとの重複チェックを行います。既存ビューが0件であることを確認し、ゲートを通過します。

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/04.png)

## 4. Phase 2 — オントロジー設計

8テーブルのカラム情報・外部キーを解析し、オントロジーの **クラス** と **リレーション** を自動設計します。

**設計されたクラス（8個）**

| クラス名 | 対応テーブル |
|---------|------------|
| CustomerLoyalty | CUSTOMER_LOYALTY |
| Country | COUNTRY |
| Franchise | FRANCHISE |
| Location | LOCATION |
| Menu | MENU |
| OrderDetail | ORDER_DETAIL |
| OrderHeader | ORDER_HEADER |
| Truck | TRUCK |

**設計されたリレーション（6個）**

| リレーション | 方向 | カーディナリティ |
|------------|------|----------------|
| has_order | OrderDetail → OrderHeader | N:1 |
| has_menu_item | OrderDetail → Menu | N:1 |
| has_truck | OrderHeader → Truck | N:1 |
| has_location | OrderHeader → Location | N:1 |
| has_customer | OrderHeader → CustomerLoyalty | N:1 |
| has_franchise | Truck → Franchise | N:1 |

FK カバレッジ 100%、ゲート通過。

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/05.png)

## 5. Phase 3 — 構造の可視化

Streamlit が利用可能な環境ではグラフィカルなビジュアライザーが起動しますが、今回の CoCo CLI 環境では利用不可のため、ASCII テキストでクラス階層とリレーション構造を表示します。

```
MY_ONTOLOGY
├── Classes (8/8)
│   ├── CustomerLoyalty
│   ├── Country
│   ├── Franchise
│   ├── Location
│   ├── Menu
│   ├── OrderDetail
│   ├── OrderHeader
│   └── Truck
└── Relations (6/6)
    ├── has_order: OrderDetail → OrderHeader
    ├── has_menu_item: OrderDetail → Menu
    ├── has_truck: OrderHeader → Truck
    ├── has_location: OrderHeader → Location
    ├── has_customer: OrderHeader → CustomerLoyalty
    └── has_franchise: Truck → Franchise
```

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/06.png)

## 6. Phase 4 — SQL 生成・デプロイ

4つの SQL ファイルを生成し、完全性チェック後にデプロイを実行します。

**生成される SQL ファイル**

| ファイル名 | 内容 |
|-----------|------|
| `02_concrete_views.sql` | L2 コンクリートビュー（8個） |
| `03_metadata_tables.sql` | メタデータテーブル（22個） |
| `04_abstract_views.sql` | L3 アブストラクトビュー（10個） |
| `05_view_generator_sp.sql` | ビュー生成ストアドプロシージャ（1個） |

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/07.png)

デプロイが完了すると、合計 **46 Snowflake オブジェクト** が `FROSTBYTE_TASTY_BYTES.RAW_POS` スキーマに作成されます。

| オブジェクト種別 | 数 |
|---------------|---|
| Concrete views | 8 |
| Abstract views | 10 |
| Unified views | 2 |
| Metadata tables | 22 |
| Action/Function tables | 4 |
| Stored procedure | 1 |
| **合計** | **46** |

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/08.png)

## 7. Phase 4.5 — ベースセマンティックビュー検証

`MY_ONTOLOGY_BASE` セマンティックビューを Cortex Analyst にデプロイし、ビジネス質問2件でテストします。

- **Q1**: What products does each customer buy? → Rodolfo Tucker 氏の購買品目を正確に集計 ✓
- **Q2**: How are customers segmented? → 性別・国・ブランド別の顧客分布を正確に集計 ✓

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/09.png)

## 8. Phase 5 — 全セマンティックビュー完成

3つのセマンティックビューがすべてデプロイされ、エンティティ分布の確認クエリが実行されます。

| セマンティックビュー | 対象テーブル数 | 主なエンティティ数 |
|--------------------|--------------|-----------------|
| MY_ONTOLOGY_BASE | 8 | OrderDetail: 673M / OrderHeader: 248M |
| MY_ONTOLOGY_ONTOLOGY_MODEL | 3 | — |
| MY_ONTOLOGY_METADATA_MODEL | 4 | CustomerLoyalty: 222K |

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/10.png)

## 9. Phase 6 — Cortex Agent 作成

`MY_ONTOLOGY_AGENT` が3ツールとルーティング設計で作成されます。

**ツール構成**

| ツール名 | セマンティックビュー | 用途 |
|---------|------------------|------|
| `base_query_tool` | MY_ONTOLOGY_BASE | エンティティ検索・集計 |
| `ontology_query_tool` | MY_ONTOLOGY_ONTOLOGY_MODEL | リレーション探索・分布 |
| `metadata_query_tool` | MY_ONTOLOGY_METADATA_MODEL | クラス定義・プロパティ参照 |

Agent はユーザーの質問を解析し、適切なツール（または複数ツールの組み合わせ）にルーティングします。

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/11.png)

## 10. Phase 7 — E2E 検証

L1〜L5 の全レイヤーについてヘルスチェックが実行されます。

- L1 Source Tables: ✓
- L2 Concrete Views: ✓
- L3 Abstract Views: ✓
- L4 Semantic Views: ✓
- L5 Cortex Agent: ✓

**最終デプロイ成果物:** 46 Snowflake オブジェクト + 3 セマンティックビュー + 1 Cortex Agent

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/12.png)

## 11. ビジュアライザーで構造を確認

CoCo CLI セッションとは別に、スキルが出力した Python スクリプトを手動実行することでオントロジーのグラフ構造を可視化できます。

```bash
pip install streamlit streamlit-agraph pyyaml
python -m streamlit run visualize_ontology.py
```

**8ノード（クラス）、6エッジ（リレーション）** のグラフが表示されます。

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/13.png)

## 12. Cortex Agent への質問検証

構築した `MY_ONTOLOGY_AGENT` に3段階の難易度で質問します。

---

### Q1（シンプル）: メニューカテゴリ別の売上集計

> 売上合計と注文件数をメニューカテゴリ別に教えて

Agent は `ORDER_DETAIL` と `MENU` を結合する SQL を自動生成し、カテゴリ別に集計します。Main カテゴリが売上約 $87 億・2.6 億件で首位でした。

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/ex1.png)

---

### Q2（中程度）: 特定顧客の購買推移

> 顧客である Rodolfo Tucker 氏の月別注文回数と累計購入金額の推移を教えて

`CUSTOMER_LOYALTY` → `ORDER_HEADER` → `ORDER_DETAIL` の3テーブル結合が必要な質問です。2019年10月〜2022年10月の約3年間の推移を、棒グラフ（月別注文回数）と折れ線グラフ（累計購入金額）で表示しました。累計購入金額は **$10,494.75** でした。

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/ex2.png)

---

### Q3（複雑）: フランチャイズ別売上 × 上位メニュー

> フランチャイズオーナーごとの総売上と、配下のトラックで最も売れたメニュー上位3品をまとめて教えて

`FRANCHISE` → `TRUCK` → `ORDER_HEADER` → `ORDER_DETAIL` → `MENU` の5テーブルを横断するネスト集計が必要な質問です。全336フランチャイズに対して、各オーナーの総売上と上位3品を正確に返しました。

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/ex3.png)

**Agent Details を確認すると、Key data objects に今回構築したセマンティックビューが実際に使われていることが分かります。** `MY_ONTOLOGY_METADATA_MODEL` がメタデータ参照ツールとして機能し、テーブル間の意味的なつながりを Agent が理解した上でクエリを生成していることが確認できます。

![](/images/i140-coco-ontology-stack-builder-tasty-bytes/ex3-1.png)

## まとめ

`ontology-stack-builder` を使うと、ソーステーブルの指定から Cortex Agent のデプロイまでを **7フェーズのガイド付きワークフロー** で完結できます。今回の検証で確認できた主なポイントを整理します。

| 観点 | 結果 |
|------|------|
| 構築時間 | CoCo プロンプト1回で7フェーズ自動実行 |
| デプロイ成果物 | Snowflakeオブジェクト46個 + セマンティックビュー3個 + Agent 1個 |
| FK 解析精度 | 8テーブル・6リレーションすべて正確に検出（カバレッジ100%） |
| 単純集計（Q1） | カテゴリ別売上を正確に集計 |
| 中程度の結合（Q2） | 3テーブル結合・時系列グラフを自動生成 |
| 複雑な集計（Q3） | 5テーブル横断のネスト集計を正確に処理 |

特に Q3 の Agent Details 画面（`ex3-1`）から読み取れるように、**Key data objects として `MY_ONTOLOGY_METADATA_MODEL` が参照されている**点が重要です。Cortex Agent は単純な SQL 生成だけでなく、オントロジー定義やメタデータモデルを参照してテーブル間の意味的なつながりを把握した上でクエリを構築しています。今回の検証の初感として、**Key data objects にオントロジー／メタデータモデルが使われていることで、Agent の回答精度が向上している**可能性が高いと感じています。

セマンティックレイヤーの整備はデータ品質や AI 回答精度の基盤になります。`ontology-stack-builder` はその構築コストを大幅に下げる実用的なスキルです。
