---
title: "Reactを学習できるサービスmosya Reactの技術的な紹介"
emoji: "👨‍💻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "service-worker", "個人開発"]
published: true
---

2024年3月15日の一粒万倍日に、mosya ReactというReactを学習できるサービスをリリースしました。

![](https://storage.googleapis.com/zenn-user-upload/3e864523bc9c-20240323.png)

こちら1年間の開発期間を経て、ようやくリリースできました！
mosyaの開発期間と合わせると約2年間の開発期間を経てのリリースとなります。
いやー、長かった！

良かったら下のリンクから試してみてください！

https://mosya.dev/react


## どんなサービスか

mosya ReactはReactをオンライン上で学習できるサービスです。
エディターに書いたコードがリアルタイムにプレビューできるようになっていて環境構築なしでReactを学習できます！
採点機能が搭載されているのでReactを自学習したい方におすすめです！

このサービスの開発で特に頑張ったのが以下の特徴です！

- 最新の技術にキャッチアップしている
- ライブラリの型がエディター上で確認できる
- Biomeを動かしていてリントエラーがエディター上で確認できる

### 最新の技術にキャッチアップしている

mosya Reactでは最新の技術を取り入れた内容になっています！
たとえばReactは当然hooksを教えていますし、TypeScriptを使ったReactの開発も学べます！
また開発が止まってしまっているRecoilではなく、Jotaiを使った状態管理も学べます！
React Routerのレッスンもあるのですが、古いバージョンのv5ではなく、v6を使ったレッスンになっています！

レッスンを作ってる最中で技術的なトレンドが変わってしまってそのレッスンを全て破棄した上で新しいレッスンを作り直すこともあり、とても大変でした。。。
個人開発だからこそできることだと思います！

### ライブラリの型がエディター上で確認できる

mosya Reactではライブラリの型がエディター上で確認できます！また自分が書いたコードの型もエディター上で確認でき、まるでVSCodeのような開発環境を提供しています！

![](https://storage.googleapis.com/zenn-user-upload/cf42e133c538-20240323.png)

チェックしたい変数や関数にカーソルを合わせると型が表示されるので、型を確認しながら開発ができます！

### Biomeを動かしていてリントエラーがエディター上で確認できる

mosya ReactではBiomeを動かしていて、リントのレッスンや実践編のレッスンでリントエラーがエディター上で確認できます！

![](https://storage.googleapis.com/zenn-user-upload/28eed235d916-20240323.png)

採点時にもリントエラーがあるとクリアできないようになっていて自然と良いコードが書けるようになります！

## 技術的な紹介

mosya Reactは以下の技術を使って開発しています。

- Service Worker
- Web Worker
- IndexedDB

特に頑張ったのがコードを書いた際にリアルタイムに反映されるプレビュ画面です！
このプレビュー画面にはService Workerを使ってReactのコードをコンパイルしています！

![](https://storage.googleapis.com/zenn-user-upload/80c29130b02f-20240323.png)

どのように実現しているかというと、Service Workerを使ってエディターに書いたコードを受け取り、そのコードをコンパイルしてプレビュー画面に表示しています！

![](https://storage.googleapis.com/zenn-user-upload/333b411d5b44-20240323.png)

詳しく説明すると、まずエディターで書いたファイルの内容をフロント側でIndexedDBに保存します。

![](https://storage.googleapis.com/zenn-user-upload/6e8c922fd645-20240323.png)

IndexedDBに保存している理由としては、Service WorkerがIndexedDBに保存されたファイルを取得できるためです。

次にコードを書き換えたタイミングでiframeをReactのkeyを使って更新します。

```tsx
<iframe key={refreshKey} />
```

iframeの中のファイルへのリクエストに対してService Workerが役に立ちます！
Service Workerの機能の一つで、リクエストに対して処理を介在して返すことができる機能を使います！

![](https://storage.googleapis.com/zenn-user-upload/144738ddbc6a-20240324.png)

通常はここでキャッシュを返すことが多いのですが、今回はService WorkerがIndexedDBに保存されたファイルを取得してコンパイルして返します！

この際に相対パスへのimport文はすべてesmのインポートとして解決するために拡張子を`.mjs`に変換した上でコードを`esbuild`を使ってコンパイルします。

例えば、`Hoge.mjs`のファイルにアクセスがあった場合、`Hoge.tsx`のファイルをIndexedDBから取得してコンパイルして、それをService Workerから返します。


### Reactなどのライブラリの解決

Reactなどのライブラリの解決はServiceWorkerではなくimportmapという機能を使って解決します。
例えば下のようなコードを書くとしましょう。

```tsx
import React from 'react'
```

これができるようにするためには、importmapを使って`react`を`https://esm.sh/react`から読み込むように設定します。

```html
<script type="importmap">
{
  "imports": {
    "react": "https://esm.sh/react"
  }
}
</script>
```

`esm.sh`は`commonJS`で書かれたライブラリーをCDN経由で`esm`に変換してくれるサービスです。
めっちゃ便利です！


### エディターのリント

エディターのリントはWeb Workerを使って実現しています。
Web Worker上で`Biome`というライブラリを使ってリントを行っています。

https://biomejs.dev/

ここで`Comlink`というライブラリを使ってWeb Workerとメインスレッドの通信を行っています。
Worker側のメソッドをフロント側で直感的に呼べてとても使いやすいです。

#### フロント側

```ts
const worker = new Worker(
new URL("../../workers/biome.worker", import.meta.url),
{
    type: "module",
}
);
const api = Comlink.wrap<BiomeWorkerApi>(worker);
const diagnostics = await api.getJSReport(code, pathName);
```

#### Worker側

```ts
// 上記省略
const workerApi = {
  getJSReport,
};

Comlink.expose(workerApi);
```

このとき`@biomejs/wasm-web`というWasm化された`Biome`のライブラリがめっちゃ役に立ちました！
ブラウザー上でリンターが動かせるのすごい！

### エディター内での型の解決

先ほど紹介したようにmosya Reactでは型を調べたい変数や関数にカーソルを合わせると型が表示されます。

![](https://storage.googleapis.com/zenn-user-upload/cf42e133c538-20240323.png)

これはあらかじめレッスンごとに`node_modules`のファイル内容を`JSON化`しておき、それを`Monaco Editor`の機能を使って読み込ませています。

大体こんな感じで読み込ませています。

```ts
const deps = await fetch(
  `${process.env.NEXT_PUBLIC_CLOUD_FLARE_PUBLIC_URL}/${lessonName}/deps.json`
);
const depsJson = await deps.json()

Object.entries(depsJson).forEach(([key, value]) => {
  monaco.languages.typescript.typescriptDefaults.addExtraLib(
    value,
    `file:///node_modules/${key}`
  );
});
```

node_modulesのファイル内容を読み込ませるだけで型を解決してくれる`Monaco Editor`は優秀ですね！

### 型のテスト

TypeScriptのレッスンでは型レベルのテストも採点時に行っています！
TypeScriptの`Compiler API`を使って型のテストを行っています！

https://github.com/steelydylan/type-tester

これを使うと以下のような感じで型のテストを書くことができます。

```ts
import { TypeTester } from "browser-type-tester";

const code = `
type Concat<T extends any[], U extends any[]> = [...T, ...U];
type Result = Concat<[1], [2]>; // expected to be [1, 2]
`;
const typeTest = new TypeTester({ code });

typeTest.it("Concat<A, B>型が正しい", () => {
  typeTest.expect("Concat<['a', 'b'], ['c', 'd']>").toBe("['a', 'b', 'c', 'd']");
});

typeTest.it("Concat<A, B>はanyを返さない", () => {
  typeTest.expect("Concat<['a', 'b'], ['c', 'd']>").not.toBeAny();
});

const results = await typeTest.run();

console.log(results);
```

詳しくは以前書いたZennの記事を見てみてください！

https://zenn.dev/steelydylan/articles/mosyatc-development

## 今後のアップデート

今後のアップデートではNext.jsを使った開発のレッスンを追加する予定です！
Next.jsに限っては上記の技術では再現不可能なので、`WebContainers`というライブラリーを使ってプレビューを実現する予定です！

https://webcontainers.io/

`WebContainers`は商用利用には許可が必要なのですが友人の協力によりなんとかDiscord上でWebContainersの開発者と繋がることができました！
1万セッションユーザーまでは無料で使えてそれ以降は支払いが発生するという形になりました。

Next.jsのレッスンはサブスクに登録いただいているユーザーのみに提供予定なのでなかなか1万セッションユーザーに達することはないと思いますが、もし達した場合はちゃんと支払いをする予定です！

WebContainersに関する記事も実は以前Zennに書いてます！

https://zenn.dev/steelydylan/articles/webcontainers


## 他のサービスの技術的な紹介

以前リリースしたmosyaやmosya&lt;TC&gt;の開発の詳細については以下のZennの記事にまとめています。

https://zenn.dev/steelydylan/articles/mosya-development

https://zenn.dev/steelydylan/articles/mosyatc-development

## 宣伝

mosya Businessという法人向けサービスも運営しています！
mosyaやmosya Reactなどのサービスが法人の社員向けに提供できるもので、社員の進捗や書いたコードを可視化できるようになっています！

### 進捗の確認画面

![](https://storage.googleapis.com/zenn-user-upload/0387b71dd188-20240324.png)

### コードの確認画面

![](https://storage.googleapis.com/zenn-user-upload/09fc25f146aa-20240324.png)

必要なレッスンも私に相談していただける窓口も用意していますので興味のある方はぜひこちらをご覧ください！
会社でもmosyaを使ってくれるととても嬉しいです！

https://mosya.dev/business
