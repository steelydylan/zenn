---
title: Container Queriesという考え方の紹介
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---


## 従来のmedia queryの問題点


CSSでWindow幅に応じて適応するスタイルを変えるために`media query`という技術がある。スマートフォンやタブレットの普及によって、レスポンシブ対応が必須になった現代においてはmedia queryは非常に便利な機能の一つである。  
ただ、この `media query` だけでは、限界がでてくる。それは使っているブラウザのwindow幅でしかCSSの切り替えが行えないということである。時にそれは実装者にとって非常に厄介な問題となりうる。

* * *

![](/archives/001/201706/6ee07732f0db2b8b2b3932d96816f091.png)

上の図のようなレイアウトのケースを考えてみよう。  
window幅が1024pxの時はメインカラムとサブカラムがある。960px幅の際にメインカラムの下にサブカラムが回りこむ。  
window幅が1024pxの際にはメインカラムとサブカラムが横並びでレイアウトされているのでcontainerの幅はwindow幅が960pxの時よりも小さくなってしまう。その際にcontainer内に配置できるグリッド数も960pxの時と比べると少なくなってしまうだろう。結果的に以下のようなgrid配置が考えられる。

*   （0 ~ 479px）container内のitemを2列表示
*   （480px ~ 767px）container内のitemを3列表示
*   （768 ~ 1023px）container内のitemを4列表示
*   （1024px ~ 1199px）container内のitemを3列表示
*   （1200px ~ ）再びcontainer内のitemを4列表示

ここで上の条件で従来のmedia queryを使ってCSSを書いてみよう

```css
.container {
  display:flex;
  flex-wrap:wrap;
}
.container .item {
  width:50%;
}
@media screen and (min-width:480px) {
  .container .item {
    width:33.333333%;
  }
}
@media screen and (min-width:768px) {
  .container .item {
    width:25%;
  }
}
@media screen and (min-width:1024px) {
  .container .item {
    width:33.333333%;
  }
}
@media screen and (min-width:1200px) {
  .container .item {
    width:25%;
  }
}
```

基準点がwindow幅なので、1024pxの時点でメインカラムとサブカラムが横並びになる際のグリッド数を再び考慮してCSSを書く必要性が出てくる。

## 2. Container Queriesとは
-----------------------

ここで、`Container Queries`の登場である。  
Container Queriesとはwindow幅が基準ではなく指定したセレクタの幅を基準にその中に記述された子要素のセレクタに対してCSSを適応していく考え方である。  
containerの幅に応じて適応するスタイルを変更すれば、メインカラムとサブカラムが横並びから縦並びになる際のことを考慮しなくていいのでもっとシンプルにCSSがかけるようになる。

```css
.container {
  display:flex;
  flex-wrap:wrap;
}
.container .item {
  width:50%;
}
.container:(min-width:480px) {
  .item {
    width:33.333333%;
  }
}
.container:(min-width:768px) {
  .item {
    width:25%;
  }
}
```

これくらいのCSSなら media query でいいじゃんと思うかもしれないが、さらに複雑なレイアウトになってくると media query だけで全体のレイアウトを管理するのは難しくなってくる。  
特に、Reactなどのライブラリの普及によって、Componentの考え方が浸透してきた今、この`Container Queries`は今後必要になってくる考え方ではないだろうか？  
しかし残念ながらContainer Queries はCSSの仕様にはないので、JavaScriptなどのサポートを頼る必要がある。

## Container Queries の応用

自分が趣味で制作しているスタイルガイドジェネレーター Atomic Labでも Container Queriesの考え方を利用している。このスタイルガイドジェネレーターではわざわざwindow幅を小さくしてスマートフォンやタブレット時のコンポーネントのレイアウトを確認しなくていいようリサイズハンドルをつけている。containerをリサイズすれば、いかにもIframeをリサイズしているような実行結果を得られる。

* * *

[![](/archives/001/201706/9d8018953c5894eefb74cde3a053a926.png)](/archives/001/201706/large-9d8018953c5894eefb74cde3a053a926.png) 

container幅 388px

* * *

![](/archives/001/201706/6fde0087d6205685d8b6aba80504b81d.png)

container幅 818px

* * *

> ![](http://steelydylan.github.io/atomic-lab/images/ogp.png)
> 
> [AtomicLab](http://steelydylan.github.io/atomic-lab/)
> 
> Component Guide Generator Based on Atomic Design
> 
> * * *

参考
--

* * *

> ![](https://ethanmarcotte.com/img/ethan-thumb.jpg)
> 
> [On container queries. ??ethanmarcotte.com](https://ethanmarcotte.com/wrote/on-container-queries/)
> 
> A number of prominent web folks have been asking for “container queries.” I think they’re right to do so, and here’s why.
> 
> * * *

container queriesを実現するためのpolyfillもいくつかgithubなどで配布されている。

* * *

> ![](https://avatars2.githubusercontent.com/u/367169?v=3&s=400)
> 
> [ausi/cq-prolyfill](https://github.com/ausi/cq-prolyfill)
> 
> GitHub
> 
> cq-prolyfill - Prolyfill for CSS Container Queries
> 
> * * *