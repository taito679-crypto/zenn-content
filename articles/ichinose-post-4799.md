---
title: "Claude Code 使い方 完全ガイド 2026年版——インストールから実戦運用まで"
emoji: "✨"
type: "tech"
topics: ["claude", "python", "macos", "llm", "kindle"]
published: true
---

⏱この記事は約 7 分で読めます


📖 目次

- [📌 はじめに](#はじめに)
- [📌 Claude Code インストール方法と初期設定](#claude-code-インストール方法と初期設定)
- [📌 Claude Code の実用的な使い方——ファイル操作から自動化まで](#claude-code-の実用的な使い方ファイル操作から自動化まで)
- [📌 Claude Code で詰まりやすい3つのポイント](#claude-code-で詰まりやすい3つのポイント)
- [📌 Claude Code × Ollama のハイブリッド構成](#claude-code--ollama-のハイブリッド構成)
- [📌 Claude Code で実際に自動化した事例](#claude-code-で実際に自動化した事例)
- [📌 まとめ](#まとめ)
- [📌 関連記事](#関連記事)


最終更新: 2026-04-28


## はじめに

Claude Code を使い始めて約4ヶ月。最初の2週間で3回詰まり、1回は気づかぬうちに$23のAPIクレジットを溶かした。この記事はそういう記録だ。


公式ドキュメントを読めばインストールはできる。でも「どこで先に詰まるか」「Ollama と組み合わせるとどう変わるか」「CLAUDE.md に何を書けば使いやすくなるか」——これは実際に数ヶ月回してみないと見えてこない。私が失敗した箇所も含めて書いていく。


## Claude Code インストール方法と初期設定

### インストール

Node.js が入っていれば1行で終わる。


```bash
npm install -g @anthropic-ai/claude-code

```

起動は `claude` コマンド。初回だけ Anthropic アカウントとの認証が走る。ここは詰まらない。詰まったのはその後だ。


`.zshrc` に `ANTHROPIC_API_KEY` を設定したままにしておくと、Claude Code がサブスクリプションではなくAPIクレジットを使い始める。私は1週間気づかなかった。Sonnet 4.6 はトークン単価が高いので、少し使っただけでも額が跳ね上がる。


対処は `.zshrc` の1行をコメントアウトするだけ。


```bash
# export ANTHROPIC_API_KEY=sk-ant-...  ← コメントアウト

```

Claude Code はサブスクで動かす前提で設計されている。APIキーを環境変数に入れっぱなしにしているとただのAPI呼び出しになる——これ、意外と見落とす。


### 使えるモデルと使い分け


モデル
向いている作業


Opus（/fast）
設計・戦略・難しい判断


Sonnet 4.6
コード実装・記事生成・レビュー


Haiku
grep・ファイル整理・短い質問・確認作業


タブを3枚開いて「左=Opus・中=Sonnet・右=Haiku」で使い分けている。Haiku で済む作業を Sonnet に投げると、トークン消費が体感で3倍になる。この差は積み重なると月末にじわじわ効いてくる。


## Claude Code の実用的な使い方——ファイル操作から自動化まで

### プロジェクトディレクトリで起動する


```bash
cd ~/Documents/AI_Automation_Base
claude

```

これだけで、そのディレクトリのコードを全部読んだ状態で会話が始まる。「このファイルを読んでから」という前置きが要らない。


### CLAUDE.md でコンテキストを固定する

プロジェクトルートに `CLAUDE.md` を置くと、毎回自動で読み込まれる。私の場合、プロジェクトの目的（AI自動化パイプライン）、使っているスタック（launchd / Ollama / Playwright）、やってほしいこと・やってほしくないこと、ファイル構成の説明——これを全部書き込んでいる。


これがあるとセッションをまたいでも文脈が復元される。「前回と違う方向に実装された」というストレスが激減した。逆に言うと、CLAUDE.md なしで使っている人は毎回同じ説明を繰り返しているはずだ。


### 実際に投げている指示の例


```
01_Scripts/sns/x_post.py のエラーを直して。
280文字オーバーになるケースがある。

```


```
blog_queue/ 以下の未投稿JSONを全部 WordPress に公開して。
REST APIは 04_Config/.env に書いてある。

```


```
inject_internal_links.py を作って。
公開記事一覧を取得→本文中の関連キーワードにリンクを挿入→更新するやつ。

```

指示は「何をしてほしいか」だけ。ファイルパスや処理の詳細は CLAUDE.md から拾ってくれる。


## Claude Code で詰まりやすい3つのポイント

### 1. 確認プロンプトが多すぎる問題

デフォルトはファイル書き込み・コマンド実行のたびに確認が入る。慣れるまでは安全でいいが、自動化のフローに組み込もうとすると邪魔になる。`.claude/settings.json` で許可リストを書けば解決する。


```json
{
  "permissions": {
    "allow": [
      "Bash(python3:*)",
      "Bash(cat:*)",
      "Bash(ls:*)"
    ]
  }
}

```

### 2. トークンが思ったより消える

実装まで丸投げすると、方向が違ったときに丸ごと直す羽目になる。「調査だけして、実装は確認が出てから」というフローに変えてから、無駄なトークン消費が減った。設計が固まる前に手を動かさせるのが一番もったいない。


### 3. モデルのデフォルトが重い

`/model` コマンドでセッション中にいつでも切り替えられる。grep やファイル確認は全部 Haiku に投げる習慣にしたら、月のトークン消費が体感で半分以下になった。最初は全部 Sonnet 4.6 に頼んでいたが、それは明らかにやりすぎだった。


## Claude Code × Ollama のハイブリッド構成

コスト重視の作業には Ollama（ローカルLLM）を使う。


```python
# ai_client.py で切り替えを一元管理
client = AIClient(prefer_claude=False)  # Ollama 優先モード
response = client.generate(prompt)

```

`prefer_claude=False` にすると全タスクが Ollama に流れる。Claude Code はコード生成・設計に使い、量産系（SNS投稿・要約・分類）は Ollama の qwen2.5:3b に任せる——それが今の棲み分けだ。MacBook Pro M5 なら qwen2.5:3b は軽快に動く。レスポンスは Sonnet 4.6 より落ちるが、SNS投稿文の量産程度なら十分だ。


タスク
使うモデル
理由


コード設計・実装
Claude Sonnet 4.6
精度が必要


SNS投稿文生成
Ollama qwen2.5:3b
量が多いのでコスト優先


品質チェック
Claude Haiku
判断の一貫性が必要


ログ解析・分類
Ollama qwen2.5:3b
ローカルで完結したい


## Claude Code で実際に自動化した事例

### WordPress ブログの一括更新

先日、ブログの整理を Claude Code に丸投げした。重複ページ（プライバシーポリシーが3本、お問い合わせが2本あった）の非公開化、テストドラフト（chunk-* / block-* / x という名前のゴミ）のゴミ箱移動、About ページのAI自動化エンジニア紹介への書き換え、内部リンク注入スクリプト（inject_internal_links.py）の作成と実行——全部まとめて「やって」の一言で動かした。


結果として公開記事が14本になり、AdSense の準備度スコアが78点まで上がった。これができるのは CLAUDE.md にコンテキストが全部入っているからで、それがなければファイルの意味を毎回説明する必要がある。


### ファイル数が多い整理作業


```bash
# Claude Code に投げた指示（実際のもの）
blog_queue/ の JSON を全部確認して、
タイトルが重複しているものを除外してから
WordPress に投稿して。
REST API の認証情報は .env にある。

```

10本の記事が自動公開された。手作業でやったら2時間かかる。実際の所要時間は23分だった。


## まとめ

Claude Code は「コードを書いてくれるツール」というより、「自分の代わりに作業してくれる専属エンジニア」として使う方が性に合っている。


CLAUDE.md にプロジェクトのコンテキストを書き込み、モデルを使い分け、Ollama とのハイブリッドで運用すれば、月のAPI費用を抑えながら自動化の範囲を広げられる。


インストールして「すごい」と思って終わる人が多い印象だ。でも本番で機能するのは、自分のプロジェクトに合った設定ファイルを育てた後——それだけは確かだ。まず `CLAUDE.md` を1つ書いてみることを勧める。それだけで、毎回同じ説明を繰り返す手間がなくなる。✨


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


- [MacにClaude Codeをインストールして初期設定するまでに詰まった話](https://ichinose-taito.com/mac%e3%81%abclaude-code%e3%82%92%e3%82%a4%e3%83%b3%e3%82%b9%e3%83%88%e3%83%bc%e3%83%ab%e3%81%97%e3%81%a6%e5%88%9d%e6%9c%9f%e8%a8%ad%e5%ae%9a%e3%81%99%e3%82%8b%e3%81%be%e3%81%a7%e3%81%ab%e8%a9%b0/)
- [MacでOllamaを動かすまでにハマった話—qwen3.5:9bが動いたときの感動を記録しておく](https://ichinose-taito.com/mac%e3%81%a7ollama%e3%82%92%e5%8b%95%e3%81%8b%e3%81%99%e3%81%be%e3%81%a7%e3%81%ab%e3%83%8f%e3%83%9e%e3%81%a3%e3%81%9f%e8%a9%b1-qwen3-59b%e3%81%8c%e5%8b%95%e3%81%84%e3%81%9f%e3%81%a8%e3%81%8d/)
- [ChatGPTとClaudeを使い分けるようになった話——自動化パイプラインで実感した判断基準](https://ichinose-taito.com/chatgpt%e3%81%a8claude%e3%82%92%e4%bd%bf%e3%81%84%e5%88%86%e3%81%91%e3%82%8b%e3%82%88%e3%81%86%e3%81%ab%e3%81%aa%e3%81%a3%e3%81%9f%e8%a9%b1-%e8%87%aa%e5%8b%95%e5%8c%96%e3%83%91/)

---

この記事はもともと私のブログで書いたものです。より詳しい実装例・失敗談はこちらで読めます。

👉 [Claude Code 使い方 完全ガイド 2026年版——インストールから実戦運用まで](https://ichinose-taito.com/?p=4799)

