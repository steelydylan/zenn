---
title: "ウィジェットの埋め込みコードを開発した際の苦労話"
emoji: "👨‍💻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "react"]
published: true
---

つい先日、ブログの継続状況をGitHubの草のように可視化できる[Blog Friends](https://blog-friends.com/)というサービスを開発しました。

https://blog-friends.com

サービスにログインし、自分のサイトのRSSを登録するだけで簡単にブログの投稿履歴を以下のように確認できます。

![](https://blog-friends.com/images/activity.png)

開発に至った経緯や技術スタックはこちらの記事にまとめております。

https://zenn.dev/steelydylan/articles/blog-friends

そして、今回はこちらのヒートマップを任意のサイトに以下のように表示できる埋め込みコードを開発しました。

![](https://storage.googleapis.com/zenn-user-upload/23cf310cf91c6180d77b8900.png)


Blog Friendsのご自身のプロフィールページにて以下のボタンをクリックしていただくことで埋め込みコードがコピーできます。

![](https://storage.googleapis.com/zenn-user-upload/2f248c4e8eb55e44ff850481.png)

この埋め込みコードをWordPressやらはてなブログやらの任意のサイトに貼り付けていただくだけで先程のウィジェットが表示されます。

この記事ではこの埋め込みコードを作る上で苦労した点や開発のコツなどを書いていきたいと思います。

## 開発で苦労した点


### 1. iframe内のコンテンツの高さを取得する

まず埋め込み先のページを作成しました。
以下のURLにアクセスしていただくとヒートマップだけが表示されたページが見えるかと思います。

https://blog-friends.com/users/steelydylan/embed


iframeは任意のサイトをパーツのように別のサイトに埋め込むことができて大変便利なのですが不便な点もあります。

それはiframeの高さをiframeの内側から決められないことです。
iframeの高さが決められないと、変にiframe内がスクロールできてしまったり逆にiframeが高くなりすぎたりしてダサくなってしまいます。

![](https://storage.googleapis.com/zenn-user-upload/f523f6662ca09e40e0b81262.png)

そこで、iframe内から自身の高さを取得してそれを`親window`に送信するようにしました。
`window.parent.postMessage`を使うと親windowに対してiframe内からメッセージを送信することができます。

iframe内では以下のように親のwindowに対して自身のサイズを送信しています。
このサイトではReactを使っているのですが、`useLayoutEffect`を使うことでDOMが更新されて表示された後に実行することで正確な大きさを取得することができました。

```ts
useLayoutEffect(() => {
  if (!ref.current) {
    return;
  }
  const rect = ref.current.getBoundingClientRect();
  if (!window.parent) {
    return;
  }
  const sendEmbedSizeInfo = () => {
    window.parent.postMessage(JSON.stringify({
      width: rect.width,
      height: rect.height,
      isEmbed: true,
      name: window.name,
    }), "*")
  }
  sendEmbedSizeInfo();
  window.addEventListener('resize', sendEmbedSizeInfo)
  return () => window.removeEventListener('resize', sendEmbedSizeInfo)
}, [])
```

埋め込み元では以下のようなスクリプトで高さを反映しています。
window.addEventListener('message')を使うことで任意のメッセージを取得することができます。
ただし、これは全てのメッセージを受け取ってしまうので何らかの判定処理を入れて条件に合致した時だけ処理を実行する必要があります。
`event.origin`でイベントの発行元が取得できるのでそれを使ってそれ以外のURLからのメッセージの際は処理しないようにしています。

```ts
window.addEventListener('message', function(event) {
  try {
    const data = JSON.parse(event.data)
    if (!data.isEmbed || event.origin !== 'https://blog-friends.com') {
      return
    }
    const iframe = document.querySelector(`[name="${data.name}"]`) as HTMLIFrameElement
    if (!iframe) {
      return
    }
    iframe.height = data.height
  } catch (e) {}
}, false);
```

### 2. 複数箇所に埋め込みコードが配置されている場合の処理

`window.addEventListener('message')`では全てのメッセージを受信してしまうためメッセージがどのiframeに対して向けられたメッセージなのかを取得するのが難しいです。
iframeだけでなく例えばChromeの拡張機能からもメッセージが送られている場合もあります。

埋め込みコードが一つだけなら、特定のクラスをiframeにつけておいてメッセージを受信したタイミングでそのiframeに対して高さを変えればいいのですが、複数箇所に埋め込みコードがあるとどのiframeの高さを変更していいかわからなくなります。

そこでiframeのname属性にユニークなidを以下のようにつけることにしました。
shortIdというライブラリを使ってユニークなidをiframeのname属性に付与しています。

```ts
iframe.name = `blog-friends-${shortid()}`;
```

iframeに名前をつけておくことでiframe内でもその名前を`window.name`から取得することが可能です。

```ts
const sendEmbedSizeInfo = () => {
  window.parent.postMessage(JSON.stringify({
    width: rect.width,
    height: rect.height,
    isEmbed: true,
    name: window.name, // ← ここでiframe.nameが取得できる
  }), "*")
}
```

あとは親windowでiframe内から送信されたメッセージよりnameを取得して`querySelector`でサイズを変更したいiframeを取得します。

```ts
window.addEventListener('message', function(event) {
  try {
    const data = JSON.parse(event.data)
    if (!data.isEmbed || event.origin !== 'https://blog-friends.com') {
      return
    }
    const iframe = document.querySelector(`[name="${data.name}"]`) as HTMLIFrameElement
    if (!iframe) {
      return
    }
    iframe.height = data.height
  } catch (e) {}
}, false);
```

### 3. 埋め込みコードの公開先

最後に埋め込みコードをどこで公開すればいいか悩みました。
[Blog Friends](https://blog-friends.com) でそのまま公開すればいいのですが、埋め込みコードスクリプトは気楽にサイトとは切り離して開発したいという思いから別プロジェクトとして管理することにしました。

実はnpmにソースコードをpublishすれば、以下のサイトでCDNとしてソースコードを公開できます。

https://unpkg.com/

今回はunpkgを埋め込みコードのホスティング先として利用させていただくことにしました。

ソースコードはこちらです！

https://github.com/steelydylan/blog-friends-embed

## まとめ

埋め込みコードの開発は簡単だと思っていたのですが、iframeやWeb Messaging Apiを知らないとハマることが多いです。
もし同じような埋め込みにチャレンジすることがあれば今回の記事を思い出していただければ幸いです。

また、宣伝になりますが、Blog FriendsはブログのRSSを登録することで活動履歴を簡単にトラッキングできますので、ぜひ発信の継続にご利用いただけると幸いです。

https://blog-friends.com/
