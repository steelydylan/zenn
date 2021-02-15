---
title: "isomorphicなマークダウンパーサー、zenn-markdown-htmlについて"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["node", "javascript", "markdown"]
published: false
---

[@steelydylan](https://twitter.com/steelydylan) です。
クラスメソッドさんに譲渡される前まで開発者の友人としてZennの開発に関わっていました。
今回は、Zenn用のマークダウンパーサーの開発の苦労話について共有させてください。

マークダウンパーサーとはマークダウンをHTMLに変換するコードのことを指しているのですが、
実はZennのマークダウンパーサーは GitHub に [zenn-markdown-html](https://github.com/zenn-dev/zenn-editor/tree/master/packages/zenn-markdown-html) として公開されています。

`zenn-markdown-html`はブラウザーでもNode.js上、Workerでも動く様に作られているのですが、そのようにisomorphicに開発するのにとても苦労しました。

# zenn-markdown-htmlの開発に使われている技術

zenn-markdown-htmlには主に [markdown-it](https://github.com/markdown-it/markdown-it)が使われています。
markdown-itには多くのプラグインが存在するので基本的にはmarkdown-itとそのプラグインを組み合わせた物が`zenn-markdown-html`です。
ただ、最初に使用しようとしていたプラグインには以下の様な問題がありました。

- XSSなどの脆弱性の問題
- プラグインの多くがNode.jsでは動くが**ブラウザーでは動かない問題**

XSSに関する脆弱性への対処は @catnose がプラグインの作者にPRをだしたりして主に修正したのですが、ブラウザーで動かない問題は自分の方で主に対応しました。例えばこんなPRです。

## markdown-imsizeがブラウザでも動作するように調整

https://github.com/zenn-dev/zenn-editor/pull/35

## markdownItAnchorをブラウザでも動作する様に調整

https://github.com/zenn-dev/zenn-editor/pull/38


## markdown-it-prismをブラウザでも動作する様に調整

https://github.com/zenn-dev/zenn-editor/pull/52

## リンクからのカード生成をブラウザでも動作する様に調整

https://github.com/zenn-dev/zenn-editor/pull/96

このように多くのプラグインや実現したいことがNode.jsでは実現できて、ブラウザでは実現できない課題がありました。
主に以下の3つの理由でした

1. ブラウザでもNode.jsでも動作するisomorphicなドキュメントパーサーが存在しない
2. `fs`などのNode.js依存のモジュールが多く使われている
3. `requrire.context`などNode.js依存の関数が多く使われている

