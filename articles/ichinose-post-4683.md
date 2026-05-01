---
title: "Python subprocessで外部コマンドを安全に呼ぶパターン集——自動化パイプラインで詰まった話"
emoji: "✨"
type: "tech"
topics: ["claude", "python", "macos", "llm", "kindle"]
published: true
---

⏱この記事は約 12 分で読めます


📖 目次

- [📌 導入](#導入)
- [📌 shell=True は原則禁止——なぜ危ないか](#shelltrue-は原則禁止なぜ危ないか)
- [📌 エラーを握りつぶさない——check=True と例外処理](#エラーを握りつぶさないchecktrue-と例外処理)
- [📌 タイムアウトを必ず設定する](#タイムアウトを必ず設定する)
- [📌 出力を逐次読みたいとき——Popen を使う](#出力を逐次読みたいときpopen-を使う)
- [📌 環境変数を安全に渡す](#環境変数を安全に渡す)
- [📌 作業ディレクトリを明示する](#作業ディレクトリを明示する)
- [📌 実用ラッパー関数——パイプライン全体で使い回す](#実用ラッパー関数パイプライン全体で使い回す)
- [📌 まとめ](#まとめ)
- [📌 関連記事](#関連記事)


最終更新: 2026-04-28


⏱この記事は約 4 分で読めます

📖 目次

- [📌 導入](#導入)
- [📌 shell=True は原則禁止——なぜ危ないか](#shelltrue-は原則禁止なぜ危ないか)
- [📌 エラーを握りつぶさない——check=True と例外処理](#エラーを握りつぶさないchecktrue-と例外処理)
- [📌 タイムアウトを必ず設定する](#タイムアウトを必ず設定する)
- [📌 出力を逐次読みたいとき——Popen を使う](#出力を逐次読みたいときpopen-を使う)
- [環境変数・パス・作業ディレクトリ管理](https://ichinose-taito.com/claude-code-hook%e3%81%a7python%e3%82%b9%e3%82%af%e3%83%aa%e3%83%97%e3%83%88%e3%81%8c%e9%bb%99%e3%81%a3%e3%81%a6%e8%90%bd%e3%81%a1%e3%82%8b%e5%95%8f%e9%a1%8c-%e3%83%91%e3%82%b9/)” style=”color:#6366f1;text-decoration:none;”>📌 環境変数を安全に渡す
- [📌 作業ディレクトリを明示する](#作業ディレクトリを明示する)
- [📌 実用ラッパー関数——パイプライン全体で使い回す 自動化パイプラインの運用と監視](#実用ラッパー関数パイプライン全体で使い回す)
- [📌 まとめ](#まとめ)


## 導入

最初にやらかしたのはffmpegを呼ぶ箇所だった。


ブログ用の動画変換スクリプトで、ファイル名を `shell=True` の文字列に直接埋め込んでいた。問題が出たのは「AI 自動化 2026」みたいにスペース入りのファイル名を処理しようとしたとき。コマンドが途中で分割されてサイレントに失敗する。出力ファイルはできていない。でもスクリプトは exit code 0 で終わる。 [自動化スクリプトのエラーハンドリング](https://ichinose-taito.com/pip-install-%e3%82%a8%e3%83%a9%e3%83%bc%e8%a7%a3%e6%b1%ba%ef%bc%9a%e8%87%aa%e5%8b%95%e5%8c%96%e3%82%b9%e3%82%af%e3%83%aa%e3%83%97%e3%83%88%e3%81%a7%e8%a9%b0%e3%81%be%e3%81%a3%e3%81%9f%e7%a7%81/)30分くらい原因を探して、やっと文字列結合が原因だと気づいた——そのとき「これ、ちゃんとパターン化しないとまたやる」と思った。


このパイプラインは Claude Code + Ollama + launchd で動いていて、スクリプトが10本を超えたあたりから同じような失敗が複数箇所で出てきた。`subprocess` の書き方が場所によってバラバラだった。エラーを握りつぶす書き方、タイムアウトなしの書き方、環境変数を引数に直接埋め込む書き方——全部踏んだ。この記事はその記録。


## shell=True は原則禁止——なぜ危ないか

まずここから話す。よく見るやつ。


```python
filename = "AI 自動化 2026.mp4"
subprocess.run(f"ffmpeg -i {filename} output.mp3", shell=True)

```

`shell=True` で文字列を渡すということは、シェルにその文字列を解釈させるということだ。スペースが入っていれば引数が分割される。もっと怖いのは、`filename` の中に `"; rm -rf ~"` のような文字列が混入したとき——シェルはそのまま実行する。自動化パイプラインはファイル名をAPIレスポンスや外部入力から生成することが多いから、これは架空の話ではない。


正しい書き方は引数をリストで渡す。


```python
filename = "AI 自動化 2026.mp4"
subprocess.run(["ffmpeg", "-i", filename, "output.mp3"])

```

シェルを経由しないので、スペースも特殊文字もそのまま1つの引数として扱われる。`shell=True` が必要になるのは、パイプ（`|`）やリダイレクト（`>`）をシェル側で処理させたいときだけ。それ以外はリスト渡し一択。[公式ドキュメントのセキュリティ説明](https://docs.python.org/3/library/subprocess.html#security-considerations)にも同じことが書いてある。


## エラーを握りつぶさない——check=True と例外処理

launchd で定期実行するスクリプトは誰も見ていない。そこに「エラーが出ても次のステップに進む」コードが紛れ込んでいると、失敗が蓄積したまま気づかない。これが一番たちが悪い。


```python
import subprocess

try:
    result = subprocess.run(
        ["ffmpeg", "-i", "input.mp4", "output.mp3"],
        check=True,
        capture_output=True,
        text=True,
    )
    print(result.stdout)
except subprocess.CalledProcessError as e:
    print(f"exit code: {e.returncode}")
    print(e.stderr)
    raise

```

`check=True` を付けると、コマンドが非ゼロで終了した瞬間に [`CalledProcessError`](https://docs.python.org/3/library/subprocess.html#subprocess.CalledProcessError) を投げる。`capture_output=True` で stdout/stderr を両方キャプチャ、`text=True` でバイト列でなく文字列として受け取れる。この3つはセットで覚えておくと後が楽だ。


私はここで末尾の `raise` を省略したことがある。`except` ブロックでログだけ出して「処理した気」になっていた。結果、次のステップが空ファイルを受け取って静かに壊れていた。例外はキャッチしたらちゃんと再送出するか、上位で明示的にハンドルする。


## タイムアウトを必ず設定する

設定し忘れるとゾンビプロセスになる——というのを身をもって経験した。


Ollama で qwen2.5:3b を呼ぶスクリプトがあって、Ollama サービスの起動が遅れたせいで `subprocess.run()` がずっとブロックし続けた。launchd のジョブは次の起動時刻まで終わらない。1時間以上後にジョブキューが詰まっているのに気づいて、ログを掘ったらここだった。


```python
import subprocess

try:
    result = subprocess.run(
        ["ollama", "run", "qwen2.5:3b", "タイトルを1行で生成して"],
        timeout=120,
        check=True,
        capture_output=True,
        text=True,
    )
    print(result.stdout)
except subprocess.TimeoutExpired as e:
    e.kill()
    print("タイムアウト: プロセスを強制終了した")
    raise

```

[`TimeoutExpired`](https://docs.python.org/3/library/subprocess.html#subprocess.TimeoutExpired) が飛んだとき、`e.kill()` を呼ばないと子プロセスがゾンビになる。これも最初は書いていなかった。


私のパイプラインでは、Ollama 呼び出しや ffmpeg 変換には `timeout=120`〜`300`、外部APIコール系には `timeout=30` を基準にしている。処理の性質によって変えるのが実態に合っている。


## 出力を逐次読みたいとき——Popen を使う

`subprocess.run()` は完了まで待つ。ffmpeg で10分の動画を変換するとき、その10分間は何も出力されない。スクリプトが死んでいるのか動いているのか、launchd のログを見ても分からなくなる。


[`Popen`](https://docs.python.org/3/library/subprocess.html#subprocess.Popen) を使うと出力を行単位で読める。


```python
import subprocess

with subprocess.Popen(
    ["ffmpeg", "-i", "input.mp4", "-vn", "-q:a", "2", "output.mp3"],
    stdout=subprocess.PIPE,
    stderr=subprocess.STDOUT,
    text=True,
) as proc:
    for line in proc.stdout:
        print(line, end="")
    proc.wait()

```

launchd のジョブログに進捗が流れるので、どこで止まっているかが分かる。`with` ブロックで囲むのは必須——`Popen` はリソース管理を自分でやる必要があるため、コンテキストマネージャに任せた方が確実に解放される。


## 環境変数を安全に渡す

APIキーをコマンドライン引数に含めると `ps aux` で丸見えになる。[`env` パラメータ](https://docs.python.org/3/library/subprocess.html#subprocess.run)経由が正しい書き方だ。


```python
import os
import subprocess

env = os.environ.copy()
env["NOTION_API_KEY"] = "secret-token-xxxx"

result = subprocess.run(
    ["notion-cli", "export", "--format", "markdown"],
    env=env,
    check=True,
    capture_output=True,
    text=True,
)

```

`os.environ.copy()` で現在の環境変数を引き継いで、必要なキーを追加する。子プロセスは親の環境変数も受け取れる。


逆に特定の環境変数を「除外」したいケースもある。私のパイプラインで Claude Code を子プロセスとして呼ぶとき、`ANTHROPIC_API_KEY` が環境変数に残っていると、サブスクリプション経由ではなく API クレジットを消費してしまう問題があった。気づくまで数日かかった。毎朝クレジットが微妙に減っていて、やっとログを追ったらここだった。


```python
import os
import subprocess

env = {k: v for k, v in os.environ.items() if k != "ANTHROPIC_API_KEY"}

result = subprocess.run(
    ["claude", "--print", "タスクを実行して"],
    env=env,
    check=True,
    capture_output=True,
    text=True,
)

```

これで Claude Code がサブスク経由で動く。


## 作業ディレクトリを明示する


```python
import subprocess
from pathlib import Path

BASE_DIR = Path("/Users/ichinosetaito/Documents/AI_Automation_Base")

subprocess.run(
    ["git", "pull"],
    cwd=BASE_DIR,
    check=True,
    capture_output=True,
    text=True,
)

```

`cwd` を指定しないと、スクリプトを起動したディレクトリで実行される。launchd から呼んだとき、作業ディレクトリが `/` になっていることがある。相対パスを使っているスクリプトは全部壊れる。


launchd の plist に `WorkingDirectory` キーを書く方法もあるが、Python コード内で指定した方がスクリプト単体でも手動実行でも同じ挙動になる。明示する、それだけ。


## 実用ラッパー関数——パイプライン全体で使い回す

毎回 `check=True, capture_output=True, text=True, cwd=BASE_DIR, timeout=60` を書くのが面倒なので、ラッパーを1つ作って使い回している。


```python
import os
import subprocess
from pathlib import Path

BASE_DIR = Path("/Users/ichinosetaito/Documents/AI_Automation_Base")


def run_cmd(
    args: list[str],
    cwd: Path = BASE_DIR,
    timeout: int = 60,
    extra_env: dict | None = None,
    strip_keys: list[str] | None = None,
) -> subprocess.CompletedProcess:
    env = os.environ.copy()
    if extra_env:
        env.update(extra_env)
    if strip_keys:
        for k in strip_keys:
            env.pop(k, None)
    return subprocess.run(
        args,
        cwd=cwd,
        timeout=timeout,
        check=True,
        capture_output=True,
        text=True,
        env=env,
    )

```

呼び出し側はシンプルになる。


```python
# Ollama でテキスト生成
result = run_cmd(["ollama", "run", "qwen2.5:3b", prompt], timeout=120)
print(result.stdout)

# Claude Code を API キーなしで呼ぶ
result = run_cmd(
    ["claude", "--print", task],
    strip_keys=["ANTHROPIC_API_KEY"],
    timeout=180,
)

```

新しいスクリプトを書くたびにこの関数を `import` するだけで、パイプライン全体が同じ安全設定で動く。


## まとめ

自動化スクリプトは「誰も見ていない状態で動く」前提で書く必要がある。`shell=True` は理由を言語化できるときだけ使う。`check=True` を省略すると失敗が検知されない。`timeout` なしはゾンビプロセスの原因になる。`capture_output=True` と `text=True` はセットで使う。APIキー等は `env` パラメータ経由で渡して、`ps aux` に見えない場所に置く。`cwd` は明示する。この6点が崩れると、launchd で動く夜間バッチが失敗したまま翌朝まで気づかない——というのを全部踏んだ。✨


今回まとめたパターンは最初から知っていたわけではなく、スクリプトが壊れるたびに1つずつ追加してきたもの。失敗ログを眺めながら少しずつ堅牢にしていくのが、「完璧より継続」のやり方だと思っている。


👨‍💻

一ノ瀬泰斗
AI自動化エンジニア / Python個人開発者
Claude Code × Ollama × launchd で SNS・ブログ・KDPを全自動化。実測データと失敗談を軸に、月5万円収益化のリアルな記録を発信中。


### 関連記事


- [Mac で使える AI ツール 比較 2026｜個人開発者が実際に使い倒した結果](https://ichinose-taito.com/?p=4801)
- [Python で WordPress 自動投稿——REST API 実装と実運用のリアル](https://ichinose-taito.com/?p=4769)
- [Playwright Mac 入門 2026——ブラウザ自動化を最速で動かすセットアップ](https://ichinose-taito.com/?p=4768)


## 関連記事


- [Claude Code と Playwright どっち使う？ブラウザ自動化の使い分けを実装経験から整理した](https://ichinose-taito.com/claude-code-%e3%81%a8-playwright-%e3%81%a9%e3%81%a3%e3%81%a1%e4%bd%bf%e3%81%86%ef%bc%9f%e3%83%96%e3%83%a9%e3%82%a6%e3%82%b6%e8%87%aa%e5%8b%95%e5%8c%96%e3%81%ae%e4%bd%bf%e3%81%84%e5%88%86%e3%81%91/)
- [launchd exit 78 エラーの原因と修正方法——設定ミスを3分で特定する](https://ichinose-taito.com/launchd-exit-78-%e3%82%a8%e3%83%a9%e3%83%bc%e3%81%ae%e5%8e%9f%e5%9b%a0%e3%81%a8%e4%bf%ae%e6%ad%a3%e6%96%b9%e6%b3%95-%e8%a8%ad%e5%ae%9a%e3%83%9f%e3%82%b9%e3%82%923%e5%88%86%e3%81%a7/)
- [MacでOllamaを動かすまでにハマった話—qwen3.5:9bが動いたときの感動を記録しておく](https://ichinose-taito.com/mac%e3%81%a7ollama%e3%82%92%e5%8b%95%e3%81%8b%e3%81%99%e3%81%be%e3%81%a7%e3%81%ab%e3%83%8f%e3%83%9e%e3%81%a3%e3%81%9f%e8%a9%b1-qwen3-59b%e3%81%8c%e5%8b%95%e3%81%84%e3%81%9f%e3%81%a8%e3%81%8d/)


### この記事を書いた人

**一ノ瀬 泰斗** — AI自動化エンジニア・KDP出版者


    Claude Code + Ollama + launchd を組み合わせ、ブログ・KDP電子書籍・SNS発信を自動化中。

    自分で作って自分で失敗して自分で直す、作業途中の生々しさが武器。

    本ブログでは実体験ベースの実測データ・失敗談・詰まりポイントを中心に発信しています。
  


    お問い合わせは [コンタクトページ](/contact/) から /

    [プライバシーポリシー](/privacy-policy/)
  


📘 この記事のテーマをさらに深掘りした本
### [Claude Codeで副業自動化——月5万円を目指した全記録](https://www.amazon.co.jp/dp/B0GX2ZV7DP?tag=taito2525-22)

非エンジニアが1年で月10万円に到達した全パイプラインと失敗談


[Kindleで読む →](https://www.amazon.co.jp/dp/B0GX2ZV7DP?tag=taito2525-22)


一ノ瀬泰斗
AI自動化エンジニア / Python個人開発者
Claude Code × Ollama × launchd で SNS・ブログ・KDPを全自動化。実測データと失敗談を軸に、月5万円収益化のリアルな記録を発信中。


💬 **自動化の相談・小規模受託も受付中**：「launchd で毎朝 AI が動く仕組みを作りたい」「KDP の自動出版を組みたい」など、[X (@taito_automate) の DM](https://x.com/taito_automate) からお気軽にどうぞ。

---

この記事はもともと私のブログで書いたものです。より詳しい実装例・失敗談はこちらで読めます。

👉 [Python subprocessで外部コマンドを安全に呼ぶパターン集——自動化パイプラインで詰まった話](https://ichinose-taito.com/?p=4683)


