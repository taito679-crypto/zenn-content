---
title: "KDP自動出版が3日詰まった原因はカテゴリー「場所」チェックボックスだった"
emoji: "✨"
type: "tech"
topics: ["claude", "python", "macos", "llm", "kindle"]
published: true
---

⏱この記事は約 8 分で読めます


📖 目次

- [📌 何が起きたか](#何が起きたか)
- [📌 環境](#環境)
- [📌 詰まったポイント](#詰まったポイント)
- [📌 解決までの手順](#解決までの手順)
- [📌 コード／設定の抜粋](#コード設定の抜粋)
- [📌 試してわかったこと](#試してわかったこと)
- [📌 まとめ](#まとめ)
- [📌 関連記事](#関連記事)


最終更新: 2026-04-28


KDP自動出版パイプラインが3日連続で詰まった。原因は「カテゴリーの場所チェックボックス」——地味すぎて逆に3日気づかなかった。


## 何が起きたか

深夜5時にlaunchdが `kdp_auto_publish.py` を自動起動している。スクリプトはEPUBをアップロードして、カテゴリー設定して、価格入力して、出版ボタンを押す——という一連を全部Playwrightで自動化している構成だ。


それが3日連続でカテゴリーページで止まった。


翌朝起きてログを確認すると「コンテンツ」か「詳細」ページ止まりで、出版まで届いていない。`04_Config/kdp_debug/` には `category_placement_fail_050131.png` というスクリーンショットが残っていて、カテゴリーモーダルが開いたままになっているのはすぐわかった。でも「なぜ閉じないのか」が、そこから2日半読めなかった。


## 環境


```
macOS Sequoia 15.x / MacBook Pro M5 32GB
Python 3.12 + Playwright (asyncio)
スクリプト: 01_Scripts/kdp/kdp_auto_publish.py  v3.0
カテゴリー設定フィールド: category_path / category_path_2 / category_placement

```

## 詰まったポイント

KDPのカテゴリーモーダルは2段階になっている。ドロップダウンで「コンピューター」「Python」と絞り込む段階と、そのあと「場所」というチェックボックス群が出てきて `コンピュータ・IT > Python` のようなノードを選ぶ段階。この2段階目の存在を、スクリプト設計のときに完全に舐めていた。


`kdp_auto_publish.py` には `category_placement` というフィールドがあって、デフォルト値がこうなっていた。


```python
placement = config.get("category_placement", "日本の小説・文芸")

```

もともと小説系の本を出版するために書いたスクリプトだから、デフォルトが「日本の小説・文芸」なのはその時点では正しかった。問題は、技術書のキューJSONに `category_placement` を書き忘れたこと。


そうするとスクリプトはPythonの技術書カテゴリーのモーダルの中から「日本の小説・文芸」というテキストを探す。当然ない。見つからないので `category_placement_fail` のスクリーンショットを保存してフォールバック処理に入る——ここがもう一つの地雷だった。


フォールバックは「最初の未チェックのチェックボックスを選択する」という実装で、モーダルスコープを `document` 全体にしていた。タイミングによってはモーダルが閉じかけている状態でPlaywrightのセレクターがモーダル外のDOM要素を掴む。掴んだチェックボックスをクリックしてもモーダルは当然閉じない。「次へ」を押しても保存スピナーが永遠に回り続ける——という流れ。


EPUBのアップロード自体は成功していた。「表紙のアップロードに成功しました」は毎回出ていた。だからなおさらカテゴリー以降を疑うのに時間がかかった。


## 解決までの手順

`04_Config/kdp_debug/` に保存された `category_placement_fail_050131.png` を開いて、カテゴリーモーダルが開いたままになっているのを確認した。ログのファイル名 `category_placement_fail` はコード内でそのタイミングにしか出てこないので、スクリプトの行番号まで5分で辿れた。


```python
# kdp_auto_publish.py:1639
await save_debug(page, "category_placement_fail")

```

ここからKDPの実UIを手動で開いて確認した。Pythonカテゴリーで「場所」チェックボックスに表示されるテキストは `コンピュータ・IT` だった。「コンピューター」でも「コンピュータ」でもなく「コンピュータ・IT」。中黒が入る。この表記でないとスクリプトがテキストマッチに失敗する。


次にキューJSONに `category_placement` を追加した。


```json
{
  "category_path": ["コンピュータ・IT", "Python"],
  "category_path_2": ["コンピュータ・IT", "プログラミング"],
  "category_placement": "コンピュータ・IT"
}

```

これだけで2日分の詰まりは解消した。ただフォールバックのバグは残ったままだったので、そちらも同時に直した。


フォールバックが `document` 全体のチェックボックスを走査していたのが根本だった。モーダルが閉じていてもページ内の別チェックボックスを掴んでクリックしていた——いわゆるゾンビクリック。


```python
# 修正前: document 全体を走査
var container = modal || document;

# 修正後: aria-hidden="false" のモーダルに限定
var modal = document.querySelector(
    '.a-popover-modal[aria-hidden="false"], [role="dialog"][aria-hidden="false"]'
);
if (!modal) return {status: 'modal_not_found'};
var container = modal;

```

モーダルが存在しない場合は即 `modal_not_found` を返して中断する。これでゾンビクリックが消えた。


再実行したら、コンテンツページで詰まらずに価格設定ページまで通った。


## コード／設定の抜粋

技術書向けキューJSONのカテゴリー設定テンプレ:


```json
{
  "category_path": ["コンピュータ・IT", "Python"],
  "category_path_2": ["コンピュータ・IT", "データサイエンス・機械学習"],
  "category_placement": "コンピュータ・IT"
}

```

`category_path` で絞り込んだ後に出てくる「場所」チェックボックスのテキストと一致させる必要がある。KDPの表記は「コンピューター」じゃなくて「コンピュータ・IT」——実UIで必ず目視確認したほうがいい。


フォールバック修正の核心部分:


```python
fallback_result = await page.evaluate("""() => {
    var modal = document.querySelector(
        '.a-popover-modal[aria-hidden="false"], [role="dialog"][aria-hidden="false"]'
    ) || document.querySelector('.a-popover-modal[style*="visible"]');
    if (!modal) return {status: 'modal_not_found'};
    var cbs = modal.querySelectorAll('input[type="checkbox"]');
    for (var i = 0; i StartCalendarInterval

    Hour5
    Minute0


```

## 試してわかったこと

KDPのカテゴリーツリーには「場所」という概念がある。ドロップダウンで選んだ値と「場所」チェックボックスのテキストが必ずしも一致しない——この仕様を把握していなかったのが3日詰まった根本だった。


`aria-hidden` 属性を信頼してスコープを絞ったら、「開いていないモーダルを操作しようとする」系のバグが全体的に減った。Playwrightで動的モーダルを扱うときは `aria-hidden="false"` を必ずスコープに含める——これは今後の原則にする。


あとデバッグスクリーンショットの命名規則は最初から入れておいて本当によかった。ファイル名が `category_placement_fail_050131.png` というだけで「カテゴリー設定のどの処理で」「何時1分31秒に」止まったかが一瞬でわかる。ログの文字列だけだと画面状態の特定にもう1〜2時間かかっていた。


## まとめ

`category_placement` のデフォルト値が小説前提のまま技術書キューに使い回されていた。フォールバックが `document` 全体を走査していてゾンビクリックが起きていた。モーダルスコープを `aria-hidden="false"` に限定したら3日分の詰まりが一気に解決した。


自動化スクリプトに汎用フォールバックを書くとき——スコープを絞らないと、エラー処理がさらなるエラーを産む。今回それを3日かけて学んだ。


一ノ瀬泰斗
AI自動化エンジニア / Python個人開発者
Claude Code × Ollama × launchd で SNS・ブログ・KDPを全自動化。実測データと失敗談を軸に、月5万円収益化のリアルな記録を発信中。


💬 **自動化の相談・小規模受託も受付中**：「launchd で毎朝 AI が動く仕組みを作りたい」「KDP の自動出版を組みたい」など、[X (@taito_automate) の DM](https://x.com/taito_automate) からお気軽にどうぞ。


### 関連記事


- [副業で月5万円稼ぐために僕がAI自動化で実装した6つの仕組み](https://ichinose-taito.com/?p=4682)
- [KDP自動化スクリプトが深夜に止まり続けた原因と5時間かけて気づいたタイミングバグの話](https://ichinose-taito.com/kdp%e8%87%aa%e5%8b%95%e5%8c%96%e3%82%b9%e3%82%af%e3%83%aa%e3%83%97%e3%83%88%e3%81%8c%e6%b7%b1%e5%a4%9c%e3%81%ab%e6%ad%a2%e3%81%be%e3%82%8a%e7%b6%9a%e3%81%91%e3%81%9f%e5%8e%9f%e5%9b%a0%e3%81%a85/)
- [AI自動化で副収入を確保する方法：ローカル環境で検証した収益化戦略](https://ichinose-taito.com/ai%e8%87%aa%e5%8b%95%e5%8c%96%e3%81%a7%e5%89%af%e5%8f%8e%e5%85%a5%e3%82%92%e7%a2%ba%e4%bf%9d%e3%81%99%e3%82%8b%e6%96%b9%e6%b3%95%ef%bc%9a%e3%83%ad%e3%83%bc%e3%82%ab%e3%83%ab%e7%92%b0%e5%a2%83%e3%81%a7/)


## 関連記事


- [launchd × BlueSky自動エンゲージ、429エラーと日次上限に詰まった記録](https://ichinose-taito.com/launchd-x-bluesky%e8%87%aa%e5%8b%95%e3%82%a8%e3%83%b3%e3%82%b2%e3%83%bc%e3%82%b8%e3%80%81429%e3%82%a8%e3%83%a9%e3%83%bc%e3%81%a8%e6%97%a5%e6%ac%a1%e4%b8%8a%e9%99%90%e3%81%ab%e8%a9%b0%e3%81%be/)
- [Claude Code hookでPythonスクリプトが黙って落ちる問題——パス・環境変数・作業ディレクトリの三重奏を全部解決した](https://ichinose-taito.com/claude-code-hook%e3%81%a7python%e3%82%b9%e3%82%af%e3%83%aa%e3%83%97%e3%83%88%e3%81%8c%e9%bb%99%e3%81%a3%e3%81%a6%e8%90%bd%e3%81%a1%e3%82%8b%e5%95%8f%e9%a1%8c-%e3%83%91%e3%82%b9/)
- [KDP自動化スクリプトが深夜に止まり続けた原因と5時間かけて気づいたタイミングバグの話](https://ichinose-taito.com/kdp%e8%87%aa%e5%8b%95%e5%8c%96%e3%82%b9%e3%82%af%e3%83%aa%e3%83%97%e3%83%88%e3%81%8c%e6%b7%b1%e5%a4%9c%e3%81%ab%e6%ad%a2%e3%81%be%e3%82%8a%e7%b6%9a%e3%81%91%e3%81%9f%e5%8e%9f%e5%9b%a0%e3%81%a85/)

---

この記事はもともと私のブログで書いたものです。より詳しい実装例・失敗談はこちらで読めます。

👉 [KDP自動出版が3日詰まった原因はカテゴリー「場所」チェックボックスだった](https://ichinose-taito.com/?p=4444)

