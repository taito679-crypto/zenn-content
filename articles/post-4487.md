---
title: "Ollama をlaunchdで常駐させたら Mac がフル稼働になった話——KeepAlive の罠と2日かかった原因特定"
emoji: "✨"
type: "tech"
topics: ["claude", "python", "macos", "llm", "kindle"]
published: true
---

⏱この記事は約 7 分で読めます


📖 目次

- [📌 何が起きたか](#何が起きたか)
- [📌 環境](#環境)
- [📌 詰まったポイント](#詰まったポイント)
- [📌 解決までの手順](#解決までの手順)
- [📌 試してわかったこと](#試してわかったこと)
- [📌 まとめ](#まとめ)


最終更新: 2026-04-28


## 何が起きたか

Ollama を launchd で常駐させてから、作業中に Mac の風切り音 [Python スクリプト常駐化の環境設定チェックリスト](https://ichinose-taito.com/claude-code-hook%e3%81%a7python%e3%82%b9%e3%82%af%e3%83%aa%e3%83%97%e3%83%88%e3%81%8c%e9%bb%99%e3%81%a3%e3%81%a6%e8%90%bd%e3%81%a1%e3%82%8b%e5%95%8f%e9%a1%8c-%e3%83%91%e3%82%b9/)が急に大きくなり始めた。Activity Monitor を開くと `ollama_llama_server` が CPU 80～110% を常時食っている。モデルを使っていないのに。私は「KeepAlive=true にしとけば楽」と思って設定した plist が、逆に Mac を常時フル稼働状態にしていたことに気づくまで2日かかった。その間、ファンは24時間回り続けた。


## 環境


- macOS 15.4 Sequoia（MacBook Pro M5 32GB）
- [Ollama](https://ollama.com/) 0.6.x（Homebrew インストール済み）
- qwen3.5:9b / gemma4:26b をメインモデルとして [Ollama モデルの性能比較と選定ガイド](https://ichinose-taito.com/ollama-%e3%81%8a%e3%81%99%e3%81%99%e3%82%81%e3%83%a2%e3%83%87%e3%83%ab-10%e9%81%b8%ef%bd%9c%e6%97%a5%e6%9c%ac%e8%aa%9e%e6%80%a7%e8%83%bd%e3%81%ae%e5%ae%9f%e6%b8%ac%e3%83%a9%e3%83%b3%e3%82%ad%e3%83%b3/)使用
- launchd plist を `~/Library/LaunchAgents/` に配置
- 自動化スクリプト: `01_Scripts/lib/ai_client.py` 経由で各パイプラインが呼び出し


## 詰まったポイント

最初に疑ったのはスクリプト側だった。`ai_client.py` がループしてるとか、パイプラインが暴走してるとか。でも全部の Python プロセスを止めてもファンは回り続けた。


私は `ps aux | grep ollama` で確認すると、`ollama_llama_server` が生きていた。[Ollama](https://github.com/ollama/ollama) はモデルをリクエストされた瞬間にサーバープロセスを起動して、デフォルトで 5分間メモリに保持する仕様だ。問題はそこじゃなくて、保持が終わった後に launchd が即座にプロセスを再起動していたこと——これが原因だった。再起動のたびにモデルの再ロードが走る。その分の CPU 消費が止まらない。


`KeepAlive=true` の意味を理解していなかった。「プロセスが落ちたら即座に再起動する」という設定なので、Ollama がモデルをアンロードしてプロセスを終了しようとすると、[launchd](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingLaunchdJobs.html) が黙って復活させる。その際にモデルの再ロードが走り、CPU 使用率が上がり、プロセスが重くなり……ループだ。


エラーログは特に出ない。`/u​sr/local/var/log/` にも `~/Library/Logs/` にも何も残っていなかった。「正常に動いているけど重い」という状態で、デバッグの手がかりが掴みにくかった。syslog を見ても launchd の再起動記録は見つからず——結局、1日目は完全に暗中模索。


## 解決までの手順

**ステップ1: 現状の plist を確認する**


まず私は以下を実行した。


```bash
cat ~/Library/LaunchAgents/com.taito.ollama.plist

```

中身は `KeepAlive=true`・`RunAtLoad=true` のシンプルな構成だった。モデルが自動アンロードされてプロセスが終了するたびに launchd が再起動する仕組み。この時点で仮説が固まった。


**ステップ2: `OLLAMA_KEEP_ALIVE` を短くセットして様子を見る**


[Ollama](https://github.com/ollama/ollama/blob/main/README.md) にはモデルの保持時間を制御する環境変数がある。デフォルトは `5m`（5分）。私はこれを `0` にしてリクエスト直後に即座に解放させてみた。


```bash
launchctl unload ~/Library/LaunchAgents/com.taito.ollama.plist
OLLAMA_KEEP_ALIVE=0 ollama serve &

```

CPU 使用率が 80% から 15% に低下した。モデルが即アンロードされてプロセスが静かになっている。手応えあり——ここで確信が深まった。


**ステップ3: `KeepAlive` を OnDemand 相当に切り替える**


launchd の `KeepAlive=true` を削除して、`RunAtLoad=false` にした。Ollama は最初から起動しない。必要な時にスクリプト側から呼び出す構成に変える。この設定変更だけで、ファンの音が 2日ぶりに止まった。


**ステップ4: plist を書き直す**


```xml


  Label
  com.taito.ollama
  ProgramArguments
  
    /u​sr/local/b​in/ollama
    serve
  
  EnvironmentVariables
  
    OLLAMA_KEEP_ALIVE
    0
  
  RunAtLoad
  
  StandardErrorPath
  /tmp/ollama.err.log
  StandardOutPath
  /tmp/ollama.out.log


```

**ステップ5: 検証と本番投入**


私は以下で新しい plist を読み込んで、1週間の本番運用で確認した。


```bash
launchctl load ~/Library/LaunchAgents/com.taito.ollama.plist

```

その1週間、ファンは Ollama 呼び出し時にだけ回転し、用が済むと止まるようになった。CPU は常に 5% 以下。メモリ不足のアラートも消えた。


## 試してわかったこと

`KeepAlive` は「常に生きてることを保証する」という意味で、単なる再起動フラグではない。launchd の仕組みそのものを知らずに「常駐」の言葉だけで有効にしてしまった——これが落とし穴だった。


`OLLAMA_KEEP_ALIVE=0` にすると、モデルがメモリから即座に消える。その後のリクエストでは再ロードが走るが、バックグラウンド稼働中の無意味な再起動・再ロードと比較すれば、無視できる程度の負荷。


ログが出ないプロセスの暴走は本当に厄介だ。Activity Monitor だけでなく、`launchctl list` と `log stream --predicate 'process == "launchd"'` を併用すると、launchd の動きが可視化できる。


## まとめ

ローカルモデルを常駐させるときは、`KeepAlive=true` が本当に必要か立ち止まるべき。私の場合、オンデマンド起動で十分だった。マシンリソースが限られた環境ほど、この判断が効く。


次は Ollama の Modelfile に `PARAMETER keep_alive` を明示的に記述する方向で検討中。個別モデルごとに保持時間を制御できるから、自動化パイプラインとの相性がいい。


📘 この記事のテーマをさらに深掘りした本
### [ローカルLLM完全攻略：M2 Macで動かすAIシステム構築術](https://www.amazon.co.jp/dp/B0GX2YQT7Y?tag=taito2525-22)

Ollamaのメモリ管理から量産パイプラインまで詰まりポイントを全解説


[Kindleで読む →](https://www.amazon.co.jp/dp/B0GX2YQT7Y?tag=taito2525-22)


一ノ瀬泰斗
AI自動化エンジニア / Python個人開発者
Claude Code × Ollama × launchd で SNS・ブログ・KDPを全自動化。実測データと失敗談を軸に、月5万円収益化のリアルな記録を発信中。


💬 **自動化の相談・小規模受託も受付中**：「launchd で毎朝 AI が動く仕組みを作りたい」「KDP の自動出版を組みたい」など、[X (@taito_automate) の DM](https://x.com/taito_automate) からお気軽にどうぞ。

---

この記事はもともと私のブログで書いたものです。より詳しい実装例・失敗談はこちらで読めます。

👉 [Ollama をlaunchdで常駐させたら Mac がフル稼働になった話——KeepAlive の罠と2日かかった原因特定](https://ichinose-taito.com/?p=4487)

