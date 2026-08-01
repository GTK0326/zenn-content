---
title: "Agent Skillのおすすめの作り方 ── 公式ガイドの基本と、実際に作って分かったこと"
emoji: "🧩"
type: "idea"
topics: ["ai", "agent", "claudecode", "skill", "llm"]
published: true
---

## この記事で分かること

Agent Skill（`SKILL.md`）を作ろうとしたとき、そもそもどう書けばいいのか分からない、書いてみても思ったとおりに動かないことがよくあります。

使ってほしい場面で呼ばれなかったり、逆に関係ない場面で勝手に動き出したりします。

この記事では、まず**公式ガイドが示している基本**を整理します。

そのうえで、実際にスキルをいくつも作ってみて「こう書いたほうがうまくいく」と感じたことを、**経験則として**書いていきます。

前半が土台、後半が実践という構成です。

![Agent Skill の作り方の主なポイント](/images/b-agent-skill-authoring/cover.png)

## 前半：公式ガイドが示している基本

まず[Anthropic の Skill authoring best practices](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/best-practices)が示している基本を確認します。

ここは守っておく必要があります。以下、「何をするか」「なぜそうするか」「公式ガイドの記述（原文）」の順に整理します。

### 1. frontmatter は `name` と `description` の2つだけにする

**なぜ**: 起動時にシステムプロンプトへ読み込まれるのは、この2つだけだからです。他のフィールドを足しても、スキルが選ばれるかどうかの判断には使われません。

**公式ガイドの記述**:

> The SKILL.md frontmatter requires two fields:
>
> `name`:
> - Maximum 64 characters
> - Must contain only lowercase letters, numbers, and hyphens
> - Cannot contain XML tags
> - Cannot contain reserved words: "anthropic", "claude"
>
> `description`:
> - Must be non-empty
> - Maximum 1,024 characters
> - Cannot contain XML tags
> - Should describe what the Skill does and when to use it

### 2. `description` に「何をするか」と「いつ使うか」の両方を書く

**なぜ**: `description` は、多数のスキルの中からどれを使うかを判断するための情報だからです。機能の説明だけでは、使うべき場面が判断できません。

**公式ガイドの記述**:

> The `description` field enables Skill discovery and should include both what the Skill does and when to use it.
>
> **Be specific and include key terms**. Include both what the Skill does and specific triggers/contexts for when to use it.
>
> Each Skill has exactly one description field. The description is critical for skill selection: Claude uses it to choose the right Skill from potentially 100+ available Skills.

### 3. `description` は三人称で書く

**なぜ**: `description` はシステムプロンプトに埋め込まれるため、視点が混ざると判断がぶれるからです。

**公式ガイドの記述**:

> **Always write in third person**. The description is injected into the system prompt, and inconsistent point-of-view can cause discovery problems.
>
> - **Good:** "Processes Excel files and generates reports"
> - **Avoid:** "I can help you process Excel files"
> - **Avoid:** "You can use this to process Excel files"

### 4. `name` は動名詞または名詞句で統一する

**なぜ**: 名前を見ただけで何をするスキルか分かるようにするためです。命名がばらつくと、スキルが増えたときに管理できなくなります。

**公式ガイドの記述**:

> Use consistent naming patterns to make Skills easier to reference and discuss. Consider using **gerund form** (verb + -ing) for Skill names, as this clearly describes the activity or capability the Skill provides.
>
> **Avoid:**
> - Vague names: `helper`, `utils`, `tools`
> - Overly generic: `documents`, `data`, `files`
> - Reserved words: `anthropic-helper`, `claude-tools`
> - Inconsistent patterns within your skill collection

### 5. SKILL.md の本体は500行未満に保つ

**なぜ**: コンテキストウィンドウは会話履歴や他のスキルと共有される資源だからです。読み込まれた瞬間から、すべてのトークンが他の情報と競合します。

**公式ガイドの記述**:

> Keep SKILL.md body under 500 lines for optimal performance. If your content exceeds this, split it into separate files using the progressive disclosure patterns described earlier.

### 6. 詳細は別ファイルに外出しし、必要なときだけ読ませる

**なぜ**: ファイルはファイルシステム上に置かれ、実際に読まれるまでトークンを消費しないからです。まとめて持たせても、使わなければコストになりません。

**公式ガイドの記述**:

> SKILL.md serves as an overview that points Claude to detailed materials as needed, like a table of contents in an onboarding guide.
>
> **No context penalty for large files:** Reference files, data, or documentation don't consume context tokens until actually read

### 7. 参照ファイルは SKILL.md から1階層までにする

**なぜ**: 参照先からさらに参照させると、途中までしか読まれないことがあるからです。

**公式ガイドの記述**:

> Claude may partially read files when they're referenced from other referenced files. When encountering nested references, Claude might use commands like `head -100` to preview content rather than reading entire files, resulting in incomplete information.
>
> **Keep references one level deep from SKILL.md**. All reference files should link directly from SKILL.md to ensure Claude reads complete files when needed.

### 8. 決定論的な処理はスクリプトにして同梱する

**なぜ**: 毎回コードを生成させると内容がぶれるうえ、トークンも時間も消費するからです。

**公式ガイドの記述**:

> **Prefer scripts for deterministic operations:** Write `validate_form.py` rather than asking Claude to generate validation code
>
> **Benefits of utility scripts:**
> - More reliable than generated code
> - Save tokens (no need to include code in context)
> - Save time (no code generation required)
> - Ensure consistency across uses

### 9. タスクの壊れやすさに応じて自由度を変える

**なぜ**: 正解が複数ある作業と、手順を1つでも間違えると壊れる作業とでは、必要な指示の細かさが違うからです。

**公式ガイドの記述**:

> Match the level of specificity to the task's fragility and variability.
>
> **High freedom** (text-based instructions): Use when: Multiple approaches are valid / Decisions depend on context / Heuristics guide the approach
>
> **Medium freedom** (pseudocode or scripts with parameters): Use when: A preferred pattern exists / Some variation is acceptable / Configuration affects behavior
>
> **Low freedom** (specific scripts, few or no parameters): Use when: Operations are fragile and error-prone / Consistency is critical / A specific sequence must be followed

---

ここまでが公式ガイドの基本です。

## 後半：実際に作って分かったこと

ここからは、実際にスキルを作って動かしてみた結果と、**すでに公開されている完成度の高いスキルをいくつも読んで観察した共通パターン**を書いていきます。

公式ガイドは「良いスキルの形」を教えてくれますが、**実際の会話でどう発火するか**という部分は、作って試すか、うまく動いているスキルの書き方を観察するかしないと見えてきません。

以下は、その2つから得られた実践的な内容です。既存のスキルに共通して見られた書き方については、その旨を明記しています。

各項目は「Context（前提）→ Practice（こう書く）→ Anti-pattern（こう書かない）」の順で整理します。

## SKILL.md の書き方：分割は「行数」ではなく「分岐」で判断する

**Context（前提）**

公式ガイドの「500行未満」は、コンテキストを圧迫しないための目安として妥当です。実際、目安なしに書き続けると確実に肥大化します。

ただし、これを「行数が増えたら分割する」という機械的なルールとして運用すると、かえって扱いにくくなることがあります。

実際に効いたのは、行数を目安として持ちつつ、**独立した分岐があるかどうか**で判断することでした。行数はあくまで「見直しのきっかけ」で、分割するかどうかは中身の構造で決めます。

**Practice（こう書く）**

手順が一直線に進むタスクは、行数が増えても1つのファイルにまとめます。分岐がない手順を無理に分けると、本来ひと続きだった流れが途切れ、途中の手順が読まれないまま先へ進んでしまうことがあります。

逆に、「新規作成」と「障害調査」のように、入口が同じでもやることが完全に別のワークフローに分かれる場合は、本体が短くても分割します。この2つを1つのファイルに同居させると、作成の手順を読みながら調査を始めるような混線が起きます。

判断の目安は次のとおりです。分割されているスキルを見ていくと、おおむねこの基準で分かれていました。

| 状況 | 判断 |
|---|---|
| 手順が一直線で分岐がない | 長くても1ファイルにまとめる |
| 独立したワークフローが3つ以上ある | 短くても分割する |
| 分岐先で読むべき知識が全く違う | 分割するか、参照ファイルに外出しする |

**Anti-pattern（こう書かない）**

行数が増えたという理由だけで、一直線の手順を機械的に切りません。ただし、目安を大きく超えたまま放置もしません。超えたときは「本当に1つの流れなのか」を見直します。

逆に、明らかに別のワークフローになっているものを、まだ短いからという理由で1ファイルに詰め込みません。

## 入口のスキル：処理を持たせない

**Context（前提）**

分割すると、入口となる SKILL.md は振り分け専用になります。ここは実際の処理を書くファイルとは、書き方が別物になります。

**Practice（こう書く）**

冒頭で「このスキル自体は処理を持たない」と宣言します。入口に手順の断片が残っていると、子スキルを読まずにその断片だけで処理を始めてしまうことがあるためです。

```markdown
このスキルは処理内容を持ちません。
必ず下表から該当する子スキルをロードしてから作業を始めてください。
```

振り分けの条件は、1つの表にまとめます。これは既存のスキルを読んでいて共通して見られた書き方で、条件を本文のあちこちに分散させると、更新したときにズレるためだと理解しました。

| 意図 | トリガー例 | ロード先 |
|---|---|---|
| 作成・変更 | 「作りたい」「変更したい」 | `./create/SKILL.md` |
| 障害調査 | 「動かない」「エラーが出る」 | `./troubleshoot/SKILL.md` |

指示は能動態で書きます。よくできているスキルは「詳しくはこちら」「参照してください」のような受け身の案内をほとんど使わず、「`./create/SKILL.md` をロードしてください」と動詞で命令する形に揃えられていました。受け身の案内は読み飛ばされやすいためだと考えられます。

子スキルが名前で解決できなかった場合の代替経路も書いておきます。環境によっては解決に失敗することがあり、そのとき自己流の処理に流れると品質が崩れるためです。

```markdown
子スキルが名前で解決できない場合は、
`./troubleshoot/SKILL.md` を直接読み込み、最後まで実行してください。
```

**Anti-pattern（こう書かない）**

入口に「よくあるケースはこう対処する」といった処理の断片を残しません。

振り分けの条件を、本文の複数箇所に分散させません。

## 末端のスキル：`description` を省く

**Context（前提）**

入口から名指しでロードされる末端のスキルには、`description` を書かないほうがうまくいきます。

`description` は自動発火のための情報であり、末端のスキルはそもそも自動発火させる必要がないためです。

**Practice（こう書く）**

入口のスキルには `description` を作り込みます。ここが呼ばれるかどうかを決めます。

末端のスキルには、何をするか・前提・出力だけを簡潔に書きます。トリガー語の列挙は不要です。

**Anti-pattern（こう書かない）**

すべての SKILL.md に、同じ密度でトリガー語を書きません。起動時に読み込まれるメタデータが増え、そのぶん他の情報と競合します。

末端のスキルに「〜のときに使う」と書いて、入口と競合させません。

## `description` の書き方：「いつ呼ぶな」を書く

**Context（前提）**

公式ガイドが言う「何をするか」「いつ使うか」は必要な情報ですが、それだけだと暴発が止まりません。実際に問題になるのは、たいてい**呼んでほしくない場面**が書かれていないからです。

`description` は、誤爆と取りこぼしを潰していく作業ログの置き場だと考えると、扱いやすくなります。

:::message
個人的には、ここが一番効きました。呼ばれない・暴発するの大半は `description` の書き方で解決します。
:::

**Practice（こう書く）**

呼ばない条件を、具体例つきで書きます。抽象的な除外条件は効きにくいため、実際に誤爆したケースの言葉をそのまま書きます。

```yaml
description: >-
  （何をするかの説明）...
  次の場合は使用しない: 特定のIDを指定した参照（詳細取得のスキルを使う）、
  すでに手元にある結果の整形、社内カタログの検索（カタログ検索ツールを使う）。
```

発火の方針には、その理由も添えます。「迷ったら呼ぶ」とだけ書くと判断がぶれるため、なぜそちらに倒すのかを書きます。

```markdown
迷った場合は呼び出す側に倒す。呼び出しのコストは小さく、見落としのコストは大きいため。
```

呼びすぎが害になるスキルなら、逆に「迷ったら呼ばない、理由は〜」と書きます。判断基準そのものより、**判断の根拠**を書くほうが安定します。

止め方も同じ `description` に書きます。積極的に発火させると、今度は「しつこい」という問題が出るためです。

```markdown
その要求が一度解決した後は、未要求のまま再提案しない。
ただしユーザーが明示的に要求した場合は、過去のやり取りに関わらず常に発火する。
```

このとき、止める範囲を会話単位にしないことが重要です。会話単位で止めると、別の話題に移ったときに二度と発火しなくなります。「同じ要求については止める、別の要求なら再び発火する」と書き分けます。

似た機能のツールがある場合は、代替なのか併用なのかを明示します。ここが曖昧だと、片方しか実行されません。

```markdown
このスキルは○○ツールの代替ではない。両方を同じターンで実行すること。
（対象とする範囲が異なり、補完関係にあるため）
```

**Anti-pattern（こう書かない）**

「〜に役立ちます」とだけ書いて、呼ばない条件を書かないままにしません。

トリガー語を並べるだけで終わりにしません。列挙は取りこぼしには効きますが、暴発を抑えるにはあまり効かない印象です。

「常に使用する」と書きながら、止め方を書かないままにしません。

## 似た機能のスキル：互いに「自分ではない」と書く

**Context（前提）**

機能が近いスキルが複数あると、奪い合いになりがちです。

これは、それぞれのスキルが**自分の担当ではないケース**を書くことで解消できます。

たとえば次の3つが並ぶ状況を考えます。

- A：単発のファイル処理
- B：継続的なパイプライン構築
- C：すでにテーブルにあるデータへの関数適用

**Practice（こう書く）**

A の `description` の末尾に「継続的に処理し続けたい場合は B を使う」と書きます。

B には「単発なら A、ファイルを伴わずテーブル上のデータだけを扱うなら C に委ねる」と書きます。

相手のスキル名は名指しで書きます。「別のスキルを使ってください」では、どこへ行けばいいか分からず解決しません。

そして双方向に書きます。片方にしか書かないと、書かれていない側が担当外の仕事を奪い続けます。

**Anti-pattern（こう書かない）**

「〜にも対応可能」と守備範囲を広く書きません。隣のスキルと衝突します。

自分の得意分野だけを書いて、境界を書かないままにしません。

## 参照ファイル：「読め」ではなく「読む前に書くな」と書く

**Context（前提）**

詳細な仕様やスキーマを外出しすること自体は公式ガイドのとおりですが、外出しした結果、読まれないまま作業が進んでしまうことがあります。

「詳細は `references/api.md` を参照」と書いても、そのまま自己流で書き始めてしまう、という状態です。

**Practice（こう書く）**

読むタイミングを、次の行動の前提として書きます。

```markdown
`references/api.md` を読む前に、このAPIを使うコードを書かないでください。
```

「参照する」ではなく「読まずに次の行動をしない」という形にすると、順序が守られます。

100行を超える参照ファイルには、冒頭に目次を付けます。部分的に読まれた場合でも、何が書いてあるかは伝わります。

**Anti-pattern（こう書かない）**

「必要に応じて参照してください」と書きません。読まれません。

参照ファイルの中から、さらに別のファイルを参照させません。

毎回必ず読む必要がある中核の手順を、外出ししません。

## スクリプト：判断させたくない処理をコードにする

**Context（前提）**

手順が固定されている処理は、スクリプトを同梱して実行させたほうが安定します。毎回コードを生成させると、少しずつ違うものが出てくるためです。

**Practice（こう書く）**

タスクの性質に応じて、指示の細かさを変えます。

| タスクの性質 | 書き方 |
|---|---|
| 正解が複数ある | 文章で方針だけを示す |
| 望ましい型がある | 擬似コードやパラメータ付きの例を示す |
| 壊れやすい・順序が重要 | 実行するスクリプトとコマンドを固定して書く |

スクリプトについては、実行させたいのか読ませたいのかを明示します。同じファイルでも意図が2通りあり、混同されると、実行してほしい場面で中身を読んで再実装が始まります。

- 実行させたい場合：「`scripts/validate.py` を実行してください」
- 参考にさせたい場合：「抽出ロジックは `scripts/extract.py` を参照してください」

エラー処理はスクリプト側に持たせます。「失敗したら呼び出し側で対処する」という書き方をすると、そこから自己流の回復処理が始まるためです。想定される失敗はスクリプト内で処理して正常系に戻すか、原因が分かるメッセージを出して止めます。

生成 → 検証 → 修正 → 再検証というループを、手順として書いておきます。検証をスクリプトにしておくと、品質の判断がぶれません。

**Anti-pattern（こう書かない）**

毎回スクリプトを生成させません。

定数の根拠を書かないままにしません。`TIMEOUT = 47` のような値は、なぜその値なのかが分からないと、変更していいのか判断できません。

選択肢を並べません。既定を1つ決めて、例外のケースだけを書きます。

## 強い言葉：理由とセットで書く

**Context（前提）**

`MANDATORY` `MUST` 「絶対に」といった強い語は、つい増えます。

ただ、強い語だけを増やしても、あまり効かない印象です。すべてのスキルが「自分が最優先」と主張しはじめると、結局どれを優先するのか決まらなくなります。

**Practice（こう書く）**

なぜそうすべきかを、理由として書きます。

これは既存のスキルにも共通して見られる書き方でした。強い語を並べるより、その指示が必要な理由を添えているものが多く、そのほうが想定外の状況に出会ったときにも判断が効くためだと理解しています。

| 書き方 | 例 |
|---|---|
| 弱い | 「必ずこの順序で実行すること」 |
| 強い | 「この順序で実行すること。前の手順の出力を次の手順が入力として使うため、順序を変えると参照先が存在しない状態になる」 |

**Anti-pattern（こう書かない）**

強い語だけを重ねません。書きたくなったら、その語を消して理由に置き換えられないかを先に考えます。

## まとめ

最後に、ここまでの内容をカテゴリごとに整理します。

**構成の決め方**

- 分割は行数ではなく、独立した分岐があるかどうかで判断する
- 手順が一直線なら、長くても1つのファイルにまとめる
- 入口のスキルには処理を持たせず、振り分けだけを書く
- 振り分けの条件は1つの表にまとめる
- 子スキルが解決できない場合の代替経路を書いておく

**呼ばれ方の制御**

- `description` には「いつ呼ぶか」だけでなく「いつ呼ぶな」を具体例つきで書く
- 発火の方針には、その理由をセットで書く
- 積極的に発火させる場合は、止め方も書く
- 止める範囲は会話単位ではなく、要求単位にする
- 似た機能のスキルどうしは、双方向に「自分ではないケース」を名指しで書く
- 明示的にロードされる末端のスキルでは、`description` を省く

**ファイルの持たせ方**

- 詳細は参照ファイルに外出しし、必要なときだけ読ませる
- 参照は「読め」ではなく「読む前に次の行動をするな」と書く
- 参照ファイルは1階層までにとどめる
- 判断させたくない処理はスクリプトにして同梱する
- スクリプトは、実行させるのか読ませるのかを明示する

**書き方・表現**

- `description` は三人称で書く
- frontmatter は `name` と `description` だけにする
- 指示は能動態で、動詞を使って書く
- 強い語は、理由とセットで書く

スキルは一度書いて終わりにはなりません。

うまく呼ばれなかったケース、逆に暴発したケースを、`description` に1つずつ書き足していくのが、結局いちばん早く安定します。

## 参考

- [Skill authoring best practices - Claude Docs](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/best-practices)
- [Extend Claude with skills - Claude Code Docs](https://docs.anthropic.com/en/docs/claude-code/skills)
