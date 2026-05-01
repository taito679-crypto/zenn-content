---
title: "自動化パイプライン監視でログに頼るな：状態管理+Discord通知に切り替えた実装例"
emoji: "✨"
type: "tech"
topics: ["claude", "python", "macos", "llm", "kindle"]
published: true
---

最終更新: 2026-04-28


# 自動化パイプライン監視でログに頼るな：状態管理+Discord通知に切り替えた実装例


⏱この記事は約 14 分で読めます


📖 目次

- [📌 ログが嘘をつく3つのパターン](#ログが嘘をつく3つのパターン)
- [📌 監視手法の比較：ログ・状態ファイル・通知](#監視手法の比較ログ・状態ファイル・通知)
- [📌 状態管理ファイルで「本当の成否」を記録する](#状態管理ファイルで本当の成否を記録する)
- [📌 Discord通知で「気づかない失敗」をゼロにする](#discord通知で気づかない失敗をゼロにする)
- [📌 ttyd + launchd でスマホから直接ターミナルを見る](#ttyd--launchd-でスマホから直接ターミナルを見る)
- [📌 ヘルスチェックスクリプトをコピペで使う](#ヘルスチェックスクリプトをコピペで使う)
- [📌 よくある質問](#よくある質問)
- [📌 まとめ](#まとめ)


自動化スクリプトを launchd で回しはじめると、最初は「ログを見ればなんとかなる」と思う。僕もそう思ってた。


実際はちがった。ログファイルを tail -f してたのに、note投稿スクリプトが2週間ほぼサイレントに失敗し続けていた。exitコードは0。エラー行もない。なのに1本も投稿されていなかった。


原因を掘り返してみると、`note_storage_state.json` に保存してたクッキーが21個中15個期限切れになっており、headless Chromiumがログインできずに「成功した風」で終了していた。ログには `投稿完了` と書いてあった。嘘だった。


この記事でわかること：

– ログ監視が自動化パイプラインで機能しない理由（実例つき）

– 「状態管理ファイル」で成否を永続化する設計

– Discord通知で異常を即時検知する実装

– スマホから監視できる ttyd + launchd 構成

– コピペで使える通知テンプレートとヘルスチェックスクリプト


## ログが嘘をつく3つのパターン

ログが役に立たないとは言わない。デバッグには使う。ただ「監視」のツールとして使うのは設計ミスだ、というのが今の考え。


実際に踏んだ地雷を3パターン紹介する。


「クッキー期限切れ問題」は最初に書いた。ログに `投稿完了` と出るのに投稿ゼロ。Playwright が `storage_state` を読み込み、ログインチェックをスキップして、実際はゲスト状態で note の投稿ボタンを押す。UIはエラーモーダルを出すが、スクリプトは `page.click()` が成功したことだけを確認してexit 0で終わる。ログには何も残らない。


2つ目は「UI変更による誤検出」。note.com がある時期に `/landing?puid=` から SvelteKit の `?app_launch=false`（下書きプレビュー用のURL）に遷移する仕様に変わった。スクリプトがこのURLを「公開完了」と判定していた。ログには `公開URL: https://note.com/...` と出ている。実際は下書きプレビューのURLだった。


3つ目は「並行実行によるログ混在」。launchd で複数のスクリプトを同時に走らせると、stdout を同じファイルに書き込むケースでログが混ざる。タイムスタンプが前後したり、あるスクリプトのエラー行が別スクリプトのブロックに紛れ込む。目視で追うのが事実上不可能になる。


## 監視手法の比較：ログ・状態ファイル・通知


手法
異常検知速度
実装コスト
誤検出リスク
スマホ対応


ログファイル監視
遅い（気づかない）
低
高
✗


状態管理JSON
中（次回起動時）
中
低
△


Discord通知
即時
中
低
◎


ヘルスチェックスクリプト
定期（設定次第）
高
低
◎


ttyd + launchd
即時（手動確認）
中
なし
◎


ログは「事後に読む」ためのものだ。「今何が起きているか」を把握するには向いていない。状態ファイルと通知を組み合わせるのが今のところ一番まともな運用になっている。


## 状態管理ファイルで「本当の成否」を記録する

`exit 0` は信用しない。そう決めてから設計が変わった。


スクリプトが「本当に目的を達成できたか」を判定するロジックを書いて、その結果を JSON に書き込む。launchd が次回スクリプトを起動するときにそのJSONを読んで、前回の状態から再開するか判断する。


```python
# コピペ可
import json
from pathlib import Path
from datetime import datetime

STATE_FILE = Path("04_Config/state/note_post_state.json")

def load_state():
    if STATE_FILE.exists():
        return json.loads(STATE_FILE.read_text())
    return {"last_success": None, "consecutive_failures": 0, "last_url": None}

def save_state(success: bool, url: str = None, error: str = None):
    state = load_state()
    now = datetime.now().isoformat()
    if success:
        state["last_success"] = now
        state["consecutive_failures"] = 0
        state["last_url"] = url
        state["last_error"] = None
    else:
        state["consecutive_failures"] = state.get("consecutive_failures", 0) + 1
        state["last_error"] = error
        state["last_failure"] = now
    STATE_FILE.write_text(json.dumps(state, ensure_ascii=False, indent=2))

```

連続失敗カウントを持つのがポイント。3回連続で失敗したら Discord に通知する閾値として使える。1回の失敗でアラートを飛ばすと通知疲れが起きるので、閾値は2〜3回が現実的だった。


## Discord通知で「気づかない失敗」をゼロにする

スクリプトが失敗しているのに気づかない状態が一番まずい。ダッシュボードを自分から見に行く運用は続かない。通知が来るから気づく、という設計にしないと機能しない。


```python
# コピペ可
import requests
import os

DISCORD_WEBHOOK = os.environ.get("DISCORD_WEBHOOK_URL")

def notify_discord(title: str, body: str, level: str = "info"):
    colors = {"info": 0x5865F2, "success": 0x57F287, "error": 0xED4245, "warn": 0xFEE75C}
    payload = {
        "embeds": [{
            "title": title,
            "description": body,
            "color": colors.get(level, 0x5865F2),
            "timestamp": datetime.now().isoformat()
        }]
    }
    requests.post(DISCORD_WEBHOOK, json=payload, timeout=10)

# 使用例
try:
    url = post_to_note(article)
    save_state(success=True, url=url)
    notify_discord("note投稿✨", f"公開完了\n{url}", level="success")
except Exception as e:
    state = load_state()
    save_state(success=False, error=str(e))
    if state["consecutive_failures"] >= 2:
        notify_discord("note投稿エラー", f"3回連続失敗\n{e}", level="error")

```

実際に運用して気づいたのは、「成功通知もほしい」という点。失敗通知だけだと、スクリプトが動いているのか止まっているのか判断できなくなる。朝の定期実行が完了したら成功通知も飛ばすようにした。スマホを開いて Discord に何も来ていなければ「動いていない」と判断できる。


## ttyd + launchd でスマホから直接ターミナルを見る

Discord 通知で「失敗した」はわかる。でも「なぜ失敗したか」を調べるには、ターミナルに入らないといけない。外出先でMacを開けないときの話だ。


ttyd をlaunchdで常時起動すると、スマホのブラウザから直接ターミナルを操作できる。


```xml


    Label
    com.taito.ttyd
    ProgramArguments
    
        /opt/homebrew/b​in/ttyd
        -p
        7681
        zsh
    
    RunAtLoad
    
    KeepAlive
    
    StandardOutPath
    /Users/taito/.ttyd/ttyd.log
    StandardErrorPath
    /Users/taito/.ttyd/ttyd.err


```

`http://192.168.x.x:7681` にスマホからアクセスすると、ブラウザ上でターミナルが開く。そこで `tail -f 04_Config/logs/note_post.log` してもいいし、状態JSONを `cat` してもいい。


ローカルネットワーク専用なのでセキュリティ上の問題は今のところ許容範囲と判断している。外出先から見たいなら Tailscale と組み合わせる手がある。


## ヘルスチェックスクリプトをコピペで使う

毎朝7時に全スクリプトの状態を確認して Discord に送るワンショットスクリプト。これをlaunchdで定期実行すれば、手動で確認しなくていい。


```bash
#!/b​in/bash
# コピペ可
# 配置場所: 01_Scripts/monitoring/quick_status.sh

STATE_DIR="04_Config/state"
WEBHOOK="${DISCORD_WEBHOOK_URL}"

summary="**パイプライン状態チェック**\n"

for f in "$STATE_DIR"/*.json; do
    name=$(basename "$f" .json)
    failures=$(python3 -c "import json; d=json.load(open('$f')); print(d.get('consecutive_failures', 0))")
    last=$(python3 -c "import json; d=json.load(open('$f')); print(d.get('last_success', 'なし'))")

    if [ "$failures" -ge 2 ]; then
        icon="❌"
    elif [ "$failures" -ge 1 ]; then
        icon="⚠️"
    else
        icon="✅"
    fi

    summary+="${icon} ${name}: 連続失敗=${failures}, 最終成功=${last}\n"
done

curl -s -X POST "$WEBHOOK" \
    -H "Content-Type: application/json" \
    -d "{\"content\": \"${summary}\"}"

```


## よくある質問

### ログファイルは完全に不要になる？

ならない。ログはデバッグに使う。「なぜ失敗したか」を掘るときにログは必要で、消していい理由はない。「監視の主軸」として使わない、という話。状態ファイルとDiscord通知で「何が起きたか」を把握して、原因調査のときにログを読む、という役割分担が機能した。


### Discord通知が多すぎてうるさくなった場合は？

連続失敗カウントの閾値を上げるか、通知チャンネルを「監視専用」と「日次サマリー」に分ける。即時アラートは本当に止まっているときだけ、それ以外は朝のサマリーに集約する運用が今のところ一番うるさくなかった。


### スマホからターミナルを開くのはセキュリティ的に大丈夫？

ローカルネットワーク内限定（192.168.x.x）なら許容範囲だと思っている。外部公開するなら Basic認証か Tailscale のVPN経由にすべき。ポート7681をファイアウォールで外部遮断しておくだけでも最低限の対策になる。


### 状態ファイルが増えすぎてわからなくなったら？

`04_Config/state/` 配下に置いて、スクリプト名と1対1で対応させると管理しやすい。状態ファイル自体を読んで一覧表示するスクリプト（上のquick_status.sh）を作っておけば、個別ファイルを開く必要がなくなる。


### launchdとcronどちらがいい？

Mac で動かすなら launchd 一択。cronは `/etc/crontab` の権限問題やフルディスクアクセスでひっかかることがある。launchd は plist で管理できてログ出力先も指定できる。`keepAlive: true` で落ちたら自動再起動もできる。


## まとめ


- ログの `exit 0` と「完了」メッセージは信用するな。本当の成否は別途確認が必要
- 状態管理JSONに「連続失敗カウント」「最終成功時刻」「最終エラー内容」を書き込む設計にする
- 成功も失敗もDiscordに通知する。通知が来ないこと自体が「止まっているサイン」になる
- ttyd + launchd でスマホから直接ターミナルに入れる環境を作ると、外出先での障害対応が現実的になる
- ヘルスチェックスクリプトを朝一定時に走らせると、朝起きた時点で全パイプラインの状態がわかる


2週間サイレントに失敗し続けていた note投稿スクリプトを発見したのは、状態ファイルを導入して3日後だった。ログを見る習慣をやめて「通知が来なければ成功」に設計を変えた。今のところその方が楽に動いている。


この記事を読んだあなたに：

– launchd 入門：cron より安定する Mac の定期実行設定 launchd-intro

– Playwright セッション切れを Brave CDP 接続で根絶した方法 playwright-brave-cdp

– Discord Bot を Python で作って自動化の監視ハブにした discord-bot-monitor


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


{"@context": "https://schema.org", "@type": "FAQPage", "mainEntity": [{"@type": "Question", "name": "ログファイルは完全に不要になる？", "acceptedAnswer": {"@type": "Answer", "text": "ならない。ログはデバッグに使う。「なぜ失敗したか」を掘るときにログは必要で、消していい理由はない。「監視の主軸」として使わない、という話。状態ファイルとDiscord通知で「何が起きたか」を把握して、原因調査のときにログを読む、という役割分担が機能した。"}}, {"@type": "Question", "name": "Discord通知が多すぎてうるさくなった場合は？", "acceptedAnswer": {"@type": "Answer", "text": "連続失敗カウントの閾値を上げるか、通知チャンネルを「監視専用」と「日次サマリー」に分ける。即時アラートは本当に止まっているときだけ、それ以外は朝のサマリーに集約する運用が今のところ一番うるさくなかった。"}}, {"@type": "Question", "name": "スマホからターミナルを開くのはセキュリティ的に大丈夫？", "acceptedAnswer": {"@type": "Answer", "text": "ローカルネットワーク内限定（192.168.x.x）なら許容範囲だと思っている。外部公開するなら Basic認証か Tailscale のVPN経由にすべき。ポート7681をファイアウォールで外部遮断しておくだけでも最低限の対策になる。"}}, {"@type": "Question", "name": "状態ファイルが増えすぎてわからなくなったら？", "acceptedAnswer": {"@type": "Answer", "text": "04_Config/state/  配下に置いて、スクリプト名と1対1で対応させると管理しやすい。状態ファイル自体を読んで一覧表示するスクリプト（上のquick_status.sh）を作っておけば、個別ファイルを開く必要がなくなる。"}}, {"@type": "Question", "name": "launchdとcronどちらがいい？", "acceptedAnswer": {"@type": "Answer", "text": "Mac で動かすなら launchd 一択。cronは  /etc/crontab  の権限問題やフルディスクアクセスでひっかかることがある。launchd は plist で管理できてログ出力先も指定できる。 keepAlive: true  で落ちたら自動再起動もできる。"}}]}

---

この記事はもともと私のブログで書いたものです。より詳しい実装例・失敗談はこちらで読めます。

👉 [自動化パイプライン監視でログに頼るな：状態管理+Discord通知に切り替えた実装例](https://ichinose-taito.com/?p=5133)


