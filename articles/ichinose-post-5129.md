---
title: "KDP自動出版 品質スコアが全冊70点になった原因と解決手順"
emoji: "✨"
type: "tech"
topics: ["claude", "python", "macos", "llm", "kindle"]
published: true
---

最終更新: 2026-04-28


# KDP自動出版 品質スコアが全冊70点になった原因と解決手順


⏱この記事は約 9 分で読めます


📖 目次

- [📌 KDP自動出版パイプラインの全体像](#kdp自動出版パイプラインの全体像)
- [📌 「全冊同スコア」になる原因3パターン比較](#全冊同スコアになる原因3パターン比較)
- [📌 原因1: JSONパースが静かに失敗していた](#原因1-jsonパースが静かに失敗していた)
- [📌 原因2: スコア集計ロジックのバグ](#原因2-スコア集計ロジックのバグ)
- [📌 実証：修正後のスコア分布](#実証修正後のスコア分布)
- [📌 コピペで使えるスコアリングプロンプト](#コピペで使えるスコアリングプロンプト)
- [📌 よくある質問](#よくある質問)
- [📌 まとめ](#まとめ)


KDP（Kindle Direct Publishing）の出版作業を自動化している。AIが本を書いて、AIが品質チェックして、スコア順に自動出版するパイプラインだ。 [KDP自動化パイプラインのPython+launchd実装例](https://ichinose-taito.com/%e8%b5%b7%e3%81%8d%e3%81%9f%e3%82%89discord%e3%81%ab%e5%9c%a8%e5%ba%ab%e3%83%ac%e3%83%9d%e3%83%bc%e3%83%88%e3%81%8c%e6%9d%a5%e3%81%a6%e3%81%9f-python%ef%bc%8blaunchd%e6%9c%9d%e3%82%a4/)


先週、そのパイプラインを本格稼働させた初日に気づいた。 [パイプライン本番稼働時の異常検知方法](https://ichinose-taito.com/%e8%87%aa%e5%8b%95%e5%8c%96%e3%83%91%e3%82%a4%e3%83%97%e3%83%a9%e3%82%a4%e3%83%b3%e7%9b%a3%e8%a6%96%e3%81%a7%e3%83%ad%e3%82%b0%e3%81%ab%e9%a0%bc%e3%82%8b%e3%81%aa%ef%bc%9a%e7%8a%b6%e6%85%8b%e7%ae%a1/)スコアが全部「70点」だった。1冊だけじゃない。キューに積んであった8冊、全部。


偶然が8回続くわけがない。バグだ。 [全冊同じスコアになるAI採点バグの事例](https://ichinose-taito.com/ai%e8%87%aa%e5%8b%95%e6%8e%a1%e7%82%b9%e3%81%8c%e5%85%a8%e5%86%8a70%e7%82%b9%e3%81%ab%e3%81%aa%e3%81%a3%e3%81%9f%e3%83%90%e3%82%b0%e3%81%ae%e5%8e%9f%e5%9b%a0%e3%81%a8%e4%bf%ae%e6%ad%a3%e6%96%b9/)


この記事でわかること：


- KDP自動出版パイプラインの品質スコアリング仕組み
- 「全冊同スコア」が起きる典型的な原因3パターン
- 実際に僕が踏んだ原因と修正コード
- そのまま使えるスコアリングプロンプトのテンプレ
- よくある詰まりポイントとFAQ


## KDP自動出版パイプラインの全体像

先に構成を整理しておく。


```
kdp_ai_book_generator.py
  └→ 本文生成（[Claude Sonnet](https://docs.anthropic.com/claude/reference/getting-started-with-the-api)）
  └→ 表紙生成（Gemini Web経由）
  └→ content_scorer.py でスコアリング
       └→ S/A/B/C/D/F グレード付け
  └→ priority_stock_manager.py でキュー管理
       └→ スコア順に並べて上から自動出版

```

スコアリングは `content_scorer.py` が担当していて、[Ollama](https://github.com/ollama/ollama)の `qwen2.5:9b` を使ってAIに採点させる。採点基準は「非エンジニア会社員が10分で読み切れるか」「第3章に手を動かせるコードがあるか」など5軸。それぞれ20点満点で合計100点満点。


稼働初日に全冊70点。採点AIが何かやらかしている。


## 「全冊同スコア」になる原因3パターン比較


パターン
症状
確認方法


プロンプト末尾に数字だけ出力させていない
JSONパース失敗→デフォルト値返却
ログに `JSONDecodeError` が出るか確認


Ollamaのthinkingモードがオン
`...` が邪魔してパース失敗
`"think": false` を設定しているか確認


スコア集計ロジックのバグ
常に同じ変数を返している
`print(scores)` を各軸の直後に入れる


僕が踏んだのは1番目と3番目の複合技だった。


## 原因1: JSONパースが静かに失敗していた

`content_scorer.py` の採点部分はこうなっていた。


```python
response = ollama_client.chat(
    model="qwen2.5:9b",
    messages=[{"role": "user", "content": prompt}],
    options={"think": False}
)
raw = response["message"]["content"]

try:
    scores = json.loads(raw)
except json.JSONDecodeError:
    scores = {"total": 70}  # フォールバック

```

フォールバックが `70` だった。


Ollamaが出力するテキストはたまに`json ...`のコードブロックで囲まれて返ってくる。[`json.loads()`](https://docs.python.org/ja/3/library/json.html) はそのまま渡すとエラーになる。エラーになったら70点。全冊。


修正はこれだけ。


```python
# コピペ可
import re

def parse_score_response(raw: str) -> dict:
    # コードブロック除

```

去

    cleaned = re.sub(r"“`(?:json)?\s*|\s*“`", "", raw).strip()

    try:

        return json.loads(cleaned)

    except json.JSONDecodeError:

        # 数字だけ抽出してtotal扱い

        numbers = re.findall(r"\b(\d{1,3})\b", cleaned)

        if numbers:

            return {"total": int(numbers[0])}

        return {"total": 0}  # 0にしてtrash行き


フォールバックを `70` にしていたのが致命的だった。0か-1にしておけばキューで目立つ。「まあ70点なら通してもいいか」という判断がAIを挟まずに発動していた形。


## 原因2: スコア集計ロジックのバグ

採点は5軸あって、それぞれ20点満点。


```python
axes = ["readability", "actionability", "structure", "uniqueness", "completeness"]
total = sum(scores.get(ax, 14) for ax in axes)  # デフォルト14

```

デフォルトが14。5軸 × 14 = 70。


Ollamaが軸名を少し変えて返してくる（`"readability"` → `"read_ability"` とか）と、全軸 `get()` が失敗してデフォルト値合計の70点になる。


修正は軸名の正規化を入れた。


```python
# コピペ可
def normalize_axis_name(name: str) -> str:
    mapping = {
        "read_ability": "readability",
        "action": "actionability",
        "struct": "structure",
        "unique": "uniqueness",
        "complete": "completeness",
    }
    for k, v in mapping.items():
        if k in name.lower():
            return v
    return name.lower()

```

Ollamaは創意工夫してキー名を変えてくる。正規化は必須だった。


## 実証：修正後のスコア分布

修正を入れて8冊を再採点した結果。


```
冊番号   修正前   修正後
---------------------------
book_001   70    82 (A)
book_002   70    61 (C)
book_003   70    91 (S)
book_004   70    55 (D)
book_005   70    78 (B)
book_006   70    44 (F) ← trash行き
book_007   70    85 (A)
book_008   70    69 (C)

```

book_004とbook_006はtrash退避になった。修正前だったら普通に出版されていた。


Fランクが実際に1冊あったのは地味にショックだった。「AIが書いた本だから品質は均一」という思い込みがあったけど、そんなことはない。出来不出来がある。


## コピペで使えるスコアリングプロンプト


```python
# コピペ可
SCORE_PROMPT = """
あなたはKindle電子書籍の品質審査員です。
以下の本文を読んで、5軸で採点してください。

採点軸（各20点満点）:
- readability: 非エンジニアが10分で読み切れるか
- actionability: 読後すぐ手を動かせる内容があるか
- structure: 章の流れが自然か
- uniqueness: 他の入門書と差別化できているか
- completeness: 章立てとして完結しているか

必ずこのJSON形式だけで返してください（説明文不要）:
{
  "readability": ,
  "actionability": ,
  "structure": ,
  "uniqueness": ,
  "completeness": ,
  "total": 
}

本文:
{content}
"""

```

「必ずJSON形式だけで返してください」の一文が効く。これがないとOllamaは感想文を付けてくる。


## よくある質問

### Ollamaのthinkingモードって何ですか？

qwen系の一部モデルは `...` タグで思考過程を出力する機能がある。これがオンだと出力の先頭が `` から始まり、JSONパースが確実に失敗する。


```python
options={"think": False}  # 必須

```

パイプライン内では全てこれを入れている。


### スコアリングにClaude使わないんですか？

API代がかかる。量産フェーズの品質チェックにクレジットを使うのは設計が悪い。Ollamaで十分な精度が出る。Claude Sonnetは本文生成だけに集中させている。


### trash退避になった本はどうなりますか？

`priority_stock_manager.py` が `02_Content/kdp_queue/trash/` に移動させる。自動出版されない。手動で確認して修正するか、そのまま放置するかは僕が判断する。


### 全部Fになったりしませんか？

スコアリングプロンプトを締めすぎると全部F、緩めすぎると全部Sになる。最初の数冊は手動採点と比較して閾値調整が必要だった。「B以上で出版」にしたら週3〜4冊ペースが保てる。


### Ollamaのモデルは何がおすすめですか？

採点用途では `qwen2.5:9b` が安定している。速度と精度のバランスがいい。`gemma4:26b` はパイプライン禁止（重すぎて他の処理を詰まらせる）。


## まとめ


- フォールバック値に「通りそうな点数」を設定するとバグが静かに潜伏する
- Ollamaのレスポンスはコードブロック・キー名ゆらぎ・thinkingタグの3点を想定してパースする
- スコアリングプロンプトには「JSON形式だけで返す」を明記する
- 採点失敗時のデフォルトは0か-1（絶対に出版されない値）にする
- 修正後にtrashが実際に発生したのは良いサイン——スコアリングが機能している証拠


全冊70点は派手なバグだったけど、初日に発見できてよかった。これが気づかれないまま1ヶ月動いていたら、Fランク本が量産されていた。


この記事を読んだあなたに：

– Ollama qwen2.5の使い方と実測速度比較 #

– KDP自動出版パイプライン全体設計と失敗した3つのこと #

– launchd でPythonスクリプトを毎日自動実行する方法 #


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


{"@context": "https://schema.org", "@type": "FAQPage", "mainEntity": [{"@type": "Question", "name": "Ollamaのthinkingモードって何ですか？", "acceptedAnswer": {"@type": "Answer", "text": "qwen系の一部モデルは  ...  タグで思考過程を出力する機能がある。これがオンだと出力の先頭が    から始まり、JSONパースが確実に失敗する。 \n  options={"think": False}  # 必須\n  \n パイプライン内では全てこれを入れている。"}}, {"@type": "Question", "name": "スコアリングにClaude使わないんですか？", "acceptedAnswer": {"@type": "Answer", "text": "API代がかかる。量産フェーズの品質チェックにクレジットを使うのは設計が悪い。Ollamaで十分な精度が出る。Claude Sonnetは本文生成だけに集中させている。"}}, {"@type": "Question", "name": "trash退避になった本はどうなりますか？", "acceptedAnswer": {"@type": "Answer", "text": "priority_stock_manager.py  が  02_Content/kdp_queue/trash/  に移動させる。自動出版されない。手動で確認して修正するか、そのまま放置するかは僕が判断する。"}}, {"@type": "Question", "name": "全部Fになったりしませんか？", "acceptedAnswer": {"@type": "Answer", "text": "スコアリングプロンプトを締めすぎると全部F、緩めすぎると全部Sになる。最初の数冊は手動採点と比較して閾値調整が必要だった。「B以上で出版」にしたら週3〜4冊ペースが保てる。"}}, {"@type": "Question", "name": "Ollamaのモデルは何がおすすめですか？", "acceptedAnswer": {"@type": "Answer", "text": "採点用途では  qwen2.5:9b  が安定している。速度と精度のバランスがいい。 gemma4:26b  はパイプライン禁止（重すぎて他の処理を詰まらせる）。"}}]}

---

この記事はもともと私のブログで書いたものです。より詳しい実装例・失敗談はこちらで読めます。

👉 [KDP自動出版 品質スコアが全冊70点になった原因と解決手順](https://ichinose-taito.com/?p=5129)


