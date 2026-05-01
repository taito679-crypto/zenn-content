---
title: "WordPressにPythonで自動投稿した話——REST APIの詰まりポイント全部まとめ"
emoji: "✨"
type: "tech"
topics: ["claude", "python", "macos", "llm", "kindle"]
published: true
---

⏱この記事は約 11 分で読めます


📖 目次

- [📌 はじめに](#はじめに)
- [📌 WordPress REST API の基本——認証だけ先に片付ける](#wordpress-rest-api-の基本認証だけ先に片付ける)
- [📌 投稿スクリプトの実装](#投稿スクリプトの実装)
- [📌 タグの扱いで詰まった話](#タグの扱いで詰まった話)
- [📌 アイキャッチ画像の設定](#アイキャッチ画像の設定)
- [📌 キューからの自動投稿フロー](#キューからの自動投稿フロー)
- [📌 MarkdownをWordPressに渡すときの注意点](#markdownをwordpressに渡すときの注意点)
- [📌 実際に動かした結果](#実際に動かした結果)
- [📌 まとめ](#まとめ)
- [📌 関連記事](#関連記事)


最終更新: 2026-04-28


## はじめに

AIに記事を書かせて、人間の手を一切介さずWordPressに公開する——。言葉にすると簡単そうだし、コードを書き始めた最初の数分はそう感じた。でも認証で30分、タグの仕様で2時間近く、画像アップロードで415エラーに引っかかって……と、詰まりポイントが3か所立て続けに来た。


私の環境はMacBook Pro M5 32GB、Python 3.12。スクリプト群は`~/Documents/AI_Automation_Base/01_Scripts/blog/`に置いていて、launchdで毎朝7時に自動実行している。今は起きたら記事が公開済みになっている状態を作れた。


ドキュメント読めば分かる話は省く。実際に詰まったところだけを書く。


## WordPress REST API の基本——認証だけ先に片付ける

エンドポイント自体はシンプルだ。`/wp-json/wp/v2/posts` にPOSTするだけで記事が作れる。認証が通っていないと何の説明もなく403が即座に返ってくる。最初の30分はここで消えた。


認証方式は2択になる。


- Application Passwords（WordPress 5.6から標準搭載、プラグイン不要）
- JWT認証（プラグイン追加が必要、トークン管理も必要）


最初にJWT認証から試みた。プラグインを入れてトークンを取得する手順自体はすぐ動いた。でもトークンに有効期限があって、切れると無言で403が返ってくる。自動化スクリプトの中にリフレッシュ処理まで書くのが面倒で、Application Passwordsに切り替えた。こちらはパスワードを一度生成すれば使い続けられる。プラグイン追加もいらない。


WordPress管理画面 → ユーザー → プロフィール → 「アプリケーションパスワード」で新規生成するだけ。生成されたパスワードを`.env`に書く。


```
WP_URL=https://example.com
WP_USER=admin
WP_APP_PASS=XXXX XXXX XXXX XXXX XXXX XXXX

```

スペースが入ったまま渡していい。Pythonの`requests`はBasic認証の仕組み上そのまま処理してくれる。


## 投稿スクリプトの実装

実際に使っているコードをそのまま載せる。


```python
import os
import requests
from requests.auth import HTTPBasicAuth
from dotenv import load_dotenv

load_dotenv("04_Config/.env")

WP_URL = os.getenv("WP_URL")
WP_USER = os.getenv("WP_USER")
WP_APP_PASS = os.getenv("WP_APP_PASS")

def post_to_wordpress(title: str, content: str, tags: list[str], status: str = "draft") -> dict:
    endpoint = f"{WP_URL}/wp-json/wp/v2/posts"
    auth = HTTPBasicAuth(WP_USER, WP_APP_PASS)

    tag_ids = get_or_create_tags(tags)

    payload = {
        "title": title,
        "content": content,
        "status": status,   # "draft" or "publish"
        "tags": tag_ids,
        "format": "standard",
    }

    resp = requests.post(endpoint, json=payload, auth=auth, timeout=30)
    resp.raise_for_status()
    return resp.json()

```

`status="draft"` にしておくと下書き保存になる。品質スコアがAランク以上のものだけ`publish`で直接上げている——というより、最初は全部`publish`で動かして変な記事が公開されてから`draft`デフォルトを覚えた。パイプラインの途中にレビューを挟みたい場合は`draft`で止めておくのが無難だ。


## タグの扱いで詰まった話

ここが一番時間を食った。体感で2時間以上。


WordPress REST APIでタグを指定するとき、タグの「名前」ではなく「ID」を渡さないといけない。これを知らずに`"tags": ["Python", "AI自動化"]`という形で送ったら、エラーも出ずにタグだけ空になる。レスポンスは200が返ってくるから、投稿が5本ほどタグなしで公開されてから気づいた。


タグIDを動的に取得して、存在しなければ作成する関数を書く必要がある。


```python
def get_or_create_tags(tag_names: list[str]) -> list[int]:
    auth = HTTPBasicAuth(WP_USER, WP_APP_PASS)
    tag_ids = []

    for name in tag_names:
        search_resp = requests.get(
            f"{WP_URL}/wp-json/wp/v2/tags",
            params={"search": name, "per_page": 1},
            auth=auth,
            timeout=10
        )
        results = search_resp.json()

        if results:
            tag_ids.append(results[0]["id"])
        else:
            create_resp = requests.post(
                f"{WP_URL}/wp-json/wp/v2/tags",
                json={"name": name},
                auth=auth,
                timeout=10
            )
            tag_ids.append(create_resp.json()["id"])

    return tag_ids

```

タグ検索は部分一致で返ってくる。「Python」で検索すると「Python入門」「Python自動化」なども引っかかる。完全一致で絞りたい場合は`results[0]["name"] == name`で確認してから使う——ここをサボったら似たタグが大量に生まれて、後から整理する羽目になった。


## アイキャッチ画像の設定

記事本文だけ上げてアイキャッチが空だと一覧ページで浮く。これも自動化した。


Gemini Webで生成した画像ファイルをWordPressのメディアライブラリにアップロードし、返ってきた`media_id`を記事の`featured_media`フィールドに設定する。流れは単純だが、アップロード部分で最初に`415 Unsupported Media Type`が出た。`requests`のデフォルトのContent-TypeはJSONになっているので、`image/jpeg`を明示しないとWordPress側が受け付けない。


```python
def upload_media(image_path: str, filename: str) -> int:
    auth = HTTPBasicAuth(WP_USER, WP_APP_PASS)
    with open(image_path, "rb") as f:
        resp = requests.post(
            f"{WP_URL}/wp-json/wp/v2/media",
            headers={
                "Content-Disposition": f'attachment; filename="{filename}"',
                "Content-Type": "image/jpeg",
            },
            data=f.read(),
            auth=auth,
            timeout=60
        )
    resp.raise_for_status()
    return resp.json()["id"]

```

PNGを上げる場合は`Content-Type: image/png`に変える。ファイルによって切り替えるなら`mimetypes.guess_type(image_path)`で取得できる。タイムアウトは60秒にしてあるが、大きい画像だと足りないことがある。実測で2MB前後のファイルが15〜20秒かかった。


## キューからの自動投稿フロー

単発スクリプトを手で叩いても意味がない。「キューに積んだ記事を順番に上げていく」仕組みが本体だ。


ディレクトリ構成はこうなっている。


```
02_Content/
  blog_queue/
    pending/    ← 投稿待ちJSON
    posted/     ← 投稿済みJSON（アーカイブ）

```

各JSONの構造。


```json
{
  "title": "記事タイトル",
  "content": "本文（Markdown）",
  "tags": ["Python", "AI自動化", "個人開発"],
  "score": "A",
  "created_at": "2026-04-16T10:30:00"
}

```

投稿スクリプトは`pending/`を読んでスコア順にソートし、上位から1本取り出して投稿する。成功したら`posted/`に移動。失敗した場合はJSONにリトライカウントを書き込んで、3回超えたらスキップ。launchdで毎朝7時に回している。


```xml

StartCalendarInterval

    Hour7
    Minute0


```

これで寝ている間に翌日分が公開される。稼働から2週間、投稿の失敗は1回——Xserverが一時的に重くなってタイムアウトした。リトライ機能が動いて翌朝に上がっていた。今のところそれだけ。


## MarkdownをWordPressに渡すときの注意点

AIが生成する本文はMarkdown形式なので、HTMLに変換してから渡す。


```python
import markdown

html_content = markdown.markdown(
    md_text,
    extensions=["fenced_code", "tables", "nl2br"]
)

```

`fenced_code`を入れないとコードブロックが崩れる。これは入れ忘れると即分かる。`nl2br`は改行を`
`に変換する設定で、WordPressのClassic Editorとの相性がいい。


Gutenberg環境で動かすと、シンプルなHTMLだと保存後に見た目が崩れることがある。``のようなブロックコメントを付けて渡せば対応できるが、コメントを自動生成するラッパーを別途書く必要があって地味に面倒だ。私はClassic Editorを使っているのでこの問題は回避している。Gutenbergで動かしたいなら、ブロックコメント挿入の処理をパイプラインに組み込む必要がある。


## 実際に動かした結果

この仕組みで14本の記事を一気に公開した。平均文字数は2,430字、テーマはすべてAI自動化系。手作業なら1〜2日かかる量が、スクリプトを走らせて寝て起きたら終わっていた。


AdSense準備度チェックスクリプトで78点が出た。Search Console登録 → AdSense申請の順に進める予定だ。ただしこの記事はAI生成丸出しと判定されて「有用性の低いコンテンツ」で弾かれた——だから書き直している。量を積んだ後に質を上げる工程が抜けていたのが原因だ。自動化と品質管理は別の問題で、パイプラインに品質フェーズを組み込む必要があると分かった。


## まとめ

WordPress REST APIはドキュメントが整っているので、仕様を知っていれば詰まらない。知らないと「エラーも出ずに処理されてしまう」タイプの罠が多い——タグが名前で動かないのもそうだし、画像アップロードのContent-Typeもそうだ。


詰まったのは4点。認証はApplication Passwords一択でシンプル、タグはID指定が必須で名前では動かない、画像アップロードはContent-Typeヘッダーを明示する、MarkdownはHTMLに変換してから渡す。この順に対処していけば動く。


「毎日記事を書く」を「毎日スクリプトが記事を書いて上げる」に変えると、継続のコストが消える。完璧な記事を1本書くより70点の記事を毎日積み上げる方が検索流入的にはいい——とは言うものの、品質フェーズなしで量だけ積んだらAdSenseに弾かれた。量と質の両立が次の課題。✨


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
- [Claude Code hookでPythonスクリプトが黙って落ちる問題——パス・環境変数・作業ディレクトリの三重奏を全部解決した](https://ichinose-taito.com/claude-code-hook%e3%81%a7python%e3%82%b9%e3%82%af%e3%83%aa%e3%83%97%e3%83%88%e3%81%8c%e9%bb%99%e3%81%a3%e3%81%a6%e8%90%bd%e3%81%a1%e3%82%8b%e5%95%8f%e9%a1%8c-%e3%83%91%e3%82%b9/)
- [launchd × BlueSky自動エンゲージ、429エラーと日次上限に詰まった記録](https://ichinose-taito.com/launchd-x-bluesky%e8%87%aa%e5%8b%95%e3%82%a8%e3%83%b3%e3%82%b2%e3%83%bc%e3%82%b8%e3%80%81429%e3%82%a8%e3%83%a9%e3%83%bc%e3%81%a8%e6%97%a5%e6%ac%a1%e4%b8%8a%e9%99%90%e3%81%ab%e8%a9%b0%e3%81%be/)

---

この記事はもともと私のブログで書いたものです。より詳しい実装例・失敗談はこちらで読めます。

👉 [WordPressにPythonで自動投稿した話——REST APIの詰まりポイント全部まとめ](https://ichinose-taito.com/?p=4623)


