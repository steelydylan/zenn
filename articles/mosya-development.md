---
title: "【個人開発】模写を通してWeb制作を学ぶmosyaというサービスを開発した話"
emoji: "👨‍💻"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "typescript", "tailwind", "planetscale", "個人開発", "cloudrun", "cloudflare"]
published: false
---

今回、個人開発で1年もの歳月をかけて`mosya`というコーディング学習サービスを開発しました。
主なターゲットは`Web制作者`を目指している方で、`Progate`の次の学習に悩んでいる方や一からWeb制作を学びたい方、企業のWeb担当者の方などを想定しています。

https://mosya.dev/

## どんなサービスか

模写を通してWeb制作の基礎を学ぶmosyaというサービスを開発しました。
専用のエディター内蔵で実際に手を動かして見本を参考にしながら模写をすることで、
体系的にWeb制作を学ぶことができます。

操作感がわかりやすいように動画を用意しましたので、ぜひご覧ください。

@[youtube](https://youtu.be/Hz2Oz1GUKck)


## なぜ作ったのか

### 動画だけではなく手を動かして体系的に学べるサービスを作りたい

Web制作を学ぶ上ですでにたくさんの教材はあるのですが、部分的な知識を学ぶに過ぎない教材が多く、
実際に見本のサイトを完成させられるようになるまでには至らないと感じていました。
なので、なるべく最初は簡単な部品をHTML, CSSでコーディングしていくことでコーディングに慣れていき、
少しずつ難易度を上げていくような教材があればいいなと思い作成しました。

エディターが内蔵されているので、その場でHTMLとCSSのコーディング結果を確認でき見本と比べられるので
動画を見て学習するだけよりも知識がより深まりやすいと思います。

### 学習コストを下げたい

オンラインスクールや学習教材などは、月額数千円から数万円というものが多く、非常に高額なものが多いです。
また、学習教材は一度購入すると期限が切れてしまうものが多いです。
mosyaはなるべくコストを抑え、一度購入すればずっと使い続けられるようにしました。
また、学習教材を使い終わった後もマニュアルが充実しているので辞書がわりとして使うこともできます。

## 使用技術

さて、ここはzennなので、サービスの紹介ではなく、サービスを作る上で使用した技術について紹介します。


### サービス構成図

このサービスを作るにあたって以下のようなサービス構成にしました。

![](https://storage.googleapis.com/zenn-user-upload/2e01ebf382b0-20230503.png)

サービス構成図はこんな感じです。最初は一つのレポジトリで開発しようと思ったのですが、`Cloud Build`のデプロイ時間が長く
ストレスなので、ビルドに時間がかかる **ユーザーのコードを採点する** ためのサービスと **本体のサービス** 、**ユーザーへの連絡用のサービス** に切り分けました。

また、レポジトリも **教材管理用のレポジトリ(mosya-lessons)**、 **本体開発のためのレポジトリ(mosya)** 、 **その他インフラ周りのレポジトリ(mosya-infra)** に分けました。

何名かの方に教材部分を手伝ってもらったのですが、レポジトリを分けたことにより **mosya** 本体のコード内容を知らなくてもマークダウンを書くだけで教材を作成できるようになりました。
結果、学習コストを下げることができその人たちのオンボーディングの時間を短縮できたので、レポジトリを分けてよかったと思っています。


#### サービス関認証の実装

![](https://storage.googleapis.com/zenn-user-upload/daae5b234b94-20230503.png)

mosyaのユーザーが書いたコードのビジュアル採点部分は小さいマイクロサービスとなっており、これは本体のサーバーである`mosya`のCloud Runからしかアクセスできないようにしています。
`GCP`には **サービス間認証** のための機能が揃っているのでこちらが簡単に実現しました。

代替以下のような手順でサービス関連系の自走ができました。

プリンシパルの設定で、呼び出し先のCloud Runに呼び出し元のCloud RunのサービスアカウントのCloud Run起動元として設定する

![](https://storage.googleapis.com/zenn-user-upload/408ebcdfda77-20230309.png =300x)


呼び出し元のCloud Runで以下のように`google-auth-library`を使って、
`auth.getIdTokenClient`に呼び出し先のURLを指定して、`client`を取得し、以下のように`Authorization`ヘッダーなどの情報を取得


```js
import { GoogleAuth } from "google-auth-library"

const auth = new GoogleAuth();
const client = await auth.getIdTokenClient(process.env.SITE_COMPARE_API)
const headers = await client.getRequestHeaders()
```

:::message
同じProjectの場合は特に認証鍵をサーバーに設置しておく必要もなしで便利です。
:::

あとはこの情報を使ってアクセスするだけ

```js
const res = await superagent
  .post(process.env.SITE_COMPARE_API)
  .set(headers)
  .send()
```

#### Cloudflareの利用

![](https://storage.googleapis.com/zenn-user-upload/de72944ba42c-20230503.png)

Cloudflareは運用コストを下げるためにとても重宝します。例えば、Cloudflareのキャッシュ機能を使うことで、あまり更新がないAPIや画像、その他アセットなどのファイルをキャッシュすることで、直接、サーバーである`Cloud Run`が叩かれる機会を減らすことができます。

画像は、Cloudflareの`R2`を利用してそこにカスタムドメインを当てることでリクエスト時間を短縮することに成功しています。

#### PlanetScaleの利用

PlanetScaleは開発用と本番用でDBのブランチを切り替えることができるのでとても便利です。`staging`環境用の開発ブランチである`development`から本番環境のブランチである`master`にマージすることで、本番環境のDBにステージング環境のDBのスキーマを簡単に反映することができます。

`GitHub Actions`を使っているのですが、公式が出しているサンプルを利用することで簡単に実装できました。

https://github.com/planetscale/deploy-deploy-request-action

こんな感じで、自動でPlanet Scaleでスキーマのマージリクエストが作成されマージされるようになっています。

![](https://storage.googleapis.com/zenn-user-upload/e1137886971e-20230503.png)

ワークフローの中でスキーマの反映が行えるのは非常に便利ですね！

#### Github Actionsの利用

サービス本体は`Cloud Build`でデプロイしていますが、その他の部分では`Cloud Build`だとデプロイ時間が長くストレスなので、`Github Actions`を利用しています。

`GitHub Actions`を使えば`git`の操作が楽に行えるので教材の反映などは差分があったファイルから、教材の反映に必要な処理だけを抽出して必要に応じて`DB`に更新をかけたり`Cloudflare`の`R2`に画像をアップロードしたりしています。

`git`の差分の抽出にはシェルスクリプトをゴリゴリ書いても良かったのですが少し面倒だったのでGitHub Actionsに公開されているこちらのアクションを利用しました。

https://github.com/technote-space/get-diff-action/tree/main

こんな感じでGitHub Actions内で差分ファイルを簡単に取得することができます！
今回のプロジェクトでは差分ファイル一覧をとりあえず`.env`に書き出して利用しています。

```
echo "DIFF_FILES=${{ env.GIT_DIFF_FILTERED }}" >> .env
```

### コーディングの採点ロジック

今回サービスを作るにあたってコーディングの採点

特に採点部分は今回かなり力を入れました。
ユーザーが課題を提出してから採点までの時間が長すぎるとストレスを感じてしまうのでなるべく採点時間を短くするために以下の手段で実現しました。

#### ロジック採点

最初、`cypress` を使ってサーバー側でユーザーの提出したコードをテストしていたのですが、それだと採点に10秒以上かかってしまうことがわかりました。
そこで、ユーザーが提出したHTMLとCSSがコード的に間違っていないかをチェックするための採点は **フロントエンド** で行うことにしました。
それにあたって、ブラウザーでHTMLとCSSを気軽にテストできるツールがなかったので`jest`に記法が似た独自のテストツールを自作しました。

https://github.com/steelydylan/html-browser-tester

こんな感じでユーザーが提出したHTMLとCSSを渡すだけでフロントエンドでテストが完結できます。

```ts
import { BrowserTester } from 'html-browser-tester'

const html = `
  <!DOCTYPE html>
  <html lang="ja">
  <head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Hello</title>
    <style>
      h1 {
        color: #000;
      }
    </style>
  </head>
  <body>
    <h1>Title1</h1>
    <h2>Title2</h2>
  </body>
  </html>
`

const main = async () => {
  const browserTester = new BrowserTester({ html, width: 980, height: 980 })

  browserTester.test('h1,h2 textContent should have right textContent', async (_, doc) => {
    const h1 = doc.querySelector('h1')
    const h2 = doc.querySelector('h2')
    browserTester.expect(h1?.textContent).toBe('Title1')
    browserTester.expect(h2?.textContent).toBe('Title2')
  })

  browserTester.test('title should have right textContent', async (_, doc) => {
    const title = doc.querySelector('title')
    browserTester.expect(title?.textContent).toBe('Hello')
  })

  browserTester.test('h2 should have red text', async (window, doc) => {
    const h2 = doc.querySelector('h2')
    browserTest.expect(window.getComputedStyle(h2).color).toBe('rgb(255, 0, 0)')
  })

  const results = await browserTester.run()

  console.log(results)
  /*
   [
    { description: 'h1,h2 textContent should have right textContent', result: true },
    { description: 'title should have right textContent', result: true },
    { description: 'h2 should have red text', result: true }
   ]
  */
}

main()
```

この場合、テストをモロにコードに含めないといけなくなり大変なので文字列を以下のように評価できるメソッドも用意しました。

```ts
browserTest.evaluate(`
  test('h1,h2 textContent should have right textContent', async (_, doc) => {
    const h1 = doc.querySelector('h1')
    const h2 = doc.querySelector('h2')
    expect(h1?.textContent).toBe('Title1')
    expect(h2?.textContent).toBe('Title2')
  })

  test('title should have right textContent', async (_, doc) => {
    const title = doc.querySelector('title')
    expect(title?.textContent).toBe('Hello')
  })

  test('h2 should have red text', async (window, doc) => {
    const h2 = doc.querySelector('h2')
    expect(window.getComputedStyle(h2).color).toBe('rgb(255, 0, 0)')
  })
`)
```

このテストを教材ごとのDBに保存しておいて、ユーザーが提出したコードを評価するときにこのテストを実行して結果を返すようにしました。
このテスト結果だけをサーバーに送信するようにしています。

#### ビジュアル採点

mosyaには実際に書いたコードと模写対象のサイトの比較のためのビジュアル採点機能があります。
提出すると以下のような表示結果で採点画面が表示されます。

ピンク色になっている部分がユーザーが書いたコードと模写対象のサイトの差分になります。

![](https://storage.googleapis.com/zenn-user-upload/272ceb4fcd63-20230503.png)

見本のサイトは`mosya-lessons`レポジトリでのpushタイミングで`GitHub Actions`でスクリーンショットを撮ってそれを`Cloudflare Workers`の`R2`にあらかじめ保存しています。

一方ユーザー側のコードはHTML,CSSをサーバーに送信してそれを`R2`にアップロードし、アップロードされて生成されたWebサイトに`playwright`でアクセスしてスクリーンショットを撮っています。

https://playwright.dev/

そのスクリーンショットデータとあらかじめアップロードしておいた見本のスクリーンショットを`Resemble.js`で比較して差分を取得しています。

https://rsmbl.github.io/Resemble.js/

この差分データを再び`R2`にアップロードしてその結果をブラウザーに返却しています。

あらかじめ見本データをR2に保存しておくことでレスポンスにかかる時間を短縮しています。

### エディターの開発

またユーザーがコーディングに使うエディターには`monaco-editor`を使っています。
VisualStudioCodeにとてもよく似たエディターで、VSCodeの拡張機能をそのまま使えるのでとても便利です。

https://microsoft.github.io/monaco-editor/

例えばこの`monaco-editor`に`markup-lint`や`emmet`などのプラグインを入れることでユーザーにコーディング体験を向上させています。

#### Markuplint

Markuplintはユーザーが書いたHTMLがマークアップ的に正しいかをチェックしてくれるツールです。

https://markuplint.dev/ja/

mosyaではこのMarkuplintを後半のレッスンで使うためにエディターに組み込んでいます。
長くなるので省略しますが代替以下のような感じで`monaco-editor`に`Markuplint`を組み込んでいます。

```ts
// ユーザーの入力したコードをlinterにかけてエラーを取得する
linter.setCode(value)
// レポートを生成
const reports = await linter.verify();
const diagnotics = await diagnose(reports);
const model = monacoEditorRef.current.getModel();
// レポート結果をmonaco-editorに反映
monaco.editor.setModelMarkers(model, 'markuplint', diagnotics);
```

リンターのルールセットにはMarkuplintのおすすめのルールセットがあるのでそれを少し上書きして使っています。

```ts
import ruleset from "@markuplint/ml-core/markuplint-recommended.json"
```

#### Emmet

ご存知だとは思いますが、EmmetはHTML,CSSを簡単に書くためのツールです。例えば、`div>ul>li*3`と書くと以下のようなHTMLが生成されます。

```html
<div>
  <ul>
    <li></li>
    <li></li>
    <li></li>
  </ul>
</div>
```

こんな感じのコードを書けばこれも簡単に`monaco-editor`に組み込むことができます。

```ts
import { emmetCSS, emmetHTML } from "emmet-monaco-es/dist/emmet-monaco.esm"

// monaco-editorのローダーからmonacoを取得
const monaco = await loader.init()
emmetHTML(monaco)
emmetCSS(monaco)
```

`monaco-editor`は拡張性が高いのがいいですね！

### プレビュー画面の開発

mosyaでは実際にユーザーが書いたコードが反映されるプレビュー画面に非常に力を入れました。
これは完成系のサイトと現在自分がコーディング中のサイトを見比べるUIです。

![](https://storage.googleapis.com/zenn-user-upload/b79677efa2cf-20230503.png)

こちらの`CSS Battle`というサイトがこの見比べるUIを採用していたので模写学習には非常にいいUIだなと思い採用しました。

https://cssbattle.dev/

#### sandpack利用の挫折

プレビュー画面の生成には最初`CodeSandbox`が開発した`Sandpack`というライブラリを使っていました。

https://sandpack.codesandbox.io/

ただ、こちらのライブラリは、`React`を書くのには向いていたのですが、`HTML`や`CSS`を書いた際にその結果を即座に反映しライブリロードするような仕組みは整っていなかったのでこちらは断念しました。


#### iframeのsrcdocの利用の挫折

次にiframeの`srcdoc`の利用を検討しました。`srcdoc`を使うと`srcdoc`に`HTML`を文字列として代入するだけで`iframe`の中にその内容が表示されて非常に便利です。
ところが、こちらユーザーが書いたcssを`link`タグとして読み込むことができないので学習体験が悪いと思い断念しました。

#### Blob化してURLとして読み込む方向で調整

最終的にユーザーが書いたHTML、CSSをBlob化してそれを`URL.createObjectURL`を使って一時的に`URL`を作成しそれをiframeで読み込む形に落ち着きました。

```ts
const jsBlob = new Blob(
  [script],
  { type: "text/javascript" }
)
const cssBlob = new Blob(
  [css],
  { type: "text/css" }
)

const jsBlobUrl = URL.createObjectURL(jsBlob)
const cssBlobUrl = URL.createObjectURL(cssBlob)
```

ユーザーが書いたHTMLのコードを正規表現で置換し、`script.js`と`style.css`の読み込みがあったらその読み込みを発行した一時的なURLに置き換えています。

大体こんな感じです👇

```ts
code
  .replace(/src=("|')(\.\/)?script\.js("|')/, `src="${jsBlobUrl}"`)
  .replace(/href=("|')(\.\/)?style\.css("|')/, `href="${cssBlobUrl}" crossorigin="anonymous"`)
```

こうすることによりユーザーにとって本当にリアルなコーディング体験を提供できるようになりました。

### Next.jsの利用

今回費用の関係で`Next.js`を`Vercel`ではなく`Cloud Run`にデプロイする形で運用しているのですが、そのAPI部分もそのまま`Next.js`の`API Routes`を使っています。
ただ、`Next.js`の`API Routes`はとても簡素で、クライアントからサーバーサイドの型情報を取得したりあらかじめ`zod`でバリデーションをかけた上で`ts`の型をAPI内で効かせることが難しかったのでそこの部分も別にライブラリを作りました。

以下のサイトのようにAPIで書いたコードがクライアント側のコードにも型として反映されるようになっていて非常に開発が捗りました。

https://stackblitz.com/edit/next-typescript-32qrbx?embed=1&file=pages/index.tsx&file=pages/api/sample/%5Bid%5D.ts&hideNavigation=1&view=editor

その件については別で`zenn`に記事を書いてます。

https://zenn.dev/steelydylan/articles/next-typed-connect

## まとめ

このように`mosya`にはユーザーに喜んでいただけるような体験を提供するための工夫が詰まっています。
またWeb歴が長い方でも、もしかしたら初耳かもしれない新しいCSSやHTMLも紹介してたりするので良かったらぜひ使ってみてください！

もう一度mosyaのリンクを貼っておきます！

https://mosya.dev/