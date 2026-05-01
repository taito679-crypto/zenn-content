---
title: "Gemini 2.0 Flash をWebから無料で使い続けて分かった限界と運用の実態"
emoji: "✨"
type: "tech"
topics: ["claude", "python", "macos", "llm", "kindle"]
published: true
---

⏱この記事は約 5 分で読めます


📖 目次

- [📌 画像生成の自動化で詰まった3点](#画像生成の自動化で詰まった3点)
- [📌 1日の運用実態](#1日の運用実態)
- [📌 無料運用の限界、正直に言うと](#無料運用の限界正直に言うと)
- [📌 Watchdog を入れてから安定した](#watchdog-を入れてから安定した)
- [📌 まとめ](#まとめ)


最終更新: 2026-04-28


なぜ「Webから無料で」にこだわったか

[Playwright](https://playwright.dev/python/) を使用して独自の `user_data_dir` を立ててみたところ、Googleアカウントのセッションが毎回切れる問題があった。

そこでBraveを`--remote-debugging-port=9222`で起動し、[Chrome DevTools Protocol](https://developer.chrome.com/docs/devtools/remote-debugging)を利用した既存のログイン済みセッションをそのまま流用する方式に切り替えた。これにより常時接続が可能となった。


`launch_brave_debug.sh` のスクリプトを作成し、`com.taito.brave-cdp`というラベルでマシン起動時に自動でBraveが立ち上がるように設定した。

その際の接続側は以下のコードを使用:


```python
from playwright.sync_api import sync_playwright
from cdp_browser import connect_cdp_sync
```

この connect_cdp_sync 関数を`cdp_browser.py`に集約して使用。


`connect_cdp_sync`関数は、ポートが死んでいた場合に例外を出してDiscordに通知するようにした。これにより [launchd と Discord Bot による朝の自動通知実装](https://ichinose-taito.com/%e6%9c%9d%e3%81%ae%e3%80%8c%e7%8a%b6%e6%b3%81%e6%8a%8a%e6%8f%a15%e5%88%86%e3%80%8d%e3%82%92%e3%82%bc%e3%83%ad%e3%81%ab%e3%81%99%e3%82%8bdiscord-bot%e9%80%9a%e7%9f%a5%e3%81%ae%e4%bd%9c%e3%82%8a/)エラー検出と対応が可能となった。


## 画像生成の自動化で詰まった3点

Gemini 2.0 Flashのプロンプトに日本語を混ぜると文字化けや生成失敗問題が発生した。 日本語テキスト入り画像生成5ツール比較から、原因はGeminiの [日本語テキスト入り画像生成の 5 ツール実測比較](https://ichinose-taito.com/kdp%e8%a1%a8%e7%b4%99-%e6%97%a5%e6%9c%ac%e8%aa%9e%e3%83%86%e3%82%ad%e3%82%b9%e3%83%88%e5%85%a5%e3%82%8a%e7%94%bb%e5%83%8f%e7%94%9f%e6%88%905%e3%83%84%e3%83%bc%e3%83%ab%e6%af%94%e8%bc%83%e3%80%902026/)画像生成エンジンが日本語テキストをレイアウトに組み込む処理が不安定か、と推測される。


これに対処するため、プロンプト形式を英語で書き、入れたい日本語テキストを明示的に指定することで改善した。


```
Create a YouTube thumbnail with dark background a​nd white text saying "Python自動化で月5万円稼いだ実録". Style: tech blog, clean, minimalist. Text must be clearly legible Japanese characters.
```

また、セレクタの変更が頻繁に起こることに対処し、テキストエリアへの入力はアリアベースのセレクタを優先するように設定した。これにより、クラス名での直指定がメンテコストが高すぎる問題が解決された。


生成完了の検出タイミングも「ダウンロードボタンが出現するまで待つ」方式に変更し、画像が完成しないとダウンロードボタンが出ることを確認することで問題を解決した。


## 1日の運用実態

実際のパイプラインは以下の流れで行われている:


```bash
$ cat generate_eyecatch_for_posts.py

/Applications/Brave\ Browser.app/Contents/MacOS/Brave\ Browser --remote-debugging-port=9222 --profile-directory="Default"&

```

プロンプトをJSONファイルに溜め、`generate_eyecatch_for_posts.py`が朝7時に起動するように設定した。


生成された画像は、`/02_Content/blog_thumbnails/`ディレクトリで保存され、その後WordPressの対象記事にアイキャッチとして紐付けられる。


## 無料運用の限界、正直に言うと

Gemini 2.0 FlashのWeb経由で使用すると、生成が壊れることを前提にコードを書く必要があり、その場合に通知して手動対応を行うフローが必要となった。


生成速度も予測不能であり、レート制限やGemini側の仕様変更によって安定しない問題が発生した。


[GoogleのAPI経由での使用](https://ai.google.dev/)を検討し、もし壊れる場合はAPIキーでフォールバックするような設計が必要となった。


## Watchdog を入れてから安定した

`brave_cdp_watchdog.py`というスクリプトを追加し、BraveのCDPポートが死んでいたら自動再起動するように設定した。これによりクラッシュによるパイプライン停止問題が解消された。


```python
import subprocess, requests, time

def check_cdp():
    try:
        r = requests.get("http://localhost:9222/json", timeout=3)
        return r.status_code == 200
    except:
        return False

def restart_brave():
    subprocess.run(["pkill", "-f", "Brave Browser"])
    time.sleep(2)
    subprocess.Popen(["/Applications/Brave\ Browser.app/Contents/MacOS/Brave\ Browser", "--remote-debugging-port=9222", "--profile-directory=Default"])
```

[Python subprocess](https://docs.python.org/3/library/subprocess.html)モジュールを使用することで、BraveのCDPポートが死んでいた場合にもパイプラインは止まらなくなった。


## まとめ

Gemini 2.0 Flash のWeb経由での使用はコストゼロで高品質な画像生成をパイプラインに組み込めるが、安定性や生成速度に対して課題がある。壊れたときに素早く対応できるフローの構築が必要となり、APIキー経由でのフォールバックも検討するべきだ。


現在、Googleの仕様変更などに備えて、壊れても直せるシステムを実装していきたい。


一ノ瀬泰斗
AI自動化エンジニア / Python個人開発者
Claude Code × Ollama × launchd で SNS・ブログ・KDPを全自動化。実測データと失敗談を軸に、月5万円収益化のリアルな記録を発信中。


💬 **自動化の相談・小規模受託も受付中**：「launchd で毎朝 AI が動く仕組みを作りたい」「KDP の自動出版を組みたい」など、[X (@taito_automate) の DM](https://x.com/taito_automate) からお気軽にどうぞ。

---

この記事はもともと私のブログで書いたものです。より詳しい実装例・失敗談はこちらで読めます。

👉 [Gemini 2.0 Flash をWebから無料で使い続けて分かった限界と運用の実態](https://ichinose-taito.com/?p=4685)

