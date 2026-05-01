---
title: "macOS の launchd で Python スクリプトを定時実行する完全設定手順【詰まりポイントまとめ】"
emoji: "✨"
type: "tech"
topics: ["claude", "python", "macos", "llm", "kindle"]
published: true
---

⏱この記事は約 13 分で読めます


📖 目次

- [📌 導入](#導入)
- [📌 launchd と cron の違い、実際のところ](#launchd-と-cron-の違い実際のところ)
- [📌 .plist ファイルの基本構造](#plist-ファイルの基本構造)
- [📌 詰まったポイント3つ](#詰まったポイント3つ)
- [📌 読み込みと確認コマンド](#読み込みと確認コマンド)
- [📌 複数スクリプトを管理するときの整理](#複数スクリプトを管理するときの整理)
- [📌 ログ設計の話](#ログ設計の話)
- [📌 まとめ](#まとめ)
- [📌 関連記事](#関連記事)


最終更新: 2026-04-28


⏱この記事は約 5 分で読めます

📖 目次

- [📌 導入](#導入)
- [📌 launchd と cron の違い、実際のところ](#launchd-と-cron-の違い実際のところ)
- [📌 .plist ファイルの基本構造](#plist-ファイルの基本構造)
- [📌 詰まったポイント3つ](#詰まったポイント3つ)
- [📌 読み込みと確認コマンド](#読み込みと確認コマンド)
- [📌 複数スクリプトを管理するときの整理](#複数スクリプトを管理するときの整理)
- [📌 ログ設計の話](#ログ設計の話)
- [📌 まとめ](#まとめ)


## 導入

cron を使っていた。macOS に移行するまでは。


移行してすぐ気づいたのが「MacBook がスリープ中に実行時間を過ぎると、起動後もスルーされる」という動作だった。毎朝 07:00 に SNS 投稿スクリプトを走らせたいのに、充電しながら蓋を閉めて寝ると朝に投稿ゼロ——という日が2週間続いた。cron の仕様というより macOS のスリープ管理との相性の問題で、調べるのに2日かかった。


launchd に乗り換えたのは去年の秋ごろ。最初の .plist ファイルを書いて実際に動かすまでに丸1日溶かした。今は 30 本以上のスクリプトを launchd で管理していて、SNS 投稿・KDP 原稿生成・ブログ記事生成まで全部自動で回っている。その経験をそのまま書く。


## launchd と cron の違い、実際のところ

cron に慣れていると、[launchd](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingLaunchdJobs.html) は最初に面食らう。`* * * * *` で書けたものを XML で書かなければならない。設定ファイルが冗長で、初見のとき「なんでこんな複雑にした」と思った。


ただ慣れると launchd の方がはるかに信頼できた。スリープ復帰後に「missed」した実行を拾ってくれるし、stdout / stderr を別ファイルに吐けるし、プロセスが落ちたら自動再起動もできる。


私のパイプラインでいうと、毎日 07:00 / 12:30 / 20:30 の3回、BSky と X と note への投稿スクリプトを走らせている。cron 時代は深夜に MacBook を充電しながら放置すると7時の投稿が飛んでいた。launchd に変えてから4ヶ月、一度もそういう取りこぼしは起きていない。


## .plist ファイルの基本構造

`~/Library/LaunchAgents/` に `.plist` を置くのが基本。ここに置いたものはログインユーザー権限で動く。サーバーデーモンとして全ユーザー向けに動かしたい場合は `/Library/LaunchDaemons/` だが、個人の Python スクリプトなら LaunchAgents で十分だし、ユーザーの環境変数やホームディレクトリに素直にアクセスできる。


```


    Label
    com.taito.sns-post

    ProgramArguments
    
        /opt/homebrew/bin/python3
        /Users/ichinosetaito/Documents/AI_Automation_Base/01_Scripts/sns/sns_post_launcher.py
    

    StartCalendarInterval
    
        Hour
        7
        Minute
        0
    

    EnvironmentVariables
    
        PATH
        /opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin
    

    WorkingDirectory
    /Users/ichinosetaito/Documents/AI_Automation_Base

    StandardOutPath
    /Users/ichinosetaito/Documents/AI_Automation_Base/04_Config/logs/sns-post.log

    StandardErrorPath
    /Users/ichinosetaito/Documents/AI_Automation_Base/04_Config/logs/sns-post_err.log


```

`Label` はユニークな[逆ドメイン形式](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingLaunchdJobs.html)で書くのが慣例。ファイル名もこれに合わせる（`com.taito.sns-post.plist`）。ラベルとファイル名がズレていると `launchctl` で管理するときに混乱する——一度やった。


## 詰まったポイント3つ

### python3 のパスが通らない

一番最初に踏んだ。ターミナルで `which python3` すると `/opt/homebrew/bin/python3` が返ってくるのに、launchd から実行すると err.log に `-bash: python3: command not found` が出た。スクリプト自体は全く問題ない——のに動かない。30分溶かした。


原因は launchd の環境変数 `PATH` が、ターミナルの `PATH` と別物だから。launchd が起動するプロセスは `.zshrc` を読まない。Homebrew も pyenv も最初から存在しない扱いになる。


`EnvironmentVariables` セクションに明示的に書く必要がある。


```
EnvironmentVariables

    PATH
    /opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin


```

加えて `ProgramArguments` の Python 指定も絶対パスにする。`/usr/bin/python3` はシステム付属の古い Python なので、Homebrew 経由のものを使っているなら `/opt/homebrew/bin/python3` を直に書く。


### 仮想環境の pip モジュールが読み込まれない

私はスクリプトごとに `.venv` を切っている。launchd はその仮想環境の存在を知らないので、`import anthropic` の行で `ModuleNotFoundError: No module named 'anthropic'` が出た。ターミナルから手動実行すると動く——launchd からだけ落ちる——という状況で、最初は原因がわからなかった。


解決策は2つある。shell スクリプトをラッパーにして `source .venv/bin/activate` してから呼ぶ方法と、仮想環境の Python インタープリターを `ProgramArguments` に直接指定する方法。私は後者にしている。ログが分散しないし、plist の見通しもいい。


```
ProgramArguments

    /Users/ichinosetaito/Documents/AI_Automation_Base/.venv/bin/python3
    /Users/ichinosetaito/Documents/AI_Automation_Base/01_Scripts/sns/sns_post_launcher.py


```

### 作業ディレクトリが違う

スクリプト内で相対パスを使っていると、launchd から実行したときに `FileNotFoundError: [Errno 2] No such file or directory: 'output/result.json'` が出る。launchd のデフォルト作業ディレクトリは `/` だから。


`WorkingDirectory` キーを plist に追加するか、スクリプト側で `Path(__file__).parent` を基準にした絶対パスに統一するか——私は両方やっている。plist 側に書いておくと、スクリプトの書き方に依存しなくていい。


```
WorkingDirectory
/Users/ichinosetaito/Documents/AI_Automation_Base

```


## 読み込みと確認コマンド

`.plist` を書いたら `launchctl` で登録する。macOS Monterey 以降は `load` より `bootstrap` が推奨されている。


```
# 登録
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.taito.sns-post.plist

# 確認
launchctl list | grep com.taito

# 手動で今すぐ実行（テスト用）
launchctl kickstart -k gui/$(id -u)/com.taito.sns-post

# アンロード
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.taito.sns-post.plist

```

`launchctl list | grep com.taito` の出力の見方。


```
# PID    Status  Label
  -      0       com.taito.sns-post      # 正常終了（PID なし = 実行完了済み）
  1842   -       com.taito.sns-post      # 実行中（PID あり）
  -      2       com.taito.sns-post      # エラー終了（Status が非ゼロ）

```

PID が `-` で終了コードが `0` なら正常完了。PID に数値が出ていれば今実行中。終了コードが `-1` や `2` になっていたらエラー——err.log を見に行く。`127` は「コマンドが見つからない」なので、だいたいパス問題。


## 複数スクリプトを管理するときの整理

今は 30 本以上あるので、命名規則を決めている。Label とファイル名を一致させること、スコープ（sns / kdp / blog / system）を2番目のセグメントに入れること——この2点だけ守れば管理はだいぶ楽になる。


```
com.taito.sns-post.plist          # SNS 定時投稿（07:00 / 12:30 / 20:30）
com.taito.kdp-generate.plist      # KDP 原稿生成（毎日 02:00）
com.taito.blog-generate.plist     # ブログ記事生成（毎日 03:30）
com.taito.discord-bot.plist       # Discord Bot（常駐）
com.taito.system-cleanup.plist    # ログローテーション（毎週日曜 04:00）

```

常駐型（KeepAlive）と定時型（StartCalendarInterval）を混在させても問題ない。Discord Bot は `KeepAlive true` で動かしていて、クラッシュしても数秒以内に launchd が再起動してくれる。✨


```
KeepAlive


```

これだけ追加すれば常駐型になる。プロセス監視ツールを別途立てる必要がなくて、構成がシンプルに保てる。


## ログ設計の話

ログを雑に設計すると、何が起きているか追えなくなる。私の設計は stdout を `04_Config/logs/{name}.log`（正常出力）、stderr を `04_Config/logs/{name}_err.log`（エラー）に分けている。


スクリプト側でも Python の `logging` モジュールを使い、実行日時・処理件数・エラー内容を必ず出力している。launchd は実行のたびにログを追記してくれるが、スクリプト側で `----` のセパレーターを入れて区切りを明示している——これをやらないと複数回の実行が混ざって、どの実行でエラーが出たか追えなくなる。3回それで時間を溶かしてから徹底した。


```
import logging
import sys

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s',
    handlers=[logging.StreamHandler(sys.stdout)]
)
logger = logging.getLogger(__name__)

print("----", flush=True)  # 実行ごとのセパレーター
logger.info("開始: sns_post_launcher.py")
# 処理...
logger.info("完了: 投稿 3 件 / スキップ 0 件")

```

tail -f で眺めると、どのスクリプトがいつ動いてどれだけ処理したか一目でわかる。障害対応の速度が全然違う。


## まとめ

cron より設定が面倒なのは本当で、最初の1本を動かすまでが一番しんどい。でも動き出したら安定していて、スリープ復帰後の取りこぼしもなく、ログも綺麗に管理できる。


「手で実行したら動くのに launchd から動かない」という問題のほぼ全部は、環境変数かパスかディレクトリの問題。この3点を最初に確認する癖をつけると解決が早い。err.log に `127` が出ていたらパス、`ModuleNotFoundError` が出ていたら venv、`FileNotFoundError` が出ていたら WorkingDirectory——この3択で大体片がつく。


今は朝7時に起きると BSky・X・note への投稿が全部済んでいて、KDP の原稿も生成されている。ここまで来るのに半年かかったけど、launchd の設定で詰まっていた時間が一番長かった気がする。誰かの1時間節約になれば。


👨‍💻

一ノ瀬泰斗
AI自動化エンジニア / Python個人開発者
Claude Code × Ollama × launchd で SNS・ブログ・KDPを全自動化。実測データと失敗談を軸に、月5万円収益化のリアルな記録を発信中。


### 関連記事


- [launchd exit 78 エラーの原因と修正方法——設定ミスを3分で特定する](https://ichinose-taito.com/?p=4766)
- [Macのcronを捨ててlaunchdに乗り換えた話——自動化パイプライン9本を移行して気づいたこと](https://ichinose-taito.com/?p=4622)
- [launchd × BlueSky自動エンゲージ、429エラーと日次上限に詰まった記録](https://ichinose-taito.com/launchd-x-bluesky%e8%87%aa%e5%8b%95%e3%82%a8%e3%83%b3%e3%82%b2%e3%83%bc%e3%82%b8%e3%80%81429%e3%82%a8%e3%83%a9%e3%83%bc%e3%81%a8%e6%97%a5%e6%ac%a1%e4%b8%8a%e9%99%90%e3%81%ab%e8%a9%b0%e3%81%be/)


## 関連記事


- [launchd × BlueSky自動エンゲージ、429エラーと日次上限に詰まった記録](https://ichinose-taito.com/launchd-x-bluesky%e8%87%aa%e5%8b%95%e3%82%a8%e3%83%b3%e3%82%b2%e3%83%bc%e3%82%b8%e3%80%81429%e3%82%a8%e3%83%a9%e3%83%bc%e3%81%a8%e6%97%a5%e6%ac%a1%e4%b8%8a%e9%99%90%e3%81%ab%e8%a9%b0%e3%81%be/)
- [副業で月5万円稼ぐために私がAI自動化で実装した6つの仕組み](https://ichinose-taito.com/%e5%89%af%e6%a5%ad%e3%81%a7%e6%9c%885%e4%b8%87%e5%86%86%e7%a8%bc%e3%81%90%e3%81%9f%e3%82%81%e3%81%ab%e5%83%95%e3%81%8cai%e8%87%aa%e5%8b%95%e5%8c%96%e3%81%a7%e5%ae%9f%e8%a3%85%e3%81%97%e3%81%9f6/)
- [Macのcronを捨ててlaunchdに乗り換えた話——自動化パイプライン9本を移行して気づいたこと](https://ichinose-taito.com/mac%e3%81%aecron%e3%82%92%e6%8d%a8%e3%81%a6%e3%81%a6launchd%e3%81%ab%e4%b9%97%e3%82%8a%e6%8f%9b%e3%81%88%e3%81%9f%e8%a9%b1-%e8%87%aa%e5%8b%95%e5%8c%96%e3%83%91%e3%82%a4%e3%83%97/)


一ノ瀬泰斗
AI自動化エンジニア / Python個人開発者
Claude Code × Ollama × launchd で SNS・ブログ・KDPを全自動化。実測データと失敗談を軸に、月5万円収益化のリアルな記録を発信中。


💬 **自動化の相談・小規模受託も受付中**：「launchd で毎朝 AI が動く仕組みを作りたい」「KDP の自動出版を組みたい」など、[X (@taito_automate) の DM](https://x.com/taito_automate) からお気軽にどうぞ。

---

この記事はもともと私のブログで書いたものです。より詳しい実装例・失敗談はこちらで読めます。

👉 [macOS の launchd で Python スクリプトを定時実行する完全設定手順【詰まりポイントまとめ】](https://ichinose-taito.com/?p=4595)

