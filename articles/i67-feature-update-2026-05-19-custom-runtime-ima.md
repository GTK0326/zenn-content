---
title: "SnowflakeのMLランタイムを自前コンテナに置き換える——CREで再現性とコンプライアンスを同時に解決"
emoji: "🐳"
type: "tech"
topics: ["snowflake", "machinelearning", "docker", "container", "mlops"]
published: true
---

## この記事で分かること

Snowflake Notebooks および Container Runtime 上の ML Jobs に、自前の Docker イメージを持ち込みたい方向けの記事です。

「Snowflake 標準ランタイムでは目的のパッケージが使えない」「社内セキュリティポリシーで承認済みイメージしか利用できない」という課題を、Custom Runtime Environment（CRE）が解決します。

Image Repository の作成から CRE の登録・参照まで、Preview 段階での制約も含めて一通りの手順を整理します。

公式リリースノート: [Snowflake Documentation](https://docs.snowflake.com/en/release-notes/2026/other/2026-05-19-custom-runtime-images-preview)

:::message
内容は記事作成時点のものです。
仕様は変更され得るため、最終的には最新の公式ドキュメントで確認ください。
:::


![](/images/i67-feature-update-2026-05-19-custom-runtime-ima/cover.png)

## Snowflake 標準ランタイムが「壁」になる3つの場面

Snowflake Notebooks や ML Jobs でモデル開発を進めると、避けがたい制約にぶつかる場面があります。

| # | 課題 | 具体的な問題 |
|---|------|------------|
| 1 | **プリインストール外のライブラリが必要** | Notebook セル内で `!pip install` を実行すれば一時的にインストールできますが、Notebook を再起動するたびに消えます。CI/CD パイプラインで再現性を担保しようとすると、この方法は根本的な解決にはなりません。 |
| 2 | **コンプライアンス要件のクリア** | 金融・医療・公共分野の組織では、「使用するコンテナイメージは社内セキュリティ審査を通過したものに限る」というポリシーが一般的です。Snowflake 管理のランタイムはソースが外部管理のため、このような審査を社内でコントロールできません。 |
| 3 | **ローカル・CI・本番の環境統一** | Snowflake がランタイムをアップデートすると、既存の学習コードが依存パッケージのバージョン変更により壊れることがあります。再現性を重視するチームほど、Snowflake 側の更新が「予期しない破壊的変更」として機能してしまいます。 |

## CRE が「自前コンテナを Snowflake で動かす」を実現する

Snowflake は 2026年5月19日、Custom Runtime Environments（CRE）を Preview として公開しました。

CRE を使うと、自分でビルドした Docker イメージを Snowflake の Image Repository に登録し、Notebooks や ML Jobs の実行環境として名前で指定できます。

前述の3つの課題に対する答えを整理します。

**パッケージの完全制御**: `Dockerfile` に任意の Python パッケージを固定バージョンで記述し、そのまま Snowflake の実行環境にできます。

**コンプライアンス対応**: 社内承認済みのベースイメージから構築したコンテナを Snowflake 上でも利用でき、イメージの出所を社内で管理できます。

**環境の固定**: イメージタグを固定すれば、何度実行しても同一のパッケージバージョンが保証されます。

CRE はアカウントレベルのオブジェクトで、スキーマには属しません。

一度登録すれば複数の Notebook や ML Job から共有して参照でき、バージョン管理は Image Repository 側で一元化できます。

## Before / After で見るランタイム指定の変化

CRE 導入前後で、実行環境の指定がどう変わるかを対比します。

**Before: Snowflake 管理のランタイム（パッケージ追加は都度 pip）**

```sql
-- ランタイム指定なし。Snowflake が管理するデフォルト環境で動く
EXECUTE NOTEBOOK mydb.myschema.analysis_notebook;
```

Notebook セル内では、次のような一時インストールが必要でした。

```python
# 再起動のたびに消えるため、再現性ゼロ
!pip install lightgbm==4.3.0 shap==0.45.0
```

**After: CRE でカスタムコンテナを指定**

```sql
-- RUNTIME パラメータで CRE を名前指定するだけ
EXECUTE NOTEBOOK PROJECT mydb.myschema.analysis_notebook
  RUNTIME = 'cre@ml_lightgbm_env';
```

イメージ側に `lightgbm==4.3.0` と `shap==0.45.0` が含まれているため、セルには純粋な分析コードだけが残ります。

`cre@` プレフィックスが CRE 参照を示す構文で、環境名を変えるだけで異なるイメージに切り替えられます。

## 実際に動かしてみよう

全体の流れは次のとおりです。

1. Image Repository を作成し、レジストリ URL を確認する
2. ローカルで Docker イメージをビルドしてプッシュする
3. CRE を登録する
4. Notebooks / ML Jobs から CRE を参照する

### ステップ1: Image Repository を作成する

:::details DB・スキーマ・Image Repository 作成 DDL

```sql
-- データベースとスキーマを作成
CREATE DATABASE IF NOT EXISTS cre_demo_db;
CREATE SCHEMA IF NOT EXISTS cre_demo_db.ml_schema;

-- Image Repository を作成（コンテナイメージの格納場所）
CREATE IMAGE REPOSITORY cre_demo_db.ml_schema.my_image_repo;
```

:::

```sql
-- レジストリ URL を確認する
SHOW IMAGE REPOSITORIES IN SCHEMA cre_demo_db.ml_schema;
```

実行結果:
```
name              | repository_url
------------------+-------------------------------------------------------------------------------------------
MY_IMAGE_REPO     | myorg-myacc.registry.snowflakecomputing.com/cre_demo_db/ml_schema/my_image_repo
```

この URL が `docker push` の宛先になります。

`myorg-myacc` の部分はアカウントごとに異なるため、必ず `SHOW` で確認してください。

### ステップ2: Docker イメージをビルドしてプッシュする

Snowflake が提供するベースイメージを拡張した `Dockerfile` を用意します。

```dockerfile
# Snowflake 提供の ML ランタイムベースイメージを拡張
FROM nvcr.io/snowflake/runtime:latest

# 必要なパッケージを固定バージョンで追加
RUN pip install --no-cache-dir \
    lightgbm==4.3.0 \
    shap==0.45.0
```

ターミナルからビルドとプッシュを実行します。

```bash
# Snowflake レジストリにログイン
docker login myorg-myacc.registry.snowflakecomputing.com

# ビルドとプッシュ（タグにバージョンを明記する）
REPO=myorg-myacc.registry.snowflakecomputing.com/cre_demo_db/ml_schema/my_image_repo
docker build -t $REPO/ml_lightgbm_env:v1.0 .
docker push $REPO/ml_lightgbm_env:v1.0
```

プッシュが完了したことを確認してから、次のステップに進んでください。

プッシュ前に CRE を作成しようとすると、サーバーサイドのイメージ存在チェックでエラーになります（ステップ3で詳述）。

### ステップ3: CRE を登録する

```sql
-- CRE はアカウントレベルのオブジェクト（DB・スキーマ指定は不要）
CREATE CUSTOM RUNTIME ENVIRONMENT ml_lightgbm_env
  IMAGE_PATH = 'cre_demo_db/ml_schema/my_image_repo/ml_lightgbm_env:v1.0'
  BASE_IMAGE_TYPE = 'ML_RUNTIME';
```

重要な注意点として、イメージが存在しない状態で実行すると `IMAGE_PATH` が見つからない旨のエラーが返ります。

これは DDL 構文の問題ではなく、Snowflake がサーバーサイドでイメージの実在を確認するためです。

Docker プッシュ後に再実行すると CRE が正常に作成されます。

### ステップ4: CRE の一覧を確認する

```sql
-- アカウント内の CRE 一覧（スキーマ指定なし）
SHOW CUSTOM RUNTIME ENVIRONMENTS;
```

CRE が作成されると、`name`・`base_image_type`・`image_path` の各列を持つ行が返ります。

スキーマを跨いで複数の CRE を登録した場合も、このコマンド1つでアカウント全体の一覧が確認できます。

### ステップ5: アクセス権を付与する

```sql
-- データサイエンティストロールに Image Repository への読み取り権を付与
GRANT READ ON IMAGE REPOSITORY cre_demo_db.ml_schema.my_image_repo
  TO ROLE data_scientist_role;

-- CRE の使用権を付与
GRANT USAGE ON CUSTOM RUNTIME ENVIRONMENT ml_lightgbm_env
  TO ROLE data_scientist_role;
```

### ステップ6: CRE を指定して Notebook を実行する

```sql
-- RUNTIME パラメータで CRE を参照する
EXECUTE NOTEBOOK PROJECT mydb.myschema.analysis_notebook
  RUNTIME = 'cre@ml_lightgbm_env';
```

`cre@ml_lightgbm_env` のうち `cre@` が CRE 参照を示すプレフィックスです。

CRE 名を変えるだけで実行環境を切り替えられるため、実験ごとにイメージを使い分けることも容易です。

## 本番導入前に知っておきたいポイント

**1. CRE 作成は「イメージプッシュ後」が必須**

`CREATE CUSTOM RUNTIME ENVIRONMENT` は実行時にサーバーサイドでイメージの存在確認を行います。

`docker push` が完了していない状態では `IMAGE_PATH` 不存在エラーが返るため、処理順序は「push → CREATE CRE」の固定順です。

Terraform などの IaC ツールでリソースを管理する場合、Image Repository と Docker プッシュステップを CRE 作成の依存先として明示的に設定する必要があります。


## 検証コード

この記事のハンズオンで使用した SQL を Jupyter Notebook（.ipynb）形式で公開しています。

Snowflake Notebooks にインポートして、そのまま自分の環境で実行できます。

[📓 検証ノートブックを開く（GitHub）](https://github.com/GTK0326/zenn-content/blob/main/notebooks/i67-feature-update-2026-05-19-custom-runtime-ima.ipynb)
## まとめ

Custom Runtime Environment（CRE）は、Snowflake の ML 実行環境に自前の Docker コンテナを持ち込む手段を正式に提供するものです。

「パッケージ制約の解消」「コンプライアンス対応」「環境の再現性」という3つの課題を、`RUNTIME = 'cre@<名前>'` という1パラメータで解決できる点が大きな特徴です。

詳細な構文と最新の制約事項は[公式ドキュメント（Custom runtime images）](https://docs.snowflake.com/en/release-notes/2026/other/2026-05-19-custom-runtime-images-preview)で確認してください。

## 参考リンク

- [Custom runtime images (Preview) — Snowflake Release Notes](https://docs.snowflake.com/en/release-notes/2026/other/2026-05-19-custom-runtime-images-preview)