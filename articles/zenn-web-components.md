---
title: "Web Components利用したZennマークダウン部分の改善について"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---

ひっそりと [@catnose](https://twitter.com/catnose99) の友人としてZennのマークダウン部分の改善やエディターを開発している[@steelydylan](https://twitter.com/steelydylan) です。
ReactやVueなどのフレームワークの影に隠れてあまり注目されていないようにみえるWeb Componentsが今回活躍したのでメモがてら記事にしておこうと思います。

# Web Componentsを使うまでのマークダウン表示の問題点

Zennのマークダウンには通常のマークダウン記法に加え独自の記法が存在します。

```md
@[tweet](tweetのURL)
```

たとえば、上記の記法でTwitterのウィジェットが表示されます。
まさにこのウィジェットの表示には今回紹介する`Web Components`が利用されています。

@[tweet](https://twitter.com/steelydylan/status/1277938362473541634)

実はZennのマークダウン記法は全てここのソースコードに集約されています笑

https://github.com/zenn-dev/zenn-editor/tree/master/packages/zenn-markdown-html

Zennではこういったオリジナルの記法を実現するためにマークダウンをただパーサーを使って、HTMLに変換して表示するのみならずJavaScriptの処理も加えていくつかのウィジェットを表示してます。TwitterやGistなど。

## パフォーマンスの問題
例えば、Twitterのウィジェットを表示するためにマークダウンをレンダーした後に、今までのZennでは、`twttr.widget.load()`を実行する必要がありました。
`twttr.widget.load()`は、要素を全ドキュメント上から検索して、`twitter-tweet`というクラスが付与されている要素をウィジェットに変換するのでパフォーマンス上少しコストがかかります。できれば、`twittr.widget.load(element)`という形で要素を直接指定して検索によるパフォーマンス低下を防ぎたいところです。

## コードの複雑性の問題
Zennでは記事の部分のみならず、記事に対するコメントの部分にもマークダウンが使用できそのコメント部分でも記事同様にZennのマークダウン記法が使えます。記事の部分とコメントの部分でそれぞれマークダウンが展開され、もし、`@[tweet](tweetのURL)`が存在すれば、その度にウィジェットに変換する必要があるため、ソースコードを見ると、複数箇所に`twttr.widget.load()`が記述されていました。Twitterだけだったらまだいいのですが今後他のJavaScriptを必要とする埋め込みウィジェットが増えてくるとコードはどんどん煩雑になりそうです。

# Web Componentsの利用

そこで、[@catnose](https://twitter.com/catnose99) と[@steelydylan](https://twitter.com/steelydylan) はWeb Componentsに目を向けました。ReactやVueのようにあらかじめ要素の振る舞いやHTMLやそのスタイルを定義でき、以下の様にHTMLタグとして利用できます。


```html
<embed-tweet src="https://twitter.com/steelydylan/status/1277938362473541634"></embed-tweet>
```

HTMLタグとしてドキュメント上に表示するだけで、属性さえ正しければ、その後の処理などを気にしなくても必ず要素が同じ振る舞いをするあたりはReactやVueと似ています。ただ、ReactやVueとは異なり、HTMLドリブンなので、DOMに追加されたタイミングで独自の処理を実行することが可能です。つまり、`@[tweet](tweetのURL)`から`<embed-tweet src="URL"></embed-tweet>`に変換するパーサーさえ作ってしまえば後の処理はすべてWeb Componentsに任せることが可能なのです。

Web Componentsでは以下の様なことが可能です。他にもたくさんのことができますが、以下はZennで利用したWeb Componentsの主な技術になります。

- Web Components内にHTMLを表示する
- Web Componentsがドキュメントに追加されたタイミングでの振る舞いを定義
- Shadow DOMを使ったスタイルの隠蔽

## Web Components内にHTMLを表示する
Web Componentsは基本的に`HTMLElements`やその他、`HTMLParagraphElement`などそれぞれの要素を起点としてクラスの`extends`をすることで作成が可能です。
また`this.innerHTML`に表示したいHTMLの内容を代入するだけで、その中にHTMLを表示することができます。非常に直感的です。
さらに、属性値は他の要素と同様に `this.getAttribute('属性名');`で取得することができます。

```ts
class EmbedTweet extends HTMLElements {
  constructor() {
    super();
    this.url = this.getAttribute('src');
    this.innerHTML = `<div>
      <a href="${this.url}">${this.url}</a>
    </div>`
  }
}
```

## Web Componentsがドキュメントに追加されたタイミングでの振る舞いを定義
ドキュメントに追加されたタイミングでの振る舞いはライフサイクルメソッドである`connectedCallback`で定義することができます。このライフサイクルメソッドのおかげで任意のタイミングで個別に`twttr.widget.load()`を記述する必要がなくなりました。

```ts
class EmbedTweet extends HTMLElements {
  connectedCallback() {
    const container = this.querySelector(`.${containerClassName}`);
    window.twttr.widgets.createTweet(this.tweetId, container, {
      align: 'center',
    })
  }
}
```

`twttr.widget.load`の代わりに`twttr.widget.createTweet`を利用して直接、ウィジェットに変換したいDOMを指定することで、DOMの検索コストを改善しています。
Web Components内のDOMの検索にもお馴染みの`this.querySelector`が使えます。

## Shadow DOMを使ったスタイルの隠蔽

また、`constructor`ないで以下の様にShadow DOM ツリーをWeb Components内に定義することもできます。

```ts
const shadowRoot = this.attachShadow({ mode: 'open' });
```

`shadowRoot`の`innerHTML`に対して表示したいHTMLを代入することで通常のDOMではなくShadow DOMとしてWeb Components内にHTMLを表示することができます。Shadow DOMとして表示することで、外部のCSSからの影響を受けず内部だけでスタイルを定義することができます。
また、`attachShadow`をする際に、`mode: 'close'`にすることで外部のJavaScriptからのアクセスさえも遮断することができ、完全なコンポーネントのカプセル化が実現します。

# まとめ

ご紹介したように、Web Componentsではライフサイクルメソッドの利用や`this`を使った属性値へのアクセスや内部のDOM取得ができるので、マークダウンやブログコンテンツとの愛称は抜群だと思います。
正直ここまでWeb Componentsが使いやすい物とは思っていませんでした笑
ReactやVueといったJavaScriptフレームワークだけに着目せずHTML本来が持っている機能にも注目していきたいですね。
