---
title: "AI自動採点が全冊70点になったバグの原因と修正方法"
emoji: "✨"
type: "tech"
topics: ["claude", "python", "macos", "llm", "kindle"]
published: true
---

最終更新: 2026-04-28


# AI自動採点が全冊70点になったバグの原因と修正方法


⏱この記事は約 7 分で読めます


📖 目次

- [📌 バグ発覚——キューの22冊が全部70点](#バグ発覚キューの22冊が全部70点)
- [📌 犯人はfallback値「70」だった](#犯人はfallback値70だった)
- [📌 なぜJSONパースが毎回失敗していたか](#なぜjsonパースが毎回失敗していたか)
- [📌 修正は10行以内で終わった](#修正は10行以内で終わった)
- [📌 再発防止——launchdでスコア分布を毎朝監視する](#再発防止launchdでスコア分布を毎朝監視する)
- [📌 まとめ](#まとめ)


KDPのキューを確認するのは毎朝6時台の習慣になっている。その日の朝、異変に気づいた。22冊並んでいる原稿のスコアがすべて「70」だった。


70点。ぴったり。1点の誤差もなく、22冊全部。


最初は「たまたまクオリティが揃ったのかも」と半信半疑だった。が、手元で2冊読み比べると3,000字のスカスカ原稿と50,000字の詳細解説本が同点という状況——これは明らかにシステムが壊れている。


content_scorer.py。Ollamaに採点JSONを返させてKDPキューの優先度を決めるスクリプトで、2週間かけて組んだやつだ。それが完全に死んでいた。原因の特定に2時間、修正に30分。詰まったのはfallback値の存在を完全に忘れていたから。同じ轍を踏む人が出ないように記録する。


## バグ発覚——キューの22冊が全部70点


まずログを掘った。


```
$ grep "score" logs/scorer_2026-04-27.log | head -10
2026-04-27 06:12:43 [INFO] Scored "Pythonで始める自動化入門": score=70
2026-04-27 06:12:51 [INFO] Scored "launchd完全ガイド": score=70
2026-04-27 06:12:59 [INFO] Scored "Claude Code実践": score=70
2026-04-27 06:13:07 [INFO] Scored "Ollama入門": score=70
...

```

22件中22件が70点。エラーログは1行も出ていない。スクリプトは正常終了している。これが一番厄介なタイプのバグだ——「壊れているのに壊れているように見えない」やつ。採点システムが静かにサボっていた4日間、私はキューが正常に積み上がっていると信じていた。


## 犯人はfallback値「70」だった


content_scorer.pyの採点関数を読み直した。コードは130行ほどで、採点ロジックの末尾にこういう記述があった。


```python
def score_content(text: str) -> int:
    try:
        response = ollama.chat(
            model="qwen2.5:3b",
            messages=[{"role": "user", "content": SCORING_PROMPT.format(text=text[:2000])}]
        )
        result = json.loads(response["message"]["content"])
        return int(result["score"])
    except Exception:
        return 70  # ← こいつが犯人

```

fallback値が70。JSONのパースが失敗するたびにサイレントで70を返す設計になっていた。エラーログが出ないのはそのせい。`except Exception` で全部飲み込んで、何事もなかったように70を返し続けていた。


「なんで70にしたんだろう」と自分のコードを読みながら思った。おそらく「中間点くらいにしておけばいいか」という雑な判断だったと思う。2週間前の自分に言いたい。


## なぜJSONパースが毎回失敗していたか


採点が正常だった時期とバグ発生日を照合すると、4日前に `ollama pull qwen2.5:3b` を走らせていた。モデル更新が怪しい。


更新前のモデルはこういうレスポンスを返していた。


```
{"score": 82, "reason": "構成が明確で技術的な正確性が高い"}

```

更新後はこうなっていた。


```
採点結果をお伝えします。

```json
{"score": 82, "reason": "構成が明確で技術的な正確性が高い"}
```

```

markdownのコードフェンスで囲まれたJSONが返ってくるようになっていた。`json.loads()` は当然失敗する。失敗すると `except Exception: return 70` が発動。全冊70点の完成。


`ollama show qwen2.5:3b` でtemplate設定を確認したら、デフォルトのシステムプロンプト設定が変わっていた。3回くらい「なんで？」と思いながらollamaのGitHubをたどって、やっとそこで気づいた——モデルのアップデートでデフォルト挙動が変わることがある。それをテストなしで本番に流していた。


## 修正は10行以内で終わった

パーサーをrobustにするだけでよかった。コードフェンスを剥がしてからJSONを抽出する処理を挟む。


```python
# 修正前
result = json.loads(response["message"]["content"])
return int(result["score"])

# 修正後
raw = response["message"]["content"]
json_match = re.search(r'\{[^{}]*"score"[^{}]*\}', raw, re.DOTALL)
if not json_match:
    logger.warning(f"JSON parse failed. raw={raw[:200]}")
    return 50  # fallbackを50に変更
result = json.loads(json_match.group())
return int(result["score"])

```

fallbackを70から50に変えた点も地味に大事。70だと「そこそこ良い原稿」に見えてキューに居座り続ける。50ならtrash行きの閾値（60点未満）を下回るので、パース失敗の原稿が混入しない。


修正後に22冊を再採点した。


```
$ python content_scorer.py --rescore-all
Scored "Pythonで始める自動化入門": score=88
Scored "launchd完全ガイド": score=91
Scored "Claude Code実践": score=74
Scored "Ollama入門": score=63
...
Distribution: min=61, max=94, mean=79.4, std=8.2

```

ようやく正常な分布になった。4日間溜まっていたキューが一気に動いた。


## 再発防止——launchdでスコア分布を毎朝監視する

同じことが起きたとき、今度は即気づきたい。スコア分布の異常を検知するスクリプトを追加した。


```python
# score_monitor.py
import json, sys
from pathlib import Path

scores = [entry["score"] for entry in json.loads(Path("queue.json").read_text())]
if len(scores) >= 5 and len(set(scores)) == 1:
    msg = f"[ALERT] 全{len(scores)}件のスコアが{scores[0]}点で完全一致。採点システムを確認してください。"
    print(msg)
    sys.exit(1)

```

launchd plistは `/Library/LaunchAgents/com.taito.score_monitor.plist` に配置して、毎朝7:05に走らせている。異常があればDiscordに通知が飛ぶ。今日までアラートは1度も出ていない——それが正常な状態だ。


## まとめ

今回の4日間ロスで学んだことは2つある。


fallback値は「一目で異常とわかる値」にする——70という中途半端な数字にしたのが失敗だった。0か-1にしておけば翌朝のキュー確認で即気づけた。


`ollama pull` は静かに本番を壊す——モデル更新後は必ずスコアリングのテストを1本走らせる。CIを組んでいなくてもスモークテスト1件あるだけで今回の2時間ロスは防げた。


content_scorer.pyは今も動いている。次に何かが壊れたとき、launchdが7:05に教えてくれる。それだけで十分だ。


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

👉 [AI自動採点が全冊70点になったバグの原因と修正方法](https://ichinose-taito.com/?p=5157)


