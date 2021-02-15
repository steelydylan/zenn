---
title: "Next.jsベースの静的サイトジェネレーターNextraが非常に便利だった話"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "markdown", "next"]
published: true
---

[steelydylan](https://twitter.com/steelydylan)です。たまにOSSを開発する機会があるのですが、以前OSSを公開する時に[VuePress](https://vuepress.vuejs.org/)のような気軽な静的サイトジェネレーターがReactにもあったらいいのにと探していました。
その時に`Nextra`というとても素敵なツールを見つけたので紹介させてください。

https://github.com/shuding/nextra

# Nextraとは

Nextraの公式サイトに行けばわかることですが日本語で簡単に解説させてください。
Nextraは、Next.jsとMDXで動作するスタティックサイトジェネレーターです。実際にNext.jsを開発しているVercelで働かれている[shuding](https://github.com/shuding)という方が作られているので信頼性は高そうです。

例えば、Reactのデータフェッチ用のhooksであるSWRというライブラリのサイトもNextraで作られています。

https://swr.vercel.app/

Nextraを使えば、Next.jsを知っている人はさらに便利に使えますが知らなくてもノーコードでマークダウンをかけるだけでドキュメントをかけちゃいます。自分もNextraを使って、以前公開したOSSの紹介サイトとしてNextraを使ってますが、ほぼほぼノーコードでした！

https://react-split-mde.vercel.app/

また、Next.jsで動作するため、Vercelを使えばGithubにpushするだけでドキュメントが公開できるというメリットもあります。

https://vercel.com/


早速使い方を紹介しちゃいます！


# インストール

```sh
$ yarn add next react react-dom
$ yarn add nextra nextra-theme-docs
```

## セットアップ

まずは、脳死でnext.config.jsに以下の様に記述します。これでNext.js上でnextraを動かす環境が調います。

```js:next.config.js
// next.config.js
const withNextra = require('nextra')('nextra-theme-docs', './theme.config.js')
module.exports = withNextra({
  webpack: config => {
    config.module.rules.push({
      test: /\.txt$/i,
      use: "raw-loader",
    })
    return config
  }
})
```

次に、公開したいサイトのメタ情報などを `theme.config.js` に記述します。以前私が公開した、[React Split MDE](https://github.com/steelydylan/react-split-mde)のテーマ設定は以下の様な感じでした。

```jsx
// theme.config.js
export default {
  repository: 'https://github.com/steelydylan/zenn-mde', // project repo
  docsRepository: 'https://github.com/steelydylan/zenn-mde-site', // docs repo
  branch: 'master', // branch of docs
  path: '/', // path of docs
  titleSuffix: ' – React Split MDE',
  nextLinks: true,
  prevLinks: true,
  search: true,
  customSearch: null, // customizable, you can use algolia for example
  darkMode: true,
  footer: true,
  footerText: 'MIT 2020 © steelydylan.',
  footerEditOnGitHubLink: true, // will link to the docs repo
  logo: <>
    <svg>...</svg>
    <span>React Split MDE</span>
  </>,
  head: <>
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <meta name="description" content="React Split MDE: the next markdown editor" />
    <meta name="og:title" content="React Split MDE: the next markdown editor" />
  </>
}
```

また、nextraのCSSを適用したい方は`pages/_app.js`に以下の様に記述します。

```jsx
import 'nextra-theme-docs/style.css'

export default function Nextra({ Component, pageProps }) {
  return <Component {...pageProps} />
}
```

あとは、package.jsonの`scripts`に以下の様な記述を追加し、`yarn start`とコマンドを入力することで確認しながら記事が書けます！

```json: package.json
...
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "dev": "next",
    "start": "next start",
    "build": "next build"
  },
...
```

# 記事を書く

`pages/**/***.mdx` としてマークダウンファイルを設置します。拡張子が `mdx`であることに要注意です。通常、Next.jsでは、jsxやtsxなどのファイルを`pages`以下に配置すると思うのですが、この`mdx`がページの代わりとなるイメージです。この`mdx`にマークダウンを記述することでそのままそのファイル名がURLとなります。たとえば、`pages/advanced.mdx`という名前でファイルを設置すると、https://example.com/advanced というURLで記述したマークダウンの内容が確認できます。
https://example.com にあたるURLには `pages/index.mdx` としてファイルを設置します。

## Frontmatter
また、以下の様なFrontmatterもマークダウンの先頭に記述することも可能です。

```
---
title: Introduction
---
```

また以下の様にFrontmatterに `full: true`と記述し、ドキュメント幅を調整することも可能です。

```
---
title: Introduction
full: true
---
```

## JSX

またmdxでは以下の様にJSも普通に記述できるので、Next.jsなら`SSR`や`ISR`といったことも簡単にできちゃいます。
これは感動です。。。

```jsx
import Markdown from 'markdown-to-jsx'
import { useSSG } from 'nextra/ssg'

export const getStaticProps = ({ params }) => {
  return fetch('https://api.github.com/repos/steelydylan/react-split-mde/releases')
    .then(res => res.json())
    // we keep the most recent 5 releases here
    .then(releases => ({ props: { ssg: releases.slice(0, 5) }, revalidate: 10 }))
}

export const ReleasesRenderer = () => {
  const releases = useSSG()
  return <Markdown>{
    releases.map(release => {
      const body = release.body
        .replace(/&#39;/g, "'")
        .replace(/@([a-zA-Z0-9_-]+)(?=(,| ))/g, '<a href="https://github.com/$1" target="_blank" rel="noopener">@$1</a>')
      return `## <a href="${release.html_url}" target="_blank" rel="noopener">${release.tag_name}</a> 
Published on ${new Date(release.published_at) .toDateString()}.\n\n${body}`}).join('\n\n')
  }</Markdown>
}

# Change Log

Please visit the [React Split MDE](https://github.com/steelydylan/react-split-mde/releases) for all history releases.

<ReleasesRenderer/>
```

# 記事の順番を決める

`meta.json`を`pages`以下のディレクトリに設置することで記事の順番を決めることができます。
キー名に記事のスラグ、バリューに記事のタイトルといった具合です。

```json:meta.json
{
  "index": "Introduction",
  "getting-started": "Getting Started",
  "docs": "Docs",
  "advanced": "Advanced",
  "change-log": "Change Log"
}
```

# まとめ

以上、Nextraの紹介でした。自作のOSSのドキュメントをサクッと作りたい時などに非常に重宝すると個人的には思っています。今後、OSSのドキュメント作成が今以上に捗りそうです！

また、今回のドキュメントはGithubに公開しているので参考にしたい方はどうぞ

https://github.com/steelydylan/react-split-mde-site

