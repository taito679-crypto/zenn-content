---
title: "KDP表紙 日本語テキスト入り画像生成5ツール比較【2026年実測】"
emoji: "✨"
type: "tech"
topics: ["claude", "python", "macos", "llm", "kindle"]
published: true
---

最終更新: 2026-04-28


# KDP表紙 日本語テキスト入り画像生成5ツール比較【2026年実測】


⏱この記事は約 11 分で読めます


📖 目次

- [📌 5ツール比較表](#5ツール比較表)
- [📌 Gemini Web経由：日本語テキストの精度が別格](#gemini-web経由日本語テキストの精度が別格)
- [📌 DALL-E 3（GPT-4o API）：日本語は後から重ねる前提で使う](#dall-e-3gpt-4o-api日本語は後から重ねる前提で使う)
- [📌 Stable Diffusion（ComfyUI）：背景生成専用機として割り切った](#stable-diffusioncomfyui背景生成専用機として割り切った)
- [📌 Pollinations.ai：日本語を投げた瞬間に沈んだ](#pollinationsai日本語を投げた瞬間に沈んだ)
- [📌 Midjourney：品質は認めるが自動化できない](#midjourney品質は認めるが自動化できない)
- [📌 夜間無人稼働で実際に動かしているカスケード構成](#夜間無人稼働で実際に動かしているカスケード構成)
- [📌 よくある質問](#よくある質問)
- [📌 まとめ](#まとめ)


KDPの表紙生成を自動化しようとして、最初の3日間は日本語テキストの問題だけで溶けた。Pollinations.aiに日本語を含むURLを投げたら即404。Stable DiffusionのComfyUIで試したらタイトル部分が豆腐の行列になった。DALL-E 3をAPIで叩いたら「10分でわかるAI副業」が「10分でわわんAI」みたいな崩れ方で返ってきた。笑えない。


毎日1本KDPを出すパイプラインを動かすつもりが、表紙の生成だけで止まり続ける。諦めるか5ツール全部叩いて比較するか——後者を選んだ記録がこの記事。日本語テキストが崩れるかどうかの実力差、完全自動化パイプラインに組み込めるかどうかの判定、詰まった場所と回避策まで書く。


## 5ツール比較表


ツール
日本語文字品質
API自動化
コスト
速度
総合評価


Gemini（Web経由）
◎ 文字崩れほぼなし
△ CDP操作必要
無料枠あり
中
A


DALL-E 3（GPT-4o）
○ Pillow合成前提で安定
◎ API直接
1枚0.04ドル
速
B


Stable Diffusion（ComfyUI）
△ 日本語はほぼ無理
◎ ローカルAPI
電気代のみ
中
C


Pollinations.ai
× 日本語で404多発
◎ 無料API
無料
速
D


Midjourney
△ 英語優先・文字弱い
× Discord限定
月10ドル〜
中
D


全部試した結果、Gemini一択だった。以下に詳細を書く。


## Gemini Web経由：日本語テキストの精度が別格

Gemini 2.0 FlashとUltraを両方試した。どちらも日本語テキストを画像内に描写させると、他ツールと比べて明らかに精度が違う。ただしプロンプトの書き方に罠がある。


最初は「日本語で書けばいいだろう」と思ってそのまま投げた。「AI副業の本の表紙、ミニマルデザイン、暗い背景、タイトルを大きく配置して」——これで試したら、文字が崩れた。確率にすると6〜7割くらいで何かがおかしくなる。「配置して」みたいな日本語動詞が混ざると特に怪しい。


解決したのは「英語でプロンプトを書いて、日本語テキストを文字列として明示する」パターン。


```
# コピペ可
Book cover design, minimalist style, dark gradient background,
Japanese title text "10分でわかるAI副業" displayed prominently at the top,
author name "一ノ瀬泰斗" at the bottom,
clean typography, professional ebook cover

```

これに変えたら文字化けがほぼ消えた。体感9割以上で正確に描写される。プロンプト全体を日本語で書くと崩れやすく、英語の文脈に日本語テキストを「文字列として埋め込む」形にするのがポイント。


自動化はPlaywright + Brave CDP経由で実装している。`launch_brave_debug.sh`でデバッグポートを開いたBraveに接続して、Gemini WebのUIを操作して画像を取得する構成。セッション切れもなく、1日20枚程度の生成量なら無料枠で収まっている。


```python
# cdp_browser.py 経由で Brave に接続
from cdp_browser import connect_cdp_sync
from playwright.sync_api import sync_playwright

with sync_playwright() as pw:
    browser, ctx = connect_cdp_sync(pw)
    page = ctx.new_page()
    page.goto("https://gemini.google.com")
    # プロンプト入力 → 画像取得
    browser.close()

```


## DALL-E 3（GPT-4o API）：日本語は後から重ねる前提で使う

APIで直接叩けるので自動化との相性はいい。ただし日本語テキストをプロンプトで指定すると、Geminiより崩れやすい。精度を上げようとプロンプトをいじり続けたが、Geminiには追いつかなかった。


試行錯誤して今落ち着いているのが「DALL-E 3で背景だけ生成して、Pillowで日本語テキストを後から合成する」ハイブリッド。背景の品質は高い。日本語テキストは任せない——その割り切りで安定した。


```python
# コピペ可
import openai
from PIL import Image, ImageDraw, ImageFont
import requests

client = openai.OpenAI()
response = client.images.generate(
    model="dall-e-3",
    prompt="Professional ebook cover, dark theme, abstract tech pattern, no text",
    size="1024x1024"
)

img = Image.open(requests.get(response.data[0].url, stream=True).raw)
draw = ImageDraw.Draw(img)
font = ImageFont.truetype("/System/Library/Fonts/ヒラギノ角ゴシック W6.ttc", 60)
draw.text((100, 100), "10分でわかるAI副業", font=font, fill="white")
img.save("cover.png")

```

コストは1枚0.04ドル。月100枚生成しても4ドル。許容できる金額だが、Geminiが無料で精度も高いので出番は少ない。カスケードパイプラインの2番手として待機させている位置づけ。


## Stable Diffusion（ComfyUI）：背景生成専用機として割り切った

ローカルで動かせるのは強みだが、日本語テキストの描写は実質あきらめた。


LoRAを3本試した。「文字LoRA」と呼ばれるモデルで、まともに動いたのは1本だけ——しかも品質のばらつきが大きくて、文字を正確に出そうとすると背景の別のところが崩れる。英語アルファベットに最適化されているものがほとんどで、日本語特化はまだ少ない印象。


今は「背景デザインだけSD、テキスト合成はPillow」で分業させている。電気代だけで動くのでコストはゼロ。夜間の無人稼働でも問題なく回っている。


```
# ComfyUI API呼び出し例
# ローカル: http://localhost:8188/prompt

import requests, json

prompt = {
    "3": {
        "class_type": "KSampler",
        "inputs": {
            "model": ["4", 0],
            "positive": ["6", 0],
            "negative": ["7", 0],
            "seed": 42, "steps": 20, "cfg": 7
        }
    }
    # ...以下省略
}
requests.post("http://localhost:8188/prompt", json={"prompt": prompt})

```


## Pollinations.ai：日本語を投げた瞬間に沈んだ

「無料で使えるAPIがある」という情報を見て期待していた。実際に試したら、日本語文字列を含むURLを投げた瞬間に404が返ってきた。最初はエンコードの問題だと思ってURLエンコードして再送信しても404。


原因を調べると、`or`や`and`、`/`、改行コード、引用符といった文字列がWAFにひっかかってブロックされるパターンだった。


```
# これは通る
https://image.pollinations.ai/prompt/book%20cover%20dark%20theme

# これは404になる
https://image.pollinations.ai/prompt/AI副業の本・表紙デザイン

```

英語プロンプトだけ使うなら速くて無料なので使い道はある。KDP表紙の日本語タイトル入りには向かない——カスケードパイプラインの3番手に置いてあるが、実際に起動することはほとんどない。


## Midjourney：品質は認めるが自動化できない

画像の完成度で言えば5ツール中一番かもしれない。ただしDiscordのコマンドで動かす仕様なので、自動化パイプラインに組み込む手段がない。サードパーティのAPIが存在するが安定性に問題があって本番運用には使えなかった。


月10ドル以上を払って手動で生成する前提なら選択肢に入る——が、夜間の無人稼働が前提の私の用途とは根本的に合わない。日本語テキストの精度も英語に比べて落ちる印象で、表紙デザインにおける採用理由が見つからなかった。


## 夜間無人稼働で実際に動かしているカスケード構成

4段カスケードの実装を抜粋する。Gemini→DALL-E→Pollinations→SDの順で、成功したら止まる設計。どこかが落ちていても次が拾う。


```python
# コピペ可 / image_gen_cascade.py 抜粋
PROVIDERS = ["gemini", "dalle", "pollinations", "stable_diffusion"]

def generate_cover(title: str, author: str) -> str:
    prompt_en = build_english_prompt(title, author)

    for provider in PROVIDERS:
        try:
            path = generate_with_provider(provider, prompt_en)
            if path:
                print(f"[{provider}] 生成成功: {path}")
                return path
        except Exception as e:
            print(f"[{provider}] 失敗: {e}, 次のプロバイダへ")

    raise RuntimeError("全プロバイダ失敗")

def build_english_prompt(title: str, author: str) -> str:
    return (
        f'Professional ebook cover, dark minimalist design, '
        f'Japanese text "{title}" at the top, '
        f'author name "{author}" at the bottom, '
        f'abstract tech background, high quality'
    )

```

Geminiが落ちていてもDALL-Eで拾える。DALL-Eのクレジットが切れていてもPollinationsが英語テキスト版を出す。どこかが通る構成にしておくと、夜間の無人稼働で止まらない——これが今の安定稼働の実態。


## よくある質問

### 日本語テキストを画像に入れる一番確実な方法は？

Geminiに「英語プロンプト+日本語テキストを文字列として明示」するパターンが今のところ最も安定している。それでも崩れる場合は、Pillowで後からテキストを描画するハイブリッド方式に切り替える。この2段構えで詰まったことは今のところない。


### Gemini WebをCDP経由で操作するのは規約的に大丈夫？

グレーゾーン、というのが正直なところ。明示的に禁止されているわけではないが、商用利用の場合はGemini API（REST）を使う方が安全。APIは課金が発生する。利用規約は自分で確認してほしい。


### KDP表紙の推奨解像度は？

KDPの推奨は縦2560px×横1600px、縦横比1.6:1、72DPI以上。DALL-E 3のデフォルト出力（1024×1024）は正方形なので、KDP向けにはリサイズか縦長プロンプトの指定が必要。


### Stable DiffusionのLoRAで日本語テキストに対応できる？

私が試した範囲では難しかった。文字LoRAは英語アルファベットに最適化されているものが多く、日本語特化のLoRAは品質のばらつきが大きい。Pillowで合成する方が確実で、実装も単純。


### Pollinationsの404を回避する方法は？

英語プロンプトだけ使うのが根本的な解決。日本語・記号・改行を全部URLエンコードしても通らない場合は、ゼロ幅スペース（U+200B）をWAFに引っかかる文字の後に挿入するバイパス手法があるが、継続的に使えるかは保証できない。英語だけに絞る方が現実的。


## まとめ

5ツールを実際に叩いた結論として、日本語テキスト入り画像生成はGeminiが一択だった。プロンプトを英語で書いて日本語テキストを文字列として明示する形式が文字化け防止に効く。それでも崩れる場合はPillowで後から合成するハイブリッドで対処できる。


自動化パイプラインには4段カスケード構成が安定する。DALL-E 3はAPI自動化に向くが、日本語テキストはPillow合成と組み合わせる方が現実的。Pollinationsは英語限定と割り切れば使い道があるが、日本語を含むURLを投げた瞬間に信頼性が落ちる。Midjourneyは品質を認めつつも、自動化目的には根本的に向かない。


この構成で夜間の無人稼働が続いている。表紙の生成で詰まっている場合の参考になれば。


この記事を読んだあなたに：

– KDP自動出版パイプラインの全体設計を解説 #

– Brave CDP接続でPlaywrightセッション切れを根絶した方法 #

– image_gen_cascade.py の実装全文と設定方法 #


📘 この記事のテーマをさらに深掘りした本
### [Claude Codeで副業自動化——月5万円を目指した全記録](https://www.amazon.co.jp/dp/B0GX2ZV7DP?tag=taito2525-22)

非エンジニアが1年で月10万円に到達した全パイプラインと失敗談


[Kindleで読む →](https://www.amazon.co.jp/dp/B0GX2ZV7DP?tag=taito2525-22)


一ノ瀬泰斗
AI自動化エンジニア / Python個人開発者
Claude Code × Ollama × launchd で SNS・ブログ・KDPを全自動化。実測データと失敗談を軸に、月5万円収益化のリアルな記録を発信中。


💬 **自動化の相談・小規模受託も受付中**：「launchd で毎朝 AI が動く仕組みを作りたい」「KDP の自動出版を組みたい」など、[X (@taito_automate) の DM](https://x.com/taito_automate) からお気軽にどうぞ。


{"@context": "https://schema.org", "@type": "FAQPage", "mainEntity": [{"@type": "Question", "name": "日本語テキストを画像に入れる一番確実な方法は？", "acceptedAnswer": {"@type": "Answer", "text": "Geminiに「英語プロンプト+日本語テキストを文字列として明示」するパターンが今のところ最も安定している。それでも崩れる場合は、Pillowで後からテキストを描画するハイブリッド方式に切り替える。この2段構えで詰まったことは今のところない。"}}, {"@type": "Question", "name": "Gemini WebをCDP経由で操作するのは規約的に大丈夫？", "acceptedAnswer": {"@type": "Answer", "text": "グレーゾーン、というのが正直なところ。明示的に禁止されているわけではないが、商用利用の場合はGemini API（REST）を使う方が安全。APIは課金が発生する。利用規約は自分で確認してほしい。"}}, {"@type": "Question", "name": "KDP表紙の推奨解像度は？", "acceptedAnswer": {"@type": "Answer", "text": "KDPの推奨は縦2560px×横1600px、縦横比1.6:1、72DPI以上。DALL-E 3のデフォルト出力（1024×1024）は正方形なので、KDP向けにはリサイズか縦長プロンプトの指定が必要。"}}, {"@type": "Question", "name": "Stable DiffusionのLoRAで日本語テキストに対応できる？", "acceptedAnswer": {"@type": "Answer", "text": "私が試した範囲では難しかった。文字LoRAは英語アルファベットに最適化されているものが多く、日本語特化のLoRAは品質のばらつきが大きい。Pillowで合成する方が確実で、実装も単純。"}}, {"@type": "Question", "name": "Pollinationsの404を回避する方法は？", "acceptedAnswer": {"@type": "Answer", "text": "英語プロンプトだけ使うのが根本的な解決。日本語・記号・改行を全部URLエンコードしても通らない場合は、ゼロ幅スペース（U+200B）をWAFに引っかかる文字の後に挿入するバイパス手法があるが、継続的に使えるかは保証できない。英語だけに絞る方が現実的。"}}]}

---

この記事はもともと私のブログで書いたものです。より詳しい実装例・失敗談はこちらで読めます。

👉 [KDP表紙 日本語テキスト入り画像生成5ツール比較【2026年実測】](https://ichinose-taito.com/?p=5125)


