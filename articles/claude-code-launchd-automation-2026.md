---
title: "Claude Code × launchd で作った全自動AI収益化パイプライン——実測データ公開"
emoji: "✨"
type: "tech"
topics: ["claudecode", "launchd", "python", "ollama", "ai"]
published: true
---

## この記事でわかること

- Claude Code を「専属CTO」として使う具体的なワークフロー
- launchd で Python スクリプトを定期実行する設定方法
- Ollama ローカルLLM と Claude API のハイブリッド運用コスト
- 実際に稼働している自動化パイプラインの全体像

---

僕が今使っているシステムは、Claude Code が設計して launchd が動かしている。コードを書くのはほぼ Claude で、僕は方向性だけ決める。

## パイプライン全体像

```
launchd (Mac)
├── 6:30  ブログ記事生成 (Claude Sonnet → WordPress)
├── 7:00  X / BlueSky 投稿 (Ollama qwen3.5)
├── 12:30 note 記事生成・投稿
├── 20:30 SNS エンゲージメント自動化
└── 毎時  KDP 売上トラッキング
```

全部で 12 個の launchd job が動いている。

## launchd の設定——exit 78 を踏んだ話

最初は crontab を使っていた。Mac が スリープすると実行されないことに気づいたのは 3 日後。launchd に移行したら `exit 78` でハマった。

```xml
<!-- com.taito.blog-article-gen.plist -->
<key>StartCalendarInterval</key>
<dict>
    <key>Hour</key><integer>6</integer>
    <key>Minute</key><integer>30</integer>
</dict>
```

exit 78 の原因はほぼ環境変数。launchd は login shell を継承しない。`PATH` と `HOME` を明示的に書かないと Python も見つからない。

```xml
<key>EnvironmentVariables</key>
<dict>
    <key>PATH</key>
    <string>/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
    <key>HOME</key>
    <string>/Users/ichinosetaito</string>
</dict>
```

## Claude Code を CTO として使う

Claude Code のセッションは `~/.claude/projects/` に自動保存される。前のセッションを引き継いで「昨日の続き」から始められる。

実際のワークフロー:

1. 「ブログ記事生成スクリプトの queue カウントがバグってる」と伝える
2. Claude がコードを読んで原因を特定する（所要 30 秒）
3. 修正案を提示→即実装
4. launchd を reload して確認

自分でデバッグすると 2 時間かかった案件が 10 分で終わった。

## Ollama × Claude のコスト比較

| タスク | モデル | 月コスト概算 |
|--------|--------|-------------|
| SNS投稿生成 (量産) | Ollama qwen3.5:9b | ¥0 |
| ブログ記事 (品質重視) | Claude Sonnet | ¥800〜 |
| KDP本文 | Claude Sonnet | ¥500〜 |
| Discord bot | Ollama qwen2.5:3b | ¥0 |

Claude API は月 ¥2,000 以下に抑えて、質が必要なところだけ使う。量産は全部ローカル。

## 実際の収益（2026年4月時点）

- KDP: 販売開始直後（データ蓄積中）
- ブログ: AdSense 審査中
- note: 記事投稿開始済み

正直まだ大きな数字じゃない。ただシステムが動いていること自体に価値があると思っていて、コンテンツが積み上がる速度が人力の 10 倍になっている。

## コピペで使える launchd ロード手順

```bash
# コピペ可
# plist を配置後にこれを実行
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.taito.blog-article-gen.plist
launchctl list | grep taito  # 確認

# 即時テスト実行
launchctl kickstart -k gui/$(id -u)/com.taito.blog-article-gen
```

## よくある質問

### launchd と crontab どっちがいい？

Mac なら launchd 一択。スリープ復帰後の実行保証、ログ管理、環境変数の明示指定ができる。

### Ollama のモデルは何がおすすめ？

日本語なら qwen3.5:9b が今のところ一番バランスがいい。M5 MacBook Pro で 8B クラスなら 1 秒以下でレスポンスが返る。

### Claude Code は月いくらかかる？

Max プラン（$200/月）で使い放題。API クレジットとは別なので、スクリプト実行時に `ANTHROPIC_API_KEY` を環境変数から除外して API 消費を防ぐ実装が必要。

### GitHub と Zenn を連携するメリットは？

Markdown を push するだけで記事が公開される。CI/CD 感覚でコンテンツ管理できる。バージョン管理もできるので「前の版に戻したい」が 1 コマンドで済む。

### launchd job が動いているか確認する方法は？

```bash
# コピペ可
launchctl list | grep taito
# 数字が PID（0なら停止中、正の数なら稼働中）
```

## まとめ

- launchd は環境変数を必ず明示的に書く
- Claude Code はコードを書く相手じゃなく「一緒に考える CTO」として使う
- Ollama で量産、Claude で品質の二段構えがコスパ最強
- システムが動き続けることが一番の資産

この記事を読んだあなたに：
- [Ollama で動くおすすめモデル比較](#)
- [Mac launchd 完全ガイド——exit 78 の原因と対処法](#)
