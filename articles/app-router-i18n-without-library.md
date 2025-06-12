

## はじめに

最近、Next.jsのAppRouterで作られたサイトを国際化対応したのですが、これが思った以上に大変でした。。。

最初はnext-i18nextやreact-i18nextといった定番ライブラリを検討していたのですが、
設定ファイルの量が意外とあったり、各ページで工夫が必要だったり...みたいな感じで、

「これ、本当に必要なのか？」と疑問に思ったんです。

そこで、Next.js 13以降のApp Routerの機能だけを使って、シンプルなi18n実装をライブラリを使わずに試してみることにしました。

結果的に、たった3つのファイルを触るだけで、追加ライブラリなしの多言語対応が実現できたので、その手法を共有したいと思います。

全体のアーキテクチャ
まず、どんな構成になるかを説明します。

```
app/
├── [locale]/        # 動的ルーティングで言語切替
│   ├── layout.tsx   # 言語別レイアウト
│   └── page.tsx     # 各ページ
├── api/             # API（言語共通）
└── middleware.ts    # 言語判定とリダイレクト
next.config.ts       # URL正規化の設定
```

対応言語は日本語（ja）、英語（en）、中国語（zh）の3つです。
英語をデフォルト言語として、URLにプレフィックスを付けない設計にしました。

例えばdocsページのURLは以下のようにしました！

```
英語: 
example.com/docs
日本語: 
example.com/ja/docs
中国語: 
example.com/zh/docs
```

英語ユーザーにはクリーンなURLを提供しつつ、他言語ユーザーにも分かりやすいURL構造になっています。

## 実装の核となる3つのファイル

### next.config.ts - URL書き換えの設定

ここが最も重要な部分です。`rewrites`機能を使って、外部からは `/docs`に見えるURLを、内部的には `/en/docs`として処理します。

```ts
export const rewrites = async () => [
  { source: '/docs/:path*', destination: '/en/docs/:path*' },
  { source: '/dashboard/:path*', destination: '/en/dashboard/:path*' },
  { source: '/', destination: '/en' }
];
```


この設定により、既存のURLを壊すことなく、段階的に翻訳が終わったページを随時切り替えることができます。

### middleware.ts - 言語検出とリダイレクト

ブラウザの`Accept-Language`ヘッダーを解析して、適切な言語ページにリダイレクトする処理を実装します。

```ts
import { NextRequest, NextResponse } from 'next/server';

const locales = ['en', 'ja', 'zh'];

function detectLocale(req: NextRequest): string {
  const acceptLanguage = req.headers.get('accept-language') ?? '';
  const browserLang = acceptLanguage.split(',')[0].split('-')[0];
  return locales.includes(browserLang) ? browserLang : 'en';
}

export function middleware(req: NextRequest) {
  const pathname = req.nextUrl.pathname;
  
  // 既に言語プレフィックスがある場合はそのまま
  const hasLocale = locales.some(locale => 
    pathname.startsWith(`/${locale}/`) || pathname === `/${locale}`
  );
  
  if (!hasLocale) {
    const locale = detectLocale(req);
    if (locale !== 'en') {
      return NextResponse.redirect(
        new URL(`/${locale}${pathname}`, req.url)
      );
    }
  }
}
```


初回アクセス時に、ユーザーのブラウザ設定に基づいて最適な言語ページに案内してくれます。

### app/[locale] - 動的ルーティングの活用

`[locale]`ディレクトリ内で、各言語に対応したページを作成します。

```ts
// app/[locale]/layout.tsx
export default function LocaleLayout({
  children,
  params: { locale }
}: {
  children: React.ReactNode;
  params: { locale: string };
}) {
  return (
    <html lang={locale}>
      <body>{children}</body>
    </html>
  );
}

// app/[locale]/page.tsx
import { dictionary, getLocale } from '@/lib/server/i18n';

export default async function Page() {
  const locale = await getLocale();
  const i18n = await dictionary(locale);
  
  return (
    <div>
      <h1>{i18n.title}</h1>
      <p>{i18n.description}</p>
    </div>
  );
}
```
翻訳はlib/server/locales/ディレクトリにJSONファイルとして配置し、サーバーサイドで読み込みます。

## 実際に運用してみた感想

この手法を本番環境で3ヶ月ほど運用してみましたが、想像以上に快適でした。

パフォーマンス面では、追加ライブラリがないためバンドルサイズが増えず、ページの読み込み速度も従来と変わりません。
開発体験としては、新しい言語を追加する際もJSONファイルを一つ作るだけで済むため、非常にシンプルです。

SEO面でも、各言語のページが独立したURLを持つため、検索エンジンからのインデックスも問題なく、各言語ごとに適切なメタタグを設定できます。

## この手法が適している場面

以下のようなプロジェクトには特におすすめです：

- 中規模のSaaSやWebアプリケーション
- ドキュメントサイト
- 段階的にi18n対応を進めたいプロジェクト

逆に、複雑な翻訳管理機能が必要だったり、数十言語に対応する大規模プロジェクトには、専用のi18nライブラリの方が適しているかもしれません。

## まとめ
App Routerの動的ルーティングと`rewrites`機能を組み合わせることで、追加ライブラリなしでシンプルなi18n実装が可能になります。

設定ファイルの複雑さに悩んでいる方や、軽量なi18n実装を探している方は、ぜひ一度試してみてください。思った以上に簡単で、かつ実用的な解決策になると思います。

実装で詰まった部分や改善点があれば、コメントで教えていただけると嬉しいです！