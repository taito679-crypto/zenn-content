---
title: "Ollama vs LM Studio 比較——ローカルLLM環境の選び方"
emoji: "✨"
type: "tech"
topics: ["claude", "python", "macos", "llm", "kindle"]
published: true
---

⏱この記事は約 9 分で読めます


📖 目次

- [📌 Ollama vs LM Studio 機能比較表](#ollama-vs-lm-studio-機能比較表)
- [📌 Ollamaをパイプラインに組み込んで実感したこと](#ollamaをパイプラインに組み込んで実感したこと)
- [📌 LM Studioで失敗してわかったこと](#lm-studioで失敗してわかったこと)
- [📌 実測：qwen2.5-3bのレスポンス速度（M5 MacBook Pro）](#実測qwen25-3bのレスポンス速度m5-macbook-pro)
- [📌 Ollamaで自動化パイプラインを動かすセットアップ](#ollamaで自動化パイプラインを動かすセットアップ)
- [📌 選定の判断基準](#選定の判断基準)


最終更新: 2026-04-28


⏱この記事は約 12 分で読めます

📖 目次

- [Ollama vs LM Studio 機能比較表](#ollama-vs-lm-studio-機能比較表)
- [Ollamaをパイプラインに組み込んで実感したこと](#ollamaをパイプラインに組み込んで実感したこと)
- [LM Studioで失敗してわかったこと](#lm-studioで失敗してわかったこと)
- [実測：qwen2.5-3bのレスポンス速度（M5 MacBook Pro） Ollamaのおすすめモデル 10選](#実測-qwen25-3bのレスポンス速度m5-macbook-pro)
- [Ollamaで自動化パイプラインを動かすセットアップ 朝の「状況把握5分」をゼロにするDiscord Bot通知の作り方](#ollamaで自動化パイプラインを動かすセットアップ)
- [選定の判断基準](#選定の判断基準)


ローカルLLMを動かすとき、OllamaとLM Studioのどちらを選ぶかで最初に詰まった。私も両方入れて試して、Ollamaに完全移行するまで3週間ほどかかっている。


移行を決めたのは夜中の障害だった。launchdで組んだ自動投稿バッチが朝4時に止まっていて [launchd exit 78 エラーの原因と修正方法](https://ichinose-taito.com/launchd-exit-78-%e3%82%a8%e3%83%a9%e3%83%bc%e3%81%ae%e5%8e%9f%e5%9b%a0%e3%81%a8%e4%bf%ae%e6%ad%a3%e6%96%b9%e6%b3%95-%e8%a8%ad%e5%ae%9a%e3%83%9f%e3%82%b9%e3%82%923%e5%88%86%e3%81%a7/)、ログに `ConnectionRefusedError: [Errno 61] Connection refused` が並んでいた。LM Studioが起動していなかった——GUIを手動で開かないとAPIサーバーが立ち上がらない設計を、完全に失念していた。その朝、修正しながら気づいた。GUIありきの設計は、自動化の文脈では最終的に詰む、と。


この記事はそこから3週間かけてOllamaに移行した過程の記録だ。M5 MacBook Pro 32GB環境での実測値と、LM Studioで3回詰まったポイントも書いておく。


## Ollama vs LM Studio 機能比較表


項目
Ollama
LM Studio


インターフェース
CLI / REST API
GUIメイン


API互換性
[OpenAI互換（/v1/chat/completions）](https://github.com/ollama/ollama/blob/main/docs/api.md)
OpenAI互換あり


モデル追加方法
`ollama pull モデル名`
GUIでダウンロード


自動化との相性
高い（launchd / cronと組みやすい）
低め（GUIの手動起動が前提）


メモリ管理
自動アンロード（デフォルト5分）
手動操作 / GUI依存


モデル対応数
[Ollamaライブラリ内（公式+コミュニティ）](https://ollama.com/library)
HuggingFace / ローカルGGUF直接読み込み


macOS launchd対応
公式サービス化済み（v0.1+）
非対応（常駐はGUI依存）


GPU加速（Apple Silicon）
[Metal対応済み](https://github.com/ollama/ollama#metal-acceleration)
[Metal対応済み](https://lmstudio.ai)


ファイル拡張設定
Modelfile（テンプレート化可能）
GUI設定（バージョン固定）


向いている用途
API自動化・常駐サービス・バッチ処理
モデル試し打ち・単発検証


## Ollamaをパイプラインに組み込んで実感したこと

移行して最初に感じたのは、セットアップの潔さだった。


`brew services start ollama` を一回叩くだけで、OS起動のたびに自動で上がってくる。launchdのplistも自動生成される（`~/Library/LaunchAgents/homebrew.mxcl.ollama.plist`）。夜中に走るPythonバッチが「Ollamaが落ちていて死んだ」という事態をほぼ気にしなくていい——これだけで移行する価値があった。


PythonからAPIを叩くコードは素直に書ける。


```python
import requests

def ask_ollama(prompt: str, model: str = "qwen2.5:3b") -> str:
    res = requests.post(
        "http://localhost:11434/api/generate",
        json={"model": model, "prompt": prompt, "stream": False},
        timeout=120,
    )
    res.raise_for_status()
    return res.json()["response"]

result = ask_ollama("今日のSNS投稿案を1つ出して")
print(result)

```

`timeout=120` の明示は省かないほうがいい。最初これを書かずに長文生成リクエストを送ったら、127秒待ってからデフォルトタイムアウトで落ちた。Requestsのデフォルトはタイムアウト無制限なので、バッチで回すと詰まり続ける——一回踏んで学んだ。


初回呼び出しはqwen2.5:3bをメモリに載せる時間込みで4.8秒。2回目以降は0.73秒。メモリ消費は生成中で2.5GB、アイドル時は自動アンロード（デフォルト5分）でゼロに戻る。複数タスクが並走するバッチでは、この自動アンロードが地味に助かっている。別プロセスがメモリを食い始めたタイミングで、Ollamaが自然に退いてくれる感じがある。


## LM Studioで失敗してわかったこと

LM Studioを使っていた3週間で、詰まったポイントが3つあった。


1つ目は冒頭の障害そのものだ。GUIを起動しないとAPIサーバーが立ち上がらない。「Local Server」タブのスタートボタンを押さないと `http://localhost:1234/v1` に何も生えない。cronやlaunchdからプロセスを起動しても、Xサーバーがない環境（VPS・Dockerコンテナ）ではそもそも動かない。自動化との相性が根本から合わない設計だと気づいたのは使い始めて2週目だった。


2つ目はメモリ管理。複数モデルを順番に叩くバッチを組んだとき、モデルを切り替えるたびにGUIの「Unload Model」を手動で押す必要があった。APIからアンロードを制御する手段がなく、私の環境では20回バッチを回したあとにメモリが徐々に膨らんでいくのが確認できた。


3つ目はVRAMの不安定性。M5移行前にM1 MacBook ProでLM Studioを使っていた時期、qwen2.5:3bの推論中にプロセスがクラッシュすることが3回続いた。ログに `Metal GPU allocation failed` が残っていた。同じモデルをOllamaで動かしたらクラッシュしなかったので、OllamaのメモリアロケーションがLM Studioより堅牢という印象がある。


LM Studioが活きるのは「HuggingFaceから直接GGUFを落として、すぐ対話したい」という単発の検証フェーズだ。GUIの操作感は確かにいいし、パラメータをスライダーで触れる直感性は試し打ちに向いている。ただし自動化を前提にした途端、制約が重なってくる。


## 実測：qwen2.5-3bのレスポンス速度（M5 MacBook Pro）

2026年4月、M5 MacBook Pro 32GB環境での計測。条件はどちらもqwen2.5:3b、プロンプトは100〜150トークン程度のSNS投稿生成タスク。


シナリオ
Ollama
LM Studio


初回生成（モデルロード込み）
4.8秒
3.2秒


2回目以降（キャッシュ済み）
0.73秒
0.81秒


メモリピーク（生成中）
2.5GB
2.6GB


メモリ（アイドル）
0（自動アンロード）
2.4GB（手動アンロードまで保持）


バッチ処理20回連続
15.2秒
18.7秒 + VRAMリセット操作


単発の速度はほぼ互角。初回ロードはLM Studioのほうがやや速かった。差が出るのはバッチだ。OllamaはGCを自動でやるので、20回連続しても15.2秒でフラットに推移する。LM Studioは途中でVRAMが逼迫してリセット操作が必要になり、18.7秒の計測値には手動介入のロスが含まれている。比較として公平ではない、という見方もできるが、その手動介入が必要になること自体が自動化との相性の問題だと思っている。


## Ollamaで自動化パイプラインを動かすセットアップ

Homebrewがあれば4ステップで終わる。


```bash
brew install ollama

```

インストール直後はまだ常駐していない。launchdに登録してOS再起動後も自動起動させる。


```bash
brew services start ollama
brew services list | grep ollama
# → ollama started /Users/username/Library/LaunchAgents/homebrew.mxcl.ollama.plist

```

`started` と表示されれば登録完了。モデルを引いてくる。


```bash
ollama pull qwen2.5:3b
# 2.0GB、初回は15分前後かかる

```

APIが生えているか確認する。


```bash
curl -X POST http://localhost:11434/api/generate \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen2.5:3b","prompt":"こんにちは","stream":false}'

```

JSONレスポンスが返ってくれば動いている。Python側は前述の `ask_ollama()` でそのまま叩ける。


自動化でモデルのパラメータを固定したいときはModelfileが便利だ。毎回呼び出し側で温度やコンテキスト長を指定するより、モデル定義にまとめておくほうが実行結果が安定する。


```dockerfile
FROM qwen2.5:3b
PARAMETER temperature 0.3
PARAMETER num_ctx 4096
PARAMETER top_p 0.9

```


```bash
ollama create my-qwen2.5 -f Modelfile
ollama run my-qwen2.5

```

バッチで「同じ設定を100回回す」前提なら、Modelfileで固めておくのが後から崩れない。呼び出しのたびにパラメータを渡す構成にすると、どこかで書き間違えたときに原因が見つけにくくなる——これも一度踏んだ。


## 選定の判断基準

私の場合、「Ollama常駐 + LM Studio検証用」の使い分けに落ち着いた。自動化を本業にするならOllamaから入るのが正解だと思っている。


Ollamaを選ぶ場面：


- launchdやcronと組んだ無人バッチが中心
- OS再起動後も自動起動させたい
- 複数モデルを回転させるメモリ効率を重視する
- CLIとAPIだけで完結させたい


LM Studioを選ぶ場面：


- HuggingFaceの新着GGUFをGUIでさっと試したい
- 対話UIでパラメータを触りながら挙動を確認したい
- 自動化は後付けで、まず手動検証フェーズにある


自動化の比率が上がるほどOllamaの優位は広がる。LM Studioに戻す理由がなくなるのは、GUIの操作が自動化の文脈で必要になった瞬間——それはたぶん来ない。


参考リンク：


- [Ollama – GitHub](https://github.com/ollama/ollama)
- [Ollama Model Library](https://ollama.com/library)
- [LM Studio 公式サイト](https://lmstudio.ai)
- [Ollama API リファレンス](https://github.com/ollama/ollama/blob/main/docs/api.md)


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

👉 [Ollama vs LM Studio 比較——ローカルLLM環境の選び方](https://ichinose-taito.com/?p=4878)

