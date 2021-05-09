---
title: Container Queriesという考え方の紹介
emoji: "📏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["css"]
published: true
---


## 従来のmedia queryの問題点

CSSでWindow幅に応じて適応するスタイルを変えるために`media query`がある。スマートフォンやタブレットの普及によって、レスポンシブ対応が必須になった現代においては`media query`は必須の機能である。  
ただ、この `media query` だけでは、限界がでてくる。それは使っているブラウザのwindow幅でしかCSSの切り替えができないということである。時にそれは実装者にとって非常に厄介な問題となりうる。

---
![](https://storage.googleapis.com/zenn-user-upload/khpef387po2ct2zf62nusztvyef0)

上の図のようなブログのレイアウトを考えてみよう。  
window幅が960pxの時はメインカラムとサブカラムが横並びで表示されているとする。
959px幅の際にメインカラムの下にサブカラムが回りこむ。  
window幅が960pxの際にはメインカラムとサブカラムが横並びでレイアウトされているので、メインカラムの幅はwindow幅が959pxの時よりも小さくなってしまう。その際にメインカラム内に配置できるグリッド数も960pxの時と比べると少なくなってしまうだろう。よって画面幅が小さいにもかかわらず中に表示するアイテムの数を増やさざるを得ない状況が生まれる。結果的に以下のような配置がありうるだろう。

-（0 ~ 479px）メインカラム内のアイテムを2列表示
-（480px ~ 767px）メインカラム内のアイテムを3列表示
-（768 ~ 1023px）メインカラム内のアイテムを4列表示
-（1024px ~ 1199px）メインカラム内のアイテムを3列表示
-（1200px ~ ）再びメインカラム内のアイテムを4列表示

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
Container Queriesとはwindow幅が基準ではなく指定した親セレクタの幅を基準にその中に記述された子要素のセレクタに対してCSSを適応していく考え方である。  
containerの幅に応じて適応するスタイルを変更すれば、メインカラムとサブカラムが横並びから縦並びになる際のことを考慮しなくていいのでもっとシンプルにCSSが記述できるようになる。

```css
.container {
  display:flex;
  flex-wrap:wrap;
  contain: layout inline-size;
}
.container .item {
  width:50%;
}
@container:(min-width:480px) {
  .item {
    width:33.333333%;
  }
}
@container:(min-width:768px) {
  .item {
    width:25%;
  }
}
```

これくらいのCSSなら media query でいいのではと思うかもしれないが、さまざまな要素が絡んでくると`media query`だけで全体のレイアウトを管理するのは難しくなってくる。  
例えば、SaaSなどでよく見かける、左側にドロワーメニューがあり折りたためるような管理画面があったとすると、ドロワーの開閉状況によってメインコンテンツの幅の大きさが変わってくるため、windowサイズだけでCSSを描くのはだいぶ大変になってくる。

![](https://storage.googleapis.com/zenn-user-upload/1l4chgqg51j1adujmcw4xo5blw3w)

特に、Reactなどのライブラリの普及によって、Componentの考え方が浸透してきた今、この`Container Queries`は今後必要になってくる考え方ではないだろうか？

## 3. Container Queryのブラウザー対応状況

残念ながら、2021年5月現在ではContainer QueryはProposedの状況なので、どのブラウザーでも標準には利用できない。唯一Chromeだけが、`chrome://flags`からお試しいただける状況だ。

https://caniuse.com/?search=container%20query

![](https://storage.googleapis.com/zenn-user-upload/ngjy5c8ugqrlr3hylfc9mnnnw9do)

Container QueryをサポートするPolyfillはあるようだ。

https://github.com/jsxtools/cqfill

記法は少し原案と違うが、独自の記法でContainer Queryをサポートしているライブラリもある

http://marcj.github.io/css-element-queries/

## 4. Container Queryのデモ

ただ、PostCSSを使うことで、Reactなどで簡単にContainer Queryを今からでも使うことができる。カードの大きさをドラッグで調整することで中身のレイアウトが変化するのが確認できるだろう。

@[codesandbox](https://codesandbox.io/embed/nervous-leaf-wif8f?fontsize=14&hidenavigation=1&theme=dark)


## 5. まとめ

ReactやVueなどのコンポーネントライブラリが主流になった現在、コンポーネントの表示のさせ方はwindowサイズではなく、親（Container）のサイズを見て判断させたほうが都合がいいケースが増えてきた。`media query`を使うよりも圧倒的にCSSの記述量が減り、メンテナンスコストが下がるはずだ。今後プロジェクトで積極的に取り入れていきたい。