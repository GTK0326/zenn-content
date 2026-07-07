# zenn-content

[Zenn](https://zenn.dev/gtk0326) の記事・画像・検証ノートブックを管理するリポジトリ。

## ディレクトリ構成

```
articles/   記事本体（.md）
images/     記事に埋め込む画像
notebooks/  記事のハンズオンで使用した SQL を Jupyter Notebook 形式で公開
```

## 命名規則

### 記事スラッグ

```
{連番}-{kebab-case-タイトル}.md
例: i145-snowflake-personal-account-initial-setup.md
```

連番は `i` + 数字。Zenn の記事番号管理に使用。

### 画像ディレクトリ

記事スラッグと同名のディレクトリを `images/` 以下に作成し、その記事専用画像を格納する。

```
images/{記事スラッグ}/cover.png   カバー画像
images/{記事スラッグ}/01.png      本文画像（連番）
```

### ノートブック

ハンズオン記事に対応する `.ipynb` ファイルを `notebooks/` に格納する。記事からは GitHub の raw URL でリンクする。

```
notebooks/{記事スラッグ}.ipynb
```

ノートブックがある記事では記事末尾に以下の形式でリンクを記載する。

```markdown
[📓 検証ノートブックを開く（GitHub）](https://github.com/GTK0326/zenn-content/blob/main/notebooks/{スラッグ}.ipynb)
```

## 記事一覧

| スラッグ | タイトル | ノートブック |
|---------|---------|------------|
| i47-feature-update-2026-05-26-custom-incremental | Snowflake Custom Incremental | ✓ |
| i67-feature-update-2026-05-19-custom-runtime-ima | Snowflake Custom Runtime IMA | ✓ |
| i68-virtual-columns-general-availability | Virtual Columns GA | ✓ |
| i73-feature-update-2026-05-28-sensitive-data-acc | Sensitive Data Access | ✓ |
| i119-snowflake-iceberg-table-migration | Iceberg Table Migration | — |
| i140-coco-ontology-stack-builder-tasty-bytes | CoCo Ontology Stack Builder | — |
| i145-snowflake-personal-account-initial-setup | 個人検証アカウント初期設定 | — |
| voice-input-with-genai-2026spring | 音声入力と生成AI | — |
