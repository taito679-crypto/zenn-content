---
title: "ChatGPT自動化が壊れる本当の理由：Brave CDP 移行で解決した実録"
emoji: "✨"
type: "tech"
topics: ["claude", "python", "macos", "llm", "kindle"]
published: true
---

最終更新: 2026-04-28


# ChatGPT自動化が壊れる本当の理由：Brave CDP 移行で解決した実録


⏱この記事は約 6 分で読めます


📖 目次

- [📌 ChatGPT 生成コードが自動化で詰まる3つの構造的な問題](#chatgpt-生成コードが自動化で詰まる3つの構造的な問題)
- [📌 headless Chromium vs Brave CDP：どちらを選ぶか](#headless-chromium-vs-brave-cdpどちらを選ぶか)
- [📌 実際に note 自動投稿が壊れていた経緯](#実際に-note-自動投稿が壊れていた経緯)
- [📌 Brave CDP 移行で実際に書いたコード](#brave-cdp-移行で実際に書いたコード)
- [📌 よくある質問](#よくある質問)
- [📌 まとめ](#まとめ)


「ChatGPT に頼んだコード、最初は動いたのにいつの間にか壊れてる」——この経験、ある人は多いと思う。


僕は 107 日間、note への投稿を手動でコピペし続けた。スクリプトは書いた。[Playwright](https://playwright.dev/python/) のコードも何度も書いた。そのたびに「ログインに失敗しました」「公開済みか判定できず」でコケた。原因がわかったのは 4 ヶ月後。クッキーの有効期限と、UI の細かい挙動変更という、ChatGPT には「見えない」問題だった。


この記事でわかること：

– ChatGPT が生成した自動化コードが壊れやすい本当の理由

– headless Chromium と Brave CDP の違いと使い分け

– note.com で実際に起きた UI 変更とその検出方法

– Brave CDP に移行してセッション切れを根治した手順

– コピペで動く Playwright + CDP 接続テンプレ


## ChatGPT 生成コードが自動化で詰まる3つの構造的な問題

ChatGPT はコードを書くのが得意だ。でも「セッションを維持する」「UI 変更を検知する」「スクリプトを長期間壊れないまま運用する」は、コード生成とは別の問題だ。


この 3 つが噛み合わないと、自動化は月単位で壊れ続ける。


### headless Chromium を使うとセッションが消える

ChatGPT に Playwright のコードを書かせると、ほぼ確実に *（略）* が出てくる。これ自体は悪くない。ただ、*（略）* や *（略）* を指定してセッションを保存する方式は、[クッキーの有効期限に完全に依存](https://playwright.dev/python/docs/api/class-browsercontext#browser-context-storage-state)する。


実際に僕の *（略）* を調べたら、21 個のクッキーのうち 15 個が期限切れだった。


（コード例は元記事を参照）


headless で起動するたびに「既存セッションを使う」つもりが、実態は「腐ったクッキーで接続しようとして失敗」になっていた。


### UI 変更をコードは知らない

2026 年 4 月、note.com の URL パターンが変わった。


以前は下書きプレビューが *（略）* だったのが、[SvelteKit](https://kit.svelte.dev/docs/introduction) 移行後は *（略）* になった。僕のスクリプトはこの URL を「公開成功」と誤検出していた。4 ヶ月間、投稿してるつもりで下書きに積み上げていた。


ChatGPT はこの変更を知らない。学習データのカットオフより後の話だから。


### セッション切れの「根治」方法を知らない

「ログインし直すコードを書いて」と ChatGPT に頼めば書いてくれる。でも 2 段階認証、reCAPTCHA、サービスごとのセッション管理の癖は、何度 prompt を調整しても完全には埋まらない。


## headless Chromium vs Brave CDP：どちらを選ぶか


項目
headless Chromium
Brave CDP


セッション管理
storage_state ファイル
[Chrome DevTools Protocol](https://chromedevtools.github.io/devtools-protocol/) でブラウザセッションを共有


ログイン維持
クッキー期限に依存
手動ログイン一回で半永久維持


セッション切れ対応
コードで再ログイン実装
Brave で手動ログインするだけ


UI 変更への対応
テスト・修正が必要
同じく必要（ここは変わらない）


初期設定コスト
低い（pip install だけ）
Brave のデバッグポート起動が必要


長期安定性
低い
高い


Brave CDP は Brave ブラウザをデバッグモードで起動して、その中のセッションを Playwright で操作する方式だ。普段 Brave でログインしている Amazon や note や X のセッションをそのままスクリプトが使える。クッキー期限が切れても、Brave を開いてログインし直せば終わり。


## 実際に note 自動投稿が壊れていた経緯

107 日間の時系列を整理するとこうなる。


（コード例は元記事を参照）


根本原因は 2 つ同時に起きていた。クッキー腐敗と URL 誤検出。どちらか片方だけ直しても動かなかったと思う。


## Brave CDP 移行で実際に書いたコード

### Brave をデバッグモードで起動する

（コード例は元記事を参照）


これを launchd に登録して、Mac 起動時に自動で立ち上がるようにしてある。


### CDP 接続の共通ライブラリ（コピペ可）

（コード例は元記事を参照）


### note の URL 誤検出を修正した部分

（コード例は元記事を参照）


[テキストロケーター](https://playwright.dev/python/docs/locators#text-locator)（*（略）*）が SvelteKit 移行後から動かなくなっていた。座標指定は不安定だけど、今は動いている。


## よくある質問

### Brave CDP って Brave ブラウザが起動してないと動かないの？

動かない。これが最大のデメリットで、Mac がスリープすると Brave が落ちることがある。launchd で *（略）* を設定して、落ちたら自動再起動させている。


### storage_state 方式はもう使えない？

note.com や X には使わない。ログイン維持が必要なサービスは全部 Brave CDP に統一した。API キーで認証するサービス（Bluesky など）は storage_state 不要なので関係ない。


### ChatGPT でコードを書くのをやめたの？

やめてない。構造の設計や「こういう処理をどう書くか」の初稿は ChatGPT や Claude で書く。ただ「このコードをそのまま本番で長期運用できるか」は別の話だ、という感覚がついた。


### テキストロケーターが動かない時の対処は？

DevTools でボタンの座標を調べて *（略）* で押す。ベストプラクティスではないけど、動く。aria-label や data-testid がついていれば *（略）* が安定する。


### 107 日間気づかなかった理由は？

手動確認を「たまにやる」から「ほぼやらない」にシフトしたタイミングが悪かった。自動化したつもりで放置していた。定期的に「本当に動いているか」を確認するモニタリングが必要だった。


## まとめ


- ChatGPT 生成の自動化コードが壊れる主因は「クッキー期限切れ」と「UI 変更」の2つ
- headless Chromium + storage_state はセッション維持が弱い。長期運用には向かない
- Brave CDP は普段使いブラウザのセッションを流用するため、ログイン切れが起きにくい
- note.com の URL パターン変更（SvelteKit 移行）を誤検出していた。実際に URL を print して確認するのが確実
- テキストロケーターが動かない場合、座標クリックは迂回策として有効
- 「動いている」の確認は月に1回でいいので必ずやる


この記事を読んだあなたに：

– Playwright + Brave CDP の環境構築手順（Mac編） #

– launchd でスクリプトを自動起動する設定テンプレ #

– note 自動投稿スクリプトの全コード公開 #


（コード例は元記事を参照）


p>


📘 この記事のテーマをさらに深掘りした本
### [Claude Codeで副業自動化——月5万円を目指した全記録](https://www.amazon.co.jp/dp/B0GX2ZV7DP?tag=taito2525-22)

非エンジニアが1年で月10万円に到達した全パイプラインと失敗談


[Kindleで読む →](https://www.amazon.co.jp/dp/B0GX2ZV7DP?tag=taito2525-22)


一ノ瀬泰斗
AI自動化エンジニア / Python個人開発者
Claude Code × Ollama × launchd で SNS・ブログ・KDPを全自動化。実測データと失敗談を軸に、月5万円収益化のリアルな記録を発信中。


💬 **自動化の相談・小規模受託も受付中**：「launchd で毎朝 AI が動く仕組みを作りたい」「KDP の自動出版を組みたい」など、[X (@taito_automate) の DM](https://x.com/taito_automate) からお気軽にどうぞ。


{"@context": "https://schema.org", "@type": "FAQPage", "mainEntity": [{"@type": "Question", "name": "Brave CDP って Brave ブラウザが起動してないと動かないの？", "acceptedAnswer": {"@type": "Answer", "text": "動かない。これが最大のデメリットで、Mac がスリープすると Brave が落ちることがある。launchd で  （略）  を設定して、落ちたら自動再起動させている。"}}, {"@type": "Question", "name": "storage_state 方式はもう使えない？", "acceptedAnswer": {"@type": "Answer", "text": "note.com や X には使わない。ログイン維持が必要なサービスは全部 Brave CDP に統一した。API キーで認証するサービス（Bluesky など）は storage_state 不要なので関係ない。"}}, {"@type": "Question", "name": "ChatGPT でコードを書くのをやめたの？", "acceptedAnswer": {"@type": "Answer", "text": "やめてない。構造の設計や「こういう処理をどう書くか」の初稿は ChatGPT や Claude で書く。ただ「このコードをそのまま本番で長期運用できるか」は別の話だ、という感覚がついた。"}}, {"@type": "Question", "name": "テキストロケーターが動かない時の対処は？", "acceptedAnswer": {"@type": "Answer", "text": "DevTools でボタンの座標を調べて  （略）  で押す。ベストプラクティスではないけど、動く。aria-label や data-testid がついていれば  （略）  が安定する。"}}, {"@type": "Question", "name": "107 日間気づかなかった理由は？", "acceptedAnswer": {"@type": "Answer", "text": "手動確認を「たまにやる」から「ほぼやらない」にシフトしたタイミングが悪かった。自動化したつもりで放置していた。定期的に「本当に動いているか」を確認するモニタリングが必要だった。"}}]}

---

この記事はもともと私のブログで書いたものです。より詳しい実装例・失敗談はこちらで読めます。

👉 [ChatGPT自動化が壊れる本当の理由：Brave CDP 移行で解決した実録](https://ichinose-taito.com/?p=5142)

