---
title: "ProgateのレッスンにChromeの開発者ツールを丸ごと埋め込んだ話"
emoji: "🛠️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["devtools", "chrome", "typescript", "chrome"]
published: false
publication_name: "progate"
---

## はじめに


HTML・CSSを学んでいると、「`margin`がなんで効かないんだろう？」「この要素ってどういう構造になってるんだろう？」と気になる場面がよくあります。普段の開発ならChromeのデベロッパーツールをパッと開いて要素を検証すればいいのですが、学習サービスのプレビューは`iframe`の中にあることが多く、普通にデベロッパーツールを開くと学習に関係のないサービス本体の要素やコンソールまで表示されてしまうという問題があります。

しかも、検証するにはブラウザー標準のデベロッパーツールを開くしかなかったので、その分だけ学習画面が圧迫されてしまうのも地味につらいところでした。こんな感じで、右側にデベロッパーツールが出てくると、肝心のエディターやプレビューの幅がぎゅっと狭くなってしまうんですよね。

![ブラウザー標準のデベロッパーツールを開いた様子。右側にツールが出る分、エディターやプレビューの幅が小さくなってしまっている](/images/progate-embed-devtools/before-narrow-width.png)

特にProgateには、自分でゼロからコードを書いて課題をクリアする「道場レッスン」があります。穴埋めではなく自力でHTML・CSSを組み立てていくぶん、「思った通りに表示されない」ときに自分で原因を調べられると学びが一気に深まるんですよね。こういう場面でこそデベロッパーツールが活きてくるはずです。

そこで今回、Progateのレッスン画面にChromeのデベロッパーツールそのものを埋め込んでしまうということをやってみました！

こんな感じで、レッスンのプレビューのすぐ下にデベロッパーツールが表示されて、書いたHTML・CSSをその場で検証できます。

![Progateのレッスン画面に埋め込まれたデベロッパーツール](/images/progate-devtools/embedded-devtools.png)

要素の検証はもちろん、スタイルや計算済みのレイアウト、コンソールまで、普段使っているデベロッパーツールとほぼ同じ感覚で使えます。

## なぜ作ったのか

実は以前、同じようなことをChiiというライブラリを使ってやった記事を書いています。

https://zenn.dev/steelydylan/articles/react-embed-devtools

このときは、デベロッパーツールをブラウザーで動かせるようにビルド済みで配布されているChiiをそのまま読み込んで、Chobitsuと組み合わせてiframeの検証を実現していました。手軽に埋め込めてとても便利だったのですが、Progateに導入しようと思うといくつか物足りない点がありました。

- ビルド済みのものをそのまま使うので、不要なタブや機能を細かく削れない
- バージョンや表示まわりを自分たちの都合に合わせて調整しにくい
- 外部のCDNから配信されたものに依存することになる

学習サービスに組み込むなら、学習に関係ない機能はできるだけ消して、必要なものだけをシンプルに見せたいという気持ちが強くありました。

そこで今回は思い切って、本家のChrome DevTools（`devtools-frontend`）をフォークして、自分たちでビルドして組み込むという方針に切り替えました。

ちなみに`devtools-frontend`はBSD-3-Clauseライセンスで公開されています。著作権表示とライセンス文を残せば、改変したものを商用サービスに組み込んで配布してもOKという、かなり緩めのライセンスなんですよね。Progateのような商用の学習サービスに組み込むうえでも安心して使えるのは、地味にありがたいポイントでした。

## 仕組みのおさらい：CDPとChobitsu

本題に入る前に、なぜこんなことが可能なのかを軽くおさらいしておきます。

### DevToolsのプロトコルは公開されている

意外と知られていないのですが、Chromeのデベロッパーツールがブラウザーとやり取りするためのプロトコルは Chrome DevTools Protocol（CDP）として仕様が公開されています。

https://chromedevtools.github.io/devtools-protocol/

デベロッパーツールは、このCDPに沿ったJSONメッセージを投げたり受け取ったりすることで動いています。たとえば「DOMツリーをちょうだい」「このスタイルを取得して」といったやり取りがすべてメッセージとして定義されているんですね。

ということは、このプロトコルを話せる相手さえ用意できれば、デベロッパーツールは本物のブラウザーに繋がっていなくても動くということになります。

### Chobitsu：CDPをpostMessage経由で話すライブラリ

そのCDPをブラウザー上で実装してくれているのが`Chobitsu`というライブラリです。

https://github.com/liriliri/chobitsu

> Chrome devtools protocol JavaScript implementation.

このライブラリをプレビュー用のiframeに読み込んでおくと、iframe内のDOMやコンソールの状態をCDPメッセージとして受け答えしてくれます。つまり、Chobitsuがブラウザー本体の代わりにデベロッパーツールの相手をしてくれるわけです。

全体の構成としては、こんな感じの3者構成になります。

![DevTools UI の iframe ⇄ 親ウィンドウ（Progate本体）⇄ プレビュー iframe + Chobitsu](/images/progate-embed-devtools/architecture.png)

DevToolsとChobitsuはお互いを知らないので、間にいる親ウィンドウが`postMessage`でメッセージを中継してあげます。

## ポイント：WebSocketの部分をpostMessageに差し替える

ここからが今回の一番のポイントです。

本家の`devtools-frontend`は、Chromeに組み込まれて動いているときは、デバッグ対象とブラウザー内部のチャンネルでやり取りしています。一方で、`devtools-frontend`を単体のWebアプリとしてホストして外部のデバッグ対象に繋ぐ構成だと、CDPをWebSocket経由でやり取りします。リモートのデバイス（実機のスマホなど）やNode.jsプロセス、ヘッドレスChromeをデバッグするときがまさにこのモードで、`?ws=...`のようなクエリパラメータで接続先のWebSocketのURLを渡す仕組みです。

https://github.com/chromedevtools/devtools-frontend


ですが今回やりたいのは、同じページ内にある別のiframe（Chobitsu）との通信です。わざわざWebSocketサーバーを立てるのは大げさですし、`postMessage`で十分やり取りできます。

そこで、`devtools-frontend`の接続を担当している[front_end/core/sdk/Connections.ts](https://github.com/ChromeDevTools/devtools-frontend/blob/main/front_end/core/sdk/Connections.ts)に、`postMessage`でCDPメッセージをやり取りする`EmbeddedConnection`というクラスを追加しました。

```ts
export class EmbeddedConnection implements ProtocolClient.InspectorBackend.Connection {
  onMessage: ((arg0: Object) => void) | null;
  private targetOrigin: string = '';
  constructor(targetOrigin: string) {
    this.targetOrigin = targetOrigin;
    this.onMessage = null;
    window.addEventListener('message', event => {
      // 想定した origin 以外からのメッセージは無視する
      if (event.origin === this.targetOrigin && this.onMessage) {
        this.onMessage(event.data);
      }
    });
  }
  setOnMessage(onMessage: (arg0: (Object|string)) => void): void {
    this.onMessage = onMessage;
  }
  sendRawMessage(message: string): void {
    // CDP メッセージを親ウィンドウに postMessage で送る
    window.parent.postMessage(message, this.targetOrigin);
  }
  setOnDisconnect(_onDisconnect: (arg0: string) => void): void {
  }
  disconnect(): Promise<void> {
    return Promise.resolve();
  }
}
```

やっていることはシンプルで、

- `sendRawMessage`：デベロッパーツールが送りたいCDPメッセージを`window.parent.postMessage`で親に投げる
- `addEventListener('message', ...)`：親から返ってきたメッセージを受け取ってデベロッパーツールに渡す

という、WebSocketの`send`/`onmessage`を`postMessage`に置き換えただけのものです。

そして、接続方法を選んでいる箇所に`?embedded=<origin>`というクエリパラメータを足して、このパラメータが付いていたら`EmbeddedConnection`を使うように分岐させます。

```ts
function createMainConnection(websocketConnectionLost: () => void): ProtocolClient.InspectorBackend.Connection {
  const wsParam = Root.Runtime.Runtime.queryParam('ws');
  const wssParam = Root.Runtime.Runtime.queryParam('wss');
  const embeddedParam = Root.Runtime.Runtime.queryParam('embedded');
  // embedded が付いていたら WebSocket ではなく postMessage で接続する
  if (embeddedParam) {
    return new EmbeddedConnection(embeddedParam);
  }
  if (wsParam || wssParam) {
    const ws = (wsParam ? `ws://${wsParam}` : `wss://${wssParam}`) as Platform.DevToolsPath.UrlString;
    return new WebSocketConnection(ws, websocketConnectionLost);
  }
  // ...
}
```

これで、デベロッパーツールを以下のようなURLで開くだけで、`postMessage`モードで起動するようになります。

```ts
// Progate 側で devtools の iframe に渡す URL
export function getDevtoolsIframeSrc(): string {
  return `${getDevtoolsOrigin()}/index.html#?embedded=${encodeURIComponent(location.origin)}`
}
```

`embedded`には親ウィンドウのoriginを渡しておき、`postMessage`の送信先や受信メッセージの検証に使っています。素性のわからないoriginとのやり取りを弾くための安全策ですね。

## 不要なタブや機能を削る

本物のデベロッパーツールには、Application、Network、Security、パフォーマンスなど、たくさんのパネルがあります。ですが、HTML・CSSの学習に使うなら、要素・コンソール・ソースくらいがあれば十分です。

`devtools-frontend`では、どのパネルを読み込むかが[front_end/entrypoints/devtools_app/devtools_app.ts](https://github.com/ChromeDevTools/devtools-frontend/blob/main/front_end/entrypoints/devtools_app/devtools_app.ts)で`import`の羅列として定義されています。なので、いらないパネルの`import`をコメントアウトするだけで機能を削れます。

```ts
import '../../panels/elements/elements-meta.js';
import '../../panels/browser_debugger/browser_debugger-meta.js';
// import '../../panels/network/network-meta.js';        // ネットワークは消す
// import '../../panels/security/security-meta.js';       // セキュリティも消す
// import '../../panels/emulation/emulation-meta.js';
import '../../panels/sensors/sensors-meta.js';
import '../../panels/accessibility/accessibility-meta.js';
// ...
// import '../../panels/application/application-meta.js'; // Application タブも消す
import '../../panels/issues/issues-meta.js';
import '../../panels/layers/layers-meta.js';
// import '../../panels/mobile_throttling/mobile_throttling-meta.js';
```

このあたりは「学習に必要なものだけ残す」という方針で、ApplicationやMobile Throttlingなど、レッスンには不要なパネルを片っ端からコメントアウトしていきました。こういう細かい調整ができるのが、自前でビルドする一番のメリットだなと思います。

## ビルドはNinjaという謎ツールで、15分くらいかかる…

さて、ソースを書き換えたらビルドが必要なのですが、ここで一つ大きな壁がありました。

`devtools-frontend`はChromium本体と同じビルドシステムを使っています。具体的には`gn`でビルド設定を生成して、`ninja`（正確には`autoninja`）でビルドする、という流れです。npmのビルドに慣れていると、初見だとなかなか「謎ツール」感があります。

https://gn.googlesource.com/gn/

`package.json`のビルドコマンドはこんな感じになっています。

```json
{
  "scripts": {
    "build": "gn gen out/Default --args=\"is_debug=false\" && autoninja -C out/Default",
    "build:fast": "gn gen out/Default --args=\"devtools_skip_typecheck=true\" && autoninja -C out/Default"
  }
}
```

`gn gen`でビルド設定（`out/Default`）を作って、`autoninja`で実際にビルドする、という2段構えです。

そして問題はビルド時間で、これがフルビルドだとだいたい15分くらいかかります…。ちょっとした修正のたびに15分待つのはなかなか厳しいものがあります。

少しでも速くするために、型チェックをスキップする`build:fast`も用意しました。`gn gen`の引数に`devtools_skip_typecheck=true`を渡すと、TypeScriptの型チェックを飛ばしてビルドだけ走らせてくれるので、試行錯誤の段階ではこちらが地味に助かりました。

それでもビルドはそこそこ時間がかかるので、「ここを直したい」というのが溜まってからまとめてビルドする、みたいな進め方になりました。Chromiumのビルドの大変さの片鱗を味わえた気がします。

## ビルドした成果物を配信する

ビルドして出来上がった成果物（`out/Default`配下の静的ファイル群）は、Progate本体とは別のオリジンから配信しています。デベロッパーツールのUIは独立した静的ファイルなので、ストレージに置いて配信するだけでOKです。

Progateでは、ビルド済みの`public`フォルダを丸ごとオブジェクトストレージにアップロードして、`devtools`用のオリジンとして配信する形にしました。配信元のoriginは設定ファイルに集約しておき、フロント側ではそれを読み込んでiframeの`src`を組み立てています。

あとは、このoriginの`index.html`に先ほどの`#?embedded=<origin>`を付けてiframeで読み込めば、デベロッパーツールが`postMessage`モードで立ち上がってくれます。

## プレビュー側とのメッセージ中継

最後に、親ウィンドウ（Progate本体）がやっている中継処理です。役割としては、2つのiframeの仲介役です。

- プレビューiframe内のChobitsuから来たメッセージ → デベロッパーツールのiframeに流す
- デベロッパーツールのiframeから来たメッセージ → プレビューiframeに`DEV`イベントとして流す

```ts
window.addEventListener('message', event => {
  // プレビュー（Chobitsu）からのメッセージを DevTools に流す
  if (event.origin === window.location.origin) {
    devtoolsWindow.postMessage(event.data, devtoolsOrigin)
  }
  // DevTools からのメッセージをプレビューに流す
  if (event.origin === devtoolsOrigin) {
    previewWindow.postMessage({event: 'DEV', data: event.data}, window.location.origin)
  }
})
```

どちらの方向も`origin`をきちんと検証していて、想定外のoriginとのやり取りは弾くようにしています。`iframe`をまたいだ`postMessage`は油断するとセキュリティ的に緩くなりがちなので、ここは丁寧にやっておきたいところです。

Chobitsu側（プレビューiframeに注入するスクリプト）では、デベロッパーツールが起動したタイミングで`DOM.enable`や`CSS.enable`といったCDPコマンドを送って、要素パネルやスタイルが正しく表示されるように初期化しています。このあたりの流れは前回のChiiの記事とほぼ同じなので、詳しく知りたい方はそちらも合わせて読んでみてください。

## 使い心地

実際に組み込んでみると、プレビューの上にある虫アイコンを押すだけでデベロッパーツールが開いて、検証したい要素をクリックするとその要素のDOM構造や適用されているスタイルがそのまま確認できます。「思った通りに表示されない」ときの原因調査にとても役立ちそうです。

学習者にとっては、普段プロのエンジニアが使っているのと同じデベロッパーツールを、学習サービスの中でそのまま触れるというのは結構嬉しいんじゃないかなと思っています。

## まとめ

今回やったことをまとめると、こんな感じです。

- 本家のChrome DevTools（`devtools-frontend`）をフォークして自前ビルドした
- WebSocket接続の部分を`postMessage`で通信する`EmbeddedConnection`に差し替えた
- CDPは公開仕様で、Chobitsuがそれを`postMessage`経由で話してくれる
- 不要なパネルは`import`をコメントアウトするだけで削れる
- ビルドは`gn` + `ninja`（autoninja）で、フルビルドは15分くらいかかる…

「ビルド済みのChiiを読み込む」から「本家をフォークして自前ビルドする」に踏み込んだことで、不要な機能を削ったり、表示を細かく調整したりといった自由度が一気に上がりました。ビルド時間との戦いはありましたが、学習サービスにちょうどいい形のデベロッパーツールを用意できたかなと思います。

## 今後の課題

- レッスンの内容に応じて、表示するパネルをさらに出し分ける
- 本家の`devtools-frontend`の更新への追従

似たようなことをやってみたい方の参考になれば嬉しいです。ぜひ試してみてください！
