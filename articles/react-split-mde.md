---
title: "Zennのためにプレビューしながら記事を書けるマークダウンエディターを開発していた話"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "markdown"]
published: true
---

クラスメソッドさんに譲渡される前に開発者の友人としてZennの開発に関わっていた[@steelydylan](https://twitter.com/steelydylan) です。
まだ、Zennには搭載されていませんが、記事の結果をプレビューしながらマークダウンをかけるエディターを開発していたので、そのエディターの供養も兼ねて記事にしておきたいと思います。

# 開発したもの

プレビューしながら記事を書けるReact製のマークダウンエディター

![](https://storage.googleapis.com/zenn-user-upload/c8pcqwvatlj0u8dqmfv3u5frk2kk)

今はオープンソースとして自分のレポジトリに公開しています！

https://github.com/steelydylan/react-split-mde

## 特徴

- スクロール同期
- TypeScript Friendly
- 改行時のリスト自動挿入
- タブキー入力時のリストのインデント
- マークダウンパーサーのカスタマイズができる
- イベントを定義できる
- 独自のショートカットコマンド定義

https://react-split-mde.vercel.app/

# そもそも何故開発していたか


## 既存のエディターがQiitaのそれに劣っている点があった。。。
@[tweet](https://twitter.com/natsume_aurlia/status/1307908917029007360)

このように、TwitterでZennに関することをエゴサしていると、プレビューしながら記事を書きたいという要望がかなりの数でありました。確かにQiitaであればHTMLへの変換結果が横に並んでいるため書きやすいというのは間違いありません。


## 既に公開されているエディターで満たせない機能

そこで、まずはそういったマークダウンとプレビューに分割されて記事を書けるエディターを探しました。
すでにいくつか公開されているエディターはいくつかありました。

https://github.com/sparksuite/simplemde-markdown-editor

Zennでは独自のマークダウン記法が多く存在するためパーサーをカスタマイズできる機能は必須です。ところが、MarkdownからHTMLの変換するためのパーサーを独自にカスタマイズできるエディターがあまりありませんでした。

### スクロール同期の問題

またマークダウン結果をカスタマイズできたとしても、記事のスクロール位置に合わせてプレビュー結果をスクロール同期できないと、結局スクロールを手動ですることになるので、不便さにはあまり変わりがありません。マークダウンの独自記法を自分で定義できてさらにスクロールも同期できるそういう都合のいいエディターは自分が探した範囲ではありませんでした。。

### 独自のショートカットコマンド定義の必要性

またZennはクールなUIが売りでもあるので、余計なエディターの装飾は削ぎ落としたいということもありました。
さらに、画像のアップロード機能やTwitterなどの埋め込みウィジェットなどの呼び出しも `ショートカットコマンド`で行いたいニッチな機能もカスタマイズすれば搭載できるようにする必要がありました。

### Twitterなどのウィジェットの再レンダリング問題

Zennでは以下のようにTwitterやGist、YouTubeのウィジェットを記事内に挿入することができます。

@[tweet](https://twitter.com/steelydylan/status/1277938362473541634)

多くのマークダウンエディターでは、記事を書くたびにすべてのプレビュー結果が再レンダリングされてしまうため、記事を書いたタイミングでプレビュー結果のスクロール位置が変わってしまうことが多いです。
つまり、今書いている記事の行だけ、再レンダリングする仕組みが必要でした。そのため、開発した、`react-split-mde`では`morphdom`というライブラリを使い編集中の要素だけ再レンダリングする工夫をしています。

# 使い方

せっかく開発したので、すこしだけ使い方を紹介させてください。まだ、完成したわけではないので仕様は今後変わる可能性はあります。

## インストール方法

```sh
$ yarn add react-split-mde
```

もしくは、

```sh
$ npm install react-split-mde
```

## シンプルな使い方

一応カスタマイズなしであればこれだけでも動きます。

```tsx
import { useState } from "react"
import { Editor } from "react-split-mde"
import "react-split-mde/css/index.css"
const defaultValue = "# React Split MDE"

export const EditorDemo = () => {
  const [value, setValue] = useState(defaultValue)
  const handleValueChange = (newValue) => {
    setValue(newValue)
  }
  return (<Editor 
    value={value}
    onChange={handleValueChange} 
  />)
}
```

## 画像挿入サンプル

`Provider`を使うと、テキスト挿入やリプレイスなどさまざまなイベントをエディターに伝えることができます。
以下は、ファイルが選択された時にそのファイルの内容をエディターに挿入する簡単なサンプルコードです。

```tsx
import { useState, useCallback } from "react"
import { Editor, useProvider } from "react-split-mde"
import "react-split-mde/css/index.css"
const defaultValue = "# React Split MDE"
export const EditorDemo = () => {
  const [emit, Provider] = useProvider()
  const [value, setValue] = useState(defaultValue)
  const handleImageUpload = useCallback(
    async (e: React.ChangeEvent<HTMLInputElement>) => {
      const uploadingMsg = "![](now uploading...)";
      emit({
        type: "insert",
        text: uploadingMsg,
      });
      await new Promise((resolve) => {
        setTimeout(() => resolve(), 1000);
      });
      emit({
        type: "replace",
        targetText: uploadingMsg,
        text: "![](https://source.unsplash.com/1600x900/?nature,water)",
      });
    },
    []
  )
  const handleValueChange = (newValue) => {
    setValue(newValue)
  }
  return (
    <Provider>
      <input type="file" onChange={handleImageUpload} />
      <Editor
        value={value} 
        onChange={handleValueChange} 
      />
    </Provider>
  )
}
```

## 独自のショートカットコマンド定義

次のコードは、Enterキーとコマンドキーの両方を同時に入力したときにイベントを発生させる方法の例になります。


```tsx
import { useState } from "react"
import { Editor, defaultCommands } from "react-split-mde"
import "react-split-mde/css/index.css"
const defaultValue = "# React Split MDE"
export const EditorDemo = () => {
  const [value, setValue] = useState(defaultValue)
  const handleValueChange = (newValue) => {
    setValue(newValue)
  }
  return (<Editor
    value={value} 
    onChange={handleValueChange}
    commands={
      ...defaultCommands,
      save: (textarea, option) => {
        const { composing, code, shiftKey, metaKey, ctrlKey } = option;
        if ((metaKey || ctrlKey) && !shiftKey) {
          if (!composing && code === EnterKey) {
            // ここに実行したい関数を定義します。
            return { stop: true, change: false };
          }
        }
      },
    }
  />)
}
```

## Web Workerとの併用

`react-split-mde`ではパーサーを独自に定義できるのでパーサー自体はWeb Workerなどで実行して、その結果をエディターに渡すといったカスタマイズも可能です。いか、Web Workerを使ったサンプルになります。

```ts:markdown.worker.ts
const ctx: Worker = self as any;
ctx.Prism = {}
ctx.Prism.disableWorkerMessageHandler = true;
// you have to disable prism worker handle option first
import markdownHTML, { enablePreview } from "zenn-markdown-html";
enablePreview();
// Respond to message from parent thread
ctx.addEventListener("message", (event) => {
  const result = markdownHTML(event.data);
  ctx.postMessage(result);
});
```

```ts:editor.tsx
import Worker from "worker-loader!./markdown.worker";
import pEvent from "p-event";
import { useState, useMemo } from "react"
import { Editor } from "react-split-mde"
import "react-split-mde/css/index.css"
const defaultValue = "# React Split MDE"
export const EditorDemo = () => {
  const worker = useMemo(() => {
    const w = new Worker();
    return w;
  }, []);
  const [value, setValue] = useState(defaultValue)
  const handleValueChange = (newValue: string) => {
    setValue(newValue)
  }
  const handleMarkdown = async (str: string) => {
    worker.postMessage(str);
    const e = await pEvent(worker, "message");
    return e.data;
  };
  return (<Editor 
    value={value}
    parser={handleMarkdown}
    onChange={handleValueChange} 
  />)
}
```

# まとめ

あまり長すぎるとただの開発したライブラリの紹介の様になってしまうのでこの辺にしておきます。
この記事でお伝えしたいこととしては、公開されている多くのマークダウンディターだと、スクロールの問題やパーサーの問題、レンダリングの問題など越えなければならない多くの壁が存在し、Zennはその壁を突破するために色々と開発を頑張っているということです笑
今はクラスメソッドさんにZennが買収されてしまったので自分は開発に関われませんが、ライブラリ自体の開発はできるのでいつか、Zenn側に`npm install`され、dependenciesとして使われる日を待ちたいと思います！