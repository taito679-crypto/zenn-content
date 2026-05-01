---
title: "Claude Code hookでPythonスクリプトが黙って落ちる問題——パス・環境変数・作業ディレクトリの三重奏を全部解決した"
emoji: "✨"
type: "tech"
topics: ["claude", "python", "macos", "llm", "kindle"]
published: true
---

⏱この記事は約 10 分で読めます


📖 目次

- [📌 何が起きたか](#何が起きたか)
- [📌 環境](#環境)
- [📌 詰まったポイント](#詰まったポイント)
- [📌 解決までの手順](#解決までの手順)
- [📌 コード／設定の抜粋](#コード設定の抜粋)
- [📌 試してわかったこと](#試してわかったこと)
- [📌 まとめ](#まとめ)
- [📌 関連記事](#関連記事)


最終更新: 2026-04-28


## 何が起きたか

Claude Code の hook 機能を使って、セッション開始時に [Claude Code 完全攻略ガイド](https://ichinose-taito.com/claude-code-%e5%ae%8c%e5%85%a8%e6%94%bb%e7%95%a5%e3%82%ac%e3%82%a4%e3%83%89-%e3%82%a4%e3%83%b3%e3%82%b9%e3%83%88%e3%83%bc%e3%83%ab%e3%81%8b%e3%82%89%e5%ae%9f%e6%88%a6%e9%81%8b/) Python スクリプトを自動起動しようとしたときの話。`settings.json` に `PreToolUse` フックを書いて `python3 myscript.py` を呼んだら、コマンド自体は実行されているのにスクリプトが黙って落ちる [設定ミスでスクリプトが落ちる場合の対処方法](https://ichinose-taito.com/launchd-exit-78-%e3%82%a8%e3%83%a9%e3%83%bc%e3%81%ae%e5%8e%9f%e5%9b%a0%e3%81%a8%e4%bf%ae%e6%ad%a3%e6%96%b9%e6%b3%95-%e8%a8%ad%e5%ae%9a%e3%83%9f%e3%82%b9%e3%82%923%e5%88%86%e3%81%a7/)。ログにも何も残らない。エラーも出ない。「あれ、動いてる？」状態が30分続いた。原因はパス・環境変数・作業ディレクトリ [Pythonパス設定エラーの解決方法](https://ichinose-taito.com/pip-install-%e3%82%a8%e3%83%a9%e3%83%bc%e8%a7%a3%e6%b1%ba%ef%bc%9a%e8%87%aa%e5%8b%95%e5%8c%96%e3%82%b9%e3%82%af%e3%83%aa%e3%83%97%e3%83%88%e3%81%a7%e8%a9%b0%e3%81%be%e3%81%a3%e3%81%9f%e7%a7%81/)の三重奏だった [Python環境のデバッグチェックリスト](https://ichinose-taito.com/python-%e7%92%b0%e5%a2%83%e3%82%ba%e3%83%ac-%e3%83%87%e3%83%90%e3%83%83%e3%82%b0%e3%83%81%e3%82%a7%e3%83%83%e3%82%af%e3%83%aa%e3%82%b9%e3%83%88%e3%81%a8%e5%86%8d%e7%99%ba%e9%98%b2%e6%ad%a2%e3%82%b3/)。 環境変数とパスの詳細なデバッグチェックリスト


## 環境


- macOS Sequoia 15.x / MacBook Pro M5 32GB
- Claude Code（最新ビルド）
- Python 3.12（[Homebrew](https://brew.sh/) 管理）
- 仮想環境: [Python venv](https://docs.python.org/3/library/venv.html)（`~/Documents/AI_Automation_Base/.venv`）
- 対象スクリプト: `01_Scripts/monitoring/session_start_hook.py`（セッション開始時に Slack 通知 + ステータスチェックを走らせるやつ）
- Ollama: `localhost:11434` で稼働中


## 詰まったポイント

フックの書き方は公式ドキュメント通りに書いた。`settings.json` に以下を追加：


```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "python3 ~/Documents/AI_Automation_Base/01_Scripts/monitoring/session_start_hook.py"
          }
        ]
      }
    ]
  }
}

```

実行ログを見ると `exit code: 0` で正常終了している。なのにスクリプトの出力が一切ない。`print()` すら出ない。


最初に疑ったのは「スクリプト自体が壊れてるか」だったが、ターミナルで直接 `python3 01_Scripts/monitoring/session_start_hook.py` を叩くと普通に動く。ここで「フック経由だと何かが違う」と気づいた。


次のエラー（推測ではなく実際に `2>&1` でキャプチャして確認した内容）：


```
/u​sr/b​in/python3: No such file o​r directory

```

`python3` の解決先が `/u​sr/local/b​in/python3` や `/opt/homebrew/b​in/python3` ではなく、フック実行時の `[PATH](https://docs.python.org/3/library/sys.html)` が削ぎ落とされた状態で `/u​sr/b​in/python3` を探しに行っていた。macOS の `/u​sr/b​in/python3` は Xcode Command Line Tools が入っていないと存在しない。


## 解決までの手順

**ステップ1: フックの出力をまずキャプチャする**


`2>&1` をつけてエラーを拾えるようにする。出力先を一時ファイルに書き出した。


```json
{
  "command": "python3 ~/Documents/AI_Automation_Base/01_Scripts/monitoring/session_start_hook.py >> /tmp/hook_debug.log 2>&1"
}

```

`/tmp/hook_debug.log` を見ると `No such file o​r directory` が出ていた。ここでようやく原因が「パス問題」と確定した。


“


**ステップ2: フルパスで python3 を指定する**


`which python3` で確認してフルパスを直書きする。


```bash
which python3
# → /opt/homebrew/b​in/python3

```


```json
{
  "command": "/opt/homebrew/b​in/python3 ~/Documents/AI_Automation_Base/01_Scripts/monitoring/session_start_hook.py >> /tmp/hook_debug.log 2>&1"
}

```

これで `No such file o​r directory` は消えた。が、次のエラーが来た。


**ステップ3: 仮想環境の依存パッケージ問題**


スクリプトが `httpx` をインポートしているが、システムの Python3 にはない。仮想環境に入れてあるやつを使わないといけない。


解決策は2つある。フックから `bash -c` 経由でシェルスクリプトを呼び、その中で `source .venv/b​in/activate` してから Python を起動する方法と、仮想環境の Python バイナリを直指定する方法。後者のほうが単純なので採用した。


```json
{
  "command": "/Users/ichinosetaito/Documents/AI_Automation_Base/.venv/b​in/python3 /Users/ichinosetaito/Documents/AI_Automation_Base/01_Scripts/monitoring/session_start_hook.py >> /tmp/hook_debug.log 2>&1"
}

```

**ステップ4: 作業ディレクトリの問題**


スクリプト内で `open("04_Config/.env")` のような相対パスを使っていた部分が `FileNotFoundError` になった。フック実行時のカレントディレクトリは Claude Code がどこで起動したかによって変わる。絶対パスに直すか、スクリプト先頭で `os.chdir()` するかで対処。


```python
import os
os.chdir(os.path.dirname(os.path.abspath(__file__)))
# これで __file__ の場所を基準に相対パスが使える

```

**ステップ5: ANTHROPIC_API_KEY の漏れ問題**


これが一番ハマった。フックから `subprocess.run(["claude", "-p", ...])` を呼んでいる箇所があり、フック実行時の環境変数に `ANTHROPIC_API_KEY` が含まれていると API クレジットを消費しようとして残高不足エラーになる。Claude Code のサブスクリプション利用なのに API クレジットを食いに行く謎挙動。


```python
# NG: 環境変数をそのまま渡す
subprocess.run(["claude", "-p", prompt])

# OK: ANTHROPIC_API_KEY を除外して渡す
env = {k: v for k, v in os.environ.items() if k != "ANTHROPIC_API_KEY"}
subprocess.run(["claude", "-p", prompt], env=env)

```

## コード／設定の抜粋

最終的な `settings.json` のフック設定：


```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "/Users/ichinosetaito/Documents/AI_Automation_Base/.venv/b​in/python3 /Users/ichinosetaito/Documents/AI_Automation_Base/01_Scripts/monitoring/session_start_hook.py >> /tmp/claude_hook.log 2>&1"
          }
        ]
      }
    ]
  }
}

```

スクリプト冒頭の定形処理：


```python
import os
import sys

# 作業ディレクトリをスクリプトの場所に固定
os.chdir(os.path.dirname(os.path.abspath(__file__)))

# claude -p を使う場合は API キーを除外
def run_claude(prompt: str) -> str:
    env = {k: v for k, v in os.environ.items() if k != "ANTHROPIC_API_KEY"}
    result = subprocess.run(
        ["claude", "-p", prompt],
        capture_output=True, text=True, env=env
    )
    return result.stdout.strip()

```

デバッグ用のシェル：


```bash
#!/b​in/bash
# フックと同じ条件でスクリプトを手動実行してテストする
env -i HOME="$HOME" PATH="/opt/homebrew/bin:/u​sr/local/bin:/u​sr/bin:/bin" \
  /Users/ichinosetaito/Documents/AI_Automation_Base/.venv/b​in/python3 \
  /Users/ichinosetaito/Documents/AI_Automation_Base/01_Scripts/monitoring/session_start_hook.py

```

`env -i` で環境変数をクリーンにした状態を再現できるので、「ターミナルでは動くのにフックで動かない」を手元で再現しやすくなる。


## 試してわかったこと

Claude Code のフックが実行されるプロセスは、ターミナルのシェルとは別の環境で動いている。`PATH` が削ぎ落とされているし、`VIRTUAL_ENV` も当然セットされていない。「ターミナルで動く = フックで動く」とはならない。


フックで何かを動かすときのチェックリストをまとめると：


- `python3` はフルパスか `.venv/b​in/python3` 直指定
- スクリプト内の相対パスは `os.chdir()` か `pathlib.Path(__file__)` 基準で管理
- `claude -p` 呼び出しがあるなら `ANTHROPIC_API_KEY` を env から抜く
- まず `>> /tmp/debug.log 2>&1` でエラーをキャプチャしてから直す


フックはサイレントに失敗する。`exit code: 0` が返ってきても「何もしなかった」だけで成功している可能性がある。ログ出力は必須。


## まとめ

Claude Code hook の Python 起動は「PATH・仮想環境・作業ディレクトリ」の三つ全部を意識しないと黙って失敗する。デバッグは `2>&1` でログをキャプチャしてから始める。`claude -p` を呼ぶスクリプトは `ANTHROPIC_API_KEY` の除外を忘れずに。


📘 この記事のテーマをさらに深掘りした本
### [Claude Codeで副業自動化——月5万円を目指した全記録](https://www.amazon.co.jp/dp/B0GX2ZV7DP?tag=taito2525-22)

非エンジニアが1年で月10万円に到達した全パイプラインと失敗談


[Kindleで読む →](https://www.amazon.co.jp/dp/B0GX2ZV7DP?tag=taito2525-22)


一ノ瀬泰斗
AI自動化エンジニア / Python個人開発者
Claude Code × Ollama × launchd で SNS・ブログ・KDPを全自動化。実測データと失敗談を軸に、月5万円収益化のリアルな記録を発信中。


💬 **自動化の相談・小規模受託も受付中**：「launchd で毎朝 AI が動く仕組みを作りたい」「KDP の自動出版を組みたい」など、[X (@taito_automate) の DM](https://x.com/taito_automate) からお気軽にどうぞ。


### 関連記事


- [Claude Code 完全攻略ガイド——インストールから実戦運用・トラブル解決まで](https://ichinose-taito.com/?p=4844)
- [Claude API vs ChatGPT API コスト比較——自動化パイプライン3ヶ月の実測データ](https://ichinose-taito.com/?p=4803)
- [Claude Code vs Cursor 比較：1ヶ月使い比べてわかった実力差](https://ichinose-taito.com/claude-code-vs-cursor-%e6%af%94%e8%bc%83%ef%bc%9a1%e3%83%b6%e6%9c%88%e4%bd%bf%e3%81%84%e6%af%94%e3%81%b9%e3%81%a6%e3%82%8f%e3%81%8b%e3%81%a3%e3%81%9f%e5%ae%9f%e5%8a%9b%e5%b7%ae/)


## 関連記事


- [launchd × BlueSky自動エンゲージ、429エラーと日次上限に詰まった記録](https://ichinose-taito.com/launchd-x-bluesky%e8%87%aa%e5%8b%95%e3%82%a8%e3%83%b3%e3%82%b2%e3%83%bc%e3%82%b8%e3%80%81429%e3%82%a8%e3%83%a9%e3%83%bc%e3%81%a8%e6%97%a5%e6%ac%a1%e4%b8%8a%e9%99%90%e3%81%ab%e8%a9%b0%e3%81%be/)
- [KDP自動出版が3日詰まった原因はカテゴリー「場所」チェックボックスだった](https://ichinose-taito.com/kdp%e8%87%aa%e5%8b%95%e5%87%ba%e7%89%88%e3%81%8c3%e6%97%a5%e8%a9%b0%e3%81%be%e3%81%a3%e3%81%9f%e5%8e%9f%e5%9b%a0%e3%81%af%e3%82%ab%e3%83%86%e3%82%b4%e3%83%aa%e3%83%bc%e3%80%8c%e5%a0%b4%e6%89%80/)
- [Ollama をlaunchdで常駐させたら Mac がフル稼働になった話——KeepAlive の罠と2日かかった原因特定](https://ichinose-taito.com/ollama-%e3%82%92launchd%e3%81%a7%e5%b8%b8%e9%a7%90%e3%81%95%e3%81%9b%e3%81%9f%e3%82%89-mac-%e3%81%8c%e3%83%95%e3%83%ab%e7%a8%bc%e5%83%8d%e3%81%ab%e3%81%aa%e3%81%a3%e3%81%9f%e8%a9%b1-k/)

---

この記事はもともと私のブログで書いたものです。より詳しい実装例・失敗談はこちらで読めます。

👉 [Claude Code hookでPythonスクリプトが黙って落ちる問題——パス・環境変数・作業ディレクトリの三重奏を全部解決した](https://ichinose-taito.com/?p=4686)

