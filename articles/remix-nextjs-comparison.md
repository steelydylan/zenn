---
title: "触ってみてわかったNext.jsと比べた時のRemixの特徴"
emoji: "😘"
type: "tech"
topics: ["remix", "nextjs", "react"]
published: true
---

本日11月23日`Remix`が正式にリリースされたので早速触ってみました。

https://remix.run/

Remixとは `react-router`というライブラリーの開発元が作ったReact製のフレームワークで、Next.jsと同じようにファイルベースでルーティングできます。
そうなると、Next.jsと一体何が違うのかと感じている人も多いと思うので、
この記事ではNext.jsと比べた時のRemixの考え方の違いをまとめたいと思います。

## SSGやISRはサポートしていない

Next.jsにはSSGやISRなどあらかじめデータをビルドしておいてその結果をリクエスト時に返すといった機能がありますが、Remixにはありませんでした。
その代わりRemixでは以下のようにページコンポーネントごとに`headers`をexportするだけで`HTTP header`をコントロールできる仕組みがあるので、そこで`Cache-Control HTTP`ヘッダーを使ったりして調整することができます。

```ts
export const headers: HeadersFunction = ({ loaderHeaders }) => {
  return {
    "Cache-Control": "max-age=3600, stale-while-revalidate=360"
  },
}
```

以下のドキュメントにあるようにRemixではSSGやISRを使ったキャッシュ戦略よりも、Cloudflare WorkersなどのEdgeで実行することや、Cache Controlなどによってパフォーマンス向上を図ることを推奨しているようですね。

https://remix.run/docs/en/v1/guides/performance

## 環境にロックインされない

もちろんNext.jsでは、GCPやServerlessなど他の環境にデプロイすることも当然可能なのですが、VercelにデプロイすることでISRや`next/image`などパフォーマンスの最適化の上でいくつか恩恵が受けられます。
Remixの場合、`Express`と連携することができたりするほか、`Vercel`はもちろん`AWS Lambda`や`Fly.io`、`Cloudflare Workers`など他の環境で動かせるアダプターが用意されているので、特定の環境にロックインされないといった点でいいですね！

## Routesごとにmetaタグやlinkタグなどが設定できる

RoutesとはNext.jsでいうところのページコンポーネントにあたるのですが、ページごとにmetaタグやlinkタグを以下のようにexportするだけで簡単に定義できます。

```tsx
export const meta: MetaFunction = () => {
  return {
    title: "About Remix"
  };
};

export const links: LinksFunction = () => {
  return [{ rel: "stylesheet", href: stylesUrl }];
};
```

Next.jsのライブラリでいうところの`next-seo`的なことが楽にできていいですね！
特定のページでのみCSSを読み込むといった時にも利用できます。

## Routesごとのエラーハンドリング

RemixではRoutesごとにエラー時の表示を`ErrorBoundary`や`CatchBoundary`コンポーネントをexportすることで設定することができます。
ErrorBoundaryはクライアントサイドのコンポーネントで意図しないエラーが発生した場合に表示でき、
CatchBoundaryはページを表示する際やPOST時にサーバーサイドからエラー系のステータスコードが返ってきた際に表示できます。

```tsx
export function ErrorBoundary({ error }) {
  return (
    <div>
      <h1>{error.message}</h1>
    </div>
  );
}
```

CatchBoundaryでは、以下のようにエラーの内容を`useCatch`を使って取り出すこともできます。

```tsx
export function CatchBoundary() {
  const caught = useCatch();

  return (
    <div>
      <h1>{caught.status} {caught.statusText}</h1>
    </div>
  );
}
```

## ページごとの部分的なルーティング

Nested Routesという機能ですが、個人的にこれが一番面白いと感じました。`Outlet`というRemixが提供しているコンポーネントを使ってネストされたURLに対して一部分にのみ適応するコンポーネントを設定することができます。
例えば、以下のようなコンポーネントが`routes/users.tsx`に定義されていたとします。

```tsx
import { Outlet } from "remix";

export default function Index() {
  return (
    <Layout>
      <Main>
        <Outlet />
      </Main>
      <Sub />
    </div>
  );
}
```

この場合、`<Outlet />`に対するコンポーネントを`routes/users/$id.tsx`などに設定することができます。
各Nested Routesにも先程の`ErrorBoundary`が設定できるので、`routes/users/$id.tsx`でエラーが起こりコンポーネントが死んだとしても、`<Outlet />`の部分のみ`ErrorBoundary`に置き換わります。
つまり、Reactでよくあるような一部書き損じたためにいきなりページ全体が死ぬといったことがなくなります。
堅牢なWebアプリケーションを作れるといった点では確かに良さそうです！

## POST処理も同じRoute内に書ける

ページコンポーネントが置かれているファイルにPOSTされた時の処理も書いておけるので見通しがいいですね。
Next.jsでいう、`pages/api/`以下に書いてた処理をページコンポーネントに持っていけるイメージです。

`remix`からFormコンポーネントと`useActionData`というhooksを利用することでこれが実現できます。

```tsx:todo.tsx
import { Form, useActionData, json } from "remix";

export async function action({ request }) {
  const body = await request.formData();
  return json({ result: 'OK' })
}

export default function Todo () {
  // ここでAPI（action）から帰ってきたデータを受け取れる
  const data = useActionData()
  return (
    <Form method="post">
      {/* ここにフォームの内容 */}
    </Form>
  )
}
```

## まとめ

触ってみた所感としてNext.jsと比べた時にSSGやISRができないことは少し不便だと思いましたが、以下のようにページごとにできることが多く、キャッシュやエラー時のハンドリングなどいろいろな戦略を立てられると感じました。

- ページごとにlinkやmetaタグを設定できる
- ページごとにheader情報を書き換えられる
- ページごとにPOST処理もかける
- ページごとにErrorBoundaryを設定できる
- Nested Routesを利用してURLごとに一部分だけコンポーネントを書き換えることもできる

ただし、以下のようなNext.jsでサポートされている機能でRemixでは使えないいくつかの機能はあるのでどちらを使うかは状況によりけりといった感じです。

- SSGやISRは使えない
- CSS Modulesなどの便利なスタイリング機能はない
- Next.jsの`pages/api/**`のようなAPI Routesがない

パフォーマンスの側面で言えば、RemixではNext.jsで提供されているようなトリッキーな機能をあえて提供せず、開発者の各々がページごとに必要に応じて`link rel="preload"`や`Cache-Control`を設定するなど、Webの標準的な技術を利用してパフォーマンスの最適化を図ることを促したい意図がRemixにあるように感じました。

Webの知識が十分ある方にはRemixで色々とチューニングできそうですが、パフォーマンス改善の知見が浅い場合には特に意識しなくても最適化をしてくれるNext.jsがお勧めですね。

まだ十分に調べられてないところがあるので何かあれば追記します。また、間違っているところなどあれば温かいご指摘などいただけると幸いです。
