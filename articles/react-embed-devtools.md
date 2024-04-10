---
title: "ブラウザーにChromeのデベロッパーツールを埋め込めるReactコンポーネントを作ってみた"
emoji: "📔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "devtools", "chrome"]
published: true
---

とてもニッチな用途で使えるコンポーネントですがその場のiframeのデバックができるReactコンポーネントを作ってみました！
まずはこちらのポストをご覧ください！

@[tweet](https://twitter.com/steelydylan/status/1775367867648938371)

このポストではChromeのデベロッパーツールを開いているわけではなく、ブラウザー内に直接デベロッパーツールが埋め込まれています！

今回はこのようなReactコンポーネントを作ってみたので、どのように作ったかをご紹介したいと思います。

## デモページ

こちらのページで実際にデモを試すことができます。

[https://react-embed-devtools.vercel.app/](https://react-embed-devtools.vercel.app/)

## なぜ作ったか

Reactをオンラインで学習できるサービスmosya Reactを先日リリースしました。

https://mosya.dev/react

このサイトではオンライン上でコードを書いてその場で書いたコードがプレビューできるようになっています。

詳しい開発記事はこちらをご覧ください！

https://zenn.dev/steelydylan/articles/mosya-react-development

ただ、プレビュー環境で少し不便だなとおもっているところがありました。
それがプレビュー(`iframe`)内の要素の検証です！
普通にデベロッパーツールを開いて要素を検証してもらってもいいのですが、
それだと、**本来学習に関係のないmosya本体の要素やコンソールが表示されてしまうという問題がありました**。

そこで、レッスンの内容に合わせてデベロッパーツールを埋め込むことができれば、学習者にとっても使いやすい環境を提供できるのではないかと考えました。

![](https://storage.googleapis.com/zenn-user-upload/9bb013d9bbac-20240410.png)

## ChiiとChobitsuを使ったReactコンポーネントの開発

今回のReactコンポーネントの開発にはChiiとChobitsuというライブラリを使いました。

### Chii

Chromeのデベロッパーツールをブラウザーで使おうという試みはChiiというライブラリで行われています。
これは本来、**リモートにあるデバイスの要素を検証したりコンソールを確認したりするためのツールです**。
このツールを使うことでiOSのSafariなど検証しにくいデバイスでもデベロッパーツールを使うことができます。

https://github.com/liriliri/chii

### Chobitsu

一方で同作者が開発している、Chobitsuは先ほどのデベロッパーツールに命令を送ることができるライブラリがあります！

このライブラリには

> Chrome devtools protocol JavaScript implementation.


という説明がされています。

すなわち、このライブラリを使うことでデベロッパーツールに対して命令を送ることができるのです！

https://github.com/liriliri/chobitsu

例えば、下のようなメソッドでローカルストレージのアイテムを全削除することができます。

```js
chobitsu.sendRawMessage(JSON.stringify({
  id: 1,  
  method: 'DOMStorage.clear',
  params: {
    storageId: {
      isLocalStorage: true,
      securityOrigin: 'http://example.com'
    }
  }
}));
```

### iframeの検証としてChiiとChobitsuを使う

この二つのライブラリの存在を知って、本来、`Chii`はリモートのデバイスをデバックするためのツールですが、この`Chobitsu`と組み合わせることでリモートでなくてもその場のiframeのデバックにも使えるのではと僕は考えました。

特に、その場でiframeの検証ができると**Playgroundやこのようなプログラミング学習サービスにはとても便利そうです！**

### デベロッパーツール側

まず、デベロッパーツール側を`Chii`を使って作成しました。
以下のように`Chii`を動かすのに必要なライブラリを一色HTMLで読み込んでいます。

```html
<meta name="referrer" content="no-referrer">
<script src="
https://cdn.jsdelivr.net/npm/requestidlecallback-polyfill@1.0.2/index.min.js
"></script>
<script src="https://unpkg.com/@ungap/custom-elements/es.js"></script>
<script type="module" src="https://cdn.jsdelivr.net/npm/chii@1.8.0/public/front_end/entrypoints/chii_app/chii_app.js"></script>
<script>
```

これをiframe内に読み込むことで、デベロッパーツールを埋め込むことができます。
ただ、srcDoc属性として読み込むとうまくデベロッパーツールが動かないので、作成したHTMLを**Blob**としてURLに変換してiframeに読み込むことでデベロッパーツールを埋め込むことができました。

```ts
export function useDevtools() {
  const [url, setUrl] = useState<string | null>(null);
  useEffect(() => {
    const devtoolsRawUrl = URL.createObjectURL(
      new Blob([devToolHTML], { type: "text/html" })
    );
    setUrl(devtoolsRawUrl);
    return () => URL.revokeObjectURL(devtoolsRawUrl);
  }, []);

  return `${url}#?embedded=${encodeURIComponent(location.origin)}`;
}
```

### プレビュー用のiframe側

次にプレビュー用のiframeには必ず`Chobitsu`からデベロッパーツールに向けて命令を送るためのスクリプトを読み込みます！
大体以下のような感じです！

::: details 埋め込みスクリプト
```html
<script src="https://cdn.jsdelivr.net/npm/chobitsu"></script>
<script type="module">
  const sendToDevtools = (message) => {
    window.parent.postMessage(JSON.stringify(message), '*');
  };
  let id = 0;
  const sendToChobitsu = (message) => {
    message.id = 'tmp' + ++id;
    chobitsu.sendRawMessage(JSON.stringify(message));
  };
  chobitsu.setOnMessage((message) => {
    if (message.includes('"id":"tmp')) return;
    window.parent.postMessage(message, '*');
  });
  const firstLocation = location.href
  window.addEventListener('message', ({ data }) => {
    try {
      const { event, value } = data;
      if (event === 'DEV') {
        chobitsu.sendRawMessage(data.data);
      } else if (event === 'LOADED') {
        sendToDevtools({
          method: 'Page.frameNavigated',
          params: {
            frame: {
              id: '1',
              mimeType: 'text/html',
              securityOrigin: location.origin,
              url: firstLocation,
            },
            type: 'Navigation',
          },
        });
        sendToChobitsu({ method: 'Network.enable' });
        sendToDevtools({ method: 'Runtime.executionContextsCleared' });
        sendToChobitsu({ method: 'Runtime.enable' });
        sendToChobitsu({ method: 'Debugger.enable' });
        sendToChobitsu({ method: 'DOMStorage.enable' });
        sendToChobitsu({ method: 'DOM.enable' });
        sendToChobitsu({ method: 'CSS.enable' });
        sendToChobitsu({ method: 'Overlay.enable' });
        sendToDevtools({ method: 'DOM.documentUpdated' });
      }
    } catch (e) {
      console.error(e);
    }
  });
</script>
```
:::

### プレビューとDevToolsを持つアプリケーション側

アプリケーション側ではDevtoolsから来たメッセージをプレビュー側に流す処理とDevtoolsから来たメッセージをプレビュー側に流す処理をします。
つまり二つのiframeの仲介役です！

```ts
const messageListener = (event: MessageEvent) => {
  // プレビューからのメッセージをDevtoolsに送信
  if (event.source === iframeRef.current?.contentWindow) {
    devtoolsIframeRef.current?.contentWindow?.postMessage(event.data, "*");
  }
  // Devtoolsからのメッセージをプレビューに送信
  if (event.source === devtoolsIframeRef.current?.contentWindow) {
    iframeRef.current?.contentWindow?.postMessage(
      { event: "DEV", data: event.data },
      "*"
    );
  }
};
window.addEventListener("message", messageListener);
```

## ライブラリ化した

これらの処理をまとめて今回`react-embed-devtools`というライブラリとして公開しました。
上の煩雑な処理がスッキリまとまっています！！

https://github.com/steelydylan/react-embed-devtools

### インストール方法

```bash
npm install react-embed-devtools
```

### 使い方

下のようなコードで簡単に使うことができます！
調査したいiframeに対して、`embedChobitsu`を使ってスクリプトを読み込むことで、実際にデベロッパーツールを使った検証が可能になります！

```tsx
import React from "react";
import { EmbedDevTools, embedChobitsu } from "react-embed-devtools";

const html = `
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Document</title>
  ${embedChobitsu()}
  <style>
    h1 {
      color: #333;
      font-size: 32px;
    }
  </style>
</head>
<body>
  <h1>Hello World</h1>
  <script>console.log('Hello World')</script>
</body>
</html>
`;

function App() {
  return (
    <EmbedDevTools
      direction="vertical"
      srcDoc={html}
      style={{ width: "100%", height: "100%" }}
      resizableProps={{
        style: { background: "rgba(0, 0, 0, 0.1)", height: "10px" },
      }}
      devToolsProps={{
        style: { width: "100%", height: "100%" },
      }}
    />
  );
}
```

### 注意点

::: message alert
iframeの読み込み先がクロスオリジンの場合、デベロッパーツールが正常に動作しないことがあります。
:::

