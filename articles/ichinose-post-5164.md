---
title: "pip install エラー解決：自動化スクリプトで詰まった私の記録"
emoji: "✨"
type: "tech"
topics: ["claude", "python", "macos", "llm", "kindle"]
published: true
---

最終更新: 2026-04-28


# pip install エラー解決：自動化スクリプトで詰まった私の記録


⏱この記事は約 11 分で読めます


📖 目次

- [📌 よくある pip エラーパターンと解決の型](#よくある-pip-エラーパターンと解決の型)
- [📌 「47行の赤文字」の読み方](#47行の赤文字の読み方)
- [📌 実際に詰まったケース：Playwright + 仮想環境の地獄](#実際に詰まったケースplaywright--仮想環境の地獄)
- [📌 仮想環境を使うようになったきっかけ](#仮想環境を使うようになったきっかけ)
- [📌 依存関係競合の実録と pip-tools 導入](#依存関係競合の実録と-pip-tools-導入)
- [📌 コピペで使えるデバッグコマンド集](#コピペで使えるデバッグコマンド集)
- [📌 よくある質問](#よくある質問)
- [📌 まとめ](#まとめ)


Python で自動化を組み始めてから半年、エラー画面を開くたびに「またか」と思いながら上から読もうとしていた。47行の赤文字——どこを読めばいいかすらわからない。Stack Overflow で似たエラーを探して、なんとなく試して、なんとなく直る。それが3回続いたところで、「このエラー、また来る気がする」と気づいた。


その予感は正しかった。


launchd + Python の自動化パイプラインを組む中で、pip 絡みのエラーに何度かやられている。この記事は、そのたびに詰まって解決した記録だ。エラーの種類、スタックトレースの読み方、仮想環境を切り直すことを覚えた経緯——実際に踏んだ順番で書く。


## よくある pip エラーパターンと解決の型

私が自動化パイプラインで踏んだエラーを種類別に整理した。遭遇するエラーの8割はこの6パターンに収まる。


エラー種別
典型的なメッセージ
解決方向


モジュール未インストール
`ModuleNotFoundError: No module named 'xxx'`
`pip install xxx`


依存関係の競合
`ERROR: pip's dependency resolver does not currently take into account all the packages`
`pip install --upgrade` or 仮想環境を切り直す


権限エラー
`Permission denied`
`--user` フラグ or sudo（後者は非推奨）


Python バージョン不一致
`python setup.py egg_info did not run successfully`
`python3 -m pip` で実行


キャッシュ壊れ
同じエラーが何度も出る
`pip cache purge`


SSL証明書
`SSLError: certificate verify failed`
`--trusted-host` or `pip install certifi`


残りの2割はライブラリ固有の問題で、ライブラリ名 + GitHub issues で調べる方が早い。pip のドキュメントより issue tracker の方が実態に近いことが多い。


## 「47行の赤文字」の読み方

最初にやらかしたのは、スタックトレースを上から律儀に読もうとしていたことだった。


Python のエラーは「下から読む」。一番最後の行に実際のエラーが書いてある。


```
Traceback (most recent call last):
  File "/Users/taito/scripts/note_poster.py", line 34, in 
    from playwright.sync_api import sync_playwright
  File "/u​sr/local/lib/python3.11/...", line 12, in ...
    ...（中略・たくさんの行）...
ModuleNotFoundError: No module named 'playwright'

```

最終行の `ModuleNotFoundError: No module named 'playwright'` だけ読めばいい。上の40行は「そこに至るまでの経路」で、今は気にしなくていい。解決策は `pip install playwright`。


ただし、依存関係の競合だけは話が違う。`ERROR: ResolutionImpossible` みたいなメッセージが中間に挟まっていたりする。そういうときは全体をコピーして Claude Sonnet 4.6 に投げるのが今の私のやり方だ。エラーの構造ごと説明してくれる。


## 実際に詰まったケース：Playwright + 仮想環境の地獄

note の自動投稿スクリプトを書いていたとき、3回連続で同じエラーが出た。


```bash
$ python3 note_poster.py
ModuleNotFoundError: No module named 'playwright'

```

`pip install playwright` を実行した。ターミナルに「Successfully installed」と出た。もう一度スクリプトを実行した。


```bash
$ python3 note_poster.py
ModuleNotFoundError: No module named 'playwright'

```

また同じ。「さっきインストールしたはずなのに」と思いながら15分くらい同じことを繰り返した。`pip install --force-reinstall playwright` も試した。直らない。


原因は Python インタープリタの混在だった。`pip install` した先の Python と、スクリプトを実行した Python が違った——それだけ。


```bash
# 確認していなかったこと
which python3
# /u​sr/b​in/python3 ← system の Python

which pip3
# /u​sr/local/b​in/pip3 ← Homebrew の Python に紐付いた pip

```

解決は `python3 -m pip install playwright` に変えること。`-m pip` を使うと「今実行しているこの python にインストールする」という意味になる。


```bash
python3 -m pip install playwright
python3 -m playwright install chromium

```

朝7時に気づいたときの安堵感は今でも覚えている。


## 仮想環境を使うようになったきっかけ

最初は venv の存在を知っていても使っていなかった。「面倒くさそう」という理由だけで。


転機は、前日まで動いていた自動化パイプラインが壊れた日だった。別プロジェクトで `pip install requests --upgrade` をしたせいで、古いバージョンに依存していたスクリプトが静かに死んでいた。launchd のエラーログを掘ったら `ImportError` が延々と積まれていた——気づくまでに半日かかった。「なぜか動かなくなった」ではなく「自分で壊した」という感覚。


それ以来、プロジェクトごとに venv を切るようにした。


```bash
cd ~/Documents/AI_Automation_Base
python3 -m venv .venv
source .venv/b​in/activate

# activate できているか確認
which python3
# → /Users/taito/Documents/AI_Automation_Base/.venv/b​in/python3

pip install playwright requests

```

launchd で自動起動するスクリプトには、plist にフルパスで書く。


```xml

ProgramArguments

  /Users/taito/Documents/AI_Automation_Base/.venv/b​in/python3
  /Users/taito/Documents/AI_Automation_Base/01_Scripts/note_poster.py


```

これをやっていないと launchd から実行したとき「モジュールが見つからない」が出続ける。launchd は venv を自動で activate してくれない——フルパスで明示するしかない。


## 依存関係競合の実録と pip-tools 導入

Playwright と selenium を同じ環境に混在させたとき、こういうメッセージが出た。


```
ERROR: pip's dependency resolver does not currently take into account all the packages
that are installed. This behaviour is the source of the following dependency conflicts.
selenium 4.1.0 requires urllib3[socks]~=1.26, but you have urllib3 2.0.3 which is incompatible.

```

`pip install --upgrade urllib3` を試したら selenium が壊れた。次に `pip install urllib3==1.26.18` を試したら今度は別のものが壊れた。30分くらい格闘して、最終的に仮想環境を作り直して両方を入れ直したら解決した。


依存関係の競合は、格闘するより環境を切り直す方が早い。頭ではわかっていても最初はなかなか割り切れなかった。


長く動かすパイプラインで競合を事前に防ぎたいなら `pip-tools` が使える。`requirements.in` に必要なライブラリ名だけ書いておくと、互換性のとれたバージョンを計算してロックファイルを生成してくれる。


```bash
pip install pip-tools

# requirements.in に必要なものを列挙
echo "playwright" >> requirements.in
echo "requests" >> requirements.in

# バージョン解決してロックファイルを生成
pip-compile requirements.in

# ロックファイルからインストール
pip-sync requirements.txt

```

毎回やるかというとやらない。「このパイプラインは半年動かし続ける」と決めたときだけ導入している。


## コピペで使えるデバッグコマンド集

詰まったときにまず叩くコマンドをまとめた。


```bash
# 今どの Python を使っているか確認
which python3 && python3 --version

# pip がどこにインストールするか確認
python3 -m pip show pip | grep Location

# インストール済みパッケージ一覧
python3 -m pip list

# 特定パッケージの情報
python3 -m pip show playwright

# キャッシュをクリア（謎のエラーが続くとき）
python3 -m pip cache purge

# 依存関係の競合チェック
python3 -m pip check

# 全パッケージを requirements.txt に書き出す
python3 -m pip freeze > requirements.txt

# requirements.txt から一括インストール
python3 -m pip install -r requirements.txt

```

`pip check` は知らなかった期間が長い。既存の環境で競合が起きているかどうかを確認できる。「なんか調子悪い気がする」というときに叩くと、問題が可視化される。


## よくある質問

### pip install したのに ModuleNotFoundError が出る

`python3 -m pip install` を使っているか確認する。単に `pip install` だと、実行している Python と異なるインタープリタにインストールされている可能性がある。venv を使っているなら `source .venv/b​in/activate` で activate できているかも確認。


### sudo pip install はだめなのか

やらない方がいい。システムの Python 環境を汚す。`--user` フラグか venv を使えば sudo は不要になる。


### pip install が途中でコネクションエラーになる

社内ネットワークやプロキシ環境で起きやすい。`pip install --trusted-host pypi.org --trusted-host files.pythonhosted.org xxx` で試す。私は外出先のモバイル回線に切り替えたら直ったことがある。


### 仮想環境の activate を忘れる

launchd のスクリプト内ではフルパス指定にする。手動実行が多いなら `.zshrc` に auto-activate のフックを書く方法もある。私は忘れがちなので、各プロジェクトの `Makefile` に `.venv/b​in/python3` で実行するターゲットを書いている。


### pip install が非常に遅い

PyPI のミラーを国内サーバーに変えると速くなる場合がある。`~/.pip/pip.conf` でデフォルトを設定しておくと毎回オプションを付けなくて済む。


## まとめ

エラーを見て「ああ、このパターンか」と思えるようになるまで時間はかかった——でも、思えるようになる。


今の私がいつも守っていることは3つだけだ。`python3 -m pip install` を使う、プロジェクトごとに venv を切る、launchd の plist にはフルパスを書く。この3つで pip 絡みのエラーの9割は消えた。残りの1割は——その都度詰まって、その都度記録している。✨


この記事を読んだあなたに：

– launchd でPythonスクリプトを自動実行する設定方法 #

– Playwright 自動化の始め方：Brave CDP 接続で詰まらなくなった #

– Ollama ローカルLLMのセットアップと自動化パイプラインへの組み込み方 #


```json


```


📘 この記事のテーマをさらに深掘りした本
### [Claude Codeで副業自動化——月5万円を目指した全記録](https://www.amazon.co.jp/dp/B0GX2ZV7DP?tag=taito2525-22)

非エンジニアが1年で月10万円に到達した全パイプラインと失敗談


[Kindleで読む →](https://www.amazon.co.jp/dp/B0GX2ZV7DP?tag=taito2525-22)


一ノ瀬泰斗
AI自動化エンジニア / Python個人開発者
Claude Code × Ollama × launchd で SNS・ブログ・KDPを全自動化。実測データと失敗談を軸に、月5万円収益化のリアルな記録を発信中。


💬 **自動化の相談・小規模受託も受付中**：「launchd で毎朝 AI が動く仕組みを作りたい」「KDP の自動出版を組みたい」など、[X (@taito_automate) の DM](https://x.com/taito_automate) からお気軽にどうぞ。


{"@context": "https://schema.org", "@type": "FAQPage", "mainEntity": [{"@type": "Question", "name": "pip install したのに ModuleNotFoundError が出る", "acceptedAnswer": {"@type": "Answer", "text": "python3 -m pip install  を使っているか確認する。単に  pip install  だと、実行している Python と異なるインタープリタにインストールされている可能性がある。venv を使っているなら  source .venv/b​in/activate  で activate できているかも確認。"}}, {"@type": "Question", "name": "sudo pip install はだめなのか", "acceptedAnswer": {"@type": "Answer", "text": "やらない方がいい。システムの Python 環境を汚す。 --user  フラグか venv を使えば sudo は不要になる。"}}, {"@type": "Question", "name": "pip install が途中でコネクションエラーになる", "acceptedAnswer": {"@type": "Answer", "text": "社内ネットワークやプロキシ環境で起きやすい。 pip install --trusted-host pypi.org --trusted-host files.pythonhosted.org xxx  で試す。私は外出先のモバイル回線に切り替えたら直ったことがある。"}}, {"@type": "Question", "name": "仮想環境の activate を忘れる", "acceptedAnswer": {"@type": "Answer", "text": "launchd のスクリプト内ではフルパス指定にする。手動実行が多いなら  .zshrc  に auto-activate のフックを書く方法もある。私は忘れがちなので、各プロジェクトの  Makefile  に  .venv/b​in/python3  で実行するターゲットを書いている。"}}, {"@type": "Question", "name": "pip install が非常に遅い", "acceptedAnswer": {"@type": "Answer", "text": "PyPI のミラーを国内サーバーに変えると速くなる場合がある。 ~/.pip/pip.conf  でデフォルトを設定しておくと毎回オプションを付けなくて済む。"}}]}

---

この記事はもともと私のブログで書いたものです。より詳しい実装例・失敗談はこちらで読めます。

👉 [pip install エラー解決：自動化スクリプトで詰まった私の記録](https://ichinose-taito.com/?p=5164)


