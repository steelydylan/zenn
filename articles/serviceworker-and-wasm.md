---
title: "ServiceWorkerとWasmを組み合わせてサーバー処理をブラウザーでリアルに再現する"
emoji: "👨‍💻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["wasm", "service-worker", "webassembly", "php"]
published: true
---

まずは動画をご覧ください。

@[tweet](https://twitter.com/steelydylan/status/1671051881723105280)

見ていただくと分かるように、ブラウザー上でPHPのコードを書くとその実行結果が右側に表示されています。
特に面白い点が、お問い合わせフォームのPOST後の処理までもブラウザー上だけで実行できているという点です。

これは`Wasm`と`ServiceWorker`を組み合わせて実現しています。
大体以下のようなプロセスで実現しています。

![](https://storage.googleapis.com/zenn-user-upload/7822028c246a-20230621.png)

Wasmはブラウザー側でも実行可能ですが、あえてServiceWorker上で実行しているのは、URLへのリクエストに対してそのリクエストにインターセプト（介入）することで、POST後の処理などもブラウザー上で実現できるようにするためです。
全てのリクエストに対してServiceWorkerを介在させることでよりリアルなサーバー処理をブラウザー上で実現できます。

例えば、以前リリースされた`WordPress`のPlaygroundもこの方法で実装されていたりします。

https://developer.wordpress.org/playground/

## なぜ作ったか

現在私は`mosya`というブラウザー上にあるエディターだけでいろんなWeb制作を学習できるサービスを開発しています。
今回紹介する技術を使うことで、よりリアルなサーバー処理をブラウザー上で実現できるようになり、より実践的な学習ができるようになると考えたからです。

https://mosya.dev/

現在は、HTMLとCSS、JavaScript、PHPの学習を対象としていますが、今後はReactやTypeScriptなどのフレームワークやライブラリを対象にしたコンテンツやサーバーサイドの学習コンテンツの追加も視野に入れています。

## Next.jsでの実装

実際にこのデモを作るにあたって、Next.js上での実装を行いました。
Next.jsにおける図にあるプロセスを解説していきます。

### 1. ServiceWorkerの登録

今回、ServiceWorkerを開発しやすくするために`next-pwa`というライブラリを使っています。

https://github.com/shadowwalker/next-pwa

`next-pwa`は`next.config.js`に以下のように設定するだけでServiceWorkerを登録できるようになります。

```js:next.config.js
const withPWA = require("next-pwa")({
  dest: "public",
});

module.exports = withPWA({
  // ここにNext.jsの設定を書く
});
```

`workers/index.ts`にServiceWorkerのコードを書くだけで自動でServiceWorkerのビルドを行ってくれるので非常に便利です。

また、ServiceWorkerに型をつけるために`tsconfig.json`に以下のように`WebWorker`を追加します。

```json:tsconfig.json
{
  "compilerOptions": {
    "lib": ["dom", "dom.iterable", "esnext", "WebWorker"],
  }
}
```

### Workerの実装

次にWoerker側の実装です。`tsconfig.json`に`WebWorker`を追加したので、`workers/index.ts`にて、`ServiceWorkerGlobalScope`という型を使うことができます。

```ts:workers/index.ts
declare let self: ServiceWorkerGlobalScope;

self.__WB_DISABLE_DEV_LOGS = true;

self.addEventListener("install", function () {
  self.skipWaiting();
});

self.addEventListener("activate", function (event) {
  event.waitUntil(self.clients.claim()); // Become available to all pages
});
```

ServiceWorkerでは`self`というグローバル変数を使うことで、ServiceWorkerのライフサイクルに応じた処理を書くことができます。

`install`と`activate`はServiceWorkerのライフサイクルにおけるイベントで、`install`はServiceWorkerがインストールされた時、`activate`はServiceWorkerが有効化された時に呼ばれます。

#### skipWaiting()

`skipWaiting()`はServiceWorkerのインストール時に呼ぶことで、すぐにサービスワーカーを有効化します。
これを呼ばないと、ServiceWorkerのインストールが完了しても、ページをリロードするまでServiceWorkerが有効化されません。

#### clients.claim()

`claim()`は`activate`イベントの中で呼ぶ必要があります。`waitUntil`で`claim`を呼ぶことで、ブラウザー側で
`navigator.serviceWorker.controller`を使ってServiceWorkerのコントローラーを取得できるようになります。
クライアント側ではこのコントローラーを介してServiceWorkerにメッセージを送ることができます。

```ts
navigator.serviceWorker.addEventListener("controllerchange", () => {
  navigator.serviceWorker.controller?.postMessage({
    // ここに送信内容
  });
});
```

### ブラウザーからメッセージが送られた時の処理

ServiceWorkerにブラウザーからメッセージが送られた時の処理は`message`イベントで受け取ることができます。
最初にお見せした図解のステップ2にあたるのですが、ここで`localforage`というライブラリを使って、ブラウザーのIndexedDBにコードを保存しています。`localStorage`のような使い勝手で非常に便利です。
保存している内容としては実行するiframeのidとそこで実行するコードと言語の情報です。

```ts
self.addEventListener("message", async function (event) {
  const { data } = event;
  await localforage.setItem(data.scopeId, {
    code: data.code,
    lang: data.lang,
  });
  // ここに続きの処理
});
```

### リクエストへの介在処理

ServiceWorkerにリクエストへの介在処理を行うためには`fetch`イベントを使います。
`fetch`イベントではXHRだけではなく画像やCSS、HTML自体のリクエストにも介在することができます。

以下の処理では`php-mock`というパスにリクエストが来た時に、`localforage`に保存しているコードを取得して、PHPを実行しています。リクエストの種類が`xhr`など`navigate`以外の場合は何もせずに終了します。
これでRest APIなどのリクエストには介在しないようにします。

ここで気をつけたいのがリクエストに介在する際に非同期処理を書けないという点です。非同期処理をイベントリスナーに書くとサービスワーカーが介在する前に普通にリクエストが行われてしまうためです。
なので、非同期処理が必要な場合は`event.respondWith`に渡す関数の中で非同期処理を行う必要があります。
今回の場合は`runPHP`という関数で非同期処理を行っています。

```ts
self.addEventListener("fetch", function (event) {
  const { request } = event;

  // only navigate request
  if (request.mode !== "navigate") return;

  // Bypass navigation requests.
  if (request.url.includes("php-mock")) {
    const scopeId = request.url.split("/").pop();

    event.respondWith(runPHP(request, scopeId));
  }
});
```

### PHPの実行

次に先ほど出てきた`runPHP`関数の実装です。`runPHP`関数では`localforage`にあらかじめ保存してあったコードを取得します。`Wasm`の実行のインターフェースとなっている`php.run`にリクエストの情報を渡してPHPを実行します。

```ts
async function runPHP(request: Request, scopeId: string) {
  const storage = (await localforage.getItem(scopeId)) as {
    code: string;
    lang: string;
  };
  if (!storage?.code || !storage?.lang) {
    return new Response("not found", {
      status: 404,
    });
  }
  const { code, lang } = storage;
  const result = await buildPHPWithRequest(
    "8.2",
    code,
    request,
    req.method === "POST" ? await request.text() : undefined
  );
  return new Response(result.body, {
    status: result.status,
    headers: result.headers,
  });
}

export async function buildPHPWithRequest(
  version: Version,
  code: string,
  request: Request,
  body?: string
) {
  const php = await initPHP(version);
  const output = await php.run({
    code: code,
    method: request.method,
    headers: Object.fromEntries(request.headers.entries()),
    body: body,
  });
  const res = new TextDecoder().decode(output.body);
  return {
    status: output.httpStatusCode,
    headers: output.headers,
    body: res,
  };
}
```

### PHPのWasmの生成

PHPの`wasm`やそれを実行する`js`の生成には`emscripten`というコンパイラーおよびDockerを利用しています。
詳しい内容についてはこの方の記事をご覧ください！

https://zenn.dev/glassmonkey/articles/ae6cadef80c6c4


### フロント側の処理

最後にフロント側の処理です。フロント側では`postMessage`という関数を用意しました。これはすでにサービスワーカーがコントロールできる状態の場合はそのままメッセージを送信しますが、そうでない場合は`controllerchange`イベントを待ってからメッセージを送信します。サービスワーカー側の`activate`イベントで`clients.claim()`を呼んでいるので、`controllerchange`イベントが発火することでサービスワーカーがコントロールできる状態になっています。

省略しますが大体以下のような処理になっています。

```ts
const postMessage = async (data: unknown) => {
  return new Promise<void>((resolve) => {
    function onReceiveMessage(event: MessageEvent) {
      navigator.serviceWorker.removeEventListener("message", onReceiveMessage);
      resolve();
    }
    function onControllerChange() {
      navigator.serviceWorker.removeEventListener(
        "controllerchange",
        onControllerChange
      );
      navigator.serviceWorker.addEventListener("message", onReceiveMessage);
    }
    if (navigator.serviceWorker.controller) {
      navigator.serviceWorker.controller.postMessage(data);
      navigator.serviceWorker.addEventListener("message", onReceiveMessage);
    } else {
      navigator.serviceWorker.addEventListener("controllerchange", onControllerChange);
    }
  });
};

postMessage({
  lang: "php",
  scopeId: "playground",
  code: debouncedCode,
}).then(() => {
  // iframeをリロードすることで再度サービスワーカーにリクエストを送る
  ref.current?.contentWindow?.location.replace(`/php-mock/playground`);
});
```

::: message
ここで気をつけたいのが、`location.reload()`を使うと、例えば、iframe内で、POSTを行った状態でリロードすると、POSTが再度行われてしまうという点です。
なので、`location.replace()`を使って、iframeを更新することで、POSTが再度行われないようにしています。
:::

## まとめ

今回はServiceWorkerとWasmを組み合わせて、ブラウザー上でPHPのコードを実行するデモを作ってみました。
`Wasm`だけではなく、`ServiceWorker`も同時に組み合わせることでよりリアルなサーバー処理をブラウザー上でエミュレートできるようになります。
この技術を応用すれば`PHP`だけではなく、`WordPress`や`Next.js`をブラウザーだけで動かすというのも可能になりそうな気がします。


## デモページ

最後に今回のデモページを作ってみたのでぜひ遊んでみてください。

https://mosya.dev/tools/php