---
title: "サービス管理者用の認証にはIAPとIdentity Platformの組み合わせが便利かもしれない"
emoji: "👨‍💻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gcp", "iap", "authentication"]
published: true
---

皆さんはサービスの管理者が使うための認証をどのように実装していますか？
`Firebase`や`Auth0`などのサービスを使ってログイン機能を実装している方も多いと思いますが、
今回は`GCP`の`IAP`と`Identity Platform`を使って管理画面のログイン機能を実装することのメリットと具体的な実装方法について紹介します。

## IAP（Identity-Aware Proxy）とは

`IAP`とは`Identity-Aware Proxy`の略で、`GCP`のサービスの一つです。
`IAP`を使うことで、サービスに対するリクエストの前段に認証を挟むことができます。

![](https://storage.googleapis.com/zenn-user-upload/1154af0180b0-20230815.png)

例えば、`GAE`や`Cloud Run`などのサービスに対して`IAP`を設定すると、`Cloud Run`にリクエストが到達する時にはすでに認証が完了している状態となりリクエストヘッダーには`IAP`が発行した`JWT`が含まれています。

この`JWT`を使うことで、そこからユーザーのメールアドレスを取得することができます。

以下は`JWT`からユーザーのメールアドレスを取得するコードの例です。

::: details 具体的なコードを確認
```js:auth.ts
import { headers } from 'next/headers';
import prisma from "#/prisma"
import { OAuth2Client } from "google-auth-library"

export const getUserByJwt = async () => {
  const headersList = headers();
  const authorization = headersList.get('x-goog-iap-jwt-assertion');
  // 開発環境時にデバッグ認証用 Header での認証を許可
  if (
    process.env.NODE_ENV === 'development' &&
    process.env.DEBUG_EMAIL
  ) {
    const email = process.env.DEBUG_EMAIL
    if (typeof email !== 'string') {
      throw new Error('Custom Header が正しくありません')
    }

    const user = await prisma.user.findUnique({
      where: {
        email,
      },
    })

    return {
      email,
      role: user?.role,
      id: user?.id,
      sub: '',
      hd: '',
    }
  }
  // Authorization ヘッダーの存在チェック
  if (!authorization) {
    throw new Error('Authorization header は必須です')
  }

  const oAuth2Client = new OAuth2Client()
  const response = await oAuth2Client.getIapPublicKeys()
  const ticket = await oAuth2Client.verifySignedJwtWithCertsAsync(
    authorization,
    response.pubkeys,
    process.env.GCP_IAP_AUDIENCE, 
    ['https://cloud.google.com/iap']
  )

  const payload = ticket.getPayload()
  if (!payload) {
    throw new Error(
      'トークンに payload 情報が含まれていません'
    )
  }


  const { email, hd, sub } = payload

  if (!email) {
    throw new Error('トークンに email 情報が含まれていません')
  }

  const mail = email.split(':')[1]

  const user = await prisma.user.findFirst({
    where: {
      email: mail
    },
  })

  return {
    email: mail,
    role: user?.role,
    id: user?.id,
    sub,
    hd,
  }
}
```
:::

ただし、IAPだけを使うことの欠点として、ユーザーの管理がIAMでロールを付与する形でしか管理できないのでプログラム的にアプリケーション側でユーザーを追加したりすることがやりづらいという点があります。
自分だけが使う管理画面だったらいいのですが、急にログインさせたいユーザーが増えた場合に少し面倒です。

## Identity Platformとは

そこで、Identity Platformの出番です。
`Identity Platform`とはGCPのアクセス管理プラットフォームで、ちょうど`Auth0`や`Firebase Auth`のようなサービスです。
以下の画像のように、`Identity Platform`を使うことで、ユーザーの管理を行うことができます。

![](https://storage.googleapis.com/zenn-user-upload/82247bec5517-20230815.png)

また認証方法もGoogleやX（旧Twitter）、メール認証など色々と選べることが魅力的です。

![](https://storage.googleapis.com/zenn-user-upload/513f09b39db6-20230815.png)

この`Identity Platform`は`Firebase`のAPIが使用できるので、`Firebase`のSDKを使ってユーザーを追加したり削除したりすることができます。

以下はFirebaseのSDKを使ってユーザーを操作するコードの例です。

ユーザーが存在するかどうかのチェック

```js
const isUserExists = await app.auth().getUserByEmail(body?.email).catch(() => false)
```

ユーザーの作成

```js
await app.auth().createUser({
  email: body?.email,
  emailVerified: false,
  password: randomUUID(),
  displayName: body?.name,
})
```

ユーザーの削除

```js
const user = await app.auth().getUserByEmail(organizationUser.email)
await app.auth().deleteUser(user.uid)
```


## IAPとIdentity Platformを組み合わせるメリット

この`IAP`と`Identity Platform`を組み合わせることで、`Identity Platform`でユーザーを管理しつつ、`IAP`で認証を行うことができます。
これによって、認証方法が限られている`IAP`の欠点を補うことができます。

![](https://storage.googleapis.com/zenn-user-upload/e8b093aa0130-20230815.png)

また、アプリケーション側にアクセスできるのはすでに認証が完了しているユーザーのみということになるので以下のメリットが得られます。

- アプリケーション側でログインなどの実装処理を書く必要がない
- アプリケーションにアクセスできる人はすでに認証済みということが前提なのでセキュリティ的に安心

今回は私（管理者）がログインして確認するための画面なので、私以外の人間がこの画面にアクセスできないIAPの仕組みがあると、
アプリケーション側の実装でセキュリティの心配をあまりする必要がなくなります。

## IAPとIdentity Platformの組み合わせのデメリット

アプリケーションにアクセスする際に初めに必ず`Identity Platform`のログイン画面が表示されるので一般のユーザーが使うサービスには向いていません。

### 実装方法

メリットについて説明しましたが、次に実際にどのように実装するのかを説明します。

#### HTTP(S)ロードバランシングの作成

まず、`IAP`を使うためには*HTTP(S)ロードバランシング*を作成する必要があります。

##### フロントエンド構成の作成

`HTTP(S)ロードバランシング`を作成するにはまず、`フロントエンド構成`を作成する必要があります。
フロントエンド構成では`IPアドレス`やポート、`プロトコル`、証明書の設定などを行います。

![](https://storage.googleapis.com/zenn-user-upload/1b7d6e14aee9-20230815.png)


##### バックエンド構成の作成

次に、*バックエンド構成*を作成します。

バックエンドタイプとしてサーバーレスネットワークエンドポイントグループを今回は選択します。

![](https://storage.googleapis.com/zenn-user-upload/1cba9cba7eef-20230815.png)

##### ルーティングルールの作成

*ルーティングルール*を作成します。今回は「単純なホストとパスのルール」を選択します。

#### IAPの有効化

次に、`IAP`を有効化します。

https://console.cloud.google.com/security/iap

上記のページにて先ほど作成したバックエンドに対して`IAP`を有効化します。

![](https://storage.googleapis.com/zenn-user-upload/78888cf1faa6-20230815.png)


#### IAPとIdentity Platformの連携

IAPとIdentity Platformを組み合わせるにはIAPの画面で認証に`IAM`ではなく、`Identity Platform`を選択します。
IAPの画面にて該当のバックエンドサービスにチェックを入れ選択します。

![](https://storage.googleapis.com/zenn-user-upload/1134d2bafe64-20230815.png)

そして、`Identity Platform`の設定画面で認証方法を選択します。
認証方法には複数選択が可能です。

![](https://storage.googleapis.com/zenn-user-upload/7ae0bbe08c15-20230815.png)

Google認証を使う場合は必要に応じて、OAuth同意画面などの設定をしましょう。

#### Cloud Runにてトリガーを選択

最後に該当の *Cloud Run* のアプリケーションに対してトリガーを設定します。
今回は作成したCloud Load Balancingからのリクエストのみ許可したいので以下のように選択します。

![](https://storage.googleapis.com/zenn-user-upload/3ef23f849bfb-20230815.png)

これで、`IAP`と`Identity Platform`を組み合わせた認証が完了しました。
`Cloud Run`にアクセスすると、`Identity Platform`のログイン画面が表示されるようになります。

#### ログイン画面のカスタマイズ

また、IAPの画面で`Identity Platform`を認証として設定すると自動で認証用のCloud Runのサービスが作成されます。
IAPの該当のバックエンドサービスにチェックを入れて選択することでそのログインURLとカスタマイズ用のURLが表示されるので、アクセスして必要に応じてカスタマイズを行いましょう。

![](https://storage.googleapis.com/zenn-user-upload/c676d99c9558-20230815.png)

ログイン画面のカスタマイズ用のCSSのURLも指定できるので、こちらも必要に応じてCSSでカスタマイズしましょう。

![](https://storage.googleapis.com/zenn-user-upload/e7f4ad6f6eef-20230815.png)

詳しい、設定の方法は以下のドキュメントを参照してみてください。

https://cloud.google.com/iap/docs/cloud-run-sign-in?hl=ja

## まとめ

今回は`IAP`と`Identity Platform`を組み合わせて管理画面のログイン機能を実装することのメリットと具体的な実装方法について紹介しました。
管理者だけがログインして使えるサービスには、アプリケーション側でログインの実装を気にせずセキュアにログイン機能が使えるこのIAPの仕組みはとても便利だと思います。
さらにIdentity Platformを使うことでログインできるユーザーの管理も簡単に行うことができます。

管理画面などを作る機会がある場合には皆さんもぜひ試してみてください。

## 参考

https://zenn.dev/ww24/articles/19099c85febe0d


