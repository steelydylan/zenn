---
title: "Next.jsのApp Routerで頑張らないi18n対応" 
emoji: "📔"
type: "tech" # tech: 技術記事 / idea: アイデア
publication_name: "progate"
topics: ["nextjs", "i18n", "react", "typescript"]
published: true
published_at: 2025-06-16 07:30 #公開日時の設定。
---

## はじめに

以前、こちらの記事を書いた際はPages Routerが前提だったのでまだ自前でi18n対応するのは簡単でした

https://zenn.dev/steelydylan/articles/nextjs-with-i18n

最近はNext.jsのAppRouterでWebアプリを作るので、作ったサイトを国際化対応したのですが、ライブラリの選定に悩みました。

最初はnext-intlやnext-i18n-routerといった定番ライブラリを検討していたのですが、
設定ファイルの量が意外とあったり、各ページで工夫が必要だったり...hooksが前提だったりみたいな感じで、
めんどくさそうだなと感じて諦めました。

https://next-intl.dev/

https://github.com/QuiiBz/next-international

https://github.com/i18nexus/next-i18n-router

このNext.jsの公式サイトで紹介されているミニマムな方法も、サーバーコンポーネントのPageコンポーネントから頑張ってdictionaryを子供のコンポーネントに渡す必要があったり、クライアントコンポーネントでの利用が面倒だったりと、なんだか複雑な印象を受けました。

https://nextjs.org/docs/app/guides/internationalization


途中で、「自分で実装した方がシンプルなんじゃないか？」と思い結局Pages Routerの時と同じように、ライブラリを使わずにi18n対応を実装することにしました。

結果的に、少しの実装でライブラリを使わずに多言語対応が実現できたので、その手法を共有したいと思います。

## 全体のアーキテクチャ

まず、どんな構成になるかを説明します。

```
app/
├── [locale]/        # 動的ルーティングで言語切替
│   ├── layout.tsx   # 言語別レイアウト
│   └── page.tsx     # 各ページ
└── middleware.ts    # 言語判定とリダイレクト
next.config.ts       # URL正規化の設定
```

対応言語は日本語（ja）、英語（en）、中国語（zh）の3つです。
英語をデフォルト言語として、URLにプレフィックスを付けない設計にしました。

例えばdocsページのURLは以下のようになります！

```
英語: aidy.jp/docs
日本語: aidy.jp/ja/docs
中国語: aidy.jp/zh/docs
```

英語ユーザーにはそのままのURLを提供しつつ、他言語ユーザーにも分かりやすいURL構造になっています。

## 実装の詳細

### next.config.ts - URL書き換えの設定

ここが最も重要な部分です。`rewrites`機能を使って、外部からは `/docs`に見えるURLを、内部的には `/en/docs`として処理します。

```ts
const nextConfig: NextConfig = {
  rewrites: async () => [
    { source: '/docs/:path*', destination: '/en/docs/:path*' },
    { source: '/dashboard/:path*', destination: '/en/dashboard/:path*' },
    { source: '/', destination: '/en' }
  ];
  // 以下省略
}
```

この設定により、既存のURLに影響なく、段階的に翻訳が終わったページを随時切り替えることができます。

### middleware.ts - 言語検出とリダイレクト

ブラウザの`Accept-Language`ヘッダーを解析し、ソーシャルメディアクローラーを考慮した上で、適切な言語ページにリダイレクトする処理例です。

ちょっとここだけ、いろんな考慮があるので長いです。。。

```ts
import { NextResponse, type NextRequest } from "next/server";
import { auth } from "@/auth";

// サポートするロケール
const locales = ['ja', 'en', 'zh'];
const defaultLocale = 'en';

// ソーシャルメディアクローラーのユーザーエージェント
const socialMediaCrawlers = [
  'facebookexternalhit', 'Facebot', 'Twitterbot', 'LinkedInBot', 'Pinterest',
  'Slackbot', 'Discordbot', 'WhatsApp', 'Googlebot', 'bingbot', 'Baiduspider', 'Yahoo'
];

// クローラー判定
function isSocialMediaCrawler(request: NextRequest) {
  const ua = request.headers.get('user-agent') || '';
  return socialMediaCrawlers.some(crawler => ua.toLowerCase().includes(crawler.toLowerCase()));
}

// Accept-Languageからロケール取得
function getLocale(request: NextRequest) {
  const acceptLanguage = request.headers.get('accept-language');
  if (!acceptLanguage) return defaultLocale;
  const locale = acceptLanguage.split(',')[0].split('-')[0];
  return locales.includes(locale) ? locale : defaultLocale;
}

export const middleware = auth(async function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;

  // 既にロケールがパスに含まれているか
  const hasLocale = locales.some(
    (locale) => pathname.startsWith(`/${locale}/`) || pathname === `/${locale}`
  );

  // クローラーはリダイレクトせず、x-localeを付与
  if (isSocialMediaCrawler(request)) {
    const headers = new Headers(request.headers);
    headers.set('x-locale', defaultLocale);
    return NextResponse.next({ request: { headers } });
  }

  // ロケールがパスにない場合のみリダイレクト判定
  if (!hasLocale) {
    const locale = getLocale(request);
    if (locale !== defaultLocale) {
      // 例: /about → /ja/about
      let redirectPath = `/${locale}${pathname.startsWith('/') ? '' : '/'}${pathname}`;
      // 末尾スラッシュ除去（任意）
      if (redirectPath.length > `/${locale}/`.length && redirectPath.endsWith('/')) {
        redirectPath = redirectPath.slice(0, -1);
      }
      return NextResponse.redirect(new URL(redirectPath, request.url));
    }
  }

  // x-localeヘッダーを付与
  const headers = new Headers(request.headers);
  const lang = locales.find((locale) => pathname.startsWith(`/${locale}`));
  headers.set('x-locale', lang || defaultLocale);

  return NextResponse.next({ request: { headers } });
});

// 静的アセット等は除外
export const config = {
  matcher: [
    '/((?!api|images|_next/static|_next/image|favicon.ico).*)'
  ]
};
```

初回アクセス時に、ユーザーのブラウザ設定やクローラー判定に基づいて最適な言語ページにリダイレクトするようにしています
言語が英語の場合は、URLに言語プレフィックスを付けずにそのままのURLを提供します。

ここでポイントとなるのは、**middlewareでリクエストヘッダーを書き換えられること**です。

この部分👇

```ts
// x-localeヘッダーを付与
const headers = new Headers(request.headers);
const lang = locales.find((locale) => pathname.startsWith(`/${locale}`));
headers.set('x-locale', lang || defaultLocale);
```

これにより、サーバーサイドの関数で`x-locale`ヘッダーを見ることで簡単に言語ごとの辞書が取得できるようになります。

```ts:lib/server/i18n.ts
import { headers } from 'next/headers';
import ja from './locales/ja.json';
import en from './locales/en.json';
import zh from './locales/zh.json';

const locales = ['ja', 'en', 'zh'];

export const getLocale = async () => {
  const h = await headers();
  return h.get('x-locale') || 'en';
};

export const dictionary = async () => {
  const locale = await getLocale();
  switch (locale) {
    case 'ja':
      return ja;
    case 'en':
      return en;
    case 'zh':
      return zh;
    default:
      return en;
  }
};
```

このように、サーバーサイドではmiddlewareで付与した`x-locale`ヘッダーをサーバーサイドで受け取り、それ用のjsonファイルを返す感じになります！

### サーバーコンポーネントでの使用例

以下は、サーバーコンポーネントで先ほどの`dictionary`関数を使って表示する例です。簡単ですね！

```ts:/app/[locale]/page.tsx
import { getLocale, dictionary } from '../../server/i18n';

export default async function Page() {
  const dict = await dictionary();

  return (
    <div>
      <h1>{dict.title}</h1>
      <p>{dict.description}</p>
    </div>
  );
}
```

このように`dictionary`関数を使うことによってサーバーコンポーネントであればどこでも辞書が呼び出せます！！
便利ですね！！

### クライアントコンポーネント

クライアントコンポーネント側ではあらかじめサーバーコンポーネントで取得した辞書をコンテキストに渡して、必要な部分で利用します。

以下は下準備のコンテキストとフックの実装例です。

```tsx:/contexts/i18nContext.ts
"use client";

import { createContext, useContext } from "react";
import { type LanguageObject } from "../server/i18n";

const I18nContext = createContext({} as {
  dictionary: LanguageObject;
  locale: string;
});

export function I18nProvider({ children, dictionary, locale }: { children: React.ReactNode; dictionary: LanguageObject, locale: string }) {
  return (
    <I18nContext.Provider value={{ dictionary, locale }}>
      {children}
    </I18nContext.Provider>
  );
}

export function useDictionary() {
  return useContext(I18nContext).dictionary;
}

export function useLocale() {
  return useContext(I18nContext).locale;
}
```

このI18nProviderを`layout.tsx`内でラップします。

```tsx:/app/[locale]/layout.tsx
import { I18nProvider } from "@/lib/hooks/i18n";
import { getLanguageObject, setLocale } from "@/lib/server/i18n";
import { notFound } from "next/navigation";

export default async function RootLayout({
  params,
  children
}: {
  params: Promise<{ locale: string }>
  children: React.ReactNode
}) {
  const { locale } = await params;

  if (locale !== 'ja' && locale !== 'en' && locale !== 'zh') {
    return notFound();
  }

  const i18n = getLanguageObject(locale);

  return (
    <I18nProvider dictionary={i18n} locale={locale}>{children}</I18nProvider>
  )
}
```

#### クライアントコンポーネントでの利用例

```tsx:/app/[locale]/components/ExampleComponent.tsx
"use client";

import { useDictionary, useLocale } from "@/contexts/i18nContext";

export function ExampleComponent() {
  const dict = useDictionary();
  const locale = useLocale();

  return (
    <div>
      <h2>{dict.exampleTitle}</h2>
      <p>{dict.exampleDescription}</p>
      <p>Current Locale: {locale}</p>
    </div>
  );
}
```

## 実際に運用してみた感想

実際に本番環境で運用していますが、非常にシンプルで使いやすいです。
パフォーマンス面では、追加ライブラリがないためバンドルサイズが増えず、ページの読み込み速度も従来と変わりません。
開発体験としても、英語の場合のみrewrites関数で新しくページを追加する必要はあるものの、普段はjsonファイルを更新するだけで済むので、非常に楽です。

## この手法が適している場面

以下のようなプロジェクトには特におすすめです：

- 中規模のSaaSやWebアプリケーション
- ドキュメントサイト
- 段階的にi18n対応を進めたいプロジェクト

逆に、複雑な翻訳管理機能が必要だったり、数十言語に対応する大規模プロジェクトには、専用のi18nライブラリの方が適しているかもしれません。

とくに、翻訳したいコンテンツの中にReactコンポーネントが含まれる場合や、動的なコンテンツが多い場合は、ライブラリを使った方が便利かもです！

## まとめ
App Routerの動的ルーティングと`rewrites`機能を組み合わせることで、追加ライブラリなしでシンプルなi18n実装が可能になります。

設定ファイルの複雑さに悩んでいる方や、軽量なi18n実装を探している方は、ぜひ一度試してみてください。思った以上に簡単で、かつ実用的な解決策になると思います。

もし改善点があれば、コメントで教えていただけると嬉しいです！