---
title: "launchd exit 78 エラーの原因と修正方法——設定ミスを3分で特定する"
emoji: "✨"
type: "tech"
topics: ["claude", "python", "macos", "llm", "kindle"]
published: true
---

⏱この記事は約 9 分で読めます


📖 目次

- [📌 exit 78 で3時間溶かした話](#exit-78-で3時間溶かした話)
- [📌 exit 78 が「設定エラー」である理由](#exit-78-が設定エラーである理由)
- [📌 私が踏んだパターン3つ](#私が踏んだパターン3つ)
- [📌 exit コードと原因の対応表](#exit-コードと原因の対応表)
- [📌 デバッグ手順——これをやれば大抵わかる](#デバッグ手順これをやれば大抵わかる)
- [📌 plist を書いたら確認する5点](#plist-を書いたら確認する5点)
- [📌 exit 78 は「環境の差異」シグナルと思えば怖くない](#exit-78-は環境の差異シグナルと思えば怖くない)
- [📌 関連記事](#関連記事)


最終更新: 2026-04-28


## exit 78 で3時間溶かした話

launchd を使い始めて最初の週、私は exit 78 に3時間費やした。スクリプトはターミナルから動く。plist の書き方も合っている気がする。なのに [launchd のパス・環境変数設定で詰まった時の解決策](https://ichinose-taito.com/claude-code-hook%e3%81%a7python%e3%82%b9%e3%82%af%e3%83%aa%e3%83%97%e3%83%88%e3%81%8c%e9%bb%99%e3%81%a3%e3%81%a6%e8%90%bd%e3%81%a1%e3%82%8b%e5%95%8f%e9%a1%8c-%e3%83%91%e3%82%b9/) `launchctl start` するたびに終了コード 78 が返ってくる——。


exit 78 は「設定が壊れている」というシグナルで、スクリプト自体のバグではない。ターミナルと launchd が別の環境で動いているせいで再現する。原因がわかってしまえば直すのは5分 [ターミナルと launchd の環境ズレをデバッグする方法](https://ichinose-taito.com/python-%e7%92%b0%e5%a2%83%e3%82%ba%e3%83%ac-%e3%83%87%e3%83%90%e3%83%83%e3%82%b0%e3%83%81%e3%82%a7%e3%83%83%e3%82%af%e3%83%aa%e3%82%b9%e3%83%88%e3%81%a8%e5%86%8d%e7%99%ba%e9%98%b2%e6%ad%a2%e3%82%b3/)もかからない。でもその「わかるまで」で半日溶かせる。


この記事では、私がパイプライン10本を launchd に乗せる過程で踏んだ失敗パターン [launchd で複数パイプラインを安定稼働させる実装例](https://ichinose-taito.com/%e6%9c%9d%e3%81%ae%e3%80%8c%e7%8a%b6%e6%b3%81%e6%8a%8a%e6%8f%a15%e5%88%86%e3%80%8d%e3%82%92%e3%82%bc%e3%83%ad%e3%81%ab%e3%81%99%e3%82%8bdiscord-bot%e9%80%9a%e7%9f%a5%e3%81%ae%e4%bd%9c%e3%82%8a/)と、デバッグを3分以内に終わらせるための手順を書く。[launchd での自動化スクリプト運用](https://ichinose-taito.com/%e6%9c%9d%e3%81%ae%e3%80%8c%e7%8a%b6%e6%b3%81%e6%8a%8a%e6%8f%a15%e5%88%86%e3%80%8d%e3%82%92%e3%82%bc%e3%83%ad%e3%81%ab%e3%81%99%e3%82%8bdiscord-bot%e9%80%9a%e7%9f%a5%e3%81%ae%e4%bd%9c%e3%82%8a/)の延長として読んでほしい。


## exit 78 が「設定エラー」である理由

launchd が報告する終了コードは、ジョブ自身が返した値をそのまま渡す。exit 78 は `sysexits.h` で定義されている `EX_CONFIG` に相当する——「設定が壊れている状態で起動しようとした」という意味だ。


ここで注意が要る。これはスクリプト内で `sys.exit(78)` を呼んだケースの話ではない。launchd 経由でのみ再現し、手動実行では問題なく動く。そういう症状の場合、犯人はほぼ必ず環境の差異だ。


参照: [Apple Developer Documentation: Creating Launch Daemons and Agents](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingLaunchdJobs.html)


## 私が踏んだパターン3つ

### パターン1: PATH がターミナルと launchd で全然違う

一番多かった。ターミナルで `which python3` を打つと `/opt/homebrew/bin/python3` が返ってくる。でも launchd のデフォルト PATH は `/usr/bin:/bin:/usr/sbin:/sbin` だけだ。Homebrew のパスが入っていない。


plist に `EnvironmentVariables` を書いていないと、Homebrew 経由でインストールしたコマンドが全部見つからなくなる。エラーログには `env: python3: No such file or directory` と出ていた。


```
EnvironmentVariables

    PATH
    /opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin

```

これを追加して `unload` → `load` で直った。追記してから動くまで10秒もかからなかった——3時間の大半はここを疑わずに過ごしていた。


### パターン2: WorkingDirectory の typo か存在しないパス

plist に `WorkingDirectory` を指定していて、そのパスが存在しないか typo している場合も exit 78 になる。


```
WorkingDirectory
/Users/ichinosetaito/Documents/AI_Automation_Base
```

確認はこれだけ。


```
ls /Users/ichinosetaito/Documents/AI_Automation_Base
```

存在確認が通らなければディレクトリ名を見直す。`~` の展開も launchd では効かないので絶対パスで書くこと。私は一度 `~/Documents/AI_Automation_Base` と書いて30分無駄にした。チルダ1文字で30分——これが2回目の沼だった。


### パターン3: venv がアクティブになっていない環境から呼ばれる

これが一番わかりにくかった。ターミナルから実行するときは仮想環境がアクティブになっているので動く。launchd から呼ばれるときはアクティブになっていないので、`import anthropic` とか `import discord` が `ModuleNotFoundError` で死ぬ。エラーログを仕掛けていなかったので、最初は何が起きているのかすら見えなかった。


解決策は venv の Python を直接指定するか、シェルスクリプトで `source activate` してから呼ぶかの2択で、私は前者に統一した。


```

ProgramArguments

    /Users/ichinosetaito/Documents/AI_Automation_Base/.venv/bin/python3
    /Users/ichinosetaito/Documents/AI_Automation_Base/01_Scripts/post_to_bsky.py

```

シェルラッパー経由だと起動のたびにシェル1本分のオーバーヘッドが乗る。数十ms の差でしかないが、パイプライン10本が並走すると地味に積み上がる。venv の Python を直接指定する方がシンプルで依存も少ない。


## exit コードと原因の対応表


exit コード
意味
よくある原因


78
EX_CONFIG（設定エラー）
PATH・WorkingDirectory・venv 未設定


127
command not found
インタープリタのパスが間違い


1
一般エラー
スクリプト内で例外発生


126
実行権限なし
chmod +x していない


0
正常終了
—


exit 78 と 127 はほぼ環境差異の問題で、exit 1 になったらスクリプト側のバグを疑う。私のパイプライン10本で集計すると、exit 78 が障害全体の約6割を占めていた。残りの約2割が exit 1 で、127 や 126 は稀だった。


## デバッグ手順——これをやれば大抵わかる

ログを見る。まずここから。


```
log show --predicate 'subsystem == "com.apple.launchd"' --last 1h | grep -i error
```

ただしこれだと出力が粗い。plist に `StandardErrorPath` を仕込んでおく方が圧倒的に速い。


```
StandardOutPath
/tmp/myjob.out.log
StandardErrorPath
/tmp/myjob.err.log
```

ロードして実行された直後に `cat /tmp/myjob.err.log` を見ると、何が足りないのかが出てくる。`ModuleNotFoundError` なら venv 問題。`No such file or directory` なら PATH か WorkingDirectory 問題。このどちらかで exit 78 の9割は説明できた。


私が最初の3時間で何をしていたかというと——`StandardErrorPath` を設定していなかったので、何も見えない状態で plist の文法ミスを探し続けていた。ログの出口を作る、それだけで世界が変わる。


## plist を書いたら確認する5点

毎回このリストを一周する癖をつけてから、exit 78 で詰まる時間がほぼゼロになった。


- [ ] `ProgramArguments` の Python パスが venv 内の絶対パスになっているか
- [ ] `WorkingDirectory` に typo がないか、ディレクトリが実在するか（`ls` で確認）
- [ ] `EnvironmentVariables` に PATH を明示しているか（Homebrew を使うなら `/opt/homebrew/bin` も含める）
- [ ] `StandardErrorPath` でエラーログを出力しているか
- [ ] plist を変更した後に `launchctl unload ~/Library/LaunchAgents/com.myjob.plist` → `launchctl load ~/Library/LaunchAgents/com.myjob.plist` しているか


最後の「変更後に reload していない」も地味に多い。変更しても反映されていないのに「直らない」と悩むやつ——私は2回やった。`unload` と `load` をセットで実行するワンライナーをシェルの alias に登録してからは起きていない。


## exit 78 は「環境の差異」シグナルと思えば怖くない

exit 78 が出たとき、スクリプトを疑うより先に環境を疑う。PATH・WorkingDirectory・venv のどれかが欠けているだけで、コード自体は正しいことがほとんどだ。


`StandardErrorPath` を必ず設定しておけばデバッグ時間が劇的に短くなる。私の場合、設定前は原因特定まで平均30分かかっていたのが、設定後は3分以内で済むようになった。体感ではなく実測の値だ。


今はパイプラインが10本以上 launchd で動いているが、exit 78 で詰まるのはたいてい新しい plist を書いたときの最初の数分だけだ。慣れたら怖くない。✨


一ノ瀬泰斗
AI自動化エンジニア / Python個人開発者
Claude Code × Ollama × launchd で SNS・ブログ・KDPを全自動化。実測データと失敗談を軸に、月5万円収益化のリアルな記録を発信中。


💬 **自動化の相談・小規模受託も受付中**：「launchd で毎朝 AI が動く仕組みを作りたい」「KDP の自動出版を組みたい」など、[X (@taito_automate) の DM](https://x.com/taito_automate) からお気軽にどうぞ。


### 関連記事


- [Macのcronを捨ててlaunchdに乗り換えた話——自動化パイプライン9本を移行して気づいたこと](https://ichinose-taito.com/?p=4622)
- [macOS の launchd で Python スクリプトを定時実行する完全設定手順【詰まりポイントまとめ】](https://ichinose-taito.com/?p=4595)
- [launchd × BlueSky自動エンゲージ、429エラーと日次上限に詰まった記録](https://ichinose-taito.com/launchd-x-bluesky%e8%87%aa%e5%8b%95%e3%82%a8%e3%83%b3%e3%82%b2%e3%83%bc%e3%82%b8%e3%80%81429%e3%82%a8%e3%83%a9%e3%83%bc%e3%81%a8%e6%97%a5%e6%ac%a1%e4%b8%8a%e9%99%90%e3%81%ab%e8%a9%b0%e3%81%be/)


## 関連記事


- [Macのcronを捨ててlaunchdに乗り換えた話——自動化パイプライン9本を移行して気づいたこと](https://ichinose-taito.com/mac%e3%81%aecron%e3%82%92%e6%8d%a8%e3%81%a6%e3%81%a6launchd%e3%81%ab%e4%b9%97%e3%82%8a%e6%8f%9b%e3%81%88%e3%81%9f%e8%a9%b1-%e8%87%aa%e5%8b%95%e5%8c%96%e3%83%91%e3%82%a4%e3%83%97/)
- macOSのlaunchdで自動化パイプラインを組んで詰まりまくった話——plistの書き方から実務の罠まで全部書く
- [副業で月5万円稼ぐためにAI自動化で実装した6つの仕組み](https://ichinose-taito.com/%e5%89%af%e6%a5%ad%e3%81%a7%e6%9c%885%e4%b8%87%e5%86%86%e7%a8%bc%e3%81%90%e3%81%9f%e3%82%81%e3%81%ab%e5%83%95%e3%81%8cai%e8%87%aa%e5%8b%95%e5%8c%96%e3%81%a7%e5%ae%9f%e8%a3%85%e3%81%97%e3%81%9f6/)

---

この記事はもともと私のブログで書いたものです。より詳しい実装例・失敗談はこちらで読めます。

👉 [launchd exit 78 エラーの原因と修正方法——設定ミスを3分で特定する](https://ichinose-taito.com/?p=4766)


