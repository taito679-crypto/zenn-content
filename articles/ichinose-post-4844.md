---
title: "Claude Code 完全攻略ガイド——インストールから実戦運用・トラブル解決まで"
emoji: "✨"
type: "tech"
topics: ["claude", "python", "macos", "llm", "kindle"]
published: true
---

⏱この記事は約 3 分で読めます


📖 目次

- [📌 導入](#導入)
- [📌 Claude Code とは何か——他のコーディング AI との違い](#claude-code-とは何か他のコーディング-ai-との違い)
- [📌 インストールと初期設定](#インストールと初期設定)
- [📌 基本操作と必須コマンド](#基本操作と必須コマンド)


## 導入

「Claude Code って結局何ができるの?」「Cursor と何が違う?」「Ollama と組み合わせてコスト削減できる?」


この3つの疑問を持ってこのページを開いたなら、全部答えられる。私は Ollama / launchd / Claude Code を実際に本番運用して、稼いだり壊したり直したりを繰り返してきたので、「実際のところ」を書く。


## Claude Code とは何か——他のコーディング AI との違い

Cursor や GitHub Copilot は IDE に統合される。Claude Code は違う。ターミナルで `claude` と打つだけで起動する CLI エージェントだ。これが何を意味するかというと、SSH 先のサーバーでも、tmux の中でも、cron や launchd から呼び出すことでも動く。


「IDE を開かなくていい」というのは思ったより大きな差で、私は launchd で夜間に自動実行させているスクリプトも、Claude Code を使えば「コードを生成してファイルに書いて git commit まで完了させる」という一連の流れをノータッチで回せる。


Claude Code の強みは「ファイル操作 × コマンド実行 × AI 判断」を一つのエージェントが担えること。弱みはオフラインで動かないこと、そして放置すると API キューリントが溶けることだ。


## インストールと初期設定

macOS の場合、Homebrew と Node.js（v18 以上）が入っていれば大抵動く。Linux（Ubuntu/Debian 系）でも同じ手順が使える。


```bash
# Node.js バージョン確認
node -v  # v18 以上であること

# npm でグローバルインストール
npm install -g @anthropic-ai/claude-code

# インストール確認
claude --version

```

`permission denied` が出る場合は `npm config set prefix ~/.npm-global` をして PATH を通す方が安全だ。`sudo npm install -g` は環境を汚すのでやめた方がいい。


## 基本操作と必須コマンド

`claude` と打つと対話セッションが始まる。ここでは自然言語でファイルの読み書き・コードの生成・コマンド実行まで依頼できる。


### ワンショット実行（自動化向け）


```bash
# -p フラグでワンショット実行
claude -p "今日の日付をファイル名に含む作業ログを作成して"

# 特定ファイルへの指示
claude -p "このスクリプトの処理速度を計測して結果を report.md に書いて" --output-format json

```

`-p` フラグを使うと対話なしで実行して結果を返してくれる。launchd や cron と組み合わせるときはこちらを使う。


### スラッシュコマンド一覧


コマンド
用途


`/help`
ヘルプ表示


`/model`
使用モデルの切り替え


`/clear`
会話履歴リセット


`/compact`
コンテキスト圧縮


`/status`
現在の状態確認


`/mcp`
MCP サーバー接続確認

[…省略]


## インストールと初期設定

macOS の場合、Homebrew と Node.js（v18 以上）が入っていれば大抵動く。Linux（Ubuntu/Debian 系）でも同じ手順が使える。


```bash
# Node.js バージョン確認
node -v  # v18 以上であること

# npm でグローバルインストール
npm install -g @anthropic-ai/claude-code

# インストール確認
claude --version

```

`permission denied` が出る場合は `npm config set prefix ~/.npm-global` をして PATH を通す方が安全だ。`sudo npm install -g` は環境を汚すのでやめた方がいい。


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

👉 [Claude Code 完全攻略ガイド——インストールから実戦運用・トラブル解決まで](https://ichinose-taito.com/?p=4844)


