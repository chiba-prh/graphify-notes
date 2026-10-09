# Graphify

| 数字 | 意味 |
|---|---|
| **約6秒** | httpxの地図を作成（72ファイル・コードのみ） |
| **1,779 / 3,756** | できた地図のノード数／エッジ数（コミュニティ106） |
| **0円** | コードの地図作成はtree-sitterで読むのでAI料金なし（グループ名付けは除く） |
| **Lv5** | 学習の到達点（全7段階中） |

## 目次

1. [Graphify Lv1：何を解決する道具か](#1-graphify-lv1何を解決する道具か)
2. [Graphify Lv2：グラフの基本用語](#2-graphify-lv2グラフの基本用語)
3. [Graphify Lv3：地図の作り方](#3-graphify-lv3地図の作り方)
4. [Graphify Lv4：AIの使い方](#4-graphify-lv4aiの使い方)
5. [Graphify Lv5：実際に触った結果](#5-graphify-lv5実際に触った結果)
6. [Graphify の基本情報](#6-graphify-の基本情報)
7. [強みと注意点](#7-強みと注意点)
8. [途中で訂正したこと](#8-途中で訂正したこと)
9. [現状と次のアクション](#9-現状と次のアクション)
10. [公式資料](#10-公式資料)

---

## 1. Graphify Lv1：何を解決する道具か

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

## 2. Graphify Lv2：グラフの基本用語

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

## 3. Graphify Lv3：地図の作り方

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

## 4. Graphify Lv4：AIの使い方

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

## 5. Graphify Lv5：実際に触った結果

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

## 6. Graphify の基本情報

| 項目 | 内容 |
|---|---|
| 役割 | 人もAIも使う**地図**（ドキュメント・PDF・画像もつなげる） |
| 最新版 | 0.9.76（PyPI `graphifyy`）／2026-07-22にMIT→Apache-2.0。YC S26の会社で、有料のクラウド版もある |
| AIへの返し方 | コマンドかMCPで、**つながりの一覧**が返る（小さい。そのあと必要なファイルを読む） |
| 使わせる仕組み | CLAUDE.mdのルール＋フックで毎回促す。strictなら1回止める |
| 更新 | `graphify update .`、またはコミット時のgitフック。`--watch` もある |
| 公式の主張 | **正確に**（ERPNextで、回答のカバー率が70.8%→82.0%。6問）。READMEのベンチは、会話の記憶の試験 |
| 得意なこと | 全体像、AとBのつながり、コメントの「なぜ」やドキュメントとの関係 |
| 注意 | 日本語の質問は、そのままだと0件。グループ名を付ける処理で、自動的にClaudeを使う |

---

## 7. 強みと注意点

### tree-sitter とは

Graphify がコードを読むのに使っている部品。Max Brunsfeld さんが GitHub 在籍中にエディタ Atom のために作り、**2018年**に正式発表された（開発は2013〜2014年ごろから）。今では GitHub のコード表示や Neovim・Zed・Helix などで使われる定番の部品。Graphify はこれを使うので、コードの地図を**ローカルで無料**で作れる。

### いちばんの強み：セッションをまたいで使い回せる

| | ふつうのAIコーディング | Graphify |
|---|---|---|
| 新しいセッション | 毎回フォルダを一から探す（grep・読み込み） | 保存済みの地図を読むだけ |
| 理解の行方 | セッションが終わると消える | ファイルとして残る |
| コスト | 毎回トークンと時間がかかる | 作るのは一度、何度も使い回す |

代わりに、**地図を最新に保つ手間**がかかる（→ Lv6 運用）。

### 逆効果になる場面

1. **地図が古い**：コードを変えたのに更新しないと、AIは「もう無い関数」や「変わったつながり」を前提に動く。何も無いより悪い。いちばん危ない
2. **小さいプロジェクト**：レポート自体にも文字数があるので、ファイル数個ならAIが直接読むほうが少ないトークンで済むことがある
3. **作るときのコスト**：コードの解析は無料だが、グループの名前付けで自動的にClaudeを使う。何度も作り直すとトークンを使う
4. **地図に頼りすぎる**：日本語の質問は0件だった。地図で見つからないと、AIが実際のコードを読まずに「無い」と判断するおそれがある

| 向いている | 向いていない |
|---|---|
| ファイルが多い | 小さい |
| 同じリポジトリで何度も作業する | 使い捨て |
| コードがあまり頻繁に変わらない | 毎日大きく作り変える |

### Lv6 運用でやること

1. **地図の更新**：`graphify update .`（手動）／gitフック（コミットのたびに自動）／`--watch`（変更を見張って自動）
2. **除外設定（`.graphifyignore`）**：テストやビルド結果を外し、地図を小さく正確にする
3. **チームでの使い方**：グラフを共有するか、各自で作るか
4. **プライバシー**：何が外部（Claudeなど）に送られるかを確認する

---

## 8. 途中で訂正したこと

- 「コードだけなら0円」は不正確だった。**地図を作るのは0円だが、グループ名を付ける処理ではClaude Codeを自動で使う**

---

## 9. 現状と次のアクション

| 項目 | 状態 |
|---|---|
| Graphifyの学習 | Lv1〜Lv5 完了（Lv6 運用、Lv7 まとめが残り） |
| Graphifyコマンド | pipxで導入済み（0.9.76）。Claude Codeのスキル・フックは未導入 |
| 練習用の地図 | httpxで作成済み。graph.htmlをブラウザで確認できる |
| READMEの解説 | 第1回（冒頭〜Prerequisites）まで。第2回以降はまだ |

- **A. graph.htmlを触ってみる**（次にやること）
  検索欄で DigestAuth を探す／名前に「Tests」が付くグループを外して本体だけ見る／god nodeをGRAPH_REPORT.mdと見比べる
- **B. Lv6 運用**（推奨）
  更新（update・gitフック・--watch）、除外設定（.graphifyignore）、チームでの使い方、プライバシー。READMEの第3〜5回の解説と合わせて進める
- **C. 自分のプロジェクトで試す**
  100ファイル以上のプロジェクトで、`graphify install --project`（そのプロジェクトだけに入れる）
- **D. Lv7 まとめ**
  Graphifyをどんな場面で使うかを、自分の言葉で決める
- **E. 片付けたいとき**
  `pipx uninstall graphifyy` と、練習用フォルダ（graphify_lab）を削除すれば元に戻る

---

## 10. 公式資料

- README：https://github.com/Graphify-Labs/graphify/blob/v8/README.md
- 仕組み：https://github.com/Graphify-Labs/graphify/blob/v8/docs/how-it-works.md
- 設計：https://github.com/Graphify-Labs/graphify/blob/v8/ARCHITECTURE.md
- ベンチ：https://github.com/Graphify-Labs/graphify/blob/v8/BENCHMARKS.md
- graph.htmlの画像：https://raw.githubusercontent.com/Graphify-Labs/graphify/v8/docs/graph-hero.png
- 公式ドキュメント：https://docs.graphify.com
- PyPI：https://pypi.org/project/graphifyy/

---

作成：2026-10-05
