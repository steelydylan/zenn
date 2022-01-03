---
title: "Remix on Cloudflare Workersの構成でハマったこと"
emoji: "🌀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["remix", "cloudflare", "react", "tailwind"]
published: true
---

この年末年始にCloudflare WorkersでRemixを触ったりしてました。
いずれも公式ドキュメントなどをしっかり読めばハマることはないのですが自分の場合少し手こずったので記事として残しておきます。はまったことは以下4つです。

1. サーバー側とフロント側で環境変数を扱いたい
2. TailWindCSSを使いたい
3. GitHub Actionsを使ってデプロイしたい
4. Node.jsのライブラリによっては利用できない

## 開発環境のセットアップ

セットアップはとても簡単でした。
Remix on Cloudflare Workersの開発環境は以下のコマンドで簡単にセットアップすることができます。

```sh
$ npx create-remix@latest
```

上記のコマンドを実行するとどの環境にデプロイしたいかを聞かれるので`Cloudflare Workers`を選択します。

```
? Where do you want to deploy? Choose Remix if you're unsure, it's easy to change deployment targets. 
  Express Server 
  Architect (AWS Lambda) 
  Fly.io 
  Netlify 
  Vercel 
  Cloudflare Pages 
❯ Cloudflare Workers 
(Move up and down to reveal more choices)
```

また、関数をビルドしたりCloudflareにデプロイしたりするためのツールとしてWranglerを別途インストール必要があります。npmを使ってインストールできます。

```sh
$ npm install -g @cloudflare/wrangler
```

## 1. サーバー側とフロント側で環境変数を扱いたい

ローカルでの開発では以下のように`.env`ファイルを使えます。
ただ、`process.env.API_KEY`のように、process.envから環境変数を取り出すわけではなく、サーバーサイドではそのまま定数がグローバルに定義されています。

そのためTypeScript利用時には`global.d.ts`などに利用する環境変数の型を定義する必要がありました。

```ts:global.d.ts
declare const SUPABASE_ANON_KEY: string
```

Cloudflare Workers側で環境変数を利用する場合は、wranglerを使って環境変数を定義しておく必要があります。

```sh
$ wrangler secret put SUPABASE_ANON_KEY
```

ただし、この環境変数はフロント側では使えないため以下のように`loader`を使ってフロントに環境変数を渡す必要がありました。これは少し不便です。

https://remix.run/docs/en/v1/guides/envvars

```tsx:root.tsx
export function loader() {
  return {
    ENV: {
      SUPABASE_URL: SUPABASE_URL,
      SUPABASE_ANON_KEY: SUPABASE_ANON_KEY,
    }
  }
}

export function Root() {
  const data = useLoaderData();
  return (
    <html>
      <head>
        <Meta />
        <Links />
      </head>
      <body>
        <Outlet />
        <script
          dangerouslySetInnerHTML={{
            __html: `window.ENV = ${JSON.stringify(
              data.ENV
            )}`
          }}
        />
        <Scripts />
      </body>
    </html>
  );
}
```

こうすることでフロント側からは、`window.ENV.SUPABASE_URL`という形で環境変数を取り出せます。
そのため、TypeScript利用時にはフロント用に`window`側にも以下のように環境変数の型を定義しておきます。

```ts:global.dts
interface Window {
  ENV: {
    SUPABASE_ANON_KEY: string
  }
}
```

## 2. TailWindCSSを使いたい

`npx create-remix@latest`を実行にpackage.jsonが自動生成されますが、そのpackage.jsonを少し修正し、`npm run start`時にtailwind.cssをビルドするようにして、その結果を`./app/styles/tailwind.css`に吐き出します。

```diff json
  "scripts": {
-    "build": "remix build",
+    "build": "npm run build:css && npm run build:remix",
+    "build:remix": "remix build",
+    "build:css": "tailwindcss --output ./app/styles/tailwind.css --minify",
    "dev": "remix watch",
    "postinstall": "remix setup cloudflare-workers",
    "build:worker": "esbuild --define:process.env.NODE_ENV='\"production\"' --minify --bundle --sourcemap --outdir=dist ./worker",
    "dev:worker": "esbuild --define:process.env.NODE_ENV='\"development\"' --bundle --sourcemap --outdir=dist ./worker",
-    "start": "miniflare --build-command \"npm run dev:worker\" --watch",
+    "start": "npm-run-all -p start:*",
+    "start:css": "tailwindcss --output ./app/styles/tailwind.css --watch",
+    "start:miniflare": "miniflare --build-command \"npm run dev:worker\" --watch",
    "deploy": "npm run build && wrangler publish"
  },
```

以下のようにroot.tsxで吐き出したcssをインポートするようにします。

```tsx:root.tsx
import tailwindStyles from "~/styles/tailwind.css";

export const links: LinksFunction = () => {
  return [
    { rel: "stylesheet", href: tailwindStyles },
  ];
};
```

## 3. GitHub Actionsを使って自動デプロイしたい

以下のようにwranglerをCI上でグローバルインストールし、`CF_EMAIL`と`CF_API_TOKEN`をGitHubの環境変数としてセットします。


```yml
steps:
  - name: checkout
    uses: actions/checkout@v1
  - name: setup Node
    uses: actions/setup-node@v1
    with:
      node-version: 16.x
      registry-url: 'https://registry.npmjs.org'
  - name: install
    run: npm ci
  - name: install wrangler
    run: npm i @cloudflare/wrangler -g
  - name: set env
    run: |
      echo "CF_EMAIL=${CF_EMAIL}" >> $GITHUB_ENV
      echo "CF_API_TOKEN=${CF_API_TOKEN}" >> $GITHUB_ENV
    env:
      CF_EMAIL: ${{secrets.CLOUDFLARE_EMAIL}}
      CF_API_TOKEN: ${{secrets.WRANGLER_API_KEY}}
  - name: publish
    run: npm run deploy
```

`CF_EMAIL`はCloud Flareにログインする際のメールアドレスで、`CF_API_TOKEN`はCloud Flareの管理画面より取得できるAPIトークンです。

### APIトークンの発行

以下のページよりトークンを発行できます。
「Cloudflare Workers を編集する」というテンプレートを選択することで簡単にトークンを生成できます。

https://dash.cloudflare.com/profile/api-tokens

### Node.jsのライブラリによっては利用できない

これは完全に自分の調査不足だったのですが、Cloudflare WorkersはNodeとは異なるためNodeの標準ライブラリである、`fs`や`net/http`などのライブラリは使用できません。なので当然それらの標準ライブラリに依存するライブラリも利用できませんでした。

https://workers.cloudflare.com/works

> Many packages that require Node dependencies, such as fs or net/http, aren't supported in Workers because Workers isn't Node—it's built on V8.


## まとめ

以上3つが私がRemixを触った上でハマったことでした。Next.jsに慣れきっているので色々と苦労しますね。
またRemix on Cloudflare Workersの構成で困ったことがあったらこの記事に追記したいと思います。
