---
title: "Mac で使える AI ツール 比較 2026｜個人開発者が実際に使い倒した結果"
emoji: "✨"
type: "tech"
topics: ["claude", "python", "macos", "llm", "kindle"]
published: true
---

⏱この記事は約 7 分で読めます


📖 目次

- [📌 この記事でわかること](#この記事でわかること)
- [📌 私の環境と前提](#私の環境と前提)
- [📌 クラウド AI ツール 実運用比較](#クラウド-ai-ツール-実運用比較)
- [📌 Ollama ローカル LLM への切り替え戦略](#ollama-ローカル-llm-への切り替え戦略)
- [📌 Mac × launchd で AI を常時稼働させる](#mac--launchd-で-ai-を常時稼働させる)
- [📌 2026 実測に基づく選定チェックリスト](#2026-実測に基づく選定チェックリスト)
- [📌 まとめ：ハイブリッド運用で月 ¥5,000 削減](#まとめハイブリッド運用で月-5000-削減)


最終更新: 2026-04-28


## この記事でわかること

毎日 Mac で 4 つの AI ツールを並行稼働させている。Claude Code、Ollama、ChatGPT Plus、GitHub Copilot——それぞれ役割が違う。「一番いい AI を 1 本選べばいい」という発想でやっていた時期があって、そのときは月 ¥15,000 を Anthropic API だけに溶かしていた。今は ¥5,000 だ。この記事は、その差が何で生まれたかをそのまま書いたものだ。


## 私の環境と前提

MacBook Pro M5 32GB。SNS 投稿・ブログ記事・KDP 出版・Discord 通知を Python + launchd で全自動化している。コードを書くのは Claude Code 任せで、私は指示文を入力するだけだ。


今月の実コストは Claude Pro ¥3,200 + Anthropic API 従量課金 ¥1,800 [Claude API と ChatGPT API の実測コスト比較](https://ichinose-taito.com/claude-api-vs-chatgpt-api-%e3%82%b3%e3%82%b9%e3%83%88%e6%af%94%e8%bc%83-%e8%87%aa%e5%8b%95%e5%8c%96%e3%83%91%e3%82%a4%e3%83%97%e3%83%a9%e3%82%a4%e3%83%b33%e3%83%b6%e6%9c%88%e3%81%ae/) で合計 ¥5,000。半年前は同じ処理量で月 ¥15,000 を超えていた。変えたのは Ollama の導入だけで、パイプラインの構造は触っていない。削れる部分は最初から決まっていた、と後になってわかった。


## クラウド AI ツール 実運用比較

外部 API・サービス系の実測値をまとめた。


ツール
月額
Mac 対応
主な用途
実測応答速度


[Claude Code](https://claude.ai)（Pro）
¥3,200
ターミナル/Web
コーディング、長文生成
初期 3 秒、ストリーミング


[ChatGPT](https://openai.com/chatgpt)（Plus）
$20
Web/アプリ
汎用、画像生成
初期 2 秒


[Gemini Advanced](https://gemini.google.com)
$19
Web
検索連携、テーブル分析
初期 4 秒


[GitHub Copilot](https://github.com/features/copilot)
$10
VSCode 拡張
コード補完のみ
補完 0.5 秒


[Cursor](https://www.cursor.com)
$20
アプリ
コーディング、エージェント
初期 2 秒


Claude Code をメインに使い始めてから、他のコーディング支援ツールはほぼ触らなくなった。指示を出したら完結まで全部やってくれる——ファイル修正、テスト実行、git コミット、全部だ。先月、450 行の Python スクリプト修正を「このファイルを修正して」の一行で頼んだ。12 分後に完了していた。修正内容の確認、エラー対応、再テストまで Claude が回した。Copilot で同じことをやると、私が手動で対応するぶんだけで 1 時間かかる計算になる。


Cursor も 2 週間試した。機能は悪くない。ただ Claude Code とコストが完全に被っている。どちらか一方を選ぶなら Claude Code だ——ターミナルで完結するから、エディタを開く必要すらない。


## Ollama ローカル LLM への切り替え戦略

[Ollama](https://ollama.ai) は Mac 上で LLM をローカル実行するためのツールだ。M5 32GB なら 9B パラメータのモデルが普通に動く。API コストはゼロ。その代わり、生成速度と品質でクラウド API に劣る。


先月、本格導入した。モデルは Qwen2.5:9b にした——3b・7b と比較して日本語の出力が体感で一段上だったからだ。実測で SNS 投稿文の生成が 1 トークン/秒前後。Claude Haiku が 100 トークン/秒近く出るので、100 倍遅い。ただし月額ゼロ。


```bash
brew install ollama
ollama run qwen2.5:9b
```

運用では `ai_config.yaml` の `prefer_claude: true/false` を切り替えるだけで、全パイプラインの LLM を差し替えられる構成にした。Claude API が落ちたときや月額を節約したい局面で、1 秒で Ollama に移行できる。


タスク
使うモデル
生成時間
理由


SNS 投稿文
Claude Haiku
3 秒
短文、高品質が必須


ブログ記事・KDP
Claude Sonnet
12 秒
長文、構成力が必須


YouTube 台本
Claude Sonnet
15 秒
構成、推敲が必須


定型レポート
Ollama Qwen2.5:9b
90 秒
テンプレート化可、コスト重視


データ整形
Ollama Qwen2.5:9b
60 秒
ルール単純、コスト重視


最初の 2 週間、コスト削減に気を取られて全タスクを Ollama に投げた。SNS 投稿文もブログ記事の要約も、全部 Qwen2.5:9b 一本。結果、BSky のいいね率が 20% 落ちた——3 日間気づかなかった。ログを掘ったら原因は単純で、Qwen が「この方法で〜できます。この方法で〜できます。」みたいな機械的な繰り返しを出力していた。SNS 系だけすぐ Claude Haiku に戻した。データで確認するまでわからなかった、というのが正直なところだ。


## Mac × launchd で AI を常時稼働させる

毎日決まった時間に AI 処理を回す基盤として、Mac の launchd を使っている。朝 7:00 に SNS 投稿文 5 本を生成する——所要約 5 分。昼 12:30 はブログ記事 1 本の書き下ろしで 18 分。夜 20:30 は KDP 1 章の完成と品質スコア算出で 12 分。この 3 本が毎日自動で回ることで、手動作業を月換算 40 時間削れた。


plist の基本構成はこれだ。


```xml


  Label
  com.user.ai-task
  ProgramArguments
  
    /opt/homebrew/bin/python3
    /path/to/generate_content.py
  
  StartCalendarInterval
  
    Hour
    7
    Minute
    0
  


```

ひとつ詰まった点がある。launchd が実行するとき、通常のシェル環境が引き継がれない。`python3` と書いたままだと「command not found」で静かに死ぬ。2 週間、スクリプトが一度も動いていなかった——`~/Library/Logs/` のログファイルを確認するまで気づかなかった。原因は Python3 の絶対パスを指定していなかっただけだ。Apple Silicon Mac なら `/opt/homebrew/bin/python3` を入れる。


## 2026 実測に基づく選定チェックリスト

ツールを選ぶ前に、自分のタスクをどれに当てはめるかだけ考えればいい。


- 品質最優先のタスク（SNS 投稿、KDP、ブログ）→ Claude Sonnet または Haiku
- 速度最優先のタスク（数秒以内に結果が必要）→ Ollama のメリットなし、Claude 一択
- コスト最優先でテンプレート化できるタスク（定型レポート、データ整形）→ Ollama Qwen2.5:9b
- IDE でのコード補完だけが欲しい → GitHub Copilot
- ターミナルから自律的にコードを書かせたい → Claude Code


## まとめ：ハイブリッド運用で月 ¥5,000 削減

月 ¥15,000 が ¥5,000 になった。やったことは Ollama の導入と、タスクの分類だけだ。


品質の低下は実測 2% 程度——SNS のいいね率も、KDP 販売数も動かなかった。つまり、最初から削れる部分があったということだ。テンプレート化できる定型処理をクラウド API に投げ続けていたのが無駄だった。気づくまでに半年かかった。


ツールを選ぶ前に、自分のタスクを「品質必須」「速度必須」「コスト最優先」の 3 つに分けてほしい。分類が終わったら、ツールの割り当てはほぼ自動で決まる。


一ノ瀬泰斗
AI自動化エンジニア / Python個人開発者
Claude Code × Ollama × launchd で SNS・ブログ・KDPを全自動化。実測データと失敗談を軸に、月5万円収益化のリアルな記録を発信中。


💬 **自動化の相談・小規模受託も受付中**：「launchd で毎朝 AI が動く仕組みを作りたい」「KDP の自動出版を組みたい」など、[X (@taito_automate) の DM](https://x.com/taito_automate) からお気軽にどうぞ。

---

この記事はもともと私のブログで書いたものです。より詳しい実装例・失敗談はこちらで読めます。

👉 [Mac で使える AI ツール 比較 2026｜個人開発者が実際に使い倒した結果](https://ichinose-taito.com/?p=4801)


