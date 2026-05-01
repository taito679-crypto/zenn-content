---
title: "Python 環境ズレ デバッグチェックリストと再発防止コマンド集"
emoji: "✨"
type: "tech"
topics: ["claude", "python", "macos", "llm", "kindle"]
published: true
---

最終更新: 2026-04-28


# Python 環境ズレ デバッグチェックリストと再発防止コマンド集


⏱この記事は約 11 分で読めます


📖 目次

- [📌 どのPythonが動いているか——環境別のパスと罠](#どのpythonが動いているか環境別のパスと罠)
- [📌 40分溶かした実体験——何が起きていたか](#40分溶かした実体験何が起きていたか)
- [📌 環境ズレが起きる4つの原因パターン](#環境ズレが起きる4つの原因パターン)
- [📌 コピペで使えるデバッグチェックリスト](#コピペで使えるデバッグチェックリスト)
- [📌 launchd 固有の落とし穴——plist 更新忘れ](#launchd-固有の落とし穴plist-更新忘れ)
- [📌 よくある質問](#よくある質問)
- [📌 まとめ](#まとめ)


noteの自動投稿スクリプトが「完走しているのに公開されない」状態に陥ったことがある。エラーはゼロ。ログに`[SUCCESS] Posted: 2026-04-27T08:01:03`が並んでいる。でもnoteのダッシュボードを開くと、記事は「下書き」のままだった。


原因を探るのに40分かかった。


`python3 -c "import sys; print(sys.executable)"` の一行で答えは出た。


この記事でわかること：

– Pythonがどの環境で動いているか一発で確認する方法

– 環境ズレが起きる原因パターン4種

– launchd/cronで実行するスクリプト固有の落とし穴

– 再発防止のためのデバッグチェックリスト（コピペ可）


## どのPythonが動いているか——環境別のパスと罠


環境
パス例
よくある罠


システムPython
/usr/b​in/python3
macOS標準、パッケージ追加不可 [Pythonのバージョン違いによるパッケージ互換性エラー](https://ichinose-taito.com/pip-install-%e3%82%a8%e3%83%a9%e3%83%bc%e8%a7%a3%e6%b1%ba%ef%bc%9a%e8%87%aa%e5%8b%95%e5%8c%96%e3%82%b9%e3%82%af%e3%83%aa%e3%83%97%e3%83%88%e3%81%a7%e8%a9%b0%e3%81%be%e3%81%a3%e3%81%9f%e7%a7%81/)


Homebrew Python
/opt/homebrew/b​in/python3
brew upgrade後にバージョンが上がる


pyenv
~/.pyenv/shims/python3
VERSION設定ミスで別バージョンが動く


venv
/path/to/.venv/b​in/python3
activate忘れ・別シェルで実行すると無効


Conda
/opt/miniconda3/envs/xxx/b​in/python
base環境で動いて気づかない


問題は「動く」という一点にある。エラーが出ないから気づけない——古い環境・古いライブラリのまま処理が完走し、意図しない挙動だけが残る。


## 40分溶かした実体験——何が起きていたか

スクリプトはnoteへの自動投稿処理だ。Playwright + Chromiumで動かしていて、記事生成→下書き保存→公開ボタンクリックという流れを自動化している。


完走する。ログも正常終了。でもnoteのダッシュボードには「下書き」が1件増えているだけ。


ログを掘り始めた。公開ボタンのURL検出ロジックを疑った。直接遷移する実装に切り替えたはずなのに、座標クリックの古い処理フローが動いている形跡がある。「あれ、コードを修正したはずでは？」——ここで気づいた。


コードは修正済みだった。新しいvenvにも配置済みだった。でもlaunchdのplistは古いPythonパスを参照したままだった。


```
# スクリプト冒頭に仕込んだログで一発でわかった
[BOOT] python = /usr/local/b​in/python3          ← 古いHomebrew Python 3.10
# 期待値:
[BOOT] python = /Users/ichinosetaito/projects/note-auto/.venv/b​in/python3
```

修正を加えたのは新しいvenvのコード。動いていたのは古いパスのコード。それだけの話。 [パス・環境変数・作業ディレクトリの全設定を見直す](https://ichinose-taito.com/claude-code-hook%e3%81%a7python%e3%82%b9%e3%82%af%e3%83%aa%e3%83%97%e3%83%88%e3%81%8c%e9%bb%99%e3%81%a3%e3%81%a6%e8%90%bd%e3%81%a1%e3%82%8b%e5%95%8f%e9%a1%8c-%e3%83%91%e3%82%b9/)


「それだけ」に40分かかった。


## 環境ズレが起きる4つの原因パターン

### パターン1: launchd/cron がシェル設定を読まない

`.bashrc`や`.zshrc`に書いたPATHも、pyenvの`eval "$(pyenv init -)"`も、launchdから実行されるスクリプトには効かない。launchdは自前の最小限環境で動くので、`python3`と書いても`/usr/b​in/python3`（macOSシステムPython）が拾われることがある。


```
# ❌ launchd plist の危ない書き方
ProgramArguments

    python3
    /Users/ichinosetaito/projects/note-auto/post_note.py


# ✅ venv内バイナリを絶対パスで指定する
ProgramArguments

    /Users/ichinosetaito/projects/note-auto/.venv/b​in/python3
    /Users/ichinosetaito/projects/note-auto/post_note.py

```

私はこれを知っていたのに、venvを作り直したときにplistの更新を忘れた。


### パターン2: venv を activate したのに別シェルで実行した

ターミナルのタブAで`source .venv/b​in/activate`、タブBで`python3 post_note.py`を実行する。タブBにはactivateが伝播していない——のでシステムのpython3が動く。エラーが出ないから気づかない。私はマルチタブ作業中にこれを3回やった。


### パターン3: Homebrew アップデート後にパスが変わる

`brew upgrade`を実行するとPythonのバージョンが上がることがある。pyenvのshimが古いバージョンを参照し続け、PATHの順番次第でどちらが動くかが変わる。


```
# アップデート後に確認する
python3 --version
# Python 3.12.3  ← 上がっていた

# pyenvとHomebrewのどちらが優先されているか
which python3
# /opt/homebrew/b​in/python3  ← pyenvより前にHomebrewのPATHが来ている

# pyenvで固定しているバージョン
pyenv version
# 3.11.9 (set by /Users/ichinosetaito/.pyenv/version)
# → shebang行のpython3.11を期待していたが、Homebrewの3.12が動いていた
```

アップデート後に`which python3`を確認する習慣がなかった。


### パターン4: どの pip でインストールしたか不明

`pip install playwright`したはずなのに`ModuleNotFoundError`が出る——これは大体、`which python3`と`which pip3`が別々の環境を指しているときに起きる。


```
# 食い違っている状態の例
which python3
# /Users/ichinosetaito/projects/note-auto/.venv/b​in/python3  ← venv

which pip3
# /opt/homebrew/b​in/pip3  ← venvのpipではない

# 解決策: python3 -m pip で実行する
python3 -m pip install playwright
# これで「今使っているpython3」に確実にインストールされる
```


## コピペで使えるデバッグチェックリスト

「なんかおかしい」と思ったらこれを上から流す。


```
# 1. どのpythonが動くか
which python3
python3 -c "import sys; print(sys.executable)"

# 2. バージョン確認
python3 --version

# 3. pipとpythonが同じ環境を指しているか
python3 -m pip --version
# pip 24.0 from /Users/ichinosetaito/projects/note-auto/.venv/lib/python3.11/site-packages/pip (python 3.11)
# ↑ このパスがpython3のパスと一致しているかチェック

# 4. 対象パッケージが今の環境に入っているか
python3 -m pip show playwright
python3 -m pip show requests

# 5. launchd plistの参照先（macのみ）
grep -A 3 "ProgramArguments" ~/Library/LaunchAgents/com.taito.note-auto.plist
```

スクリプトの冒頭に仕込んでおくと、ログファイルを見ればどの環境で動いたか一発でわかる：


```
import sys, os, platform

print(f"[BOOT] python  = {sys.executable}", flush=True)
print(f"[BOOT] version = {platform.python_version()}", flush=True)
print(f"[BOOT] cwd     = {os.getcwd()}", flush=True)
```

launchd経由のスクリプトには必ずこれを入れるようにした。ログの先頭行を見るだけで、plistが正しいパスを参照しているかどうかわかる。


## launchd 固有の落とし穴——plist 更新忘れ

今回の事故の根本はここだった。venvを作り直したとき、plistを更新しなかった。


plistを変更したらリロードが必要だ。unload→loadを忘れると古いplistが有効なまま動き続ける：


```
# plist変更後は必ずこれ
launchctl unload ~/Library/LaunchAgents/com.taito.note-auto.plist
launchctl load  ~/Library/LaunchAgents/com.taito.note-auto.plist

# ロードされているか確認
launchctl list | grep taito
```

「変更が反映されていないのに反映されたと思う」状態がいちばん厄介だ。エラーが出ない、ログが綺麗に流れる、だから動いていると思い込む。


## よくある質問

### venv を activate したのに効いていないのはなぜ？

`source .venv/b​in/activate`はそのシェルセッション内にのみ有効だ。launchd・cron・別プロセスからの呼び出しには伝播しない。解決策はvenv内のPythonバイナリを絶対パスで指定すること——`/path/to/.venv/b​in/python3 script.py`で直接実行するか、スクリプトのshebang行に`#!/path/to/.venv/b​in/python3`と書く。


### pip install したのに ModuleNotFoundError が出る

`python3 -m pip --version`と`python3`のパスが一致しているか確認する。食い違っていれば別環境にインストールしている。`python3 -m pip install パッケージ名`で実行すれば、今の`python3`に紐づくpipが使われるので確実。


### pyenv と venv はどちらを使えばいいか？

用途が違う。pyenvは「Pythonのバージョン」を切り替えるもの、venvは「同じバージョン内でパッケージを分離」するものだ。私はpyenvでプロジェクトごとにバージョンを固定したうえで、プロジェクトごとにvenvを切る構成にしている。launchdから実行するスクリプトはvenv内のバイナリを絶対パスで指定する——これだけで大半のトラブルは防げる。


### スクリプトを修正したのに古い挙動のまま

「どの環境のコードを実行しているか」と「どの環境に修正を加えたか」がズレている可能性がある。`python3 -c "import sys; print(sys.executable)"`でPythonのパスを確認し、そのパスのプロジェクトディレクトリに最新コードが入っているかを確認する。launchd plistが古いパスを参照していないかも見る。今回の私の事故はまさにこれだった。


### Homebrew の Python と pyenv が混在して混乱している

`pyenv doctor`で警告を確認する。PATHの順番が問題のことが多い。`.zshrc`で`export PATH="$HOME/.pyenv/bin:$PATH"`を先頭に置くとpyenvが優先される。ただしlaunchdは`.zshrc`を読まないので、plistには絶対パスを書くのが前提だ。


## まとめ

「動いているのに結果がおかしい」ときは`sys.executable`と`python3 -m pip --version`を最初に疑う。それだけで8割は解決する。


launchd/cronは`.zshrc`を読まない。絶対パス指定が前提で、plist変更後は必ずunload→loadが必要だ。venvのパスを変えたらplistも同時に更新する——片方だけ変えると今回の事故が再現する。


スクリプト冒頭に`sys.executable`をログ出力しておくと、ログファイルで環境を即確認できる。`python3 -m pip install`を使う習慣にすれば、pipとpythonが食い違う問題は起きなくなる。


40分は取り戻せないけど、チェックリストは残った。✨


この記事を読んだあなたに：

– launchd で Python スクリプトを自動実行する設定方法 /launchd-python-autorun

– pyenv + venv の環境構築を30分で終わらせる手順 /pyenv-venv-setup

– Python 自動化スクリプトのエラーログを Discord に飛ばす実装 /python-discord-error-log


📘 この記事のテーマをさらに深掘りした本
### [Claude Codeで副業自動化——月5万円を目指した全記録](https://www.amazon.co.jp/dp/B0GX2ZV7DP?tag=taito2525-22)

非エンジニアが1年で月10万円に到達した全パイプラインと失敗談


[Kindleで読む →](https://www.amazon.co.jp/dp/B0GX2ZV7DP?tag=taito2525-22)


一ノ瀬泰斗
AI自動化エンジニア / Python個人開発者
Claude Code × Ollama × launchd で SNS・ブログ・KDPを全自動化。実測データと失敗談を軸に、月5万円収益化のリアルな記録を発信中。


💬 **自動化の相談・小規模受託も受付中**：「launchd で毎朝 AI が動く仕組みを作りたい」「KDP の自動出版を組みたい」など、[X (@taito_automate) の DM](https://x.com/taito_automate) からお気軽にどうぞ。


{"@context": "https://schema.org", "@type": "FAQPage", "mainEntity": [{"@type": "Question", "name": "venv を activate したのに効いていないのはなぜ？", "acceptedAnswer": {"@type": "Answer", "text": "source .venv/b​in/activate はそのシェルセッション内にのみ有効だ。launchd・cron・別プロセスからの呼び出しには伝播しない。解決策はvenv内のPythonバイナリを絶対パスで指定すること—— /path/to/.venv/b​in/python3 script.py で直接実行するか、スクリプトのshebang行に #!/path/to/.venv/b​in/python3 と書く。"}}, {"@type": "Question", "name": "pip install したのに ModuleNotFoundError が出る", "acceptedAnswer": {"@type": "Answer", "text": "python3 -m pip --version と python3 のパスが一致しているか確認する。食い違っていれば別環境にインストールしている。 python3 -m pip install パッケージ名 で実行すれば、今の python3 に紐づくpipが使われるので確実。"}}, {"@type": "Question", "name": "pyenv と venv はどちらを使えばいいか？", "acceptedAnswer": {"@type": "Answer", "text": "用途が違う。pyenvは「Pythonのバージョン」を切り替えるもの、venvは「同じバージョン内でパッケージを分離」するものだ。私はpyenvでプロジェクトごとにバージョンを固定したうえで、プロジェクトごとにvenvを切る構成にしている。launchdから実行するスクリプトはvenv内のバイナリを絶対パスで指定する——これだけで大半のトラブルは防げる。"}}, {"@type": "Question", "name": "スクリプトを修正したのに古い挙動のまま", "acceptedAnswer": {"@type": "Answer", "text": "「どの環境のコードを実行しているか」と「どの環境に修正を加えたか」がズレている可能性がある。 python3 -c \"import sys; print(sys.executable)\" でPythonのパスを確認し、そのパスのプロジェクトディレクトリに最新コードが入っているかを確認する。launchd plistが古いパスを参照していないかも見る。今回の私の事故はまさにこれだった。"}}, {"@type": "Question", "name": "Homebrew の Python と pyenv が混在して混乱している", "acceptedAnswer": {"@type": "Answer", "text": "pyenv doctor で警告を確認する。PATHの順番が問題のことが多い。 .zshrc で export PATH=\"$HOME/.pyenv/bin:$PATH\" を先頭に置くとpyenvが優先される。ただしlaunchdは .zshrc を読まないので、plistには絶対パスを書くのが前提だ。"}}]}

---

この記事はもともと私のブログで書いたものです。より詳しい実装例・失敗談はこちらで読めます。

👉 [Python 環境ズレ デバッグチェックリストと再発防止コマンド集](https://ichinose-taito.com/?p=5161)


