---
title: "Discord Bot に音声読み上げを実装した話——AivisSpeech + FFmpeg で詰まった3つのポイント"
emoji: "✨"
type: "tech"
topics: ["claude", "python", "macos", "llm", "kindle"]
published: true
---

⏱この記事は約 10 分で読めます


📖 目次

- [📌 そもそもなぜ音声にしたのか](#そもそもなぜ音声にしたのか)
- [📌 構成の全体像](#構成の全体像)
- [📌 AivisSpeech で音声を生成する](#aivisspeech-で音声を生成する)
- [📌 VC への接続と音声送信——ここで詰まった](#vc-への接続と音声送信ここで詰まった)
- [📌 AivisSpeech が落ちているときの処理](#aivisspeech-が落ちているときの処理)
- [📌 launchd で常時起動させる](#launchd-で常時起動させる)
- [📌 実際に動かして気づいたこと](#実際に動かして気づいたこと)
- [📌 まとめ](#まとめ)
- [📌 関連記事](#関連記事)


最終更新: 2026-04-28


## そもそもなぜ音声にしたのか

ターミナルで別の作業をしていると、Discord のテキスト通知は完全に視界の外に落ちる。launchd で回しているパイプラインが夜中に「note 投稿失敗」を吐いていても、翌朝まで気づかない。先月だけでそれが3回あった。


声で来ればさすがに気づく。ローカルで動く TTS を探した結果、AivisSpeech が一番シンプルだった。無料、ローカル完結、REST API が2エンドポイントだけ——選んだ基準はそれだけ。dmg を落として起動するだけでポート 10101 に API が生えるので、セットアップは5分かからなかった。声質はデフォルトだと少しロボット感があるが、速度と抑揚をパラメータで調整すれば許容できる範囲に収まる。


そういう経緯で `bot.py` に音声読み上げを突っ込むことにした。


## 構成の全体像


```
[Discord Bot] → !voice コマンド or 自然言語
     ↓
AivisSpeech API (localhost:10101)
     ↓ WAV
FFmpeg で PCM に変換
     ↓
discord.py FFmpegPCMAudio → VC 送信

```

依存は3つ。`discord.py`（voice オプション付き）、`ffmpeg`、AivisSpeech のローカルサーバー。


```bash
pip install "discord.py[voice]"
brew install ffmpeg

```

AivisSpeech は起動しておくだけでいい。ただし、Bot 起動直後に叩くと 503 が返ることがある——AivisSpeech のロードが数秒かかるためで、後述のチェック関数でその辺は吸収している。


## AivisSpeech で音声を生成する

まず TTS だけ単独で動かして確認した。`audio_query` でテキスト解析、`synthesis` で WAV bytes を受け取る、2ステップの構成。


```python
import requests

def synthesize(text: str, speaker_id: int = 888753760) -> bytes:
    query_res = requests.post(
        "http://localhost:10101/audio_query",
        params={"text": text, "speaker": speaker_id}
    )
    query = query_res.json()

    synth_res = requests.post(
        "http://localhost:10101/synthesis",
        params={"speaker": speaker_id},
        json=query
    )
    return synth_res.content  # WAV bytes

```

最初に `speaker_id=0` を指定した。WAV は返ってくる。無音。30分、コードを見直し続けて、最終的に `GET /speakers` を叩いて気づいた——speaker_id は 0 が有効とは限らない。


```bash
curl http://localhost:10101/speakers | python3 -m json.tool | grep '"id"'
```

私の環境では `888753760` が最初に来た。環境によって違うので必ず確認すること。無音ファイルを生成しながら30分を溶かした経験から言っている。


## VC への接続と音声送信——ここで詰まった

### voice_client を掴む部分


```python
@bot.command()
async def voice(ctx, *, text: str):
    if ctx.author.voice is None:
        await ctx.send("VC に入ってから呼んでください")
        return

    channel = ctx.author.voice.channel
    vc = ctx.guild.voice_client

    if vc is None:
        vc = await channel.connect()
    elif vc.channel != channel:
        await vc.move_to(channel)

```

ここは素直に動いた。問題はこの後。


### BytesIO に包んでも駄目だった

AivisSpeech から返ってくる WAV bytes を直接 `FFmpegPCMAudio` に渡したら「ファイルパスかファイルオブジェクトが必要」と怒られた。なら `io.BytesIO` で包めばいいだろうと試したら、それも受け付けなかった。`FFmpegPCMAudio` は BytesIO を受け取れない——ドキュメントに書いてあるにはあるが、見落としやすい場所にある。


一時ファイルに書き出す方法が現状一番確実だった。


```python
import tempfile
import os

async def play_voice(vc, text: str):
    wav_bytes = synthesize(text)

    with tempfile.NamedTemporaryFile(suffix=".wav", delete=False) as f:
        f.write(wav_bytes)
        tmp_path = f.name

    try:
        source = discord.FFmpegPCMAudio(tmp_path)
        vc.play(source, after=lambda e: os.unlink(tmp_path) if e is None else None)
    except Exception as e:
        os.unlink(tmp_path)
        raise e

```

`after` コールバックで再生完了後に一時ファイルを消している。これを忘れると `/tmp` 以下に WAV が溜まっていく。launchd で常駐させていると気づかぬうちにディスクを圧迫するので、必ず入れる。


### libopus で2時間溶かした

次に出たエラーがこれ。


```
discord.errors.ClientException: opus shared library is not found.
```

「ffmpeg が見つからない」ならわかる。でもメッセージは ffmpeg ではなく opus の話だった。`brew install opus` はすでにしていた。それでも「not found」。


原因は `DYLD_LIBRARY_PATH` だった。`/opt/homebrew/lib` に libopus.dylib はあるのに、launchd で起動したプロセスにはそのパスが通っていない。ターミナルから直接 python を起動すると動く、launchd 経由だと opus not found——この差に気づくまで2時間かかった。エラーメッセージが opus なのに ffmpeg の再インストールを試み続けていたのが敗因。


plist の `EnvironmentVariables` に明示的に書くことで解決した。


```xml
EnvironmentVariables

    DYLD_LIBRARY_PATH
    /opt/homebrew/lib


```


## AivisSpeech が落ちているときの処理

本番で一度、AivisSpeech が落ちた状態で `!voice` を叩いて Bot ごとクラッシュした。起動チェックをサボっていた代償。


```python
def is_aivisspeech_running() -> bool:
    try:
        r = requests.get("http://localhost:10101/version", timeout=1)
        return r.status_code == 200
    except Exception:
        return False

```

timeout=1 にしているのは理由がある。AivisSpeech が完全に死んでいるとき、タイムアウトなしの requests は数十秒待ち続ける。1秒で切ることで Bot の応答がもたつかない。このチェックをコマンドの先頭に挟んで、落ちていたらテキスト通知に降格させて終了する。


```python
@bot.command()
async def voice(ctx, *, text: str):
    if not is_aivisspeech_running():
        await ctx.send(f"[音声不可] {text}")
        return
    # ... VC接続・再生処理

```


## launchd で常時起動させる

既存の `com.taito.discord-bot.plist` に2行追加するだけ。


```xml
KeepAlive

StandardErrorPath
/path/to/logs/discord_bot.log

```

AivisSpeech 自体は launchd で管理していない。Mac アプリとして常駐させている。Bot より先に AivisSpeech が起動する保証がないので、Bot 側で `is_aivisspeech_running()` を毎回チェックする設計にしている。起動順の問題を「チェック関数で逃げる」のは正直なところ応急処置に近い。もう少し使い続けたら launchd に移行するか検討する。


## 実際に動かして気づいたこと

デフォルトの読み上げ速度が速い。普通に聞いていると巻き舌のアナウンサーみたいな感じで、内容が頭に入ってこない。`audio_query` のレスポンスに `speedScale` と `intonationScale` があるので、合成前に書き換える。


```python
query["speedScale"] = 0.9
query["intonationScale"] = 1.1  # 抑揚は少し上げる

```

0.9 でちょうどよかった。0.8 まで落とすと間延びして逆に聞きづらくなった。


もう一つ。長い文章を渡すと合成に時間がかかる。100字を超えるテキストは `synthesis` の応答が3〜5秒かかることがあって、その間に VC 接続がタイムアウトするケースがあった。今は「通知用途なので100字以内に収める」という運用で対処している。文章を分割して連続再生する実装は後回し——そこまで凝った使い方はまだしていない。


使い始めると、「note 投稿完了」「エラー発生: syntax error line 42」みたいな通知が声で来るのは地味に便利だった。作業が途切れない。続けて使いながら直していく。✨


## まとめ

AivisSpeech + FFmpeg + discord.py の組み合わせは、ローカル TTS を Discord VC に流す今のところ最短ルートだと思っている。


詰まる場所は3つ。speaker_id は 0 ではなく `GET /speakers` で確認すること、WAV bytes は BytesIO ではなく一時ファイル経由で渡すこと、launchd 起動時に `DYLD_LIBRARY_PATH=/opt/homebrew/lib` を明示すること。この記事を先に読んでいれば、私が溶かした3時間は取り戻せる。


一ノ瀬泰斗
AI自動化エンジニア / Python個人開発者
Claude Code × Ollama × launchd で SNS・ブログ・KDPを全自動化。実測データと失敗談を軸に、月5万円収益化のリアルな記録を発信中。


💬 **自動化の相談・小規模受託も受付中**：「launchd で毎朝 AI が動く仕組みを作りたい」「KDP の自動出版を組みたい」など、[X (@taito_automate) の DM](https://x.com/taito_automate) からお気軽にどうぞ。


### 関連記事


- [Mac で使える AI ツール 比較 2026｜個人開発者が実際に使い倒した結果](https://ichinose-taito.com/?p=4801)
- [Python で WordPress 自動投稿——REST API 実装と実運用のリアル](https://ichinose-taito.com/?p=4769)
- [Playwright Mac 入門 2026——ブラウザ自動化を最速で動かすセットアップ](https://ichinose-taito.com/?p=4768)


## 関連記事


- [Playwright Mac 入門 2026——ブラウザ自動化を最速で動かすセットアップ](https://ichinose-taito.com/playwright-mac-%e5%85%a5%e9%96%80-2026-%e3%83%96%e3%83%a9%e3%82%a6%e3%82%b6%e8%87%aa%e5%8b%95%e5%8c%96%e3%82%92%e6%9c%80%e9%80%9f%e3%81%a7%e5%8b%95%e3%81%8b%e3%81%99%e3%82%bb%e3%83%83/)
- [個人ブログを毎日自動投稿する仕組みを全部Pythonで組んだ話](https://ichinose-taito.com/%e5%80%8b%e4%ba%ba%e3%83%96%e3%83%ad%e3%82%b0%e3%82%92%e6%af%8e%e6%97%a5%e8%87%aa%e5%8b%95%e6%8a%95%e7%a8%bf%e3%81%99%e3%82%8b%e4%bb%95%e7%b5%84%e3%81%bf%e3%82%92%e5%85%a8%e9%83%a8python%e3%81%a7/)
- [WordPressにPythonで自動投稿した話——REST APIの詰まりポイント全部まとめ](https://ichinose-taito.com/wordpress%e3%81%abpython%e3%81%a7%e8%87%aa%e5%8b%95%e6%8a%95%e7%a8%bf%e3%81%97%e3%81%9f%e8%a9%b1-rest-api%e3%81%ae%e8%a9%b0%e3%81%be%e3%82%8a%e3%83%9d%e3%82%a4%e3%83%b3%e3%83%88/)

---

この記事はもともと私のブログで書いたものです。より詳しい実装例・失敗談はこちらで読めます。

👉 [Discord Bot に音声読み上げを実装した話——AivisSpeech + FFmpeg で詰まった3つのポイント](https://ichinose-taito.com/?p=4628)


