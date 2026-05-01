---
title: "個人ブログを毎日自動投稿する仕組みを全部Pythonで組んだ話"
emoji: "✨"
type: "tech"
topics: ["claude", "python", "macos", "llm", "kindle"]
published: true
---

⏱この記事は約 8 分で読めます


📖 目次

- [📌 なぜ手動投稿をやめたのか](#なぜ手動投稿をやめたのか)
- [📌 全体像——どんな構造になっているか](#全体像どんな構造になっているか)
- [📌 生成層——記事の自動生成とスコアリング](#生成層記事の自動生成とスコアリング)
- [📌 キュー層——blog_queue/ の管理](#キュー層blog_queue-の管理)
- [📌 投稿層——WordPress REST APIで公開](#投稿層wordpress-rest-apiで公開)
- [📌 詰まったポイント3つ](#詰まったポイント3つ)
- [📌 AdSense申請に向けた準備状況](#adsense申請に向けた準備状況)
- [📌 launchdで全自動化](#launchdで全自動化)
- [📌 まとめ](#まとめ)
- [📌 関連記事](#関連記事)


最終更新: 2026-04-28


## なぜ手動投稿をやめたのか

最初の2週間、毎日手動で書いた。そして3日連続でサボった。


「今日は疲れてるから明日でいい」——この言い訳は思ったより早く来た。2週間目の木曜日だった。翌日も来て、翌々日も来た。気づいたらダッシュボードの最終更新日が4日前になっていた。個人ブログの死に方として、これ以上典型的なパターンはないと思う。


意志に頼るのはやめた。仕組みで担保する。記事の生成から公開まで全部Pythonで回す、自動投稿パイプラインを作ることにした。


今のブログは私が何もしない日も更新される。先週1週間で14本の記事が公開された。平均2,430字。全部AI自動化テーマ。AdSense準備度スコアは78/100まで来ている。


## 全体像——どんな構造になっているか

パイプラインは3層に分かれている。


```
[生成層]
  ↓ トピック決定 → LLM記事生成 → 品質スコアリング
[キュー層]
  ↓ blog_queue/ にJSONで蓄積 → スコア順で並び替え
[投稿層]
  ↓ daily_blog_poster.py → WordPress REST API → 公開

```

生成と投稿を分離したのがこの設計の核だった。最初は「生成してすぐ投稿」にしようとしていた。シンプルだし、コードも少なくて済む。ただそうすると品質チェックを挟む場所がない。出来の悪い記事がそのまま公開される。試しに1週間やってみたら、明らかに構成の崩れた記事が3本出た。キューを間に入れることで、生成・品質確認・投稿を非同期で動かせるようになった。


## 生成層——記事の自動生成とスコアリング

記事生成は `ai_client.py` 経由でClaude Sonnet 4.6を叩いている。Ollamaのqwen2.5:3bでも動くが、2,000字超の記事では構成が崩れる。具体的には「h3見出しが4つ並んで全部同じ長さ」になりやすい。Sonnet 4.6に切り替えてからこの症状はほぼ出なくなった。


```python
# ai_client.py の呼び出しイメージ
from lib.ai_client import AIClient

client = AIClient()
article = client.generate(
    task="blog_article",
    prompt=f"トピック: {topic}\n文体: 一ノ瀬泰斗ペルソナ",
    prefer_claude=True
)

```

生成した記事は `multi_agent_quality.py` でスコアリングする。S/A/B/C/D/Fの6段階。チェックポイントは「脱AI文体ルール準拠」「キーワード密度」「体験ベース文章か」の3軸で、各軸0〜33点の合計100点。スコアが低い記事はキューの後方に回される。人間がチェックしなくても、品質の低い記事が先に出ることはない。


## キュー層——blog_queue/ の管理

生成済み記事はJSON形式で `blog_queue/` に溜まる。ファイル名のフォーマットは品質管理システムと共通にしている。


```
s085_A_20260416_python_launchd_cron.json
s072_B_20260415_ollama_memory.json
s068_B_20260414_comfyui_setup.json

```

先頭の `s085` がスコア（0〜100）。投稿スクリプトはこの数値でソートして上から消費する。ファイルを開かなくてもスコアが見える——これは地味に助かっている。


先日、10本まとめて記事を追加予約したとき、キューの深さが20本を超えた。投稿は1日1本なので3週間分のストックだ。この量になると心理的に変わる。「今日生成できなかったらどうしよう」という焦りが消える。


## 投稿層——WordPress REST APIで公開

`daily_blog_poster.py` がメインスクリプト。launchdで毎朝7時に起動する。


```python
# 投稿処理の核心部分
def post_article(article_data: dict) -> bool:
    payload = {
        "title": article_data["title"],
        "content": article_data["content"],
        "status": "publish",
        "tags": resolve_tag_ids(article_data["tags"]),
        "categories": [CATEGORY_AI_AUTOMATION],
    }
    resp = requests.post(
        f"{WP_URL}/wp-json/wp/v2/posts",
        json=payload,
        auth=(WP_USER, WP_APP_PASSWORD),
    )
    return resp.status_code == 201

```

WordPressにはアプリケーションパスワードを発行してBasic認証で叩く。OAuthを想定していたが調べたら不要だった。WordPress管理画面の「ユーザー → プロフィール → アプリケーションパスワード」で30秒で発行できる。投稿成功後はファイルをキューから削除して投稿ログに記録、失敗した場合は3回リトライしてDiscordに通知が飛ぶ。


## 詰まったポイント3つ

一番ハマったのは「重複ページ問題」だった。


WordPressを移行した際、プライバシーポリシーが3つ、お問い合わせページが4つ存在していた。スラッグが `privacy-policy`、`privacy-policy-2`、`privacy-policy-3` と枝番付きで増殖していた。Google的にはduplicate contentの扱いになるし、AdSense審査でも引っかかりやすい。非公開化を自動化したが、「スラッグにハイフン+数字が付いているページを全部拾う」というロジックが想定より厄介で、REST APIのフィルタリングを3回書き直した。


次に詰まったのは「テストドラフトの掃除」。開発中に作った `chunk-1`、`chunk-2`、`block-test` みたいなドラフトが30本以上溜まっていた。REST APIでゴミ箱に移すだけなのだが、1回のリクエストで複数投稿を一括削除できない。やむなくループで1件ずつ処理した。30本で約2分かかった。


最後は「内部リンクの自動注入」。`inject_internal_links.py` を作って、記事本文内のキーワードに関連記事へのリンクを自動で埋め込む仕組みを試している。SEO効果はまだ測定できていないが、回遊率への影響は2週間後に確認できる見込みだ。


## AdSense申請に向けた準備状況

現時点で公開記事は14本、全部AI自動化テーマに統一した。プライバシーポリシー・お問い合わせ・Aboutページは揃っている。Aboutページは旧来のMBTI紹介ページから「AI自動化エンジニア一ノ瀬泰斗」の内容に刷新済みで、プライバシーポリシーにはAdSense対応の行動ターゲティング広告条項を入れた。


スコア78/100は「合格見込み」の水準と判断している。Search Console登録が終わったら申請に進む。


MBTI系の古いページは全部非公開にした。テーマが混在していると審査側が「このブログは何について書いているのか」を判断しづらくなる。一貫性を作るのが先だった。旧コンテンツを消すことに少し迷ったが、残しておくメリットはなかった。


## launchdで全自動化

plistはこの構成で走っている。


```xml
StartCalendarInterval

    Hour
    7
    Minute
    0


```

毎朝7時に `daily_blog_poster.py` が起動して、キューの先頭から1本を公開、Discordに結果通知——それだけだ。cronと違ってMacがスリープ中でも起動タイミングをずらして実行してくれるのがlaunchdの地味な利点で、7時に起きていなくても朝の投稿が飛んでいる。失敗したときもDiscordに通知が来るので、翌朝スマホで確認するだけで昨日の投稿状況がわかる。


## まとめ

個人ブログの毎日更新を意志の力でやろうとすると、私の場合2週間が限界だった。


生成→品質チェック→キュー蓄積→自動投稿の4層構造にしたら、何もしない日でもブログが更新されるようになった。今のストックは20本以上ある。この仕組みが動いている限り、更新が止まることはない。次はSearch Console登録とAdSense申請だ。✨


一ノ瀬泰斗
AI自動化エンジニア / Python個人開発者
Claude Code × Ollama × launchd で SNS・ブログ・KDPを全自動化。実測データと失敗談を軸に、月5万円収益化のリアルな記録を発信中。


💬 **自動化の相談・小規模受託も受付中**：「launchd で毎朝 AI が動く仕組みを作りたい」「KDP の自動出版を組みたい」など、[X (@taito_automate) の DM](https://x.com/taito_automate) からお気軽にどうぞ。


### 関連記事


- [Mac で使える AI ツール 比較 2026｜個人開発者が実際に使い倒した結果](https://ichinose-taito.com/?p=4801)
- [Python で WordPress 自動投稿——REST API 実装と実運用のリアル](https://ichinose-taito.com/?p=4769)
- [Playwright Mac 入門 2026——ブラウザ自動化を最速で動かすセットアップ](https://ichinose-taito.com/?p=4768)


## 関連記事


- [Python で WordPress 自動投稿——REST API 実装と実運用のリアル](https://ichinose-taito.com/python-%e3%81%a7-wordpress-%e8%87%aa%e5%8b%95%e6%8a%95%e7%a8%bf-rest-api-%e5%ae%9f%e8%a3%85%e3%81%a8%e5%ae%9f%e9%81%8b%e7%94%a8%e3%81%ae%e3%83%aa%e3%82%a2%e3%83%ab/)
- [WordPressにPythonで自動投稿した話——REST APIの詰まりポイント全部まとめ](https://ichinose-taito.com/wordpress%e3%81%abpython%e3%81%a7%e8%87%aa%e5%8b%95%e6%8a%95%e7%a8%bf%e3%81%97%e3%81%9f%e8%a9%b1-rest-api%e3%81%ae%e8%a9%b0%e3%81%be%e3%82%8a%e3%83%9d%e3%82%a4%e3%83%b3%e3%83%88/)
- [Discord Bot に音声読み上げを実装した話——AivisSpeech + FFmpeg で詰まった3つのポイント](https://ichinose-taito.com/discord-bot-%e3%81%ab%e9%9f%b3%e5%a3%b0%e8%aa%ad%e3%81%bf%e4%b8%8a%e3%81%92%e3%82%92%e5%ae%9f%e8%a3%85%e3%81%97%e3%81%9f%e8%a9%b1-aivisspeech-ffmpeg-%e3%81%a7%e8%a9%b0%e3%81%be/)

---

この記事はもともと私のブログで書いたものです。より詳しい実装例・失敗談はこちらで読めます。

👉 [個人ブログを毎日自動投稿する仕組みを全部Pythonで組んだ話](https://ichinose-taito.com/?p=4681)


