---
title: "ChatGPTサブスクをCLIから使い切る：画像生成・Excel・ブラウザ操作の全実装"
emoji: "✨"
type: "tech"
topics: ["claude", "python", "macos", "llm", "kindle"]
published: true
---

# ChatGPTサブスクをCLIから使い切る：画像生成・Excel・ブラウザ操作の全実装


⏱この記事は約 13 分で読めます


📖 目次

- [📌 Codex CLIとは何か、なぜClaude Code環境で使うのか](#codex-cliとは何かなぜclaude-code環境で使うのか)
- [📌 gpt-image-2でKDP表紙とサムネを生成——失敗3回を超えて辿り着いた設定](#gpt-image-2でkdp表紙とサムネを生成失敗3回を超えて辿り着いた設定)
- [📌 browser-useで実際のブラウザを自動操作——できることとできないこと](#browser-useで実際のブラウザを自動操作できることとできないこと)
- [📌 Excel自動生成——Pythonとの組み合わせでレポート出力を完全自動化](#excel自動生成pythonとの組み合わせでレポート出力を完全自動化)
- [📌 ChatGPT Plus vs APIコスト：3ヶ月間の実測値](#chatgpt-plus-vs-apiコスト3ヶ月間の実測値)
- [📌 参考・出典](#参考・出典)
- [📌 よくある質問](#よくある質問)
- [📌 まとめ](#まとめ)


最終更新: 2026-04-29


ChatGPT Plusを契約しているのにブラウザでしか使っていない——その非効率に気づいたのは、月額$20を払い始めて3ヶ月後だった。


Claude Codeをメイン作業環境にしている私が発見したのは、「Codex CLIを経由すれば、gpt-image-2もbrowser-useもExcel生成も、全部ターミナル一本で動く」という事実だ。KDPの表紙16枚をまとめて生成し、Excelレポートを週次で自動出力し [KDP表紙日本語テキスト入り画像生成5ツール比較【2026年実測】](https://ichinose-taito.com/kdp%e8%a1%a8%e7%b4%99-%e6%97%a5%e6%9c%ac%e8%aa%9e%e3%83%86%e3%82%ad%e3%82%b9%e3%83%88%e5%85%a5%e3%82%8a%e7%94%bb%e5%83%8f%e7%94%9f%e6%88%905%e3%83%84%e3%83%bc%e3%83%ab%e6%af%94%e8%bc%83%e3%80%902026/)、BSkyへの投稿フォーム操作まで試した。


この記事でわかること:


- Codex CLIをClaude Code環境に組み込む最短手順
- gpt-image-2をCLIから呼んでKDP表紙・サムネを生成する方法
- browser-useでWebブラウザ操作を自動化するときの実際の制限 [Playwright Mac入門 2026——ブラウザ自動化を最速で動かすセットアップ](https://ichinose-taito.com/playwright-mac-%e5%85%a5%e9%96%80-2026-%e3%83%96%e3%83%a9%e3%82%a6%e3%82%b6%e8%87%aa%e5%8b%95%e5%8c%96%e3%82%92%e6%9c%80%e9%80%9f%e3%81%a7%e5%8b%95%e3%81%8b%e3%81%99%e3%82%bb%e3%83%83/)
- ExcelファイルをCLI経由で自動生成するPythonパターン
- ChatGPT Plus vs APIコスト——3ヶ月分の実測比較 [Claude API vs ChatGPT API コスト比較——自動化パイプライン3ヶ月の実測データ](https://ichinose-taito.com/claude-api-vs-chatgpt-api-%e3%82%b3%e3%82%b9%e3%83%88%e6%af%94%e8%bc%83-%e8%87%aa%e5%8b%95%e5%8c%96%e3%83%91%e3%82%a4%e3%83%97%e3%83%a9%e3%82%a4%e3%83%b33%e3%83%b6%e6%9c%88%e3%81%ae/)


## Codex CLIとは何か、なぜClaude Code環境で使うのか

↑ ChatGPTサブスクをCLIから使い倒す（gpt-image-2生成）（Codex gpt-image-2で生成）
### Codex CLIの立ち位置と私の使い方

[Codex CLIはOpenAIが公開しているターミナル上のAIエージェント](https://github.com/openai/codex)だ。単なるコード補完ツールではなく、ファイル読み書き・シェル実行・画像生成・ブラウザ操作・スプレッドシート操作まで、プラグイン経由で幅広く使える。


私の環境はClaude Codeをメインにしている。戦略立案・コードレビュー・長期設計はClaude Code（Sonnet/Opus）で、「これ試してきて」という実行系タスクにCodex CLIを使う分業になった。ClaudeはAnthropicのモデルで動いているので、OpenAI固有のgpt-image-2やbrowser-useは直接呼べない。そこをCodex CLIで補完する構成だ。


### CLI vs GUI vs 直接API——使い分けの実態


機能
GUI（ChatGPT）
Codex CLI
直接API呼び出し


gpt-image-2画像生成
○（手動）
○（自動化可）
○（コード要）


browser-use
△（限定的）
○（プラグイン）
✕


Excel生成
○（手動DL）
○（ファイル出力）
△（コード要）


バッチ処理
✕
○
○


launchd連携
✕
○
○


月次コスト感
$20固定
使った分（私は$5〜10）
使った分


インストールは `npm install -g @openai/codex` 一発。ただしOpenAIのAPIキーが必要で、ChatGPT Plusのサブスクとは「別に」API残高が要る点は注意が必要だ。


```bash
# インストールと認証
npm install -g @openai/codex
export OPENAI_API_KEY="sk-..."

# 動作確認
codex "現在のディレクトリの内容を教えて"

```

初回起動時、モデルを `gpt-4o-mini` に設定していたが、画像生成を使おうとした段階で「gpt-image-2はgpt-4o-miniでは使えない」というエラーが出た。モデル指定の落とし穴は次のセクションで詳しく書く。


## gpt-image-2でKDP表紙とサムネを生成——失敗3回を超えて辿り着いた設定

### 日本語プロンプトと引用符という2つの鬼門

[OpenAIの公式ドキュメント](https://platform.openai.com/docs/guides/images)によると、gpt-image-2は `dall-e-3` の後継として位置付けられており、より高解像度・より指示追従性が高い。Codex CLIからモデルを明示的に指定して使う。


私が最初に踏んだ失敗は2つある。


「日本語プロンプトを渡したら壊れた」のが1つ目だ。半分英語・半分日本語のプロンプトを入れたら、テキストが文字化けした画像が出てきた。Pollinations APIで同じ問題を踏んでいたので、画像生成系は英語プロンプト固定が正解と判断した。


「引用符の問題」が2つ目。プロンプトにダブルクォートを含めると、シェルがエラーを吐く。ヒアドキュメント方式に切り替えてから安定した。


```bash
# NG: 引用符がシェルに食われる
codex "generate image: book cover with the text "CLAUDE CODE" glowing"

# OK: ヒアドキュメントで渡す # コピペ可
codex exec "$(cat 
## browser-useで実際のブラウザを自動操作——できることとできないこと

↑ browser-use：画面遷移ゼロ（gpt-image-2生成）（画面遷移なし）（Codex gpt-image-2で生成）
### プラグインの使い方と典型的なユースケース

[browser-use](https://github.com/browser-use/browser-use)はPlaywrightをバックエンドにしたブラウザ自動化ライブラリで、Codex CLIのブラウザ操作プラグインとして機能する。


私が試したのは「BSkyの投稿フォームを操作して下書きを投稿する」というユースケースだ。Claude Code側でコンテンツを生成し、そのテキストをCodexに渡してブラウザ操作させる構成になる。


```bash
# browser-useを使った投稿自動化の例 # コピペ可
POST_TEXT=$(cat output/bsky_post_draft.txt)

codex exec "$(cat 
## Excel自動生成——Pythonとの組み合わせでレポート出力を完全自動化

↑ Excel自動生成ダッシュボード（gpt-image-2生成）（Codex gpt-image-2で生成）
### 2段ロケット方式の設計

Excel生成はCodex CLIが直接xlsxファイルを書き出すのではなく、「openpyxlを使ったPythonスクリプトを生成・実行する」という2段構成になる。Codexに直接「Excelを作れ」と言うと出力が不安定だった。「Pythonスクリプトを書いて実行しろ」という指示に変えてから安定した。


```python
# Codexが生成したExcelレポートスクリプトの実例 # コピペ可
import openpyxl
from openpyxl.styles import Font, PatternFill, Alignment
from datetime import datetime
import json

wb = openpyxl.Workbook()
ws = wb.active
ws.title = "月次KDPレポート"

headers = ["タイトル", "ASIN", "販売数", "ロイヤリティ(¥)", "更新日"]
for col, h in enumerate(headers, 1):
    cell = ws.cell(row=1, column=col, value=h)
    cell.font = Font(color="FFFFFF", bold=True)
    cell.fill = PatternFill("solid", fgColor="2C3E50")
    cell.alignment = Alignment(horizontal="center")

# データはJSONキューから読み込む（実際の構成）
with open("output/kdp_monthly_data.json") as f:
    records = json.load(f)

for rec in records:
    ws.append([
        rec["title"], rec["asin"], rec["units"],
        rec["royalty_jpy"], datetime.now().date()
    ])

ws.column_dimensions["A"].width = 30
filename = f"kdp_report_{datetime.now().strftime('%Y%m')}.xlsx"
wb.save(filename)
print(f"[DONE] {filename} を出力しました")

```

### launchd連携で週次レポートを自動化した結果

毎週月曜8:00に自動実行するlaunchd plistを設定した。設定後の実態は「月曜の朝にファイルが存在している」だけ——確認作業が消えた。


自動化の積み重ねについては[非エンジニアがAI開発で収益化するまでの1年ログ](https://ichinose-taito.com/%e9%9d%9e%e3%82%a8%e3%83%b3%e3%82%b8%e3%83%8b%e3%82%a2%e3%81%8cai%e9%96%8b%e7%99%ba%e3%81%a7%e5%8f%8e%e7%9b%8a%e5%8c%96%e3%81%99%e3%82%8b%e3%81%be%e3%81%a7%e3%81%ae1%e5%b9%b4%e3%83%ad%e3%82%b0/)にも書いたが、「1個の自動化で回収できる時間」を積み上げていくと、半年後に体感が変わる。


## ChatGPT Plus vs APIコスト：3ヶ月間の実測値

### 使用データから見えたコスト構造

ChatGPT Plusを$20/月で契約しながらAPIも使う、という二重コスト状態が2ヶ月続いた。実測値を並べると以下になる。


項目
ChatGPT Plus
Codex CLI（API）
備考


基本料金
$20/月
$0
APIは従量課金


画像16枚生成
込み
$1.84
gpt-image-2実測


テキスト生成
込み
$2〜5/月
用途次第


ファイル/Excel操作
込み
$0.5〜1/月
実行回数次第


ブラウザ操作
✕（未対応）
$1〜3/月
操作量次第


月額合計感
$20固定
$5〜10（私の用途）
—


ヘビーユーザーならPlusの方が安い。私の使い方（1日10〜20回の生成）ではAPIの方が明らかに安かった。Plusは3ヶ月で解約した。


ただし「UIでサッと試したい」場面ではGUIの方が速い。CLI化は「繰り返す処理」に特化する——これが今の結論だ。


## 参考・出典


- [Codex CLI 公式リポジトリ（OpenAI / GitHub）](https://github.com/openai/codex) — インストール方法・プラグイン一覧・設定ファイルの構造はここが一次情報
- [OpenAI Images API ドキュメント](https://platform.openai.com/docs/guides/images) — 公式では gpt-image-2 のサポートモデル・解像度・フォーマット指定が詳しく書いてある
- [browser-use 公式リポジトリ（GitHub）](https://github.com/browser-use/browser-use) — Playwright連携の設定・対応ブラウザ・エラーパターンはここで確認した


## よくある質問

### Q: ChatGPT PlusのサブスクはCLI自動化に必要ですか？

必要ない。Codex CLIはOpenAIのAPIキーで動く。ChatGPT PlusはGUIのWebサービスなので、CLI自動化にはAPIの残高だけあればいい。Plusを解約してAPIだけにした方がコストが下がるケースが多い。


### Q: Claude Code（Anthropic）だけではなく、なぜCodex CLIを追加で使うのですか？

Claude CodeはAnthropicのモデルで動いている。gpt-image-2はOpenAIのモデルなので、Claude Codeから直接は呼べない。OpenAI固有の機能を使いたいとき、Codex CLIを呼び出す形で補完する構成にしている。


### Q: browser-useはBraveブラウザ以外でも動きますか？

動く。ChromiumベースのブラウザならPlaywrightが対応している。ただし私の環境はBraveに統一しているので、Chrome・Edge での安定性は未確認だ。


### Q: ExcelではなくGoogleスプレッドシートに直接書き込む方法はありますか？

`gspread` ライブラリを使ったPythonスクリプトをCodexに生成させる方法で対応できる。GoogleのOAuth認証が必要になるため初回セットアップに30分ほどかかったが、一度通れば以降は完全自動だ。


### Q: Apple Silicon（M5）のMacでCodex CLIは動きますか？

動く。M5環境で問題なく動作している。Node.jsのバージョンは18以上が必要で、nvm経由でインストールするのが安定する。


## まとめ


- Codex CLIはChatGPT Plusとは別で、OpenAI APIキーがあればCLI自動化に使える
- gpt-image-2は英語プロンプト・ヒアドキュメント方式で安定する（日本語・引用符は鬼門）
- browser-useはDOM変更に弱く、週1回のセレクタ確認が現実的なコストとして発生する
- Excel生成はopenpyxl+Pythonスクリプトの2段ロケット方式が安定する
- 「繰り返す処理」に絞ればAPIはPlusより安い（私の場合は月$5〜10）


関連記事:

– [非エンジニアがAI開発で収益化するまでの1年ログ](https://ichinose-taito.com/%e9%9d%9e%e3%82%a8%e3%83%b3%e3%82%b8%e3%83%8b%e3%82%a2%e3%81%8cai%e9%96%8b%e7%99%ba%e3%81%a7%e5%8f%8e%e7%9b%8a%e5%8c%96%e3%81%99%e3%82%8b%e3%81%be%e3%81%a7%e3%81%ae1%e5%b9%b4%e3%83%ad%e3%82%b0/) — 非エンジニアがどうやってAI自動化で収益化まで辿り着いたかを時系列で書いた

– [朝の「状況把握5分」をゼロにするDiscord Bot通知の作り方【Python/launchd】](https://ichinose-taito.com/%e6%9c%9d%e3%81%ae%e3%80%8c%e7%8a%b6%e6%b3%81%e6%8a%8a%e6%8f%a15%e5%88%86%e3%80%8d%e3%82%92%e3%82%bc%e3%83%ad%e3%81%ab%e3%81%99%e3%82%8bdiscord-bot%e9%80%9a%e7%9f%a5%e3%81%ae%e4%bd%9c%e3%82%8a/) — launchd×Python×Discordで朝の確認作業を完全に消した実装例


```json


```


一ノ瀬泰斗
AI自動化エンジニア / Python個人開発者
Claude Code × Ollama × launchd で SNS・ブログ・KDPを全自動化。実測データと失敗談を軸に、月5万円収益化のリアルな記録を発信中。


💬 **自動化の相談・小規模受託も受付中**：「launchd で毎朝 AI が動く仕組みを作りたい」「KDP の自動出版を組みたい」など、[X (@taito_automate) の DM](https://x.com/taito_automate) からお気軽にどうぞ。


{"@context": "https://schema.org", "@type": "FAQPage", "mainEntity": [{"@type": "Question", "name": "Q: ChatGPT PlusのサブスクはCLI自動化に必要ですか？", "acceptedAnswer": {"@type": "Answer", "text": "必要ない。Codex CLIはOpenAIのAPIキーで動く。ChatGPT PlusはGUIのWebサービスなので、CLI自動化にはAPIの残高だけあればいい。Plusを解約してAPIだけにした方がコストが下がるケースが多い。"}}, {"@type": "Question", "name": "Q: Claude Code（Anthropic）だけではなく、なぜCodex CLIを追加で使うのですか？", "acceptedAnswer": {"@type": "Answer", "text": "Claude CodeはAnthropicのモデルで動いている。gpt-image-2はOpenAIのモデルなので、Claude Codeから直接は呼べない。OpenAI固有の機能を使いたいとき、Codex CLIを呼び出す形で補完する構成にしている。"}}, {"@type": "Question", "name": "Q: browser-useはBraveブラウザ以外でも動きますか？", "acceptedAnswer": {"@type": "Answer", "text": "動く。ChromiumベースのブラウザならPlaywrightが対応している。ただし私の環境はBraveに統一しているので、Chrome・Edge での安定性は未確認だ。"}}, {"@type": "Question", "name": "Q: ExcelではなくGoogleスプレッドシートに直接書き込む方法はありますか？", "acceptedAnswer": {"@type": "Answer", "text": "gspread  ライブラリを使ったPythonスクリプトをCodexに生成させる方法で対応できる。GoogleのOAuth認証が必要になるため初回セットアップに30分ほどかかったが、一度通れば以降は完全自動だ。"}}, {"@type": "Question", "name": "Q: Apple Silicon（M5）のMacでCodex CLIは動きますか？", "acceptedAnswer": {"@type": "Answer", "text": "動く。M5環境で問題なく動作している。Node.jsのバージョンは18以上が必要で、nvm経由でインストールするのが安定する。"}}]}

---

この記事はもともと私のブログで書いたものです。より詳しい実装例・失敗談はこちらで読めます。

👉 [ChatGPTサブスクをCLIから使い切る：画像生成・Excel・ブラウザ操作の全実装](https://ichinose-taito.com/?p=5433)

