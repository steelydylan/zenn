---
title: "Next.jsで多言語対応のサイトを作るのが簡単すぎた件"
emoji: "🗺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "i18n"]
published: true
---

## Next.jsで多言語対応を試みた経緯

以前趣味でブログのRSSを登録するとブログの投稿率をGitHubの草のようなヒートマップ形式で表示でき他のユーザーと継続率を競えるサービス、[Blog Friends](https://blog-friends.com)を開発しました。

今回のこのサービスを[ProductHunt](https://www.producthunt.com/)（海外の自分が作ったWebサービスやアプリを投稿できるサイト）に提出しようと思い英語対応をしました。

日本語サイトがこちらで、

https://blog-friends.com/

英語サイトはこちらです。

https://blog-friends.com/en

ProductHuntに提出してみた結果は以下のように12Upvotedでなんとも言えない結果でしたが、多言語サイトを作る上で勉強になったので後悔はしていません。

https://www.producthunt.com/posts/blog-friends


## Next.jsで多言語サイトを作る方法

### ルーティングを理解する

公式サイトにもあるのですが、Next.jsで多言語サイトを作る方法として以下の二つのルーティング方法があります。

https://nextjs.org/docs/advanced-features/i18n-routing

- Domain Routing
- Sub-path Routing

#### Domain Routing

ドメインごとに言語を切り替えるルーティング。例えば、

`example.com/blog` では日本語を表示し、`example.en/blog` では英語を表示する方法

以下のようにnext.config.jsで言語ごとのドメインを設定します。

```js
// next.config.js
module.exports = {
  i18n: {
    locales: ['en', 'ja'],
    defaultLocale: 'ja',
  },
  domains: [
    {
      domain: 'example.com',
      defaultLocale: 'ja',
    },
    {
      domain: 'example.en',
      defaultLocale: 'en',
    },
  ]
}
```

#### Sub-path Routing

ドメインではなくパスで言語を切り替える方法。

例えば、`example.com/en` 以下では英語を表示するようなイメージです。

今回、私はドメインを余分に用意する余裕もないので、Sub-path Routingを採用しました。

Sub-path Routingの場合はnext.config.jsに以下のように設定するだけです。

```js
// next.config.js
module.exports = {
  i18n: {
    locales: ['en', 'ja'],
    defaultLocale: 'ja',
  },
}
```

これだけで自動でサイトの`/en`以下にアクセスされた場合はロケールが`en`判定されます。

### ロケールごとに言語を返すhooksを作成

ルーティングの理解が済んだら次にロケールごとにその言語セットを返すhooksを作ってみます。

この記事がとても役に立ちました。
https://zenn.dev/yuyao17/articles/5cfa65d7e7eb11

10分とは言いませんが、おかげで数時間でサイトを英語対応させることができました。

以下のように`useRouter`から`locale`を取り出して、それぞれのlocaleに応じて翻訳セットを返すhooksを作っておきます。

```ts
import { useRouter } from 'next/router'
import en from "../locales/en";
import ja from "../locales/ja";

export const useLocale = () => {
  const { locale } = useRouter();
  const t = locale === "en" ? en : ja;
  return { locale, t };
}
```

```ts:en.ts
export default {
  RANKED_HIGH_CONDITION: "Among all bloggers, the users who are doing their best to update the blog are ranked high.",
  UNCATEGORIZED_RANKED_HIGH_CONDITION: "Among Uncategorized bloggers, the users who are doing their best to update the blog are ranked high.",
  SEE_ALL_BLOGGERS: "See all bloggers",
  ...
}
```

```ts:ja.ts
export default {
  RANKED_HIGH_CONDITION: "全ブロガーの中で頑張ってブログを更新しているユーザーを上位表示します",
  UNCATEGORIZED_RANKED_HIGH_CONDITION: "未分類のブロガーの中で頑張ってブログを更新しているユーザーを上位表示します",
  SEE_ALL_BLOGGERS: "全ブロガーをもっとみる",
  ...
}
```

お陰で以下のようなhooksの使い方で簡単にそれぞれの言語に翻訳することができました。

```tsx
const SignupButton = () => {
  const { t } = useLocale()
  return (<Button>{t.SIGNUP}</Button>)
}
```

### ISRやSSGの時の対策

getStaticPropsを使ってる場合は以下のようにpathsを余計に返すことに注意する必要がありました。

```js
// pages/blog/[slug].js
export const getStaticPaths = ({ locales }) => {
  return {
    paths: [
      // if no `locale` is provided only the defaultLocale will be generated
      { params: { slug: 'post-1' }, locale: 'en' },
      { params: { slug: 'post-1' }, locale: 'ja' },
    ],
    fallback: true,
  }
}
```

以下のように初めに一つの言語でpathを作っておいてそれを元に別の言語を複製する方法が楽でおすすめです。

```ts
export const getStaticPaths: GetStaticPaths = async () => {
  const paths = categories.map(c => ({
    params: {
      slug: c.slug as string,
    },
    locale: 'ja',
  }))
  // 英語に対してもpathを作成
  paths.push(...paths.map(p => ({ ...p, locale: 'en' })))

  return {
    paths,
    fallback: false
  }
}
```

### 言語切り替え

また英語ユーザーの人が誤って日本語環境に訪れてしまった場合にスムーズに言語を切り替えられるよう以下のような言語切り替えメニューを用意しました。
![](https://storage.googleapis.com/zenn-user-upload/fbe5a337082871f7591b0009.png)

言語切り替えは`next/link`に対して以下のように`locale`を指定することで簡単に行うことができます。

```tsx
<Link href="/" locale="en" passHref>...</Link>
```

例えば、上記のリンクでは、`/en`に遷移します。

```tsx
<Link href="/categories/all" locale="en" passHref>...</Link>
```

また、上記の場合では、`/en/categories/all`に遷移します。

`locale`を指定しない`Link`は現在のlocaleを考慮して遷移します。
例えば、`en`の`locale`にいるユーザーが下記のようなリンクを踏んだ場合、

```tsx
<Link href="/categories/all" passHref>...</Link>
```

`/en/categories/all`に遷移します。

つまり普段は`Link`で`locale`を意識する必要はなく、ロケールを跨った画面遷移をしたい場合には`locale`を`Link`で指定することになります。

## まとめ

Next.jsでは特別プラグインやライブラリを導入しなくても既にi18nが考慮された設計がされていてとても多言語対応が楽でした。
もし、Next.jsで作ったサイトを多言語対応する必要が出てきた際にはこちらの記事を思い出していただけると幸いです。
