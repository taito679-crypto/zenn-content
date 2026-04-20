---
title: "非エンジニアがClaude Codeに全部作らせた自動収益化システムの全貌——264スクリプト・66 launchd jobの構成を公開"
emoji: "✨"
type: "tech"
topics:
  - "claudecode"
  - "python"
  - "launchd"
  - "ollama"
  - "ai"
published: true
---

## この記事でわかること

- コードを書けない人間がAI自動収益化システムをゼロから作った全工程
- 264本のPythonスクリプト・66個のlaunchd jobの実際の構成
- Claude（品質）+ Ollama（量産）のハイブリッド運用コスト
- KDP・ブログ・note・SNSを全自動で回す仕組み
- 同じシステムを自分で作るときの手順と落とし穴

---

僕はコードが書けない。「print('hello')」は知ってるけど、それ以上を自分の手で書いたことはない。

それでも今、264本のPythonスクリプトと66個のlaunchd jobが毎日勝手に動いて、KDPで本を出版し、ブログに記事を投稿し、SNSに発信し続けている。全部Claude Codeが作った。

---

## システムの全体像

```
Mac（M5 MacBook Pro）
│
├── launchd（66 jobs）
│   ├── 03:00 KDP本生成（Claude Sonnet）
│   ├── 05:00 KDP自動出版（Playwright → Brave CDP）
│   ├── 06:30 ブログ記事生成（Claude Sonnet / Ollama）
│   ├── 07:00 SNS投稿（BSky / X / note）
│   ├── 08:00 エンゲージメント自動化
│   └── 毎時 品質チェック・キュー管理
│
├── LLM層
│   ├── Claude Sonnet → KDP・ブログ・note（品質重視）
│   └── Ollama qwen3.5:9b → SNS量産・ログ解析（コスト0）
│
├── 収益チャネル
│   ├── KDP（Amazon電子書籍）
│   ├── WordPress（AdSense）
│   ├── note（有料記事・マガジン）
│   ├── Zenn（技術記事・有料本）
│   └── BSky / X（認知・集客）
│
└── 品質管理
    └── AIスコアリング S/A/B/C/D/F → キュー優先順位制御
```

---

## なぜ「コードを書かない」で動くのか

Claude Code はターミナルで動くAIエージェント。「このスクリプトを作って」と日本語で伝えると、ファイルを読み・書き・テストし・デバッグまでする。

実際のやりとりはこんな感じ。

```
僕: ブログのキューカウントがバグってる。
    blog_queueに49件あるのに「0件」と判定されてスキップされる

Claude: blog_article_gen.sh を読みました。
        問題はここです。ファイル総数を数えていますが、
        posted_titles.jsonと照合していません。
        インラインPythonで修正します。

[10秒後]
Claude: 修正しました。posted_titles.jsonに載っていない
        ファイルだけをカウントするよう変更しました。
        launchctl kickstart で確認してください。
```

デバッグから修正まで10分。自分でやったら2時間コース。

---

## launchd 構成の実際

66個のjobを管理するのに覚えておくべきことは3つだけ。

**1. 環境変数は必ず明示する**

```xml
<key>EnvironmentVariables</key>
<dict>
    <key>PATH</key>
    <string>/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
    <key>HOME</key>
    <string>/Users/ichinosetaito</string>
</dict>
```

これを書かないと `exit 78` が量産される。launchd は login shell を継承しない。

**2. ガードファイルで多重起動を防ぐ**

```bash
GUARD_FILE="/tmp/my_job_$(date +%Y%m%d).lock"
if [ -f "$GUARD_FILE" ]; then
  echo "今日実行済み、スキップ" >> "$LOG"
  exit 0
fi
touch "$GUARD_FILE"
trap "rm -f '$GUARD_FILE'" EXIT
```

**3. ログは絶対に取る**

```xml
<key>StandardOutPath</key>
<string>/path/to/logs/job_name.log</string>
<key>StandardErrorPath</key>
<string>/path/to/logs/job_name.log</string>
```

---

## Ollama × Claude のコスト設計

| タスク | モデル | 月コスト |
|--------|--------|---------|
| KDP本文（20,000字） | Claude Sonnet | 約¥50/冊 |
| ブログ記事（3,000字） | Claude Sonnet | 約¥5/記事 |
| SNS投稿（140字） | Ollama qwen3.5:9b | ¥0 |
| ログ解析・仕分け | Ollama qwen3.5:9b | ¥0 |
| Discord bot | Ollama qwen2.5:3b | ¥0 |

Claude APIは月 ¥2,000〜¥3,000 に抑える。量産は全部ローカル。品質が必要なところだけClaudeを使う。

---

## KDP自動出版パイプライン

一番作るのが大変だったのがここ。Amazonのサイトに対してPlaywrightでブラウザ操作をする。

```python
# Brave CDP経由でKDPにアクセス（セッション維持）
from cdp_browser import connect_cdp_sync

with sync_playwright() as pw:
    browser, ctx = connect_cdp_sync(pw)
    page = ctx.new_page()
    page.goto("https://kdp.amazon.co.jp/en_US/title-setup/kindle/new")
    # タイトル入力・EPUB アップロード・価格設定・出版
```

ポイントは**BraveのCDPポートを使って既存のセッションに接続する**こと。毎回ログインしなくていいし、Amazon のbot検知も通りやすい。

---

## 品質スコアリングの仕組み

コンテンツが多くなると「とりあえず投稿」が増えてクオリティが下がる。これを防ぐためにAIが全記事を採点する。

```python
# 採点基準（100点満点）
評価項目 = {
    "文体の自然さ": 25,      # AI臭がないか
    "情報の具体性": 25,      # 数字・事例があるか
    "構成の論理性": 25,      # 読みやすいか
    "キーワード適合": 25,    # SEO観点
}
```

S級（90点以上）から順にキューに並べて投稿する。D級以下はtrashに退避。

平均スコアは現在76.6点。最高はS級90点のYouTubeショート台本。

---

## コピペで始める最小構成

```bash
# コピペ可
# 1. Ollama をインストール
brew install ollama
ollama pull qwen2.5:3b

# 2. Claude Code を入れる
npm install -g @anthropic-ai/claude-code

# 3. Claude にシステムを作らせる
claude
> Python で指定フォルダのMarkdownファイルをWordPressに自動投稿する
> スクリプトを作って。REST API使用、Basic認証、エラーログあり。
```

あとはClaudeが全部書く。

---

## よくある質問

### Claude Code は月いくらかかる？

Maxプラン $200/月（約¥30,000）。API従量課金より使い放題プランの方が安い。スクリプト生成・デバッグで1日2〜3時間使っても収まる。

### Mac が スリープするとジョブが止まる？

launchd は Mac がスリープ中に時刻が来たジョブを、起動後に自動で実行する。crontab と違ってスリープ対応済み。

### Ollamaの推奨モデルは？

日本語なら `qwen3.5:9b`（バランス型）、軽量なら `qwen2.5:3b`。M5 MacBook Proなら 8B クラスで体感1秒以下。

### KDP自動出版は規約違反にならない？

Amazonの利用規約はAI生成コンテンツを禁止していない（2026年4月時点）。ただし品質ガイドラインがある。スコアリングで低品質を弾くのはその対策でもある。

### システム全体の構築期間は？

基盤（ブログ自動投稿+SNS）で1〜2ヶ月。KDP出版パイプラインで+1ヶ月。毎日Claude Codeと1〜2時間作業した場合の目安。

---

## まとめ

- コードが書けなくてもClaude Codeが全部作る
- launchd は環境変数を明示、ガードファイルで多重起動を防ぐ
- Claude（品質）+ Ollama（量産）で月コスト¥3,000以下
- 品質スコアリングで「量産=低品質」を回避する
- KDP自動出版はBrave CDP + Playwrightが安定

現時点では「仕組みはできたが収益はこれから」という段階。ただシステムが毎日動いてコンテンツが積み上がり続けている。コンテンツは資産なので、積み上がった先に収益がついてくると信じてる。

この記事を読んだあなたに：
- [launchd exit 78 の原因と完全解決法](#)
- [Ollama で日本語が使えるモデル比較 2026年版](#)
- [Claude Code × KDP で電子書籍を自動出版する方法](#)
