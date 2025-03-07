---
title: "僕のElectronアプリアーキテクチャ"
emoji: "👨‍💻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["electron", "wasm", "react", "postgresql", "trpc", "個人開発"]
publication_name: "progate"
published: true
---

## はじめに

最近、趣味でElectronを使ったアプリを開発しています。その過程で、アーキテクチャ設計についていくつか悩んだポイントがありました。本記事では、実際に採用したアーキテクチャや解決策を紹介します。
今回はサーバーにデータを置かないローカルファーストなデスクトップアプリを作っているということもあり、特に以下の点で悩みました。

- フレームワークどうする？
- WebのようなREST APIが使えない
- データベースどうしよう
- アプリのアップデートどうしよう
- アプリのコード署名むずい

## フレームワークどうする

まずはじめにフレームワークをどうするかという問題がありました。

最初はNext.jsのApp Routerを使ってアプリを作っていました。しかし、Electronアプリは最終的に静的ファイルになるため、Next.jsのServer ComponentsやServer Actionsの恩恵を受けられないという問題がありました。

そのため、Next.jsを使わずにVitest + React Routerで頑張ることにしました。

最終的にElectronではファイルを参照するためURLはfile://になるのですが、React Routerの場合、ハッシュタグに対応した、HashRouterを使うことで、Electronでも快適にページ遷移ができるようになりました。

Electronアプリの場合はNext.jsのようなガッチリしたフレームワークを使うよりも、Vite + React Routerのような軽量なフレームワークを使う方が良いと思います。

## WebのようなREST APIが使えない

次に悩んだのは、ElectronアプリはWebアプリと違い、REST APIが使えないということです。
開発中は、ローカルサーバーを立てて、そこにリクエストを送ることでAPIを使うことができますが、リリース後は、ローカルサーバーを立てることができないので、REST APIを使うことができません。

そこで、`electron-trpc`というライブラリを使って、Electronアプリ内で`tRPC`を使うことにしました。

`tRPC`とは、TypeScriptで書かれた型安全なRPCライブラリで、サーバーとクライアントの間で型安全な通信を行うことができます。

https://trpc.io/

`tRPC`は普通のWebアプリケーションだけの用途に閉じず、いろんな環境間での通信に使えるので、Electronアプリでも使いやすいです。

下のようにmainプロセスで`tRPC`のサーバーを立てて、rendererプロセスで`tRPC`のクライアントを使うことで、Electronアプリ内でAPIを使うことができます。

```ts:main.ts
import { createIPCHandler } from 'electron-trpc/main'
import { initTRPC } from '@trpc/server'
import { z } from 'zod'
// 省略
export const t = initTRPC.create({ isServer: true })
const router = t.router({
  hello: t.procedure.input(z.object({ name: z.string() })).query(async ({ name }) => {
    return `Hello, ${name}!`
  }),
})

createIPCHandler({ router, windows: [mainWindow] })
```

また、rendererプロセスで、tRPCクライアントを使えるようにするために、`preload.ts`で`exposeElectronTRPC`を実行します！

```ts:preload.ts
import { exposeElectronTRPC } from 'electron-trpc/main'
process.once('loaded', async () => {
  exposeElectronTRPC()
})
```

クライアント側は以下のような感じです。

僕は`tanstack`の`react-query`を使っているので、`react-query`と`tRPC`を組み合わせるために、`@trpc/react-query`というライブラリを使っています。

```ts:renderer/index.ts
import React, { useState } from 'react'
import { ipcLink } from 'electron-trpc/renderer'
import { createTRPCReact } from '@trpc/react-query'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import type { AppRouter } from '@main/api'

export const trpcReact = createTRPCReact<AppRouter>()

export function TrpcProvider({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(() => new QueryClient())
  const [trpcClient] = useState(() =>
    trpcReact.createClient({
      links: [ipcLink()]
    })
  )

  return (
    <trpcReact.Provider client={trpcClient} queryClient={queryClient}>
      <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
    </trpcReact.Provider>
  )
}
```

## データベースどうしよう

Electronアプリで複雑なデータ管理をしたい場合どうするか悩みました。
例えばテーブルの結合や、unique制約やその他の機能を使いたい場合、`IndexedDB`や`localStorage`では難しいです。
そこで、WASMの力を借りて、`@electric-sql/pglite`というPostgreSQLをブラウザで使えるライブラリを使うことにしました。

https://electric-sql.com/

今回はブラウザーというよりもmainプロセスでこのライブラリを利用しています。
WASMのいいところはポータブルな技術なので、Macアプリだけじゃなく、Windowsアプリでも正常に動作することです。
環境差異を気にせず気軽に導入できるのがWASMのいいところです。

`@electric-sql/pglite`を使うと、PostgreSQLのようなデータベースを使うことができます。

```ts:main/db.ts
import { PGlite } from '@electric-sql/pglite'
import { drizzle } from 'drizzle-orm/pglite'
import { app } from 'electron'
import path from 'path'

const isDev = process.env.NODE_ENV === 'development'
const dbName = isDev ? 'db-dev' : 'db'

const dbPath = path.join(app.getPath('userData'), dbName)
const client = new PGlite(dbPath)

export const db = drizzle(client)
```

dbはアプリが利用している`userData`の中に物理ファイルとしてデータが書き込まれる点がポイントです。

また、開発中のdbと本番のdbが衝突する問題があったので、`isDev ? 'db-dev' : 'db'`というように、開発中と本番でdb名を変えるようにしています。

### マイグレーション

問題は、データベースのマイグレーションです。スキーマを変えたくなった場合にWebではなく、アプリ内でデータベースのマイグレーションを行う必要があります。

そこで、アプリ起動時に必ずマイグレーションが実行されるようにしました。

```ts:main.ts
app.whenReady().then(async () => {
  await runMigrate()
})
```

`runMigrate`は以下のように書いています。

```ts:main/db/migrate.ts
import { migrate } from 'drizzle-orm/pglite/migrator'
import { db } from './db'
import path from 'path'

export const runMigrate = async () => {
  console.log('⏳ Running migrations...')
  const start = Date.now()
  await migrate(db, { migrationsFolder: path.join(__dirname, '../../drizzle') })
  const end = Date.now()
  console.log('✅ Migrations completed in', end - start, 'ms')
}
```

`drizzle-orm/pglite/migrator`を使うことで、mainプロセスでマイグレーションが実行できるようになりました。
ちゃんと物理ファイルとして定義したスキーマが書き込まれます！

ただし、`drizzle`で開発者は事前に`npm run db:generate`を実行して、`drizzle`のスキーマを生成しておく必要があります。

`drizzle-kit`というライブラリがあるので、これを使うと、`drizzle`のスキーマを簡単に生成できます。

```json:package.json
{
  "scripts": {
    "db:generate": "drizzle-kit generate"
  }
}
```

drizzle.config.tsはこんな感じ

```ts
import { defineConfig } from 'drizzle-kit'

export default defineConfig({
  schema: './src/main/db/schema.ts',
  out: './drizzle',
  dialect: 'postgresql',
  dbCredentials: {
    url: 'database.sqlite'
  },
  driver: 'pglite'
})
```

## アプリのアップデートどうしよう

アプリをApp Storeにリリースする場合は、App Storeの仕組みを使ってアプリのアップデートを行うことができますが、
自分で配布する場合は、アプリのアップデートの仕組みを自分で実装する必要があります。

### electron-updaterの利用

Electronアプリのアップデートは、`electron-updater`の`autoUpdater`を使うことで実装できました。

GitHub Actionsでアプリをビルドして、ビルドしたら特定のURLにアップロードするようにして、アプリをアップデートできるようにしました。

`electron-builder.yml`に正しい情報を書いておかないと、アップデートができないので地味に調整が大変でした。

S3などにアップする場合はproviderとして`s3`を使うことができますが、自分は`generic`を使って、特定のURLにアップロードしています。（R2）

```yml:electron-builder.yml
publish:
  - provider: generic
  - url: https://example.com/
build:
  "afterSign": "scripts/notarize.js"
```

### アップデート情報の保存

そして、アプリがアップデートされた場合は、アップデート情報をデータベースに保存することに成功しました。

```ts
export async function setUpdateInfo({
  version,
  releaseDate,
  releaseNotes
}: {
  version: string
  releaseDate: string
  releaseNotes: string
}) {
  try {
    await db.delete(update)
    return await db.insert(update).values({
      version,
      releaseDate,
      releaseNotes
    })
  } catch (error) {
    console.error('Failed to set update info in database')
    throw error
  }
}

mainWindow.once('ready-to-show', async () => {
  // ★★★最新バージョンがあるか、チェックしてダウンロード★★★
  await autoUpdater.checkForUpdates()
})
autoUpdater.addListener('update-downloaded', (e) => {
  setUpdateInfo({
    version: e.version,
    releaseDate: e.releaseDate,
    releaseNotes: (e.releaseNotes ?? '') as string
  })
  return true
})
```

### アップデートの実行

そして、rendererプロセスから、アプリを更新したいというメッセージを埋めとった際にこのようにアップデートを実行するようにしています。

```ts:main.ts
ipcMain.handle('update', () => {
  if (!mainWindow) return
  // ダイアログを表示して更新があることを伝える
  const result = dialog.showMessageBoxSync(mainWindow, {
    type: 'info',
    buttons: [i18n.t('update.CANCEL'), i18n.t('update.CONTINUE')],
    defaultId: 1,
    title: i18n.t('update.TITLE'),
    message: i18n.t('update.MESSAGE'),
    detail: i18n.t('update.DETAIL')
  })
  // アプリを終了してインストール
  if (result === 1) {
    autoUpdater.quitAndInstall()
  }
})
```

## アプリのコード署名むずい

最後に苦労したのがアプリのコード署名です！せっかくアプリを作ったのに、コード署名がないと、ユーザーがアプリをインストールする際に警告が出てしまいます。
Macの場合、警告どころか、インストールしようとするとセキュリティの設定でブロックされて即ゴミ箱送りにされてしまうので、コード署名は必須です。

コード署名は、Apple Developer Programに登録して、証明書を取得する必要があります。

### Apple Developer Programに登録

年間$99でApple Developer Programに登録することでAppleのコード署名を受けることができます。

登録後、証明書をDeveloper -> Certificates, Identifiers & Profiles よりダウンロードできます！
とりあえず、`Mac Development`を選択します！もし、Mac App Storeにリリースする場合は、`Mac App Distribution`を選択しましょう！

![](https://storage.googleapis.com/zenn-user-upload/dbc7ef2cde78-20250307.png)

### 証明書をキーチェーンにインストール

証明書をダウンロードしたら、ダブルクリックしてキーチェーンにインストールします！
インストールしたら、キーチェーンアクセスを開いて、証明書を右クリックしてエクスポートします！
形式は`p12`でパスワードを設定して書き出します。

![](https://storage.googleapis.com/zenn-user-upload/dfa01511c49e-20250307.png)


### Apple Accountでアプリのパスワードを設定

次に、Apple Developer Accountにログインして、`App-specific password`を設定します！

https://account.apple.com/

ここからアプリのパスワードを設定します！

![](https://storage.googleapis.com/zenn-user-upload/93e78c22fb7b-20250307.png)

設定したパスワードは後で、GitHub ActionsのSecretsに`APPLE_APP_SPECIFIC_PASSWORD`として登録します！

### notarize.jsを作成

次に署名に必要な`notarize.js`を作成します！
下のように書いて、`electron-builder.yml`の`afterSign`に指定します！

```ts:notarize.js
const { notarize } = require('electron-notarize');

exports.default = async function notarizing(context) {
  const { electronPlatformName, appOutDir } = context;
  if (electronPlatformName !== 'darwin') {
    return;
  }

  const appName = context.packager.appInfo.productFilename;

  return await notarize({
    appBundleId: '***.***.***',
    appPath: `${appOutDir}/${appName}.app`,
    appleId: process.env.APPLE_ID,
    appleIdPassword: process.env.APPLE_APP_SPECIFIC_PASSWORD,
    teamId: process.env.APPLE_TEAM_ID
  });
};
```

### GitHub Actionsでのコード署名

次にこの証明書をGitHub Actionsで使えるようにします！
先ほど書き出した`p12`ファイルをBase64エンコードして、GitHubのSecretsに登録します！

```sh
base64 -i distribution.p12 | pbcopy
```

このようにクリップボードにコピーしたBase64エンコードされた文字列をGitHubのSecretsに`CERTIFICATES_P12`として登録します！先ほど設定したパスワードは`CERTIFICATES_P12_PASSWORD`として登録します！
さらに、


::: details GitHub Actionsでのコード署名の設定
```yml
name: Distribute App

on:
  workflow_dispatch:

jobs:
  build:
    name: Build and Distribute App
    runs-on: macos-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3
        with:
          fetch-depth: 2

      - name: Install dependencies
        run: npm install

      - name: Apple Codesigning
        uses: apple-actions/import-codesign-certs@v1
        with:
          p12-file-base64: ${{ secrets.CERTIFICATES_P12 }}
          p12-password: ${{ secrets.CERTIFICATES_P12_PASSWORD }}

      - name: Build for Mac
        run: npm run build:mac
        env:
          GH_TOKEN: ${{ secrets.GH_TOKEN }}
          APPLE_ID: ${{ secrets.APPLE_ID }}
          APPLE_APP_SPECIFIC_PASSWORD: ${{ secrets.APPLE_APP_SPECIFIC_PASSWORD }}
          APPLE_TEAM_ID: ${{ secrets.APPLE_TEAM_ID }}

      - name: Delete Unnecessary files (macOS)
        run: |
          rm -rf dist/mac
          rm -rf dist/mac-arm64
          rm -rf dist/mac-universal
```

最初このActionsを実行した際に実際にAppleによってコードサインされるまでに20分ほどかかりましたが、その後はアプリのアップデートがスムーズに行えるようになりました！

## まとめ

ローカルファーストなデスクトップアプリを作るために色々と試行錯誤しましたが、最終的にはElectron + Vite + React + tRPC + PGLiteでアプリを作ることができました！
今後の僕の備忘録のため、そして、同じような悩みを持つ人の参考になればと思い、この記事を書きました！

ぜひ、参考にしてみてください！