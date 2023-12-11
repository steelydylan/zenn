---
title: "マークダウンの中でJSXを使えるmarkdown-it-safe-jsxを作った"
emoji: "👨‍💻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "javascript", "markdown"]
published: true
---

早速ですがマークダウンの中でJSXを使えるようにするmarkdown-itのプラグインを作りました。

https://github.com/steelydylan/markdown-it-safe-jsx

下の画像のようにマークダウンの中でJSXを使えるようになります。

![](https://storage.googleapis.com/zenn-user-upload/84a96134fadf-20231211.png)

そうすると、下の画像のように記事の中に下のようにインタラクティブに動くコンポーネントを埋め込むことができます。

![](https://storage.googleapis.com/zenn-user-upload/f6e22bdf39e5-20231211.gif)

個人の技術ブログ用途にはぴったりなのではないかと思います！

## なぜ作ったか

記事の好きな場所にインタラクティブな要素を設置することができればかなり目を惹くものになりますし、単調な記事よりも飽きがこないので最後まで記事を読んでもらえる可能性が高まると考えたからです！

技術者であれば記事の中でも自分の技術を存分に発揮したいですよね！！

さらにあらかじめサーバーサイドで出力しておいたHTMLをフロント側で`hydrate`する仕組みになっているので、
ガクッとした動きもなく、`CLS`を意識した実装になっていて、これもライブラリで実現したかったことの一つです！

以前は`Lit`を使ってカスタムコンポーネントをマークダウンに埋め込むアプローチを採用していたのですが、`Lit`はサーバーサイドレンダリングが少し複雑なのでフロントエンドだけで動かしていました。ただ`Lit`エレメントがready状態になるまでは記事内に表示されないので、記事がガクッと表示されてしまっていました。
今回はReactをサーバーサイド、でレンダリングして、クライアントでhydrateしているのでその問題を解消できました。

この実行結果は mosya.dev という私が開発したプログラミング学習サービスのブログで確認することができます👇

https://mosya.dev/blog/clippath

## 使い方

使い方はとても簡単です！
下のコードのようにmarkdown-itで使えるコンポーネントを登録できるようにしています。あらかじめ登録しておいたコンポーネントのみマークダウンで使えるので安全です！

```js
import MarkdownIt from "markdown-it";
import { safeJsx } from "markdown-it-safe-jsx";

const md = new MarkdownIt();
md.use(safeJsx, {
  MyComponent: ({ text }) => <MyComponent text={text} />,
});

const html = md.render(`
# Hello

<MyComponent text="Hello" />
`);
```

### コンポーネントのhydrate

ただ、これだけだと、マークダウンの中で書かれたJSXが文字列として出力されるだけです！
そこで、`useHydrate`というフックスタイルのAPIを提供しています。

`useHydrate`を使うと、マークダウンの中で書かれたJSXを**実際に動くコンポーネントに変換**してくれます！


```js
import MarkdownIt from 'markdown-it'
import { safeJsx, useHydrateJsx } from 'markdown-it-safe-jsx'

import { TestComponent } from './TestComponent'

const components = {
  TestComponent,
}

function Markdown({ text }: { text: string }) {
  const ref = useRef<HTMLDivElement>(null);

  useHydrateJsx({
    components,
    ref,
    unsafeHydrateFunction: true,
  }, []);

  const result = useMemo(() => {
    const md = markdownIt({
      breaks: true,
      linkify: true,
      html: true,
    });
    md.use(safeJsx, components, {
      unsafeRenderFunction: true,
    })
    return md.render(text);
  }, [text]);

  return (
    <div dangerouslySetInnerHTML={{ __html: result }} ref={ref}></div>
  )
}
```

## 他のアプローチとの違い

### MDXとの違い

マークダウンの中でJSXをかけるようにするアプローチはすでにいくつかあります。

例えば有名なライブラリだとMDXがあります。

https://mdxjs.com/

これはマークダウンファイルを一つのWebpackのモジュールとして扱い、コンパイル時にJSXに変換するアプローチです。

これだと、たとえばマークダウンのデータがAPIから取得されるような場合には使えません。

私は mosya.dev というプログラムの学習サービスを作っているのですが、このサービスの教材に必要なマークダウンのデータはAPIから取得しています。
なので、バンドル前提のMDXは少し使いづらくて採用を見送りました。

### next-mdx-remoteとの違い

そこで、next-mdx-remoteというAPIから取得したマークダウンをJSXに変換するライブラリも検討しました。

https://github.com/hashicorp/next-mdx-remote

ただ、私はマークダウンのカスタマイズはこのZennにも使われている`markdown-it`をベースにするのに慣れていて、なんとかこの`markdown-it`のプラグインとしてJSXを使えるようにしたいと思っていました笑

ちょこっとだけZennの開発も手伝わせてもらってた時期もあるので`markdown-it`のプラグインを作るのには慣れていました。

さらに`markdown-it`はその拡張性から多くのプラグインが存在していて、それらのプラグインとも組み合わせて使えるようにしたかったので、`next-mdx-remote`は使わないことにしました。

## どうやって実現しているのか

最初は`TypeScript`のパーサーを使おうと思ったのですが、`TypeScript`のパーサーを使うのは流石にバンドルサイズが大きくなりすぎるのでやめました。
そこで今回は`acorn`というJavaScriptのパーサーを採用しました！

https://github.com/acornjs/acorn

acornは私の尊敬するmarijnh氏という人が作ったパーサーでこの方はCodeMirrorというエディターの作者でもあります。

https://codemirror.net/5/

acornはJavaScriptのコードからそのコードをASTというデータ構造に変換してくれます。
このデータ構造を使えば、JSXのコードにどんなpropsが使われているのかを取得することができます。

TypeScriptと比べると118KBとかなり軽量なので、ブラウザーでも気軽に使えるのがとても便利です。

![](https://storage.googleapis.com/zenn-user-upload/225098588449-20231211.png)

### acornを使ってJSXのpropsを取得する

まず、`acorn`でJSXを解析できるようにするために`acorn-jsx`というライブラリを使います。

```jsx
import { Node, Parser } from "acorn";
import jsx from "acorn-jsx";

const JSXParser = Parser.extend(jsx()); // JSXを解析できるようにする
```

次に`acorn-walk`というライブラリを使います。これはASTを再帰的に探索するためのライブラリです。

```jsx
import { extend } from "acorn-jsx-walk";
import { base, simple } from "acorn-walk";

extend(base);
```

マークダウンの中で正規表現で引っ掛けたJSXのコードを`acorn`で解析します。
`simple`という関数を使うと再起的にASTを探索してくれるので、`JSXOpeningElement`というノードを見つけたらその中のpropsを取得します。

```jsx
function extractPropsFromJSX(jsxString: string) {
  const JSXParser = Parser.extend(jsx());
  const ast = JSXParser.parse(jsxString, {
    ecmaVersion: 2020,
    sourceType: "module",
  });
  const props: Record<string, unknown> = {};

  simple(ast, {
    JSXOpeningElement: (node) => {
      node.attributes.forEach((attr) => {
        if (attr.type === "JSXAttribute" && attr.value) {
          const propName = attr.name.name;
          let propValue;

          if (attr.value.type === "JSXExpressionContainer") {
            propValue = evaluateExpression(attr.value.expression, jsxString);
          } else {
            propValue = evaluateExpression(attr.value, jsxString);
          }

          props[propName] = propValue;
        }
      });
    },
  });

  return props;
}
```

`evaluateExpression`という関数をつくってJSXの中のpropsの内容を取得するようにします。

```jsx
function evaluateExpression(node: Node, source: string) {
  if (node.type === "Literal") {
    return node.value;
  }
  if (node.type === "ArrayExpression") {
    return node.elements.map((element) =>
      evaluateExpression(element, source)
    );
  }
  if (node.type === "ObjectExpression") {
    const obj: Record<string, unknown> = {};
    node.properties.forEach((prop) => {
      if (prop.type === "Property") {
        obj[prop.key.value] = evaluateExpression(prop.value, source);
      }
    });
    return obj;
  }
  if (node.type === "ArrowFunctionExpression" || node.type === "FunctionExpression") {
    const functionString = getSourceCode(node, source);
    const func = new Function(`return ${functionString}`)();
    return func;
  }
  if (node.type === "TemplateLiteral") {
    let templateValue = "";
    node.quasis.forEach((part, index) => {
      templateValue += part.value.raw;
      if (index < node.expressions.length) {
        const exprValue = evaluateExpression(node.expressions[index], source);
        templateValue += exprValue;
      }
    });
    return templateValue;
  }
  return undefined;
}
```

これで大体の型のpropsを取得することができます。

この結果を元に`renderToString`して結果を文字列として出力します。

```jsx
const props = extractPropsFromJSX(allMatch);
const renderedComponent = ReactDOMServer.renderToString(
  <Component {...props} />
);
```

### hydrateをつかってフロントエンドで命を吹き込む

サーバーサイドからJSXを文字列として出力することができました。
さらにフロントエンドでは実際にこのJSXが動くようにしたいです。
ただ、今のままだとただの結果のみが文字列として出力されるだけなので何を頼りにJSXを動かせばいいのかわかりません。
そこであらかじめmarkdown-it側でヒントをマークダウンの中に埋め込んでおくことにしました。

```jsx
const renderedComponent = ReactDOMServer.renderToString(
  <div
    data-component={componentName}
    data-props={JSON.stringify(props)}
  >
    <Component {...props} />
  </div>
);
```

これで、フロントエンドでは`data-component`と`data-props`を参照すればどのコンポーネントがどのpropsで呼ばれていたのかを知ることができます。

この情報をもとに`hydrateRoot`関数を使って、実際にJSXを動かすことができました。

```jsx
import { hydrateRoot } from "react-dom/client";

export function useHydrateJsx(
  options: {
    components: Record<string, React.ComponentType<any>>;
    ref: React.RefObject<HTMLElement>;
    unsafeHydrateFunction?: boolean;
  },
  deps: React.DependencyList
) {
  const { components, ref, unsafeHydrateFunction } = options;
  useEffect(() => {
    if (!ref.current) return;
    const elements = ref.current.querySelectorAll("[data-component]");
    elements.forEach((element) => {
      const componentName = element.getAttribute("data-component");
      if (!componentName) return;
      const propsString = element.getAttribute("data-props");
      const Component = components[componentName];
      const props = propsString
        ? JSON.parse(propsString)
        : {};
      if (Component) {
        hydrateRoot(element, <Component {...props} />);
      }
    });
  }, deps);
}
```

`hydrateRoot`関数は`ReactDOM.render`と同じように使えますが、`ReactDOM.render`と違って、既にDOMが存在し、そのDOMを再利用することができます。
さらにそのDOMが、再びレンダリングしようとしているコンポーネントのHTMLと完全一致させる必要があります。

はじめて使いましたが`CLS`を意識してるので、あらかじめサーバーサイドで出力したHTMLを再利用することができるのはとても便利でした。

## 制約事項

まだ以下のような機能が実装されていません。

- セルフクロージングタグのサポート
- ネストされたJSXのサポート
- 壊れたJSXのエラーハンドリングはない。。。

### セルフクロージングタグのサポート

以下のような感じのJSXはマークダウンの中で使えません。

```jsx
<MyComponent />
```

代わりに以下のように書く必要があります。

```jsx
<MyComponent></MyComponent>
```

### ネストされたJSXのサポート

まだ、以下のようにJSXをネストすることはできません。

```jsx
<MyComponent>
  <MyComponent2 />
</MyComponent>
```

これはASTを再起的に探索させることになってちょっとめんどくさかったので、今後の課題として残しておきます。

### 壊れたJSXのエラーハンドリングはない。。。

壊れたJSXをマークダウンの中で書いてしまうと、そのマークダウンを読み込んだときにエラーが発生してしまうのでそこは自己責任でお願いします。
個人用のブログなどで使うのが一番ちょうどいいかもしれません。

## まとめ

今回はマークダウンの中でJSXを使えるようにするmarkdown-itのプラグインを作りました。
JavaScriptを解析できる`acorn`がとても便利でした。
このようにASTを扱えるライブラリが使えると今回のようなこと以外にも選択肢が広がると思うので、ぜひ使ってみてください！

まだテストも書けてなく、型も不完全なので、もし興味があればPRを送ってもらえると嬉しいです！

https://github.com/steelydylan/markdown-it-safe-jsx
