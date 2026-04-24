---
title: "Claude Code × Discord で作る「情報収集の全自動パイプライン」"
emoji: "📡"
type: "tech"
topics: ["claudecode", "discord", "ai", "llm", "自動化"]
published: false
---

## はじめに

X（旧Twitter）で見つけた技術記事やAI論文のURLを、Discordにポンと投げるだけで——

1. ツイートの内容を取得
2. リンク先の記事や論文も読み込み
3. カテゴリを自動判定
4. 詳細な要約Markdownを生成
5. GitHubリポジトリにcommit & push
6. Discordに簡潔な要約を返信

これが全自動で完了します。

この記事では、Claude CodeのDiscordプラグインとSkills機能を組み合わせて作った「情報収集の全自動パイプライン」の仕組みと、1,000件以上のDiscord過去履歴を一気に整理した実践例を紹介します。

---

## なぜ作ったか - 経緯

もともと、X(旧Twitter)で見つけた技術記事や「あとで読もう」と思った論文を、Discordに自分用のサーバーを立てて、ジャンル別チャンネルに投げ込んで管理してました。AI、プログラミング、デザイン、論文…とチャンネルを分けて、URLをポンポン貼っていく感じです。

ただ、貼るだけで満足しちゃって、結局見返さない「あとで読む墓場」になっていたんですよね。

そんなときに **Claude CodeがDiscordプラグインを公開** しました。「DiscordとClaude Codeが繋がる」と聞いた瞬間、——自分が情報収集に使ってるDiscordチャンネルと、整理用のGitHubリポジトリを繋げばうまくいけるじゃん、と気づきます。

### 試行錯誤

最初はDiscordの **個人チャット(DM)** でClaude Codeとやり取りする形で試しました。シンプルだし、すぐ動くかなと。

でも実際使ってみると、AIも論文もプログラミングのURLも **全部DMに混ざる形** になって、めちゃくちゃ使いにくかったんですよね。せっかくチャンネル分けして整理してきたのに、それを台無しにしちゃう感じ。

そこで方針転換して、**Discordサーバーのチャンネル側にClaude Codeを呼ぶ** 形にしました。さらに「特定のスレッドだけ見る」設定にして、必要なときだけ反応するように。

これで **チャンネルごとに整理しつつ、見たいスレッドだけClaude Codeが処理する** という今の運用に落ち着きました。

---

## 全体アーキテクチャ

```
┌─────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   Discord    │────▶│   Claude Code    │────▶│  GitHub Repo    │
│  (スマホ/PC) │◀────│  (常駐ターミナル)  │     │  (自動push)     │
└─────────────┘     └──────────────────┘     └─────────────────┘
                           │
                    ┌──────┴──────┐
                    │  fxtwitter  │
                    │    API      │
                    └─────────────┘
```

ユーザーがやることは **DiscordにURLを貼るだけ** です。

---

## セットアップ

### 1. Claude Code Discordプラグインの導入

Claude Codeには公式のDiscordプラグインがあり、Discordチャンネルとの双方向通信が可能です。

```bash
# Claude Codeのプラグイン設定（settings.json）
{
  "enabledPlugins": {
    "discord@claude-plugins-official": true
  }
}
```

Discord Botを作成し、トークンを設定してペアリングすると、Discordからのメッセージがリアルタイムでclaude codeに届くようになります。

### 2. fxtwitter APIによるツイート取得

X(Twitter)のツイート内容を取得するために、無料で認証不要な **fxtwitter API** を使います。

```
https://api.fxtwitter.com/i/status/{TWEET_ID}
```

ツイートIDをURLから抽出し、このAPIにアクセスするだけで投稿者名、本文、画像、引用元など全ての情報がJSON形式で取得できます。

:::message alert
**⚠️ fxtwitter APIの利用に関する注意事項**

fxtwitter（FxEmbed）はMITライセンスのOSSですが、X公式APIキーを使用せず、ゲストトークンを取得してアクセスする仕組みです。**Xの利用規約（2023年9月改定）では、事前の書面による同意なくスクレイピング・クローリングを行うことを明示的に禁止しています。** そのため、fxtwitter API経由でのデータ取得はX社の規約に抵触する可能性があります。

**APIの乱用は禁止です。** 大量リクエストの連続送信は避け、個人的な小規模利用の範囲に留めてください。

本記事ではfxtwitter APIを紹介していますが、規約遵守を重視する場合は以下の代替手段を検討してください：

- **X公式API v2（Pay-Per-Use）**: 2026年2月から従量課金モデルが開始。クレジットを事前購入し、リクエストごとに消費する方式。合法的にデータ取得が可能。
- **手動での内容確認**: URLを開いて自分で内容を確認し、オリジナルの文章で要約を記述する方式。ツイート本文をそのままコピーせず、自分の言葉で要約すればデータ再配布の制限にも抵触しにくい。

なお、X公式APIでもデータ再配布にはルールがあり、完全なツイートオブジェクトの再配布は禁止されています（Post IDのみ許可）。学術機関による非商用研究目的であればPost ID再配布数に制限はありません。
:::

### 3. リポジトリのディレクトリ構成

```
discord/
├── research/          # カテゴリ別・月別に整理
│   ├── AI全般/
│   │   └── 2026/
│   │       └── 3月/
│   │           └── 記事名.md
│   ├── Claude/
│   ├── プログラミング/
│   ├── デザイン/
│   └── ...
└── paper/             # 論文PDF + 要約MDのペア
    └── AI全般/
        ├── 論文名.pdf
        └── 論文名.md
```

### 4. Skillsによるワークフロー定義

Claude CodeのSkills機能を使い、このワークフロー全体を `SKILL.md` に定義します。

```
~/.claude/skills/discord-record/SKILL.md
```

Skillsに書くことで、Claude Codeが「X URLが来たらこの手順で処理する」という判断を自律的に行えるようになります。

---

## Skillsの設計

実際には **2つのスキル** に分けて運用しています:

- **`discord-record`**: DiscordチャンネルでX URLを検知 → 取得 → 要約 → 記録 → push → 返信のメインワークフロー
- **`fetch-tweet`**: ツイート取得部分を独立した user-invocable スキルとして切り出し(`/fetch-tweet {URL}` で単発呼び出し可)

両方とも **ユーザーレベル**(`~/.claude/skills/`)に配置しているので、全プロジェクトから使えます。

```
~/.claude/skills/
├── discord-record/
│   └── SKILL.md   ← メインのワークフロー
└── fetch-tweet/
    └── SKILL.md   ← ツイート取得(切り出し)
```

:::message
SKILL.md の全文は記事末尾の **付録** に掲載しています。コピペで再利用OKです。
:::

### discord-record スキル(概要)

```yaml
---
name: discord-record
description: DiscordからのX URLを取得し、要約してリポジトリに記録
alwaysApply: false
---
```

スキルの中身には以下の6ステップを定義しています：

**Step 1: ツイート内容の取得**
- URLからツイートIDを抽出
- fxtwitter APIで本文・投稿者・引用元を取得

**Step 2: リンク先コンテンツの取得**
- Zenn、Qiita、note等の記事URLがあればWebFetchで内容も取得
- arXiv論文の場合はPDFダウンロード+詳細要約も作成

**Step 3: カテゴリ自動判定**
- 内容に基づいて11カテゴリ（AI全般、Claude、デザイン等）に自動分類

**Step 4: MDファイル作成**
- Discord返信用の簡潔な要約とは別に、MDファイルには詳細な内容を記述
- リンク先の記事・論文・ツールの内容をしっかり調べて記録

**Step 5: Git commit & push**
- 自動でステージング→コミット→プッシュ

**Step 6: Discord返信**
- カテゴリ + 1行要約 + 2-3行の要点 + MDファイルのGitHubリンク
- 簡潔に。長文禁止

### Discord返信とMDの詳細度の分離

ここが設計上のポイントです。

- **Discord返信** → 簡潔に（カテゴリ+要約+要点のみ）
- **MDファイル** → 詳細に（リンク先の記事・ツール・論文の内容まで調べて記述）

Discordはあくまで通知・確認の場であり、詳しい内容はGitHubのMDを見に行く設計です。

---

## 実践例: 1,000件の過去履歴を一気に整理

### 背景

Discordサーバーに1年以上蓄積されたXのURL群（約1,055件）を、全てこのフォーマットで詳細な要約として記録しました。

### チャンネルと件数

| チャンネル | 件数 |
|-----------|------|
| program | 301件 |
| ai | 402件 |
| デザイン | 104件 |
| 大学 | 95件 |
| claude | 81件 |
| 配信動画 | 58件 |
| unity | 38件 |
| 論文 | 34件 |
| gemini | 22件 |
| openai | 16件 |

### 処理の流れ

1. **Discordデータパッケージ**（package.zip）からmessages.jsonを抽出
2. 月別にX URLのツイートIDを抽出
3. **並列サブエージェント**（最大12件/バッチ × 5-6バッチ並列）でfxtwitter APIから一気に取得
4. 月別の詳細版まとめMDファイルを作成
5. git commit & push

### サブエージェントへのプロンプト設計

```
以下の12個のX tweet IDについて、
https://api.fxtwitter.com/i/status/{ID} にWebFetchでアクセスし
内容を取得してください。
IDs: {カンマ区切りのID}

各結果を以下の形式で返してください（エンゲージメント数値は不要）:
### YYYY-MM-DD | タイトル
- **投稿者**: 名前 (@handle)
- **URL**: https://x.com/handle/status/{ID}
- **内容**:
  - 3-5行の詳しい説明
```

**重要**: サブエージェントにはファイルパスの参照ではなく、**IDを直接プロンプトに含める**必要があります。サブエージェントからは親エージェントの`/tmp/`ファイルが見えないためです。

### 論文チャンネルの特別処理

論文チャンネル（34件）では追加で以下を実施：

1. ツイートから元論文のarXiv IDを特定（WebSearchで検索）
2. arXivからPDFをダウンロード（`discord/paper/AI全般/`に保存）
3. PDFを読んで詳細な要約MDを作成（背景・手法・実験結果・意義の構造）
4. PDF + 要約MDのペアで保存

13件の論文PDFと要約を自動生成しました。

---

## リプライチェーンの取得

X APIのPay-per-use（従量課金）を使い、自分のリプライチェーンを取得する機能も実装しています。

```python
# scripts/fetch_self_replies.py
# conversation_id + from: フィルタでリプライチェーンを取得
```

Discordから `（りぷらい）URL` の形式で送ると、リプライチェーン全体を取得して1つのMDファイルにまとめます。

---

## 運用してみて

### よかったこと

- **情報の散逸を防げる**: Discordに流れていくURLが全て構造化されて残る
- **後から検索できる**: GitHubのリポジトリ検索やgrepで過去の情報を探せる
- **要約の質が高い**: Claude Codeがリンク先の記事まで読んで要約するので、ツイート本文だけでは分からない詳細が記録される
- **スマホからでも記録できる**: DiscordにURL貼るだけなので、電車の中でも情報収集が完了

### 改善したいこと

- 画像の内容も取得・記録したい（現在は画像付きツイートの画像内容は記録されない場合がある）
- カテゴリ判定の精度向上（まれに誤分類がある）
- 重複チェック（同じURLを2回送った場合の検出）

---

## まとめ

Claude CodeのDiscordプラグイン + Skills + fxtwitter APIの組み合わせで、「URLを貼るだけで情報が整理される」パイプラインを構築しました。

1,000件以上の過去履歴も含めて全て詳細な要約として記録でき、研究活動やキャッチアップに大きく貢献しています。

Claude CodeのSkills機能は、こうした定型ワークフローの自動化に非常に強力です。一度スキルを定義すれば、あとはClaude Codeが自律的に判断して処理してくれます。

---

## 使用技術

- **Claude Code** (Opus 4.6) — AI処理の中核
- **Claude Code Discord Plugin** — Discord双方向通信
- **Claude Code Skills** — ワークフロー定義
- **fxtwitter API** — ツイート内容取得（無料・認証不要）
- **X API v2** — リプライチェーン取得（従量課金・（りぷらい）コマンド時のみ）
- **Git/GitHub** — バージョン管理・公開

---

## 付録: 実際の SKILL.md 全文

ここまで紹介した内容は、フィクションではなく実際に動かしている2つのSKILL.mdとして実装しています。コピペで再利用できるよう、全文を載せておきます。

:::details discord-record/SKILL.md(メインのワークフロー)

````markdown
---
name: discord-record
description: DiscordからのX/Twitter URLを取得し、内容を要約してリポジトリに記録し、git push後にDiscordに返信するワークフロー
alwaysApply: false
---

# Discord記録スキル

DiscordチャンネルからX(Twitter)のURLが送られた際に、内容を取得→カテゴリ判定→MDファイル作成→git commit & push→Discord返信を自動で行う。

## トリガー

Discordメッセージに `x.com` または `twitter.com` のURLが含まれている場合にこのスキルを適用する。

## ワークフロー

### Step 1: ツイート内容の取得

X URLからツイートIDを抽出し、fxtwitter APIで内容を取得する。

```
WebFetch: https://api.fxtwitter.com/i/status/{TWEET_ID}
```

取得する情報:
- 投稿日時、投稿者名、ハンドル
- ツイート本文（全文）
- 引用元ツイートの内容（あれば）
- リンク先URL（あれば）

### Step 2: リンク先コンテンツの取得（該当する場合）

ツイートにZenn記事、note記事、Qiita記事、arXiv論文等のURLが含まれる場合は、WebFetchでその内容も取得して要約に含める。

arXiv論文の場合は:
- PDFを `discord/paper/{カテゴリ}/` にダウンロード
- 詳細な要約MDも作成

### Step 3: カテゴリ判定

内容に基づいてカテゴリを自動判定する:

| カテゴリ | ディレクトリ | 判定基準 |
|---------|------------|---------|
| AI全般 | `discord/research/AI全般/` | AI、LLM、機械学習、生成AI関連 |
| Claude | `discord/research/Claude/` | Claude Code、Anthropic、Skills関連 |
| OpenAI | `discord/research/OpenAI/` | ChatGPT、GPT、OpenAI関連 |
| Gemini | `discord/research/Gemini/` | Gemini、Google AI関連 |
| プログラミング | `discord/research/プログラミング/` | プログラミング全般、Web開発 |
| デザイン | `discord/research/デザイン/` | UI/UX、CSS、Figma、フォント |
| ゲームエンジン | `discord/research/ゲームエンジン/` | Unity、Godot等 |
| 配信・動画 | `discord/research/配信・動画/` | 動画編集、配信、YouTube |
| 論文 | `discord/research/論文/` | 論文執筆、研究手法、学術 |
| 大学課題 | `discord/research/大学課題/` | 就活、大学関連 |
| その他 | `discord/research/その他/` | 上記に当てはまらないもの |

### Step 4: MDファイル作成

保存先: `discord/research/{カテゴリ}/{月}/` （3月なら `3月/`）

ファイル名: 内容の簡潔な要約（例: `Claude_Codeおすすめエージェントスキル10選.md`）

MDフォーマット:
```markdown
# タイトル

## 基本情報
- **投稿者**: 名前 (@handle)
- **元URL**: https://x.com/handle/status/{ID}
- **記事URL**: （あれば）
- **投稿日**: YYYY-MM-DD
- **記録日**: YYYY-MM-DD

## 内容

（3-10行の詳しい説明。具体的な技術名、ツール名を含む。箇条書き推奨）
```

### Step 5: Git commit & push

```bash
cd <リポジトリパス>
git add "discord/research/{カテゴリ}/{月}/"
git commit -m "Discord記録: {タイトル短縮版}"
git push
```

論文PDFがある場合は `discord/paper/` もaddする。

### Step 6: Discord返信

以下の簡潔な形式でDiscordに返信する:

```
カテゴリ: {カテゴリ名}
内容: {1行の要約タイトル}

{2-5行の要点箇条書き}

https://github.com/{owner}/{repo}/blob/main/discord/research/{カテゴリ}/{月}/{ファイル名}.md
```

**注意:**
- エンゲージメント数値（いいね、RT、ブックマーク、閲覧数）は不要
- コミットハッシュURLではなく、MDファイルの直接リンクを送る
- **Discord返信は簡潔に**。カテゴリ+1行要約+2-3行の要点のみ。長文禁止
- **MDファイルは詳細に**。リンク先の記事・論文・ツールの内容をしっかり調べて記述する。Discord返信とMDの詳細度は明確に分ける

## 特殊ケース

### 複数URLが同時に送られた場合
各URLを個別のMDファイルとして作成し、まとめて1回のコミットで処理する。

### X以外のURL（Zenn、note、YouTube等）
WebFetchで内容を取得し、同じフォーマットで記録する。元URLフィールドにそのURLを記載。

### 論文関連のツイート
元論文がarXivにある場合:
1. PDFを `discord/paper/{カテゴリ}/` にダウンロード
2. 要約MDを同ディレクトリに作成（PDF+MDペア）
3. `discord/research/論文/{月}/` にも簡易記録を作成

### （りぷらい）コマンド付きURL
`scripts/fetch_self_replies.py` を使ってリプライチェーンを取得し、全リプライの内容を含めて記録する。
````

:::

:::details fetch-tweet/SKILL.md(ツイート取得を切り出した補助スキル)

````markdown
---
name: fetch-tweet
description: X(Twitter)のURLからツイート内容（テキスト・画像）を取得して表示する。XのURLが含まれるメッセージを受け取ったときに使用する。
argument-hint: [X/TwitterのURL]
allowed-tools: WebFetch, Read, Bash
user-invocable: true
---

# X(Twitter) ツイート取得スキル

X(Twitter)のURLからfxtwitter APIを使ってツイート内容を取得・表示する。

## 手順

1. **URLからツイートIDを抽出**
   - 引数として渡されたURLから、ツイートID（数字部分）を抽出する
   - 対応形式: `https://x.com/*/status/{id}`, `https://twitter.com/*/status/{id}`

2. **fxtwitter APIでテキスト情報を取得**
   - `https://api.fxtwitter.com/i/status/{ツイートID}` にWebFetchでアクセス
   - 以下の情報を抽出:
     - 投稿者名・ハンドル
     - ツイート本文
     - 投稿日時
     - いいね数・RT数・ブックマーク数・閲覧数
     - 画像URLがあればすべて取得
     - **外部リンクURL**（論文、記事など）があればすべて取得

3. **画像がある場合は取得・読み取り**
   - 取得した画像URLにWebFetchでアクセスしてダウンロード
   - ダウンロードされたファイルをReadツールで読み取り、内容を認識する

4. **論文が関連する場合**
   - ツイート内容や画像から論文のarxiv IDやタイトルが特定できる場合:
     - arxivからPDFを探す（`https://arxiv.org/pdf/{arxiv_id}`）
     - `pdftotext` でテキスト抽出して要約
   - Discordで添付PDFがある場合はそちらも処理

5. **結果を整形して返答**
   - 以下の形式で返答する:

```
**{投稿者名}** (@{ハンドル}) — {投稿日時}

{ツイート本文}

{画像がある場合: 画像の内容説明}

いいね: {数} / RT: {数} / ブックマーク: {数} / 閲覧: {数}
```

## 引数

$ARGUMENTS
````

:::

### 配置場所と再現手順

```bash
# 1. ディレクトリ作成
mkdir -p ~/.claude/skills/discord-record
mkdir -p ~/.claude/skills/fetch-tweet

# 2. 上記の各 SKILL.md をそれぞれ配置
# (リポジトリパスやGitHub URLは自分の環境に合わせて書き換える)

# 3. Claude Code Discordプラグインを設定
# /discord:configure コマンドで対話的にセットアップ
```

ユーザーレベル(`~/.claude/skills/`)に置くことで、全プロジェクトから利用できます。プロジェクト固有にしたい場合は `<project>/.claude/skills/` でもOKです。
