---
title: "Macのcronを捨ててlaunchdに乗り換えた話——自動化パイプライン9本を移行して気づいたこと"
emoji: "✨"
type: "tech"
topics: ["claude", "python", "macos", "llm", "kindle"]
published: true
---

⏱この記事は約 11 分で読めます


📖 目次

- [📌 cron で詰まった日のこと](#cron-で詰まった日のこと)
- [📌 そもそも launchd と cron は何が違うのか](#そもそも-launchd-と-cron-は何が違うのか)
- [📌 実際に使っているplistの構造](#実際に使っているplistの構造)
- [📌 cronでは絶対ハマる PATH 問題](#cronでは絶対ハマる-path-問題)
- [📌 スリープ中に実行できるかどうか](#スリープ中に実行できるかどうか)
- [📌 launchd の使い方——3コマンド覚えれば十分](#launchd-の使い方3コマンド覚えれば十分)
- [📌 実際に詰まったポイント](#実際に詰まったポイント)
- [📌 cronを使う場面が残るとしたら](#cronを使う場面が残るとしたら)
- [📌 まとめ](#まとめ)
- [📌 関連記事](#関連記事)


最終更新: 2026-04-28


## cron で詰まった日のこと

去年の秋、BlueSky自動投稿スクリプトを夜中に動かそうとして、crontabを書いた。


```
30 7 * * * /u​sr/b​in/python3 /Users/taito/scripts/post_bsky.py

```

動かない。ターミナルで叩くと普通に動く。cron越しだと何も起きない。ログもない。本当に、何も。


3時間調べた結果、地雷が3つ同時に踏まれていた——「macOSはスリープ中にcronが実行されない」「PATHが違う」「.zshrcが読まれないので `ollama` のパスが通っていない」。どれか1つなら30分で直せた。3つ同時は堪えた。


その日launchdに書き直したら、30分で動いた。それ以来、Macで定期実行するものはlaunchd一択にしている。今は9本のパイプラインが全部launchdで回っている。


## そもそも launchd と cron は何が違うのか

Macにおけるcronは「動く場合もある」という存在だと思ったほうがいい。Apple的にはlaunchdが正式な後継で、macOS 10.4 Tigerのころから推奨されている。cronは後方互換で残っているだけ——という立ち位置だ。


launchdで何が変わるかというと、ログ取得が `.plist` に2行書くだけで終わる。


```xml
StandardOutPath
/tmp/my_job.log
StandardErrorPath
/tmp/my_job_err.log

```

cronでこれをやるには、スクリプトごとに `>> /tmp/log 2>&1` を末尾に書くか、スクリプト内でloggingを仕込むかの二択になる。設定ファイルに書き忘れて、ジョブが死んでも何のエラーも出ない状態を1週間放置したことがある。launchdに移ってから、そういう事故はなくなった。


## 実際に使っているplistの構造

今動いているパイプラインのひとつ、note記事の自動投稿ジョブはこういう形になっている。


```xml


  Label
  com.taito.note-poster

  ProgramArguments
  
    /u​sr/b​in/python3
    /Users/taito/Documents/AI_Automation_Base/01_Scripts/sns/post_note_from_queue.py
  

  StartCalendarInterval
  
    
      Hour7
      Minute0
    
    
      Hour12
      Minute30
    
    
      Hour20
      Minute30
    
  

  StandardOutPath
  /Users/taito/Documents/AI_Automation_Base/04_Config/logs/note_poster.log
  StandardErrorPath
  /Users/taito/Documents/AI_Automation_Base/04_Config/logs/note_poster_err.log

  EnvironmentVariables
  
    PATH
    /opt/homebrew/bin:/u​sr/local/bin:/u​sr/bin:/bin
    HOME
    /Users/taito
  


```

`StartCalendarInterval` に配列で複数時刻を渡せるのが地味に使いやすい。7:00 / 12:30 / 20:30の3回投稿を、1つのplistだけで管理できる。cronだと `0 7,12,20 * * *` で書けるといえば書けるが、3ヶ月後に見直したとき「これ何時だっけ」となる。plistは冗長だが読める。


## cronでは絶対ハマる PATH 問題

ターミナルで打つと動くのに、cron経由だと `command not found` になる——これが最初の壁だ。


原因はcronがlogin shellを起動しないこと。`~/.zshrc` も `~/.zprofile` も読まれない。Homebrewで入れたツール（python3、node、各種CLI）は `/opt/homebrew/bin` にあるが、cronの実行環境のPATHにそこは含まれていない。私が詰まったのも `ollama` コマンドが見つからないというエラーだった。ターミナルでは動く、cron経由では死ぬ。典型的なやつ。


launchdでも同じ問題は起きる。ただ解決が明示的でわかりやすい。plistの `EnvironmentVariables` に書くだけで、そのジョブの環境変数が固定できる。一度テンプレを作れば、9本のパイプライン全部に流用できる。


cronで同じことをやるには、crontabの先頭に `SHELL=/bin/zsh` や `PATH=...` を書くか、スクリプトの冒頭で `source ~/.zshrc` を呼ぶかになる。後者はCI環境で予期しない挙動を引き起こすことがあって、やりたくない。


## スリープ中に実行できるかどうか

MacBook Pro M5で運用しているので、これは実際に問題になった。


cronはスリープ中に指定時刻を過ぎると、そのジョブを静かにスキップする。起きてから実行されることはない。気づかないまま1週間、BSky投稿が止まっていたことがある。ログが一切なかったので、止まっていること自体に気づくまで3日かかった。


launchdも厳密にはスリープ中は動かない。ただ `RunAtLoad` と `StartInterval` を組み合わせると、「起動・復帰後に経過時間ベースで実行する」という動きをさせられる。


```xml
RunAtLoad


```

完璧な解決ではないが、cronの「黙ってスキップ」よりはずっとマシだ。今はMacが復帰したタイミングで遅れ分を流す構成にしている。


## launchd の使い方——3コマンド覚えれば十分

設定ファイルの置き場所はここ一択（ユーザースコープの場合）:


```
~/Library/LaunchAgents/

```

操作は3つのコマンドだけ覚えれば回る。


```bash
# 登録（起動）
launchctl load ~/Library/LaunchAgents/com.taito.note-poster.plist

# 停止・解除
launchctl unload ~/Library/LaunchAgents/com.taito.note-poster.plist

# 動いているか確認
launchctl list | grep com.taito

```

`launchctl list` の出力は `PID\t終了コード\tLabel` の3列。PIDが数字なら今動いている、0なら停止中、終了コードが0以外なら異常終了——という読み方をする。終了コードが78で詰まったこともあるが、それは別の記事に書いた。


## 実際に詰まったポイント

半年運用して、詰まったポイントは主に2つだった。


ひとつはplistのXML構文ミス。閉じタグを1つ忘れただけで、`launchctl load` がエラーも出さずに無視する。静かに失敗するので、しばらく気づかなかった。`plutil -lint` で事前チェックする習慣を付けてからは、このパターンでは詰まらなくなった。


```bash
plutil -lint ~/Library/LaunchAgents/com.taito.note-poster.plist

```

もうひとつはカレントディレクトリの問題。launchd経由で実行すると、カレントディレクトリが `/` になる。スクリプト内で `open('data.json')` のような相対パスを書いていると、`FileNotFoundError` で死ぬ。エラーログを見て「ファイルがない？さっきターミナルで確認したのに」と首をかしげていた。原因がカレントディレクトリだとわかったのは30分後。以来、絶対パスか `os.path.dirname(__file__)` しか使わない。


## cronを使う場面が残るとしたら

正直、今の構成でcronを使う場面はほぼない。


強いて言えば、LinuxのVPSとMacの両方で同じスクリプトを動かしたい場合だ。launchdはMac専用なので、同一の定期実行設定を両環境に持ち込みたいならcronで書いておくほうが移植コストが下がる。ただそういうケースは私の環境ではほとんど発生していない。


Mac専用のパイプラインであれば、launchd一択だと思っている。ログが取れる、PATH問題を明示的に解決できる、plistで設定が自己文書化される——この3点だけで、移行する理由として十分だった。


## まとめ

cron → launchdの移行で一番変わったのは、「自動化が壊れていることに気づける」環境になったことだ。


以前はcronがスキップしてもログが何もなくて、気づかず1週間投稿が止まっていた。launchdになってからはエラーログをDiscord Botが拾って、朝イチで「昨夜の実行でエラー出てます」と通知が来る構成になっている。9本のパイプラインが毎朝ちゃんと動いているかどうかを、コーヒーを飲みながら確認できる。


自動化は動いて終わりじゃない——壊れたときに気づける仕組みまで込みで、完成だと思っている。✨


📘 この記事のテーマをさらに深掘りした本
### [Claude Codeで副業自動化——月5万円を目指した全記録](https://www.amazon.co.jp/dp/B0GX2ZV7DP?tag=taito2525-22)

非エンジニアが1年で月10万円に到達した全パイプラインと失敗談


[Kindleで読む →](https://www.amazon.co.jp/dp/B0GX2ZV7DP?tag=taito2525-22)


一ノ瀬泰斗
AI自動化エンジニア / Python個人開発者
Claude Code × Ollama × launchd で SNS・ブログ・KDPを全自動化。実測データと失敗談を軸に、月5万円収益化のリアルな記録を発信中。


💬 **自動化の相談・小規模受託も受付中**：「launchd で毎朝 AI が動く仕組みを作りたい」「KDP の自動出版を組みたい」など、[X (@taito_automate) の DM](https://x.com/taito_automate) からお気軽にどうぞ。


### 関連記事


- [launchd exit 78 エラーの原因と修正方法——設定ミスを3分で特定する](https://ichinose-taito.com/?p=4766)
- [macOS の launchd で Python スクリプトを定時実行する完全設定手順【詰まりポイントまとめ】](https://ichinose-taito.com/?p=4595)
- [launchd × BlueSky自動エンゲージ、429エラーと日次上限に詰まった記録](https://ichinose-taito.com/launchd-x-bluesky%e8%87%aa%e5%8b%95%e3%82%a8%e3%83%b3%e3%82%b2%e3%83%bc%e3%82%b8%e3%80%81429%e3%82%a8%e3%83%a9%e3%83%bc%e3%81%a8%e6%97%a5%e6%ac%a1%e4%b8%8a%e9%99%90%e3%81%ab%e8%a9%b0%e3%81%be/)


## 関連記事


- [macOS の launchd で Python スクリプトを定時実行する完全設定手順【詰まりポイントまとめ】](https://ichinose-taito.com/macos-%e3%81%ae-launchd-%e3%81%a7-python-%e3%82%b9%e3%82%af%e3%83%aa%e3%83%97%e3%83%88%e3%82%92%e5%ae%9a%e6%99%82%e5%ae%9f%e8%a1%8c%e3%81%99%e3%82%8b%e5%ae%8c%e5%85%a8%e8%a8%ad%e5%ae%9a%e6%89%8b/)
- [launchd × BlueSky自動エンゲージ、429エラーと日次上限に詰まった記録](https://ichinose-taito.com/launchd-x-bluesky%e8%87%aa%e5%8b%95%e3%82%a8%e3%83%b3%e3%82%b2%e3%83%bc%e3%82%b8%e3%80%81429%e3%82%a8%e3%83%a9%e3%83%bc%e3%81%a8%e6%97%a5%e6%ac%a1%e4%b8%8a%e9%99%90%e3%81%ab%e8%a9%b0%e3%81%be/)
- [launchd exit 78 エラーの原因と修正方法——設定ミスを3分で特定する](https://ichinose-taito.com/launchd-exit-78-%e3%82%a8%e3%83%a9%e3%83%bc%e3%81%ae%e5%8e%9f%e5%9b%a0%e3%81%a8%e4%bf%ae%e6%ad%a3%e6%96%b9%e6%b3%95-%e8%a8%ad%e5%ae%9a%e3%83%9f%e3%82%b9%e3%82%923%e5%88%86%e3%81%a7/)

---

この記事はもともと私のブログで書いたものです。より詳しい実装例・失敗談はこちらで読めます。

👉 [Macのcronを捨ててlaunchdに乗り換えた話——自動化パイプライン9本を移行して気づいたこと](https://ichinose-taito.com/?p=4622)

