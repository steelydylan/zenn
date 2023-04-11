---
title: "Zodファーストで型安全なNext.js用のルーティングライブラリを作った話"
emoji: "📔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "zod", "typescript"]
published: true
---

今までNext.jsのAPIでちょっとしたルーティングを実現するために下記のライブラリを使っていました。

https://github.com/hoangvvo/next-connect

ただ、こちらのライブラリでは、**APIの型情報をクライアントに共有すること**や、**Zodなどであらかじめサーバーサイドで受け取るパラメータの型を縛ったりすること**ができませんでした。

そこで、今回ZodファーストなNext.js用のルーティングライブラリを作ってみました。

https://github.com/steelydylan/next-typed-connect

trpcやGraphQLなどを使えばクライアントへの型共有は行えるのですが、そこまで複雑なものにしたくなかったので、軽量なZodファーストな型安全なNext.js用のルーティングライブラリを作りたいというのがモチベーションとしてありました。

## 使い方

Zodファーストなフレームワークで、APIの型情報も、サーバーサイドでのランタイムでの型チェックも、クライアントサイドでの型チェックも、すべてZodで行うことができます。

`/pages/api/**.ts`でzodを使って以下のようにサーバーサイドを定義します。
`GET`,`POST`,`PUT`,`DELETE`,`PATCH`,`PUT`,`DELETE`のHTTPメソッドに対応しています。

### サーバーサイド

#### zodを使ってバリデーションを定義

```ts
import { z } from "zod";

// POSTメソッド用のバリデーション
const postValidation = {
  // Request bodyの型を定義
  body: z.object({
    foo: z.string(),
  }),
  // URLパラメータの型を定義
  query: z.object({
    bar: z.string().optional(),
  }),
  // Response bodyの型を定義
  res: z.object({
    message: z.string(),
  }),
}

// GETメソッド用のバリデーション
const getValidation = {
  query: z.object({
    bar: z.string().optional(),
  }),
  res: z.object({
    message: z.string(),
  }),
}  
```

#### バリデーションをルーティングに渡す

次にvalidationをルーティングに渡します。`validate`関数を使うことで、型チェックを行い、型が合わない場合はエディターでエラーが表示されると同時に、サーバーサイドでランタイムエラーが発生します。
zodを使っているので型チェックとランタイムのチェックを同時に行えることが特徴です。

それぞれのAPIルートで`validate`関数とともにルーティング処理を行います。

```diff ts:pages/api/sample.ts
import { createRouter, validate } from "next-typed-connect";
import { postValidation, getValidation } from "./validation";

const router = createRouter();

+   router.post(
+     validate(postValidation), async (req, res) => {
+       const { foo } = req.body;
+       const { bar } = req.query;
+       // この時もしbodyとqueryの型が合わない場合はエディターでエラーが表示されます。
+       res.json({ message: "Hello World" });
+     }
+   );

+   router.get(
+     validate(getValidation), async (req, res) => {
+       const { bar } = req.query;
+       res.json({ message: "Hello World" });
+     }
+   );

+   export default router.run()
```

ここで、`router.get`や`router.post`には幾つでもコールバック関数を渡すことができます。

:::message
`router.use`を使うことで、全てのHTTPメソッドに対して共通の処理を行うことができます。
:::

### 型情報のエクスポート

次に現在のファイルにて各メソッドごとに型情報をエクスポートします。これにより、クライアントサイドでの型チェックが可能になります。

```diff ts:pages/api/sample.ts
+   import { createRouter, validate, ApiHandler } from "next-typed-connect";
import { postValidation, getValidation } from "./validation";

const router = createRouter();

router.post(
  validate(postValidation), async (req, res) => {
    const { foo } = req.body;
    const { bar } = req.query;
    // この時もしbodyとqueryの型が合わない場合はエディターでエラーが表示されます。
    res.json({ message: "Hello World" });
  }
);

router.get(
  validate(getValidation), async (req, res) => {
    const { bar } = req.query;
    res.json({ message: "Hello World" });
  }
);

export default router.run()

+   export type PostHandler = ApiHandler<typeof postValidation>;
+   export type GetHandler = ApiHandler<typeof getValidation>;
```

ここで、型情報は以下のような名前でそれぞれエクスポートするようにしてください。

- PostHandler
- GetHandler
- PutHandler
- DeleteHandler
- PatchHandler

### 型定義ファイルの生成

次にコマンドを使ってexportされた型情報からクライアント用の型情報を生成します。

```bash
$ npx next-typed-connect
```

監視モードを使うことで、ファイルの変更を検知して自動的に型情報を生成することもできます。

```bash
$ npx next-typed-connect -w
```

これで`/pages/api/**.ts`にて定義した型情報が`.node_modules/.next-typed-connect/**/**d.ts`に生成されます。

### クライアントサイド

特に生成された型情報を気にしなくても、`next-typed-connect`にある`getApiData`や`postApiData`を使うことで、自動的にクライアントサイドで型情報を読み込んでくれます。


```ts
import { client } from "next-typed-connect";

// 型定義がされていることでパスの補完や
// サーバーサイドに送るパラメータの型チェックが簡単にできます。
const { data, error } = await client.get('/api/sample', {
  query: {
    bar: 'baz',
  }
})
```

```ts
import { client } from "next-typed-connect";

const { data, error } = await client.post('/api/sample', {
  body: {
    foo: 'bar',
  },
  query: {
    bar: 'baz',
  }
})
```

### オプション

`next-typed-connect`にはいくつかのコマンドがあります。

特にNext.jsは使い方によって`pages`のディレクトリ構造が変わるので、`--pagesDir`オプションを使って`pages`のディレクトリを指定する点に注意してください。

```bash
next-typed-connect --pagesDir=src/pages
```

| オプション | 説明 | デフォルト値 |
| --- | --- | --- |
| --pagesDir | ページディレクトリへのパス | pages |
| --baseDir | プロジェクトのパス | . |
| --distDir | 生成した型情報の出力先	 | node_modules/.next-typed-connect |
| --moduleNameSpace | Type definition file module name | .next-typed-connect |

## 苦労した点、工夫した点

### TypeScript Compiler APIの利用
TypeScript Compiler APIの使用を始めて試みました。TypeScript本体の慣れないメソッド名と格闘しました。

### コマンド化
`.bin`を使って`npx`や`yarn`で`next-typed-connect`を実行できるようにしました。またいくつかのオプションを用意するために`commander`を使いました。

### ファイルの監視
`-w`で監視モードを使えるようにしました。`chokidar`を使うことでファイルの変更を検知して型定義ファイルを生成することに成功しました！

### `.next-typed-connect`のモジュール名空間

`getApiData`や`postApiData`などを使うときにそれぞれのAPIの型定義ファイルを読み込む必要があるのですが、そのために`.next-typed-connect`というモジュール名空間を作成する必要がありました。
この時、`node_modules/.next-typed-connect`ディレクトリに自動的に型定義ファイルが生成されるようになってます。

この時、特に利用者側が`tsconfig.json`を変更しなくても自動的にこの名前空間に`getApiData`や`postApiData`などのメソッドがアクセスできるようにするための調整が本当に大変でした。

### `zod`から型情報の取得
`zod`を使った静的型チェックを実装しました。`zod`は型チェックとランタイムのチェックを同時に行えるので便利なのですが、それをどのようにルーティングに組み込むか非常に難しかったです。

`z.infer`を使うことでzodSchemaから型情報を取得することができました。

```ts
import { z } from "zod";

export type ApiHandler<T extends ApiZodSchema> = {
  body: T["body"] extends z.ZodSchema<any> ? z.infer<T["body"]> : never;
  query: T["query"] extends z.ZodSchema<any> ? z.infer<T["query"]> : never;
  res:  T["res"] extends z.ZodSchema<any> ? z.infer<T["res"]>: never;
}
```

## 参考にしたZennの記事

https://zenn.dev/takepepe/articles/nextjs-typesafe-api-routes

## 今後の展望

- `zod`以外の型チェックライブラリに対応する
- `headers`の型チェックを実装する
- `next-typed-connect`のテストを書く
- GitHub Actionsを使った自動リリース

よかったらGitHubでスターください！

https://github.com/steelydylan/next-typed-connect
