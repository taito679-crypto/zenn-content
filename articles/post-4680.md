---
title: "AivisSpeechをMacにインストールしてPythonから呼び出すまでに詰まった話"
emoji: "✨"
type: "tech"
topics: ["claude", "python", "macos", "llm", "kindle"]
published: true
---

⏱この記事は約 12 分で読めます


📖 目次

- [📌 動くまでに3時間かかった話](#動くまでに3時間かかった話)
- [📌 AivisSpeechとは何か](#aivisspeechとは何か)
- [📌 Macへのインストール手順](#macへのインストール手順)
- [📌 Pythonから呼び出す](#pythonから呼び出す)
- [📌 Discord BotからVCに流す](#discord-botからvcに流す)
- [📌 launchdで常駐させる](#launchdで常駐させる)
- [📌 結局どこで時間が溶けたか](#結局どこで時間が溶けたか)
- [📌 launchdはまだ試行錯誤中](#launchdはまだ試行錯誤中)
- [📌 関連記事](#関連記事)


最終更新: 2026-04-28


## 動くまでに3時間かかった話

AivisSpeechをMacに入れたのは4月の頭、Discord BotにVC読み上げを付けたかったからだ。テキストをAPIに投げて音声データを返してもらうだけ——構造はシンプルなのに、動くまでに3時間かかった。


理由が3つ重なった。Gatekeeperに弾かれる、エンジンが起動しない、Pythonのhttpxのタイムアウトが短すぎる。どれも単独なら10分で解決できる話だが、3つ同時に踏むと何が原因かわからなくなる。エラーメッセージも「Connection refused」しか出ないので、どこで詰まっているのかすら最初は判断できなかった。


この記事はその詰まりを発生した順番で書いた。完成形から言うと今はlaunchdで常駐させてPythonのHTTP越しに叩いている。1〜2秒でwavが返ってきて、そのままDiscordのVCに流れる——安定はしている。でも「安定するまで」が長かった。


## AivisSpeechとは何か

VOICEVOXと同じアーキテクチャのローカルTTSエンジンだ。APIサーバーがlocalhost:10101で起動して、HTTPでテキストを投げると音声データが返ってくる。クラウドに出ない。完全ローカル。


OpenAI TTSは1000文字あたり0.015ドルかかる。Googleも月1万文字を超えると課金が始まる。私の用途——Botが1日数十回喋る、テキスト量は数百〜数千文字——だとローカルのほうが圧倒的にコストが低い。レイテンシも実測で1〜2秒以内に収まっていて、会話のテンポを壊さない。


商用利用の条件はキャラクターごとに違うので確認は必要だが、個人の自動化用途であれば基本的に問題ない。


## Macへのインストール手順

### ダウンロードと起動、そしてGatekeeperの壁

公式サイト（aivisspeech.jp）からMac版のdmgを落とす。Apple Silicon対応ビルドが別にある場合はそちらを選ぶ。インストール自体は普通のMacアプリと変わらない。


問題は起動した瞬間に来た。「開発元を確認できません」というダイアログ。Gatekeeperだ。ここで10分溶かした。ダイアログの「キャンセル」を押しては開こうとして、Finderの右クリックから「開く」を試して、それでも弾かれた。結局ターミナルから叩くのが確実だった。


```bash
# Gatekeeperの隔離フラグを外す
xattr -dr com.apple.quarantine /Applications/AivisSpeech.app

```

これを実行してから普通にダブルクリックすると起動する。GUIで起動すると右上に「エンジン起動中…」と出る。そこから30秒〜1分待つ。焦ってcurlを叩きに行くと当然 Connection refused が返ってくるので、少し待つ。確認は：


```bash
curl http://localhost:10101/version

```

JSONが返ってきたら成功だ。


### それでもエンジンが起動しない場合——マイク設定という盲点

Gatekeeperを突破しても、私のMacBook Pro M5ではエンジンが上がらないことがあった。症状は「アプリは動いているのに localhost:10101 に接続できない」——curlを打っても Connection refused しか返ってこない。


原因を探って20分かかった。アプリのログを見ても有益なエラーは出ていない。ふと「プライバシー設定かもしれない」と思ってシステム設定を開き、「プライバシーとセキュリティ → マイク」を見たら、AivisSpeechの項目があった。オフになっていた。許可したら即座にエンジンが起動した。


TTS（音声出力）なのになぜマイクのアクセス許可が要るのか、私もよくわかっていない。でもこれが原因だったのは確かで、同じ症状が出たら真っ先に確認する場所だ。要求ダイアログが画面に出ないパターンもあるので、手動で確認しに行くほうが早い。


## Pythonから呼び出す

### 2ステップのAPI構造

AivisSpeechのAPIはVOICEVOXと構造が同じで、音声生成は2回のHTTPリクエストで完結する。`/audio_query`にテキストとspeaker_idをPOSTして合成クエリを取得し、そのクエリを`/synthesis`に投げてwavデータを受け取る——それだけだ。


```python
import httpx
import json

AIVISSPEECH_URL = "http://localhost:10101"

def synthesize(text: str, speaker_id: int = 888753760) -> bytes:
    query_resp = httpx.post(
        f"{AIVISSPEECH_URL}/audio_query",
        params={"text": text, "speaker": speaker_id},
        timeout=30,
    )
    query_resp.raise_for_status()
    query = query_resp.json()

    audio_resp = httpx.post(
        f"{AIVISSPEECH_URL}/synthesis",
        params={"speaker": speaker_id},
        content=json.dumps(query),
        headers={"Content-Type": "application/json"},
        timeout=60,
    )
    audio_resp.raise_for_status()
    return audio_resp.content

if __name__ == "__main__":
    wav = synthesize("テスト音声です")
    with open("/tmp/test.wav", "wb") as f:
        f.write(wav)
    print("生成完了: /tmp/test.wav")

```

最初 timeout を 10 秒にしていたら synthesis でタイムアウトが頻発した。文字数が多いと合成に数秒かかることがあるので、synthesis 側は余裕を持って 60 秒にしている。


### speaker_idの確認方法

speaker_id はAivisSpeechのGUIに表示されている番号で、キャラクターによって値が違う。コマンドで一覧を出すなら：


```bash
curl http://localhost:10101/speakers | python3 -m json.tool | grep -A2 '"name"'

```

これで全キャラクターとIDが並ぶ。GUIを開くのが面倒なときはこちらのほうが早い。


## Discord BotからVCに流す

単体でwavが生成できたら、次はDiscord Botとつなぐ。discord.pyのFFmpegAudioSourceを使ってVCに流す構成だ。


```python
import discord
import asyncio
import httpx
import json
import tempfile
import os

AIVISSPEECH_URL = "http://localhost:10101"
SPEAKER_ID = 888753760

async def speak_in_vc(vc: discord.VoiceClient, text: str):
    async with httpx.AsyncClient() as client:
        query_resp = await client.post(
            f"{AIVISSPEECH_URL}/audio_query",
            params={"text": text, "speaker": SPEAKER_ID},
            timeout=30,
        )
        query = query_resp.json()

        audio_resp = await client.post(
            f"{AIVISSPEECH_URL}/synthesis",
            params={"speaker": SPEAKER_ID},
            content=json.dumps(query),
            headers={"Content-Type": "application/json"},
            timeout=60,
        )

    with tempfile.NamedTemporaryFile(suffix=".wav", delete=False) as f:
        f.write(audio_resp.content)
        tmp_path = f.name

    try:
        source = discord.FFmpegPCMAudio(tmp_path)
        vc.play(source)
        while vc.is_playing():
            await asyncio.sleep(0.1)
    finally:
        os.unlink(tmp_path)

```

FFmpegが入っていないとここで詰まる。`brew install ffmpeg` で入れておく。私はこれを忘れていて「FileNotFoundError: [Errno 2] No such file or directory: ‘ffmpeg’」が出た——見ればわかるエラーだが、音声が出ない原因がAivisSpeech側だと思い込んでいたので15分無駄にした。


## launchdで常駐させる

GUIアプリをlaunchdで自動起動させるのは少し癖がある。`/Applications/AivisSpeech.app`をそのまま指定しても動かないことがある。実体のバイナリをフルパスで指定する。


```xml


    Label
    com.taito.aivisspeech
    ProgramArguments
    
        /Applications/AivisSpeech.app/Contents/MacOS/AivisSpeech
    
    RunAtLoad
    
    KeepAlive
    
    StandardOutPath
    /tmp/aivisspeech.log
    StandardErrorPath
    /tmp/aivisspeech_err.log


```

これを `~/Library/LaunchAgents/com.taito.aivisspeech.plist` に置いて：


```bash
launchctl load ~/Library/LaunchAgents/com.taito.aivisspeech.plist

```

`KeepAlive: true` にしたら落ちた後に再起動ループに入った。`/tmp/aivisspeech_err.log` が5分で100MB超えていた。今は `KeepAlive: false` にして、エンジンが落ちていたらPythonスクリプト側から起動チェックをかける構成にしている。GUIアプリをlaunchdに乗せるのはそれ自体が一仕事だと思っておいたほうがいい。


## 結局どこで時間が溶けたか

一番時間がかかったのはエンジンが起動しない問題で、原因はマイクのプライバシー設定だった。コードを書くのは10分で済んだのに、環境の問題で2時間半以上溶かした——これが実態だ。


Gatekeeperのほうはエラーメッセージが明確なのでまだいい。プライバシー設定はダイアログが出ないパターンがあるので発見が遅れた。「curlでバージョンが返ってきた瞬間」にやっと動いていることがわかる——あそこまでが長い。


API自体はシンプルで、VOICEVOXの記事がそのまま参考になる。speaker_idさえ合っていれば、コードは素直に動く✨


## launchdはまだ試行錯誤中

AivisSpeechのPython連携は「dmgインストール → Gatekeeper回避 → プライバシー設定確認 → curlでバージョン確認 → audio_query+synthesisの2リクエスト」で動く。詰まりやすいのはコードより環境側の話がほとんどだ。


launchdで常駐させるのは今も試行錯誤中で、GUIアプリ特有の挙動がある。無理にlaunchdに乗せるより、Botの起動スクリプト内でサブプロセスとして立ち上げるほうがシンプルかもしれない。私は今のところGUI手動起動＋Bot自動起動で運用している——launchdはもう少し詰めてから改めて書く。


一ノ瀬泰斗
AI自動化エンジニア / Python個人開発者
Claude Code × Ollama × launchd で SNS・ブログ・KDPを全自動化。実測データと失敗談を軸に、月5万円収益化のリアルな記録を発信中。


💬 **自動化の相談・小規模受託も受付中**：「launchd で毎朝 AI が動く仕組みを作りたい」「KDP の自動出版を組みたい」など、[X (@taito_automate) の DM](https://x.com/taito_automate) からお気軽にどうぞ。


### 関連記事


- [Mac で使える AI ツール 比較 2026｜個人開発者が実際に使い倒した結果](https://ichinose-taito.com/?p=4801)
- [Python で WordPress 自動投稿——REST API 実装と実運用のリアル](https://ichinose-taito.com/?p=4769)
- [Playwright Mac 入門 2026——ブラウザ自動化を最速で動かすセットアップ](https://ichinose-taito.com/?p=4768)


## 関連記事


- [Claude Code 使い方 完全ガイド 2026年版——インストールから実戦運用まで](https://ichinose-taito.com/claude-code-%e4%bd%bf%e3%81%84%e6%96%b9-%e5%ae%8c%e5%85%a8%e3%82%ac%e3%82%a4%e3%83%89-2026%e5%b9%b4%e7%89%88-%e3%82%a4%e3%83%b3%e3%82%b9%e3%83%88%e3%83%bc%e3%83%ab%e3%81%8b%e3%82%89/)
- [WordPressにPythonで自動投稿した話——REST APIの詰まりポイント全部まとめ](https://ichinose-taito.com/wordpress%e3%81%abpython%e3%81%a7%e8%87%aa%e5%8b%95%e6%8a%95%e7%a8%bf%e3%81%97%e3%81%9f%e8%a9%b1-rest-api%e3%81%ae%e8%a9%b0%e3%81%be%e3%82%8a%e3%83%9d%e3%82%a4%e3%83%b3%e3%83%88/)
- [Discord Bot に音声読み上げを実装した話——AivisSpeech + FFmpeg で詰まった3つのポイント](https://ichinose-taito.com/discord-bot-%e3%81%ab%e9%9f%b3%e5%a3%b0%e8%aa%ad%e3%81%bf%e4%b8%8a%e3%81%92%e3%82%92%e5%ae%9f%e8%a3%85%e3%81%97%e3%81%9f%e8%a9%b1-aivisspeech-ffmpeg-%e3%81%a7%e8%a9%b0%e3%81%be/)

---

この記事はもともと私のブログで書いたものです。より詳しい実装例・失敗談はこちらで読めます。

👉 [AivisSpeechをMacにインストールしてPythonから呼び出すまでに詰まった話](https://ichinose-taito.com/?p=4680)

