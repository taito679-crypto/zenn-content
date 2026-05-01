---
title: "Python で WordPress 自動投稿——REST API 実装と実運用のリアル"
emoji: "✨"
type: "tech"
topics: ["claude", "python", "macos", "llm", "kindle"]
published: true
---

⏱この記事は約 13 分で読めます


📖 目次

- [📌 この記事でわかること](#この記事でわかること)
- [📌 WordPress REST API の基本——まずここだけ押さえる](#wordpress-rest-api-の基本まずここだけ押さえる)
- [📌 タグ・カテゴリの ID を取得する](#タグ・カテゴリの-id-を取得する)
- [📌 キューから読み込んで一括投稿する実装](#キューから読み込んで一括投稿する実装)
- [📌 実際に動かしたときのデータ](#実際に動かしたときのデータ)
- [📌 詰まったポイント3つ](#詰まったポイント3つ)
- [📌 launchd で定期実行する](#launchd-で定期実行する)
- [📌 比較: 手動投稿 vs API自動化](#比較-手動投稿-vs-api自動化)
- [📌 まとめ](#まとめ)
- [📌 関連記事](#関連記事)


最終更新: 2026-04-28


⏱この記事は約 4 分で読めます

📖 目次

- [📌 この記事でわかること](#この記事でわかること)
- [📌 WordPress REST API の基本——まずここだけ押さえる](#wordpress-rest-api-の基本まずここだけ押さえる)
- [📌 タグ・カテゴリの ID を取得する](#タグ・カテゴリの-id-を取得する)
- [📌 キューから読み込んで一括投稿する実装](#キューから読み込んで一括投稿する実装)
- [📌 実際に動かしたときのデータ](#実際に動かしたときのデータ)
- [📌 詰まったポイント3つ](#詰まったポイント3つ)
- [📌 launchd で定期実行する](#launchd-で定期実行する)
- [📌 比較: 手動投稿 vs API自動化](#比較-手動投稿-vs-api自動化)
- [📌 まとめ](#まとめ)


## この記事でわかること

WordPress REST API を Python から叩いて記事を自動投稿する実装をそのまま書く。認証の罠、キューの設計、同じ記事が3本できたやらかし話、Xserver の WAF に蹴られたときの対処——「とりあえず1本動く」から「毎日勝手に動き続ける」に持っていくまでの差分に絞った記録だ。


## WordPress REST API の基本——まずここだけ押さえる

WP 5.6 以降、Application Passwords という機能が標準で入っている。管理画面の「ユーザー → プロフィール」を一番下までスクロールすると「アプリケーションパスワード」という枠がある。名前を適当につけて「新しいアプリケーションパスワードを追加」を押すと、スペース区切りの24文字が生成される。これとユーザー名を組み合わせて Basic 認証に使う——それだけで記事が公開できる。


最初に動かしたコードはこれだけだ。


```python
import requests
from requests.auth import HTTPBasicAuth

WP_URL = "https://ichinose-taito.com"
USERNAME = "taito"
APP_PASSWORD = "AbCd EfGh IjKl MnOp QrSt UvWx"  # スペースはそのまま渡す

def create_post(title: str, content: str, tags: list = [], categories: list = []) -> dict:
    res = requests.post(
        f"{WP_URL}/wp-json/wp/v2/posts",
        json={
            "title": title,
            "content": content,
            "status": "publish",
            "tags": tags,
            "categories": categories,
        },
        auth=HTTPBasicAuth(USERNAME, APP_PASSWORD),
        timeout=30,
    )
    res.raise_for_status()
    return res.json()

```

シンプル。拍子抜けするくらい。ただし、この後すぐに罠にはまる。


## タグ・カテゴリの ID を取得する

tags フィールドにタグ名の文字列を渡してもエラーにならない。ただ、タグが何もつかない。WordPress は tags に「数値 ID の配列」を要求する。最初の10本はタグなしで公開してしまっていた。


毎回 API でタグ名 → ID を引く関数を作ると、後からタグを追加しても壊れないので最終的にこの形に落ち着いた。


```python
def get_or_create_tag_id(name: str) -> int:
    res = requests.get(
        f"{WP_URL}/wp-json/wp/v2/tags",
        params={"search": name, "per_page": 1},
        auth=HTTPBasicAuth(USERNAME, APP_PASSWORD),
    )
    data = res.json()
    if data:
        return data[0]["id"]
    # 存在しないなら作る
    r = requests.post(
        f"{WP_URL}/wp-json/wp/v2/tags",
        json={"name": name},
        auth=HTTPBasicAuth(USERNAME, APP_PASSWORD),
    )
    return r.json()["id"]

def resolve_tag_ids(tag_names: list[str]) -> list[int]:
    return [get_or_create_tag_id(n) for n in tag_names]

```

search パラメータの部分一致に注意。「Python」で検索すると「Python自動化」もヒットする場合がある。per_page=1 で先頭だけ取ってきているので、タグ名は完全一致に近い形で渡したほうが安全だ。


## キューから読み込んで一括投稿する実装

私の環境では、Claude Sonnet 4.6 で生成した記事を JSON ファイルとして `queue/` ディレクトリに溜めている。ファイル名は `20260428_001_python-wp-api.json` のようにタイムスタンプ + 連番にしている。これを sorted() で読むだけで投稿順序が決まる。


```python
import json
import shutil
import time
import markdown2
from pathlib import Path

QUEUE_DIR = Path("/Users/ichinosetaito/Documents/AI_Automation_Base/queue/blog")
DONE_DIR  = Path("/Users/ichinosetaito/Documents/AI_Automation_Base/done/blog")
DONE_DIR.mkdir(parents=True, exist_ok=True)

def post_from_queue():
    files = sorted(QUEUE_DIR.glob("*.json"))
    if not files:
        print("queue empty")
        return

    for f in files:
        data = json.loads(f.read_text(encoding="utf-8"))
        html_content = markdown2.markdown(
            data["content"],
            extras=["fenced-code-blocks", "tables", "break-on-newline"],
        )
        tag_ids = resolve_tag_ids(data.get("tags", []))
        result = create_post(
            title=data["title"],
            content=html_content,
            tags=tag_ids,
        )
        print(f"[OK] {result['link']}")
        shutil.move(str(f), str(DONE_DIR / f.name))
        time.sleep(3)  # Xserver WAF 対策、1秒だと詰まる

if __name__ == "__main__":
    post_from_queue()

```

処理済みファイルを done/ に移動しているのは、二重投稿を防ぐため——これをさぼって一度やらかした。同じ記事が3本並んだ。消すのも手作業になるので、shutil.move は必ず入れる。


## 実際に動かしたときのデータ

先週、queue/ に溜まっていた AI 自動化テーマの記事10本をまとめて流した。MacBook Pro M5 32GB + Xserver の組み合わせで計測した数字がこれだ。


項目
数値


投稿数
10本


平均文字数
2,430字


所要時間
約4分


エラー
0件


累計公開記事
14本


4分で10本。手動でやっていたとき、コピペ・タグ設定・カテゴリ選択だけで1本5分はかかっていた。コードを書く時間を含めても、最初の週でペイした。


AdSense 準備スコアが 78 まで上がったのはこのタイミングだ。記事数が増えたのと、About・プライバシーポリシー・お問い合わせのページ構成が揃ったのが重なった。記事の量と質は切り離せない——どちらか一方では数字が動かない、というのが私の実感だ。


## 詰まったポイント3つ

### 1. 401 が返り続けて30分溶けた

Application Password のスペースを除去して渡していた。`"AbCdEfGhIjKlMnOpQrStUvWx"` と渡すと 401。`"AbCd EfGh IjKl MnOp QrSt UvWx"` と渡すと通る。公式ドキュメントには「スペースは無視される」と書いてあるが、それは UI 上の話で、送信時はスペース込みが正解だ。エラーメッセージが `{"code":"rest_forbidden","message":"Sorry, you are not allowed to do that."}` と素っ気なく、認証の問題なのかパーミッションの問題なのか区別がつかない。結局 HTTPBasicAuth に渡す文字列を生のままにして解決した。


### 2. Markdown をそのまま渡すと改行が全滅する

content フィールドに Markdown のテキストを直接渡すとブロックエディタ側でレンダリングされない。改行も段落も消える。markdown2 ライブラリで HTML に変換してから渡す必要がある。


```python
import markdown2

html = markdown2.markdown(
    raw_md,
    extras=["fenced-code-blocks", "tables", "break-on-newline"],
)

```

`fenced-code-blocks` を extras に含め忘れると、コードブロックが ````` の記号ごと表示される。地味にハマった。インストールは `pip install markdown2` で終わる。


### 3. Xserver の WAF が連続リクエストを弾く

10本を一気に投げると途中から `403` が返ってくる場面があった。Xserver の WAF が SQL インジェクション風のパターンや連続 POST を検知して遮断するらしい。投稿ごとに `time.sleep(3)` を挟むと安定した。1秒だと不安定だったので、3秒は経験値として固定している。WordPress.com ホスティングなら公式のレート制限があるのでそちらも要注意だ。


## launchd で定期実行する

Mac 環境なので crontab ではなく launchd を使っている。plist ファイルを書いて `~/Library/LaunchAgents/` に置くだけ。cron と違ってスリープ復帰後にも動いてくれるのがいい。


```xml


  Label
  com.taito.wp-poster
  ProgramArguments
  
    /opt/homebrew/bin/python3
    /Users/ichinosetaito/Documents/AI_Automation_Base/01_Scripts/blog/post_from_queue.py
  
  StartCalendarInterval
  
    Hour9
    Minute0
  
  StandardOutPath
  /tmp/wp-poster.log
  StandardErrorPath
  /tmp/wp-poster.err
  EnvironmentVariables
  
    PATH
    /opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin
  


```


```bash
launchctl load ~/Library/LaunchAgents/com.taito.wp-poster.plist

```

毎朝9時に動く。起きたら `cat /tmp/wp-poster.log` で確認して「今日は何本公開されたか」を把握するだけになった。仕組みを作れば、自分が何もしなくても記事が増える——これが「完璧より継続」の実体だと思っている。


注意点として、ProgramArguments の python3 パスは `which python3` で確認すること。launchd はログインシェルの PATH を引き継がないので、homebrew 環境だと `/usr/bin/python3` を指定するとライブラリが見つからずに落ちる。最初これで詰まった。


## 比較: 手動投稿 vs API自動化


観点
手動投稿
API自動化


1本あたりの時間
5〜10分
数秒


タグ・カテゴリのミス
たまに起きる
コードで統一される


定期投稿の安定性
忘れる
launchd が忘れない


初期コスト
ゼロ
実装に3〜5時間


特殊フォーマットへの対応
すぐ対応できる
コード変更が必要


手動の方が融通が利くのは本当で、画像の配置が複雑な記事や特殊なショートコードを使う場合は今でも手で調整している。ただ、「毎日1本」という継続を担保するには自動化しかなかった。意志の力で続けようとすると、3日目か4日目で必ず崩れる。


## まとめ

WordPress REST API + Python の自動投稿、実装自体は半日で終わる。ハードルになるのは認証の挙動とキュー管理の設計——この2点だけだ。最初からきれいに作ろうとしないで、「1本投稿できる50行」から始めて壊しながら直していくのが現実的だと思っている。私は同じ記事を3本公開するやらかしを経て今の形になった。


Search Console の登録が終わり次第、AdSense 申請に進む。その結果もここに書く。✨


👨‍💻

一ノ瀬泰斗
AI自動化エンジニア / Python個人開発者
Claude Code × Ollama × launchd で SNS・ブログ・KDPを全自動化。実測データと失敗談を軸に、月5万円収益化のリアルな記録を発信中。


### 関連記事


- [Mac で使える AI ツール 比較 2026｜個人開発者が実際に使い倒した結果](https://ichinose-taito.com/?p=4801)
- [Playwright Mac 入門 2026——ブラウザ自動化を最速で動かすセットアップ](https://ichinose-taito.com/?p=4768)
- [Gemini 2.0 Flash をWebから無料で使い続けて分かった限界と運用の実態](https://ichinose-taito.com/?p=4685)


## 関連記事


- [WordPressにPythonで自動投稿した話——REST APIの詰まりポイント全部まとめ](https://ichinose-taito.com/wordpress%e3%81%abpython%e3%81%a7%e8%87%aa%e5%8b%95%e6%8a%95%e7%a8%bf%e3%81%97%e3%81%9f%e8%a9%b1-rest-api%e3%81%ae%e8%a9%b0%e3%81%be%e3%82%8a%e3%83%9d%e3%82%a4%e3%83%b3%e3%83%88/)
- [Claude Code vs Cursor 比較：1ヶ月使い比べてわかった実力差](https://ichinose-taito.com/claude-code-vs-cursor-%e6%af%94%e8%bc%83%ef%bc%9a1%e3%83%b6%e6%9c%88%e4%bd%bf%e3%81%84%e6%af%94%e3%81%b9%e3%81%a6%e3%82%8f%e3%81%8b%e3%81%a3%e3%81%9f%e5%ae%9f%e5%8a%9b%e5%b7%ae/)
- [Playwright Mac 入門 2026——ブラウザ自動化を最速で動かすセットアップ](https://ichinose-taito.com/playwright-mac-%e5%85%a5%e9%96%80-2026-%e3%83%96%e3%83%a9%e3%82%a6%e3%82%b6%e8%87%aa%e5%8b%95%e5%8c%96%e3%82%92%e6%9c%80%e9%80%9f%e3%81%a7%e5%8b%95%e3%81%8b%e3%81%99%e3%82%bb%e3%83%83/)


### この記事を書いた人

**一ノ瀬 泰斗** — AI自動化エンジニア・KDP出版者


    Claude Code + Ollama + launchd を組み合わせ、ブログ・KDP電子書籍・SNS発信を自動化中。

    自分で作って自分で失敗して自分で直す、作業途中の生々しさが武器。

    本ブログでは実体験ベースの実測データ・失敗談・詰まりポイントを中心に発信しています。
  


    お問い合わせは [コンタクトページ](/contact/) から /

    [プライバシーポリシー](/privacy-policy/)
  


一ノ瀬泰斗
AI自動化エンジニア / Python個人開発者
Claude Code × Ollama × launchd で SNS・ブログ・KDPを全自動化。実測データと失敗談を軸に、月5万円収益化のリアルな記録を発信中。


💬 **自動化の相談・小規模受託も受付中**：「launchd で毎朝 AI が動く仕組みを作りたい」「KDP の自動出版を組みたい」など、[X (@taito_automate) の DM](https://x.com/taito_automate) からお気軽にどうぞ。

---

この記事はもともと私のブログで書いたものです。より詳しい実装例・失敗談はこちらで読めます。

👉 [Python で WordPress 自動投稿——REST API 実装と実運用のリアル](https://ichinose-taito.com/?p=4769)


