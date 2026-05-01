---
title: "MacでOllamaを動かすまでにハマった話—qwen3.5:9bが動いたときの感動を記録しておく"
emoji: "✨"
type: "tech"
topics: ["claude", "python", "macos", "llm", "kindle"]
published: true
---

⏱この記事は約 9 分で読めます


📖 目次

- [📌 最初に](#最初に)
- [📌 Homebrewでのインストールとサービス起動](#homebrewでのインストールとサービス起動)
- [📌 qwen3.5:9bを入れるまで](#qwen359bを入れるまで)
- [📌 Python から叩く—APIの使い方](#python-から叩くapiの使い方)
- [📌 launchd でサービスを常時起動させる](#launchd-でサービスを常時起動させる)
- [📌 Claude と Ollama のハイブリッド運用](#claude-と-ollama-のハイブリッド運用)
- [📌 詰まりポイントまとめ](#詰まりポイントまとめ)
- [📌 まとめ](#まとめ)
- [📌 関連記事](#関連記事)


最終更新: 2026-04-28


## 最初に

Ollamaを最初に入れたのは半年くらい前だった。「ローカルでLLMが動く」という話を聞いて、深く考えずに`brew install ollama`を叩いて、そのまま放置した。なにが動かせるかもよくわかってなかったし、当時はClaudeのAPIを使えば済む話だと思っていた。


でも今は毎日使っている。


BSky投稿の量産、note記事の下書き生成、ログの要約——これ全部Ollamaが担っている。APIコストが実質ゼロになった分、気兼ねなく大量処理できるようになった。MacBook Pro M5に載せてみたら思ったより速くて、正直こんなに使えるとは思っていなかった。


この記事では、Ollamaのインストールから`qwen3.5:9b`が動くまでの流れを、詰まりポイントも含めて書いておく。


## Homebrewでのインストールとサービス起動


```bash
brew install ollama

```

これだけ。終わり、というわけでもない。


インストール後にすぐ`ollama run llama3`を試したら「connection refused」になった。サービスが起動していなかっただけなんだけど、最初はわからなくてターミナルを再起動したり、Ollamaのバイナリパスを確認したりで30分くらい溶かした。


正解はこれ。


```bash
brew services start ollama

```

これでバックグラウンドでサービスが立ち上がる。`http://localhost:11434`が疎通確認の基本。


```bash
curl http://localhost:11434

```

`Ollama is running`と返ってくればOK。返ってこなければサービスが起動していない。


## qwen3.5:9bを入れるまで

モデルを引っ張るのは`pull`コマンド。


```bash
ollama pull qwen3.5:9b

```

最初に入れたのはllama3だったんだけど、日本語の品質が微妙だった。英語の質問には強いけど、日本語のブログ記事を生成させると文体が崩れる。


`qwen3.5:9b`は日本語の精度が体感でかなり上。中国のAlibabaが作ったモデルで、アジア言語への対応が強い。9Bなので重いわけでもなく、M5のメモリ32GBに対して余裕がある。


ダウンロードには時間がかかる。5〜6GBくらいある。Wi-Fi環境で20〜30分は見ておいた方がいい。進捗はターミナルに表示されるので放置でOK。


動作確認はこれ。


```bash
ollama run qwen3.5:9b "Pythonで簡単なHello Worldを書いて"

```


```python
print("Hello, World!")

```

が返ってくれば動いている。当たり前の結果だけど、最初にこれが動いたときは妙に嬉しかった。


## Python から叩く—APIの使い方

Ollamaはローカルでのチャット以外に、REST APIとして叩ける。これが本番環境での使い方になる。


```python
import requests

def ask_ollama(prompt: str, model: str = "qwen3.5:9b") -> str:
    res = requests.post(
        "http://localhost:11434/api/generate",
        json={
            "model": model,
            "prompt": prompt,
            "stream": False,
            "options": {"think": False}
        }
    )
    return res.json()["response"]

result = ask_ollama("launchdとcronの違いを100字で説明して")
print(result)

```

`stream: False`にしているのは、レスポンスを一括で受け取りたいから。stream処理も書けるけど、パイプラインに組み込む場合は一括の方がシンプル。


`think: False`は`qwen3.5`系モデル向けの設定。このフラグを省略すると推論ステップが出力に混入することがある。パイプラインに組み込む場合は必ず入れておく。


## launchd でサービスを常時起動させる

`brew services start ollama`だとMacが再起動したときに起動するかが不安定なことがある。確実に常駐させるなら`launchd`のplistを自分で書いた方がいい。


```xml


    Label
    com.taito.ollama
    ProgramArguments
    
        /opt/homebrew/b​in/ollama
        serve
    
    RunAtLoad
    
    KeepAlive
    
    StandardOutPath
    /tmp/ollama.log
    StandardErrorPath
    /tmp/ollama_err.log


```

これを`~/Library/LaunchAgents/com.taito.ollama.plist`に置いて、


```bash
launchctl load ~/Library/LaunchAgents/com.taito.ollama.plist

```

Mac起動後に自動でOllamaが立ち上がるようになる。`KeepAlive: true`を入れているのでクラッシュしても自動再起動してくれる。


ちなみに`ollama serve`のパスは`which ollama`で確認すること。Apple Siliconだと`/opt/homebrew/b​in/ollama`になるけど、Intelだと`/u​sr/local/b​in/ollama`になる。ここを間違えるとplistが何も起動しないまま無言で失敗する。


## Claude と Ollama のハイブリッド運用

今の僕の構成は、品質が必要な出力にはClaude（Haiku/Sonnet）、量産・ログ処理・下書きにはOllamaを使い分けている。


パイプライン全体を`ai_client.py`という薄いラッパーで統一していて、設定ファイルを変えるだけでどちらのモデルを使うかを切り替えられるようにしてある。


```yaml
# ai_config.yaml
prefer_claude: true   # false にすれば全Ollama

tasks:
  bsky_post: haiku
  note_article: sonnet
  log_analysis: qwen3.5:9b

```

`prefer_claude: false`に変えれば全タスクがOllamaに流れる。API費用が心配な月はこれで乗り切る。


Ollamaだけで回してみた週があった。体感品質は落ちるけど、量産が目的の作業なら十分だった。「完璧より継続」の精神でいえば、動いていることの方が大事。✨


## 詰まりポイントまとめ

実際に引っかかった場所を書いておく。


サービスが起動していない問題は、`brew services list`で`ollama`のステータスを確認すれば一発でわかる。`started`になっていれば問題なし。`stopped`なら`brew services start ollama`。


モデルのダウンロード中に`network timeout`になる場合は、再度`ollama pull`を実行すれば途中から再開してくれる。一から落とし直しにはならないので安心していい。


Pythonから叩いたときに`ConnectionRefusedError`が出る場合は、まず`curl http://localhost:11434`を叩いてサービスの死活を確認する。Ollamaが生きていればPython側の問題（URLミスや依存関係）。


`qwen3.5:9b`が日本語を返してくれない場合は、プロンプトを日本語で明示的に書くか、「日本語で答えてください」を一行追加すると大体解決する。


## まとめ

Ollamaは入れてしまえば単純。詰まるのは最初の`connection refused`と、`qwen3.5`の`think`フラグ周りくらい。


使い始めてから、量産系の処理をClaude APIに頼ることがほぼなくなった。月のAPIコストが目に見えて下がった。その分、品質が必要なところへClaude（Haiku）を集中させられるようになった。


ローカルLLMを試していない人は、まず`ollama pull qwen3.5:9b`の一行から始めてみるといいと思う。動いた瞬間の感覚は、試してみた人にしかわからない。


```json


```


📘 この記事のテーマをさらに深掘りした本
### [Ollamaでローカルに動かすAIの全技術](https://www.amazon.co.jp/dp/B0GX3RW56V?tag=taito2525-22)

qwen3.5から gemma4 まで、無料で使いこなす実践ガイド


[Kindleで読む →](https://www.amazon.co.jp/dp/B0GX3RW56V?tag=taito2525-22)


一ノ瀬泰斗
AI自動化エンジニア / Python個人開発者
Claude Code × Ollama × launchd で SNS・ブログ・KDPを全自動化。実測データと失敗談を軸に、月5万円収益化のリアルな記録を発信中。


💬 **自動化の相談・小規模受託も受付中**：「launchd で毎朝 AI が動く仕組みを作りたい」「KDP の自動出版を組みたい」など、[X (@taito_automate) の DM](https://x.com/taito_automate) からお気軽にどうぞ。


### 関連記事


- [Ollama おすすめモデル 10選｜日本語性能の実測ランキング](https://ichinose-taito.com/?p=4800)
- [Ollama をlaunchdで常駐させたら Mac がフル稼働になった話——KeepAlive の罠と2日かかった原因特定](https://ichinose-taito.com/ollama-%e3%82%92launchd%e3%81%a7%e5%b8%b8%e9%a7%90%e3%81%95%e3%81%9b%e3%81%9f%e3%82%89-mac-%e3%81%8c%e3%83%95%e3%83%ab%e7%a8%bc%e5%83%8d%e3%81%ab%e3%81%aa%e3%81%a3%e3%81%9f%e8%a9%b1-k/)
- [AIが会話しながら自分のペルソナを学ぶアプリを作った話——ローカルLLM+音声で実現した『関係性学習』](https://ichinose-taito.com/ai%e3%81%8c%e4%bc%9a%e8%a9%b1%e3%81%97%e3%81%aa%e3%81%8c%e3%82%89%e8%87%aa%e5%88%86%e3%81%ae%e3%83%9a%e3%83%ab%e3%82%bd%e3%83%8a%e3%82%92%e5%ad%a6%e3%81%b6%e3%82%a2%e3%83%97%e3%83%aa%e3%82%92%e4%bd%9c/)


## 関連記事


- [WantedlyのPlaywright自動化で3回連続404——CDPセッション越しにフォームへ辿り着くまでの1時間](https://ichinose-taito.com/wantedly%e3%81%aeplaywright%e8%87%aa%e5%8b%95%e5%8c%96%e3%81%a73%e5%9b%9e%e9%80%a3%e7%b6%9a404-cdp%e3%82%bb%e3%83%83%e3%82%b7%e3%83%a7%e3%83%b3%e8%b6%8a%e3%81%97%e3%81%ab%e3%83%95/)
- [Ollama をlaunchdで常駐させたら Mac がフル稼働になった話——KeepAlive の罠と2日かかった原因特定](https://ichinose-taito.com/ollama-%e3%82%92launchd%e3%81%a7%e5%b8%b8%e9%a7%90%e3%81%95%e3%81%9b%e3%81%9f%e3%82%89-mac-%e3%81%8c%e3%83%95%e3%83%ab%e7%a8%bc%e5%83%8d%e3%81%ab%e3%81%aa%e3%81%a3%e3%81%9f%e8%a9%b1-k/)
- [ローカルLLMとClaude APIを月1000回使い続けて、コストの現実が見えてきた話](https://ichinose-taito.com/%e3%83%ad%e3%83%bc%e3%82%ab%e3%83%abllm%e3%81%a8claude-api%e3%82%92%e6%9c%881000%e5%9b%9e%e4%bd%bf%e3%81%84%e7%b6%9a%e3%81%91%e3%81%a6%e3%80%81%e3%82%b3%e3%82%b9%e3%83%88%e3%81%ae%e7%8f%be%e5%ae%9f/)

---

この記事はもともと私のブログで書いたものです。より詳しい実装例・失敗談はこちらで読めます。

👉 [MacでOllamaを動かすまでにハマった話—qwen3.5:9bが動いたときの感動を記録しておく](https://ichinose-taito.com/?p=4594)

