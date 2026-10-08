# Graphify 段階学習ノート

| 数字 | 意味 |
|---|---|
| **62%** | CodeGraph公式ベンチのトークン減（合計） |
| **67k / 18k** | CodeGraphの弱点。最後に残る文脈（VS Code） |
| **約6秒** | Graphifyでhttpxの地図を作成（72ファイル・コードのみ） |
| **Lv5** | 学習の到達点（全7段階中） |

## 目次

1. [CodeGraphの公式ベンチを読み解く](#1-codegraphの公式ベンチを読み解く)
2. [Graphify Lv1：何を解決する道具か](#2-graphify-lv1何を解決する道具か)
3. [Graphify Lv2：グラフの基本用語](#3-graphify-lv2グラフの基本用語)
4. [Graphify Lv3：地図の作り方](#4-graphify-lv3地図の作り方)
5. [Graphify Lv4：AIの使い方](#5-graphify-lv4aiの使い方)
6. [Graphify Lv5：実際に触った結果](#6-graphify-lv5実際に触った結果)
7. [CodeGraph と Graphify の比較](#7-codegraph-と-graphify-の比較)
8. [途中で訂正したこと](#8-途中で訂正したこと)
9. [現状と次のアクション](#9-現状と次のアクション)
10. [公式資料](#10-公式資料)

---

## 1. CodeGraphの公式ベンチを読み解く

### テストの条件

- 実在のOSS 7つ（VS Code 約11k、Django 約3k、Tokio、OkHttp、Excalidraw、Gin、Alamofire）を使い、Claude Opus 4.8で `claude -p` を自動実行
- 各リポジトリで、**「○○はどう動いている？」という構造の質問を1つ**だけ聞く。「あり」と「なし」を4回ずつ実行し、中央値をとった（2026-08-05に再測定）
- **codegraphコマンド（CLI）は両方で禁止**した
  - 禁止しないと、「なし」側が28回中26回、Bash経由でこっそりCodeGraphを使っていた
  - コマンド経由の呼び出しはツール回数に数えられないのに、結果は文脈に入る。そのため比較がゆがむ
  - 今回は28回すべて止められ、混ざったのは0回

### 結果

ツール呼び出し**88%減**・**53%速い**・トークン**62%減**・**44%安い**。ありの側は、7つ全部でファイルを1回も読まずに答えた。

| リポジトリ | ツール回数（あり vs なし） | 料金 |
|---|---|---|
| Excalidraw | 2 vs 43 | 78%安い（$0.54 vs $2.43） |
| VS Code | 2 vs 28 | 71%安い（$0.53 vs $1.80） |
| Django | 3 vs 14 | 13%安いだけ |
| Gin | 1 vs 7 | ほぼ同じ（$0.31 vs $0.31） |

> **差を決めるのは、ファイル数ではなく「その質問でどれだけ探し回るか」。** Alamofire（110ファイル）では大きく効き、Django（3,000ファイル）では13%だけだった。

### 「精度」の意味

原文はprecisionで、**必要なコードだけを的確に取ってこられること（探し物の的中率）**を指す。コーディングがうまくなる、という意味ではない。ベンチで測ったのは時間・回数・トークン・料金で、書いたコードの品質は測っていない。

### 合計は少ないのに、最後に残る量は多い

AIは1回やり取りするたびに、それまでの会話を全部読み直している。

| | 意味 | 効くもの |
|---|---|---|
| 処理したトークンの合計 | 机の上の量 × 読み直した回数、の積み重ね | 料金 |
| 最後に残る量 | 作業が終わった時点で、机の上にある量 | 空き容量 |

- **grep方式**：小さい結果を何十回も取る。古い結果はClaude Codeが片付ける（マイクロコンパクション）ので、机はすっきりしている。ただし往復が多いので、合計は大きい
- **CodeGraph**：分厚い資料を1〜4回で持ってくる。往復が少ないので、合計は小さい。ただし資料が机に残る（VS Codeで6.7万対1.8万）
- 公式の言葉では、速さと机の狭さは**同じ仕組みの表と裏**
- 公式の数字は「1つの質問の中」の合計。**その後も同じセッションで作業を続けると、残った分を毎回読み直す**。このコストは数字に含まれていない（ただし、大半はキャッシュ読み取りなので安い）

### 使い方の結論

- **作業の入口**（流れをつかむとき）と、**影響範囲の確認**で、要所に1回ずつ使う。大きな調べ物が終わったら、セッションを区切る
- 直す場所がわかっている小さな修正や、全部読める小さなプロジェクトでは、使わなくてよい
- **サブエージェントに調べ物を任せると、効果が消える**。サブエージェントは結局ファイルを読むから。CodeGraphは、メインのAIが直接使ったときだけ効く

---

## 2. Graphify Lv1：何を解決する道具か

**プロジェクトの「地図」を1回作ってファイルに残し、セッションをまたいで使い回す道具。**

- 同じ会話の中なら、一度読んだものは探し直さない。ただし、会話の記憶は**一時的で、読んだ部分だけ**。新しいセッション、片付け（コンパクト）のあと、まだ見ていない場所では、また1から探す
- 地図は**ファイルとして残り、プロジェクト全体をカバー**する（会話の記憶＝今日歩いて覚えた道、地図＝紙の地図）
- **地図を作るのも、地図から取り出すのもGraphify（普通のプログラム）**。AIは、取り出された一部を受け取るだけなので軽い。graph.json本体はAIの文脈に入らない（図書館の目録と同じ）
- AIの仕事が「探す＋読む」から「読むだけ」になる。ただし、地図が返すのは場所とつながりなので、コードを直すときは該当ファイルを読む
- 画像やPDFも、一度地図にすれば使い回せる。ただし残るのは「何が描かれていて、何とつながるか」。見た目の細部は、元の画像を見る必要がある

```
[地図なし]  AI：grep → 読む → grep → 読む …
[地図あり]  Graphify：地図から取り出す → AI：必要なファイルだけ読む
```

---

## 3. Graphify Lv2：グラフの基本用語

| 用語 | 意味 | graph.htmlでは | 路線図で言うと |
|---|---|---|---|
| ノード | 1つの「もの」（関数・クラス・ファイル・見出し・画像の中の概念） | 小さな点 | 駅 |
| エッジ | つながり（calls・imports・inherits・uses・method など） | 細い線 | 線路 |
| コミュニティ | つながりの濃いまとまり（Leiden法で自動） | 点の色 | 路線 |
| god node | 特にたくさんつながっている中心 | 線が集中する点 | 大きな乗換駅 |
| 次数（Degree） | 1つのノードにつながる線の数 | `Degree: 47` | 乗り入れる路線の数 |

### 確かさの印（3種類）

| 印 | 意味 |
|---|---|
| `EXTRACTED` | ソースにはっきり書いてあった（確かさ1.0） |
| `INFERRED` | 推測でつないだ。0.95（ほぼ確実）〜0.55（当て推量）の点数付き |
| `AMBIGUOUS` | あいまい。レポートで「人が確認して」と知らせる |

> **確認Q：** 「Degreeがとても大きいノード」を変えるとき、なぜ注意が必要？
> **A：** 影響する範囲が広いから

---

## 4. Graphify Lv3：地図の作り方

```
①探す detect → ②読む extract（3段階） → ③つなぐ build → ④グループ分け cluster → ⑤分析 analyze → ⑥書き出し export
```

| 段階 | 対象 | 誰が読むか | 料金 |
|---|---|---|---|
| Pass 1 | コード | tree-sitter（ローカル） | 0円 |
| Pass 2 | 動画・音声 | faster-whisper（ローカルの文字起こし）。god nodeを文字起こしのヒントに使う | 0円 |
| Pass 3 | ドキュメント・PDF・画像 | Claudeのサブエージェントを並列で起動 | トークンがかかる |

### なぜコードはLLMなしで読めるのか

プログラムは**文法が厳密**なので、ルールに当てはめるだけで「関数がある」「呼んでいる」が決まる（日本語の「彼はそれを渡した」は、文脈がないとわからない）。tree-sitterは、ソースを構文木（AST）に分解する。そのため、速い・無料・毎回同じ結果・外に送らない。

わからないのは、書いた人の意図と、実行するまで決まらない呼び出し。

### キャッシュ

読んだファイルは、中身の指紋（SHA256）で記録される（`graphify-out/cache/`）。再実行すると、変わったファイルだけ読み直す。画像やPDFの料金がかかるのは、最初の1回だけ。

> **確認Q：** コードだけのプロジェクトで地図を作るとき、AIの料金はかかる？
> **A：** かからない。コードはtree-sitterで読むから
> 補足：`/graphify` スキル経由だと、Claudeが手順書を読む分の少しのトークンは使う。さらにLv5で、グループ名を付ける処理にClaudeが使われることが判明した（後述）。

---

## 5. Graphify Lv4：AIの使い方

```
【入口】①スキル /graphify ……………… 地図を作る・質問を受け付ける
【引く】②query / path / explain …… 地図から一部を取り出すコマンド
        ③MCPサーバー（任意）……………… ②をツールとして呼べるようにしたもの
【促す】④CLAUDE.mdのルール＋フック …… 「grepの前に地図を使って」と促す
        ⑤strictモード ………………………… 最初の1回だけ読み取りを止める
```

### ② 3つのコマンド

| コマンド | 使いどころ | たとえ |
|---|---|---|
| `query "質問"` | 広く聞く。BFS（広く）が初期設定、`--dfs` で深く。`--budget` で返す量の上限を決める | 周りの路線を見せて |
| `path A B` | AからBへどうつながる？ | 乗り換え案内 |
| `explain X` | Xについて（定義場所とつながり） | 駅の案内板 |

> ⚠️ **queryは、文字列の部分一致で探す。同義語も、言語をまたいだ一致もない。**
> 英語の関数名でできた地図に、日本語でそのまま聞くと0件になる。そのためスキルは、地図の単語一覧を作り、その中から質問に合う単語を最大12個選んでから検索するよう、AIに指示している。

### ④ 促す仕組みの正体

`.claude/settings.json` に、PreToolUseフックが書き込まれる。

```json
"PreToolUse": [
  { "matcher": "Bash|Grep", "hooks": [{ "command": "graphify hook-guard search" }] },
  { "matcher": "Read|Glob", "hooks": [{ "command": "graphify hook-guard read" }] }
]
```

- AIがツールを使う**直前に毎回**、「MANDATORY：先に `graphify query` を使って」という**お願いの文章**が、AIの会話に差し込まれる（`additionalContext`）。サブエージェントにも同じルールを伝えるよう指示している
- ツールの実行は止めない（＝促すだけ）。地図がない・プロジェクトの外・地図が古い、というときは出さないか、言い方を弱める。エラーのときは何もせずに通す（fails open）
- たとえると、CLAUDE.md＝朝礼での注意、フック＝作業のたびに肩をたたく人、strict＝最初の1回だけ通さない門番

### ⑤ strictモード

- Graphifyのオプション（`graphify install --project --strict`）。Claude Codeのフックという「コンセント」を借りている
- セッションで最初の生ファイル読み取りを、1回だけ `permissionDecision: "deny"` で拒否し、地図に誘導する。2回目からは、促すだけに戻る
- 切り替えは `GRAPHIFY_HOOK_STRICT=1`／`0`。Claude Code向けの機能

---

## 6. Graphify Lv5：実際に触った結果

- ✅ `pipx install graphifyy` → 0.9.76（Claude Codeの設定には触れていない）
- ✅ 題材として httpx（Pythonファイル60個）を `graphify_lab` にダウンロード
- ✅ `graphify extract . --code-only` → **約6秒**。ノード1,779・エッジ3,756・コミュニティ106
- ✅ `graphify cluster-only .` → 約28秒で、GRAPH_REPORT.mdとgraph.htmlができた

> ⚠️ **予想外：グループ名を付ける処理に、Claudeが使われた。**
> APIキーがないとき、PCにClaude Codeがあれば、自動で `claude -p` を呼ぶ作りだった（モデルを指定しないとOpus）。入力約9.7万・出力約1,300トークンを、Claude Codeの利用枠から使った。
> 避けるには `--no-label`、安くするなら `GRAPHIFY_CLAUDE_CLI_MODEL=haiku`。

### 試したコマンド

```
$ graphify explain "DigestAuth"
Node: DigestAuth
  Source:    httpx/_auth.py L175
  Community: Digest Authentication
  Degree:    13
  --> Auth [inherits] [EXTRACTED]
  --> Response [uses] [INFERRED]
  --> .auth_flow() [method] [EXTRACTED] httpx/_auth.py:L193

$ graphify path "DigestAuth" "Response"
  DigestAuth --uses [INFERRED]--> Response        （1段で直接つながる）

$ graphify query "digest auth flow" --budget 1500
  164ノード見つかり、上限に合わせて54個だけ返す（省略したという警告付き）

$ graphify query "ダイジェスト認証の流れ"
  No matching nodes found.                        （日本語だと0件）
```

queryが返すのは、ファイル名・行番号・グループの一覧で、ソースそのものではない。日本語で0件になったのは、Lv4の「言葉が一致しないと見つからない」の実証。

---

## 7. CodeGraph と Graphify の比較

| | CodeGraph | Graphify |
|---|---|---|
| 役割 | AIが引く**索引** | 人もAIも使う**地図**（ドキュメント・PDF・画像もつなげる） |
| 最新版 | 1.6.1（npm）／MIT | 0.9.76（PyPI `graphifyy`）／2026-07-22にMIT→Apache-2.0。YC S26の会社で、有料のクラウド版もある |
| AIへの返し方 | MCPツール1つ（explore）で、**ソースそのもの**が返る（数万トークン） | コマンドかMCPで、**つながりの一覧**が返る（小さい。そのあと必要なファイルを読む） |
| 使わせる仕組み | MCPの説明文で誘導。`~/.claude/CLAUDE.md` にも追記する | CLAUDE.mdのルール＋フックで毎回促す。strictなら1回止める |
| 更新 | 保存から約2秒で自動 | `graphify update .`、またはコミット時のgitフック。`--watch` もある |
| 公式の主張 | **安く速く**（料金44%減など） | **正確に**（ERPNextで、回答のカバー率が70.8%→82.0%。6問）。READMEのベンチは、会話の記憶の試験 |
| 得意なこと | 「○○はどう動いている？」に一発で答える | 全体像、AとBのつながり、コメントの「なぜ」やドキュメントとの関係 |
| 注意 | 文脈を多く占める。使用統計の送信が最初からオン（`codegraph telemetry off` で止める） | 日本語の質問は、そのままだと0件。グループ名を付ける処理で、自動的にClaudeを使う |

---

## 8. 途中で訂正したこと

- CodeGraphの「トークン62%減」は出典なしと書いたが、**公式READMEのベンチ（8/5）に載っていた**
- 動画の要約で「20,537字」と書いたが、概要欄では**25,307字**だった（自動字幕の聞き間違い）
- 「CLIをブロック」した理由を「MCPとコマンドが混ざるから」と推測したが、原文では**「コマンド経由だと回数に数えられないのに、文脈は消費する」**から
- 機械翻訳の「7で達成したのとほぼ同じ」は、正しくは**「grep方式が7回で答えられた質問（Gin）では、コストはほぼ同じ」**
- 「コードだけなら0円」は不正確だった。**地図を作るのは0円だが、グループ名を付ける処理ではClaude Codeを自動で使う**

---

## 9. 現状と次のアクション

| 項目 | 状態 |
|---|---|
| Graphifyの学習 | Lv1〜Lv5 完了（Lv6 運用、Lv7 まとめが残り） |
| Graphifyコマンド | pipxで導入済み（0.9.76）。Claude Codeのスキル・フックは未導入 |
| 練習用の地図 | httpxで作成済み。graph.htmlをブラウザで確認できる |
| CodeGraph | 未導入（資料を読んだだけ） |
| READMEの解説 | 第1回（冒頭〜Prerequisites）まで。第2回以降はまだ |

- **A. graph.htmlを触ってみる**（次にやること）
  検索欄で DigestAuth を探す／名前に「Tests」が付くグループを外して本体だけ見る／god nodeをGRAPH_REPORT.mdと見比べる
- **B. Lv6 運用**（推奨）
  更新（update・gitフック・--watch）、除外設定（.graphifyignore）、チームでの使い方、プライバシー。READMEの第3〜5回の解説と合わせて進める
- **C. 自分のプロジェクトで試す**
  100ファイル以上のプロジェクトで、`graphify install --project`（そのプロジェクトだけに入れる）。CodeGraphにも同じ質問をして、ReadやGrepの回数とトークンを比べる
- **D. Lv7 まとめ**
  CodeGraphとの使い分けを、自分の言葉で決める
- **E. 片付けたいとき**
  `pipx uninstall graphifyy` と、練習用フォルダ（graphify_lab）を削除すれば元に戻る

---

## 10. 公式資料

### CodeGraph

- ベンチマーク：https://github.com/colbymchenry/codegraph#benchmark-results
- 最後に残る文脈：https://github.com/colbymchenry/codegraph/blob/main/docs/benchmarks/residual-context-occupancy.md
- 使用統計：https://github.com/colbymchenry/codegraph/blob/main/TELEMETRY.md

### Graphify

- README：https://github.com/Graphify-Labs/graphify/blob/v8/README.md
- 仕組み：https://github.com/Graphify-Labs/graphify/blob/v8/docs/how-it-works.md
- 設計：https://github.com/Graphify-Labs/graphify/blob/v8/ARCHITECTURE.md
- ベンチ：https://github.com/Graphify-Labs/graphify/blob/v8/BENCHMARKS.md
- graph.htmlの画像：https://raw.githubusercontent.com/Graphify-Labs/graphify/v8/docs/graph-hero.png
- 公式ドキュメント：https://docs.graphify.com
- PyPI：https://pypi.org/project/graphifyy/

---

作成：2026-10-05
