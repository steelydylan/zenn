---
title: "ブラウザー上でリアルタイムに書いたReactやTypeScriptを実行するためのライブラリを作った"
emoji: "👨‍💻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "react", "npm"]
published: true
---

通常、ReactやTypeScriptを使って開発する場合は、ローカル環境で開発して、ビルドして、ブラウザーで表示するという流れになります。
ただ、昨今のブラウザーの性能はかなり高くなっており、ES Modulesをうまく使うことで、ノーバンドルでReactやTypeScriptをリアルタイムにブラウザー上で反映させることができるのではないかと考えました。

この案をもとに、何番煎じかわかりませんが、ブラウザー上でリアルタイムにReactやTypeScriptをバンドルするライブラリを作成しました。
以下のようにコードを書くだけで、ブラウザーで実行可能なJavaScriptコードが生成されます。

```ts
import { browserBundle } from "browser-bundle";

const code = `
import React from "react";
import ReactDOM from "react-dom";
import { Hello } from "./hello.tsx";

const App = () => {
  return (<div><Hello /></div>)
}

ReactDOM.render(<App />, document.getElementById("root"));
`

const result = await browserBundle(code, {
  files: {
    "./hello.tsx": `import React from "react";
    export const Hello = () => {
      return (<div>Hello World</div>)
    }`,
  }
})
```

これを`ES Modules`を使ってブラウザー上で実行することができます。
今回はiframe内でそのコードを実行しています。

```ts
const { code: bundleCode } = await browserBundle(code)
iframe.srcdoc = `
  <html>
    <head>
      <meta charset="utf-8">
    </head>
    <body>
      <div id="root"></div>
      <script type="module">
      ${bundleCode}
      </script>
    </body>
  </html>
`
```

## なぜ作ったのか

先日個人開発でリリースした模写して学ぶコーディング学習サービス、mosyaというサービスがあります。

https://mosya.dev/

こちらのサービスは現在、HTML,CSS,JavaScriptの学習を対象としていますが、今後ReactやTypeScriptなどのフレームワークやライブラリを対象にしたコンテンツを作成していく予定です。

そこで、mosyaでエディターにコードを書いている時に、ブラウザー上でリアルタイムでReactやTypeScriptをバンドルして表示するライブラリを作ってみました。

https://github.com/steelydylan/browser-bundler

また、実際にこのライブラリを試していただけるPlaygroundも用意しました。

https://mosya.dev/tools/react-ts

## どのように動いているか

このライブラリでは、コードのビルドにTypeScriptのライブラリとES Modules、`esm.sh`というES ModulesをCDNで配信してくれるサービスを利用しています。

https://esm.sh/

順を追って、どのように動いているかを説明していきます。

### 1. コードをTypeScriptでJavaScriptに変換

まず初めに入力されたコードを`ts.transpileModule`というAPIを利用して`ES Modules`として実行可能なJavaScriptに変換します。

:::message alert
ただし、`TypeScript`のバージョンはブラウザーで動かすために`5.0`以上である必要があります！
:::

CompilerOptionsは以下の通りです。

```ts
import { CompilerOptions, JsxEmit, ModuleKind, ScriptTarget, transpileModule } from 'typescript';

const defaultCompilerOptions: CompilerOptions = {
  jsx: JsxEmit.React,
  target: ScriptTarget.ESNext,
  module: ModuleKind.ESNext,
};

const jsCode = transpileModule(code, {
  compilerOptions: opt,
});
```

### 2. 相対パスなしでimportされたモジュールは`esm.sh`を利用してCDNから取得

生成されたコードに相対パスなしの`import`が含まれている場合は、`esm.sh`を利用してCDNからモジュールを取得します。

```ts
const importRegex = /import\s+.*\s+from\s+['"][^'.].*['"];?/g
const importStatements = code.match(importRegex)
let importCode = ""
importStatements?.forEach((importStatement) => {
  const convertedCode = importStatement.replace(/from\s*['"]([^'"]*)['"]/g, function(_match, p1) {
    return `from 'https://esm.sh/${p1}'`;
  });
  importCode += convertedCode + "\n"
})
```

これで、reactやreact-domなど多くのライブラリが`ES Modules`として実行可能となります。

### 3. 相対パスとして読み込まれたモジュールは`Blob URL`に変換

ここが肝となっています。実は`ES Modules`の`import`文ですが、スクリプトのURLに`Blob`や`data`スキームを指定することができます。

```ts
import { Hello } from "blob:https://example.com/0b3e2b5e-5b7e-4b1e-8b9e-9e0e2b5e7e4b"
```

この特性を利用して、相対パスで読み込まれたモジュールは`Blob URL`に変換して、`import`文を書き換えます。
大体、以下のようなコードになりますが、詳細はソースコードを参照してください。
importされているファイルを再起的に`Blob URL`に変換していく処理を書いています。

```ts
const relativeImportRegex = /import\s+.*\s+from\s+['"][.].*['"];?/g
const relativeImportStatements = code.match(relativeImportRegex)
if (relativeImportStatements) {
  await Promise.all(relativeImportStatements.map(async (relativeImportStatement) => {
    const convertedCode = await replaceAsync(relativeImportStatement, /from\s*['"]([^'"]*)['"]/g, async (_match, p1) => {
      const file = files[p1]
      const { code: converted } = await transformCode(file, options)
      if (file) {
        const blob = new Blob([converted], { type: 'text/javascript' })
        const blobURL = URL.createObjectURL(blob)
        return `from '${blobURL}'`;
      } else {
        return `from '${p1}'`;
      }
    });
    importCode += convertedCode + "\n"
  }))
}
```

### 4. 出来上がったコードを`script type="module"`の中に埋め込む

最終的に出来上がったソースコードを`script type="module"`の中に埋め込めばこれらのコードを実行することができます。
`iframe`の`srcDoc`に出来上がったコードを入れ込む使い方をおすすめしています。

```ts
const { code: bundleCode } = await browserBundle(code)
iframe.srcdoc = `
  <html>
    <head>
      <meta charset="utf-8">
    </head>
    <body>
      <div id="root"></div>
      <script type="module">
      ${bundleCode}
      </script>
    </body>
  </html>
`
```

## まとめ

今回は、ブラウザー上でReactやTypeScriptをリアルタイムにバンドルして実行するライブラリを作ってみました。
作ってみて気づいたことは最近のブラウザーの機能は本当にすごいということです。
昔はWebpackなどを使って、ビルドして、ブラウザーで実行するというのが当たり前でしたが、今はブラウザー上だけでも工夫すれば、バンドルなしでコードを実行できますね！