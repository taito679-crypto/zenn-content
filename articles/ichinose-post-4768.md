---
title: "Playwright Mac 入門 2026——ブラウザ自動化を最速で動かすセットアップ"
emoji: "✨"
type: "tech"
topics: ["claude", "python", "macos", "llm", "kindle"]
published: true
---

⏱この記事は約 10 分で読めます


📖 目次

- [📌 この記事に書いてあること](#この記事に書いてあること)
- [📌 Playwright を Mac に入れる](#playwright-を-mac-に入れる)
- [📌 私が詰まった3つのポイント](#私が詰まった3つのポイント)
- [📌 CDP 経由でブラウザに乗っかる](#cdp-経由でブラウザに乗っかる)
- [📌 launchd で毎朝7時に走らせる（Mac 限定）](#launchd-で毎朝7時に走らせるmac-限定)
- [📌 Playwright か Selenium か——2026 時点の判断](#playwright-か-selenium-か2026-時点の判断)
- [📌 動いてから考える](#動いてから考える)
- [📌 関連記事](#関連記事)


最終更新: 2026-04-28


## この記事に書いてあること

Playwright を Mac に入れて3日目、私は朝7時の自動投稿がぜんぶ止まっていることに気づいた。ログイン状態が毎回リセットされる仕様——知らなかった。記事はそこまでの話と、そこから先の話を書く。インストールから始めて、3回詰まったポイントを通過して、「CDP 経由で既存セッションを流用する」という私が最終的に行き着いた構成まで。公式チュートリアルをなぞるだけでは実務でここに詰まるので、そこを重点的に補足する。


## Playwright を Mac に入れる

Python 3.11.9 を使っている。3.10 以下は async 周りの挙動が微妙に変わることがあるので避けた方がいい。


```bash
pip install playwright
playwright install chromium

```

最初に `playwright install` とだけ打ったら、Chromium・Firefox・WebKit の3バイナリを全部ダウンロードし始めた。合計で1.8GB 近い。Chromium しか使わないなら最初から `playwright install chromium` と指定する。時間も容量も節約できる——気づいたのは終わった後だった。


動作確認はこれだけ。


```python
from playwright.sync_api import sync_playwright

with sync_playwright() as pw:
    browser = pw.chromium.launch(headless=False)
    page = browser.new_page()
    page.goto("https://example.com")
    print(page.title())
    browser.close()

```

`headless=False` にして目の前でブラウザが開けば成功。ここで詰まる場合、`playwright install` をまだやっていないか、仮想環境が噛み合っていないかのどちらか。`which python` と `pip show playwright` を並べて確認するのが早い。


## 私が詰まった3つのポイント

### headless モードで要素が取れない

`headless=True` にした途端、動いていたセレクタが `playwright._impl._errors.TimeoutError: Timeout 30000ms exceeded` で落ちた。JavaScript のレンダリングタイミングがズレている。SPA 系のページ（React・Vue）では `page.wait_for_load_state("networkidle")` を挟むのがほぼ必須になる。


```python
page.goto("https://example.com")
page.wait_for_load_state("networkidle")
element = page.locator("div.content")

```

`networkidle` の定義は「500ms 間ネットワークリクエストが2件以下の状態が続いた」というもの。それでも拾えないことがある。バックグラウンドでロングポーリングしているページで、`networkidle` が永遠に来ない——私はそれで30分溶かした。その場合は `domcontentloaded` に切り替えて `page.wait_for_selector` と組み合わせた方が確実なことが多い。


### ログインが毎回切れる

Playwright はデフォルトで実行のたびに新しいブラウザプロファイルを作る。毎回ログイン状態がゼロになる。X や note を自動化しようとするとここで必ず引っかかる。


対策は2つある。`storage_state` にセッションを保存して次回以降に読み込む方法と、CDP 経由で常時起動中のブラウザに接続する方法。前者はクッキーの有効期限が切れると再取得が必要で、夜間スクリプトが朝まで静かに失敗し続けるパターンにはまりやすい。私は後者の CDP 方式に統一した。Brave を CDP モードで常時起動しておけば X も note も Amazon も普段使いのセッションがそのまま使えるから——詳しくは次の節で。


### `TimeoutError` が連発する

デフォルトのタイムアウトは30秒。ネットワークが遅い環境や、要素の出現に時間がかかるページで頻繁に落ちる。`page.set_default_timeout(60000)` で伸ばせる。要素ごとに指定することもできる。


```python
page.set_default_timeout(60000)  # ms 単位、全体に適用
# または要素ごとに
page.locator("button.submit").click(timeout=60000)

```

ただし5〜10秒で TimeoutError が出るなら、タイムアウト不足ではなくセレクタの問題が多い。要素の構造が変わっているか、ページロードが完了していないか。タイムアウト値を増やす前に `headless=False` で実際に動かして目で確認する方が早い。


## CDP 経由でブラウザに乗っかる

私の構成の核心はここにある。Brave を `--remote-debugging-port=9222` で起動して、Python からそのブラウザに接続する。


```bash
# Brave を CDP モードで起動（一度だけ。あとは常時起動したまま）
/Applications/Brave\ Browser.app/Contents/MacOS/Brave\ Browser \
  --remote-debugging-port=9222 \
  --no-first-run \
  --no-default-browser-check

```

接続コードはこう。


```python
from playwright.sync_api import sync_playwright

with sync_playwright() as pw:
    browser = pw.chromium.connect_over_cdp("http://localhost:9222")
    ctx = browser.contexts[0]  # 既存コンテキストを流用
    page = ctx.new_page()
    page.goto("https://x.com")
    # ← ここで既にログイン済み状態
    browser.close()

```

`connect_over_cdp` は今開いているブラウザにそのまま乗っかる。セッションが切れても Brave で手動ログインすれば即復旧——それだけ。この設計にしてから「セッション切れで夜中のスクリプトが全滅していた」という事故がほぼ消えた。現在8本のスクリプトをこの構成で動かしている。


注意点が一つある。Brave を起動した直後でタブが何も開いていないと、`browser.contexts[0]` が空のこともある。その場合は `ctx.new_page()` で新規タブを開けばいいが、既存タブを操作したいなら `ctx.pages` でリストを確認してから使うこと。


## launchd で毎朝7時に走らせる（Mac 限定）

cron は macOS がスリープ中に止まる。launchd なら起動後に補完実行してくれる。plist の最小構成に、ログ出力を追加したもの。


```xml


  Label
  com.taito.playwright-autopost
  ProgramArguments
  
    /usr/bin/python3
    /Users/taito/scripts/autopost.py
  
  StartCalendarInterval
  
    Hour
    7
    Minute
    0
  
  StandardOutPath
  /tmp/playwright-autopost.log
  StandardErrorPath
  /tmp/playwright-autopost.err


```

`~/Library/LaunchAgents/` に置いて `launchctl load ~/Library/LaunchAgents/com.taito.playwright-autopost.plist` で登録する。`StandardOutPath` と `StandardErrorPath` は最初から入れておく。省いた状態で2週間運用したら、スクリプトがサイレントに失敗し続けていたことに気づかなかった——ログがなければ原因がまったく追えない。


## Playwright か Selenium か——2026 時点の判断

新しく始めるなら理由なく Playwright でいい。Selenium が今も使われている理由はほぼ「昔のコードがそのままある」だけだと思っている。判断が分かれる場面を整理すると。


- 新規プロジェクト → Playwright 一択
- Python + async で複数タブを並列処理したい → Playwright async API。Selenium では構造的につらい
- CDP 接続が必要 → Playwright のみ対応。Selenium はこれができない
- 既存の Selenium テストがある → 急がなければ Playwright に移行推奨。移行コストが高ければ共存も現実的


CDP 接続の可否だけで私は Playwright 一択になった。「既存セッションを流用する」という設計が Selenium では根本的にできない。


## 動いてから考える

インストールは `pip install playwright && playwright install chromium` の2行で終わる。詰まるのはその後——ログイン維持、タイムアウト、headless でのレンダリング待ち。大半はこの3点に集約される。


CDP 方式は最初の設定だけ少し手間がかかる。でも一度固めてしまえば運用が本当に楽になる。セッション切れ起因の障害がほぼゼロになったのは体感として大きかった。


エラーが出たら Playwright のメッセージをそのまま読む。他のツールと比べてエラーが親切な方なので、焦らず読めば大体解決できる。✨


📘 この記事のテーマをさらに深掘りした本
### [Claude Codeで副業自動化——月5万円を目指した全記録](https://www.amazon.co.jp/dp/B0GX2ZV7DP?tag=taito2525-22)

非エンジニアが1年で月10万円に到達した全パイプラインと失敗談


[Kindleで読む →](https://www.amazon.co.jp/dp/B0GX2ZV7DP?tag=taito2525-22)


一ノ瀬泰斗
AI自動化エンジニア / Python個人開発者
Claude Code × Ollama × launchd で SNS・ブログ・KDPを全自動化。実測データと失敗談を軸に、月5万円収益化のリアルな記録を発信中。


💬 **自動化の相談・小規模受託も受付中**：「launchd で毎朝 AI が動く仕組みを作りたい」「KDP の自動出版を組みたい」など、[X (@taito_automate) の DM](https://x.com/taito_automate) からお気軽にどうぞ。


### 関連記事


- [Mac で使える AI ツール 比較 2026｜個人開発者が実際に使い倒した結果](https://ichinose-taito.com/?p=4801)
- [Python で WordPress 自動投稿——REST API 実装と実運用のリアル](https://ichinose-taito.com/?p=4769)
- [Gemini 2.0 Flash をWebから無料で使い続けて分かった限界と運用の実態](https://ichinose-taito.com/?p=4685)


## 関連記事


- [Mac で使える AI ツール 比較 2026｜個人開発者が実際に使い倒した結果](https://ichinose-taito.com/mac-%e3%81%a7%e4%bd%bf%e3%81%88%e3%82%8b-ai-%e3%83%84%e3%83%bc%e3%83%ab-%e6%af%94%e8%bc%83-2026%ef%bd%9c%e5%80%8b%e4%ba%ba%e9%96%8b%e7%99%ba%e8%80%85%e3%81%8c%e5%ae%9f%e9%9a%9b%e3%81%ab%e4%bd%bf/)
- [Claude Code vs Cursor 比較：1ヶ月使い比べてわかった実力差](https://ichinose-taito.com/claude-code-vs-cursor-%e6%af%94%e8%bc%83%ef%bc%9a1%e3%83%b6%e6%9c%88%e4%bd%bf%e3%81%84%e6%af%94%e3%81%b9%e3%81%a6%e3%82%8f%e3%81%8b%e3%81%a3%e3%81%9f%e5%ae%9f%e5%8a%9b%e5%b7%ae/)
- [Mac の Brave を Playwright から CDP で操作した話——セッション切れとの戦いに終止符を打つまで](https://ichinose-taito.com/mac-%e3%81%ae-brave-%e3%82%92-playwright-%e3%81%8b%e3%82%89-cdp-%e3%81%a7%e6%93%8d%e4%bd%9c%e3%81%97%e3%81%9f%e8%a9%b1-%e3%82%bb%e3%83%83%e3%82%b7%e3%83%a7%e3%83%b3%e5%88%87%e3%82%8c/)

---

この記事はもともと私のブログで書いたものです。より詳しい実装例・失敗談はこちらで読めます。

👉 [Playwright Mac 入門 2026——ブラウザ自動化を最速で動かすセットアップ](https://ichinose-taito.com/?p=4768)


