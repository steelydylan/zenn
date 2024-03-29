---
title: "Service WorkerとWasmを組み合わせてサーバー処理をブラウザーでリアルに再現する"
emoji: "👨‍💻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["wasm", "service-worker", "webassembly", "php"]
published: true
---

今回の話は`Wasm`というよりも`Service Worker`の話がメインになりますが、`Wasm`と`Service Worker`を組み合わせることで、ブラウザー上でサーバー処理をリアルに再現することができるので、このタイトルにしています。

まずは動画をご覧ください。

@[tweet](https://twitter.com/steelydylan/status/1671051881723105280)

見ていただくと分かるように、ブラウザー上でPHPのコードを書くとその実行結果が右側に表示されています。
特に面白い点が、お問い合わせフォームのPOST後の処理までもブラウザー上だけで実行できているという点です。

これは`Wasm`と`Service Worker`を組み合わせて実現しています。
大体以下のようなプロセスで実現しています。

![](https://storage.googleapis.com/zenn-user-upload/1880fc0953b1-20230623.png)

Wasmはブラウザー側でも実行可能ですが、あえてService Worker上で実行しているのは、URLへのリクエストに対してそのリクエストにインターセプト（介入）することで、POST後の処理などもブラウザー上で実現できるようにするためです。
全てのリクエストに対してService Workerを介在させることでよりリアルなサーバー処理をブラウザー上で実現できます。

例えば、以前リリースされた`WordPress`のPlaygroundもこの方法で実装されていたりします。

https://developer.wordpress.org/playground/

## なぜ作ったか

現在私は`mosya`というブラウザー上にあるエディターだけでいろんなWeb制作を学習できるサービスを開発しています。
今回紹介する技術を使うことで、よりリアルなサーバー処理をブラウザーだけで実現できるようになり、より実践的な学習ができるようになると考えたからです。

https://mosya.dev/

学習者は自分で環境を用意する必要がないし、コードはリアルタイムに反映できるので、Wasmを使った学習体験はかなり良いと筆者は思っています。

::: message
また、Service WorkerとWasmの環境を提供することで開発者にとっても学習対象の言語ごとに（例えばGoであったりPHPであったり）のサーバーを個別に建てる必要がなくなるのでメンテナンスもしやすいです！
Service Workerが動くブラウザーであれば、オフラインでも動かすことができます！
:::

## Service Workerとは何か

まず、Service Workerとは何か軽くおさらいです。
Service Workerとは、ブラウザーのバックグラウンドで動作するスクリプトのことです。以下のようなことができるのが特徴です。

- ブラウザーのバックグラウンドで動作する
- ブラウザーのリクエストにインターセプト（介入）できる 👈これがアツイ！！
- キャッシュを操作できる
- プッシュ通知を送信できる

特に今回利用した機能がリクエストに介入できるという点です。
ブラウザー側からのリクエストに対して、Service Workerを介在させることで、サーバーにリクエストを送信する前にService Workerで処理を行い、その結果をブラウザーに返すことができます。
これを活用すると、サーバーに一切のアクセスを行わないでリクエストに対してレスポンスを返すことができるので、オフラインでも動作させることができます。

![](https://storage.googleapis.com/zenn-user-upload/bad6abe8091c-20230623.png)

## Wasmとは何か

Wasmとは何かについても触れておきましょう。
Wasmとは、ブラウザー上で動作するバイナリフォーマットのことです。
Wasmはさまざまな環境に持ち運び可能で、ブラウザーだけではなく、Service Worker上やNode.js
また最近では、CDNなどのエッジコンピューティング上やDockerでも実行することができます。

また最近では、`emscripten`というコンパイラーを使って、CやC++などから作られた言語をWasmに変換することで言語自体をブラウザーで動かそうという面白い試みもされています。

https://ja.wikipedia.org/wiki/Emscripten

`Docker`イメージから直接Wasmを生成するこんなツールも出てきています。

https://www.publickey1.jp/blog/23/dockerwebassemblywebcontainer2wasm03.html

言語だけでなく、`PostgreSQL`をブラウザーで動かすためのツールが`Supabase`から発表されていたりもしています。

https://www.publickey1.jp/blog/22/postgresqlwebpostgre-wasmwebx86.html


## Next.jsでの実装

実際にこのデモを作るにあたって、Next.js上での実装を行いました。
Next.jsにおける図にあるプロセスを解説していきます。

### Service Workerの登録

今回、Service Workerを開発しやすくするために`next-pwa`というライブラリを使っています。

https://github.com/shadowwalker/next-pwa

`next-pwa`は`next.config.js`に以下のように設定するだけでService Workerを登録できるようになります。

```js:next.config.js
const withPWA = require("next-pwa")({
  dest: "public",
});

module.exports = withPWA({
  // ここにNext.jsの設定を書く
});
```

`workers/index.ts`にService Workerのコードを書くだけで自動でService Workerのビルドを行ってくれるので非常に便利です。

また、Service Workerに型をつけるために`tsconfig.json`に以下のように`WebWorker`を追加します。

```json:tsconfig.json
{
  "compilerOptions": {
    "lib": ["dom", "dom.iterable", "esnext", "WebWorker"],
  }
}
```

### ブラウザー側の処理

ブラウザー側ではあらかじめサービスワーカー側で実行するPHPのコードを`localForage`というライブラリを使ってコードが保存されるたびに`IndexedDB`に保存しています。

https://github.com/localForage/localForage

`localForage`は`localStorage`のような使い勝手でIndexedDBに値を保存することができるので非常に便利です。
IndexedDBに保存している内容としては実行するiframeのidとそこで実行するコードと言語の情報です。


```ts
await localforage.setItem(scopeId, {
  code: code,
  lang: "php",
});
```

IndexedDBはブラウザーからでもService Workerからでもアクセス可能なのでこの特徴を利用します。
さらに保存できるデータ量が`localStorage`よりも大きいのも特徴です。

#### iframeの更新

IndexedDBへのデータの保存が終わったタイミングで、iframeを更新します。
更新することで、リクエストに対してService Workerの処理が走るようになります。
Service Worker側は`IndexedDB`からコードを取得してPHPを実行します。

```ts
await localforage.setItem(scopeId, {
  code: code,
  lang: "php",
});
ref.current?.contentWindow?.location.replace(`/php-mock/playground`);
```

::: message
ここで気をつけたいのが、`location.reload()`を使うと、例えば、iframe内で、POSTを行った状態でリロードすると、POSTが再度行われてしまうという点です。
なので、`location.replace()`を使って、iframeを更新することで、POSTが再度行われないようにしています。
:::


### Workerの実装

次にWoerker側の実装です。`tsconfig.json`に`WebWorker`を追加したので、`workers/index.ts`にて、`Service WorkerGlobalScope`という型を使うことができます。

```ts:workers/index.ts
declare let self: Service WorkerGlobalScope;

self.__WB_DISABLE_DEV_LOGS = true;

self.addEventListener("install", function () {
  self.skipWaiting();
});

self.addEventListener("activate", function (event) {
  event.waitUntil(self.clients.claim()); // Become available to all pages
});
```

Service Workerでは`self`というグローバル変数を使うことで、Service Workerのライフサイクルに応じた処理を書くことができます。

`install`と`activate`はService Workerのライフサイクルにおけるイベントで、`install`はService Workerがインストールされた時、`activate`はService Workerが有効化された時に呼ばれます。

#### skipWaiting()

`skipWaiting()`はService Workerのインストール時に呼ぶことで、すぐにサービスワーカーを有効化します。
これを呼ばないと、Service Workerのインストールが完了しても、ページをリロードするまでService Workerが有効化されません。

#### clients.claim()

`claim()`は`activate`イベントの中で呼ぶ必要があります。`waitUntil`で`claim`を呼ぶことで、ブラウザー側で
`navigator.serviceWorker.controller`を使ってService Workerのコントローラーを取得できるようになります。
クライアント側ではこのコントローラーを介してService Workerにメッセージを送ることができます。

```ts
navigator.serviceWorker.addEventListener("controllerchange", () => {
  navigator.serviceWorker.controller?.postMessage({
    // ここに送信内容
  });
});
```

### リクエストへの介在処理

Service Workerにリクエストへの介在処理を行うためには`fetch`イベントを使います。
`fetch`イベントではXHRだけではなく画像やCSS、HTML自体のリクエストにも介在することができます。

以下の処理では`php-mock`というパスにリクエストが来た時に、`IndexedDB`に保存しているコードを取得して、PHPを実行しています。リクエストの種類が`xhr`など`navigate`以外の場合は何もせずに終了します。
これでRest APIなどのリクエストには介在しないようにします。

気をつけたいのがリクエストに介在する際にイベントリスナーのコールバック自体を非同期処理を書けないという点です。非同期処理をイベントリスナーに書くとサービスワーカーが介在する前に普通にリクエストが行われてしまうためです。
なので、非同期処理が必要な場合は`event.respondWith`に渡す関数の中で非同期処理を行う必要があります。
今回の場合は`runPHP`という関数で非同期処理を行っています。

```ts
self.addEventListener("fetch", function (event) {
  const { request } = event;

  // only navigate request
  if (request.mode !== "navigate") return;

  // 特定のパスの時のみ介在する
  if (request.url.includes("php-mock")) {
    const scopeId = request.url.split("/").pop();

    event.respondWith(runPHP(request, scopeId));
  }
});
```

### PHPの実行

次に先ほど出てきた`runPHP`関数の実装です。`runPHP`関数では`IndexedDB`にあらかじめ保存してあったコードを取得します。`Wasm`の実行のインターフェースとなっている`php.run`にリクエストの情報を渡してPHPを実行します。

ここでリクエストのメソッド情報やヘッダー情報、リクエストボディなどを渡すことで、より本番のサーバー処理に近いものを再現することができます。

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

## まとめ

今回はService WorkerとWasmを組み合わせて、ブラウザー上でPHPのコードを実行するデモを作ってみました。
`Wasm`だけではなく、`Service Worker`も同時に組み合わせることで **お問い合わせフォームの実行など** よりリアルなサーバー処理をブラウザー上でエミュレートできるようになります。
この技術を応用すれば`PHP`だけではなく、`WordPress`や`Next.js`をブラウザーだけで動かすというのも可能になりそうな気がします。


## デモページ

最後に今回のデモページを作ってみたのでぜひ遊んでみてください。

https://mosya.dev/tools/php

以下のgistのコードを貼り付けることでフォームが正しく動作しているのが確認できます。

https://gist.github.com/steelydylan/bf533878d57b6f6d94d52530634b2dd9