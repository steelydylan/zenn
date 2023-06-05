---
title: "esm.shとmonaco-editorを使ってVS Codeのようなフロントエンドの開発体験を実現する"
emoji: "👨‍💻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "react", "npm"]
published: true
---

この記事は前回書いた[ブラウザー上でReactやTypeScriptのコードをコンパイルして動かすツールを作った](https://zenn.dev/ryuu/articles/browser-bundler)の続きです。

https://zenn.dev/ryuu/articles/browser-bundler

まずは動画をご覧ください。

@[youtube](https://youtu.be/uMu8whYWtHM)

下のURLを触っていただいたらわかるようにエディター上でちゃんと読み込んだライブラリの型のインテリセンスが効いていて、ブラウザー上でReactやTypeScriptのコードを実行することができます。

https://monaco-browser-bundler.vercel.app/

また、`package.json`に読み込みたいライブラリを記述することで、そのライブラリをブラウザー上で実行することもできます。もちろんエディター上でエディター上で型が解決され補完も効きます。

## どのように実装しているか

今回の実装には`monaco editor`と`esm.sh`を使っています。
`monaco editor`は`VS Code`のようなエディターをブラウザー上で実現するためのライブラリで、`esm.sh`は`CDN`から`npm`のライブラリを取得するためのサービスです。

### 1. monaco editorでTypeScriptのセットアップ

monaco-editorには内部に`TypeScript`のコンパイラが含まれているので、そのコンパイラでまずはTypeScriptのオプションを設定します。
この時、`typeRoots`を`node_modules/@types`に設定しておくことがポイントです。

```ts
export const setCompilerOptions = () => {
  monaco.languages.typescript.typescriptDefaults.setCompilerOptions({
    target: monaco.languages.typescript.ScriptTarget.ES2016,
    allowNonTsExtensions: true,
    moduleResolution: monaco.languages.typescript.ModuleResolutionKind.NodeJs,
    module: monaco.languages.typescript.ModuleKind.CommonJS,
    jsx: monaco.languages.typescript.JsxEmit.React,
    noEmit: true,
    allowJs: true,
    esModuleInterop: true,
    typeRoots: ["node_modules/@types"],
    lib: ["es2016", "dom"],
  });
  monaco.languages.typescript.typescriptDefaults.setDiagnosticsOptions({
    noSemanticValidation: false,
    noSyntaxValidation: false,
  });
};
```

### 2. `package.json`で読み込んだライブラリの型を取得する

エディターでは読み込んだライブラリーでちゃんと型の補完が効くのですが、これは`package.json`に記述されたライブラリの型を取得しているからです。
以下のように`esm.sh`というサービスを使って、`package.json`に記述されたライブラリの型を取得しています。

```ts
export const resolveModuleType = async (
  module: string,
): Promise<void> => {
  const res = await fetch(`https://esm.sh/${module}`);
  const dtsUrl = res.headers.get("x-typescript-types");
  if (!dtsUrl) return;
  const dts = await fetch(dtsUrl);
  const dtsText = await dts.text();
  monaco.languages.typescript.typescriptDefaults.addExtraLib(
    dtsText,
    `file:///node_modules/@types/${module}/index.d.ts`
  );
};

// reactの型を取得
resolveModuleType("react");
```

`esh.sh`のサイトでライブラリを取得するとその`header`に`x-typescript-types`に型情報へのURLが含まれているので、それを取得して、`monaco`に追加しています。

`monaco.languages.typescript.typescriptDefaults.addExtraLib`メソッドを使って先ほど取得した型情報を追加します。この時に先ほどTypeScriptの`typeRoots`に`node_modules/@types`を設定しておいたので、そこに取得した型情報を格納します。`file:///`から始まるパスは`monaco`の仕様で、`node_modules/@types`に格納されている型情報を取得するために必要です。

### 3. monaco editorでJSXをハイライトする

`monaco-editor`は`jsx`のハイライトに標準で、現在のところ対応していないので、`monaco-textmate`というライブラリを使って`jsx`のハイライトを実現します。
コードは以下のようになります。

```ts
import { wireTmGrammars } from "monaco-editor-textmate";
import { Registry } from "monaco-textmate"; // peer dependency
import { loadWASM } from "onigasm"; // peer dependency of 'monaco-textmate'

const textmatePath =
  "https://esm.sh/githubusercontent/microsoft/vscode/main/extensions";

const registry = new Registry({
  getGrammarDefinition: async (scopeName) => {
    if (scopeName === "source.tsx") {
      const response = await fetch(
        `${textmatePath}/typescript-basics/syntaxes/TypeScriptReact.tmLanguage.json`
      );
      return {
        format: "json",
        content: await response.text(),
      };
    }
    if (scopeName === "source.ts") {
      const response = await fetch(
        `${textmatePath}/typescript-basics/syntaxes/TypeScriptReact.tmLanguage.json`
      );
      return {
        format: "json",
        content: await response.text(),
      };
    }
    if (scopeName === "source.js") {
      const response = await fetch(
        `${textmatePath}/javascript/syntaxes/JavaScriptReact.tmLanguage.json`
      );
      return {
        format: "json",
        content: await response.text(),
      };
    }
    if (scopeName === "source.jsx") {
      const response = await fetch(
        `${textmatePath}/javascript/syntaxes/JavaScriptReact.tmLanguage.json`
      );
      return {
        format: "json",
        content: await response.text(),
      };
    }
    if (scopeName === "source.css") {
      const response = await fetch(
        `${textmatePath}/css/syntaxes/css.tmLanguage.json`
      );
      return {
        format: "json",
        content: await response.text(),
      };
    }
    if (scopeName === "text.html.basic") {
      const response = await fetch(
        `${textmatePath}/html/syntaxes/html.tmLanguage.json`
      );
      return {
        format: "json",
        content: await response.text(),
      };
    }
    if (scopeName === "source.json") {
      const response = await fetch(
        `${textmatePath}/json/syntaxes/JSON.tmLanguage.json`
      );
      return {
        format: "json",
        content: await response.text(),
      };
    }
    return {
      format: "json",
      content: "",
    };
  },
});

export const initEditor = async () => {
  await loadWASM("https://esm.sh/onigasm@2.2.5/lib/onigasm.wasm");
  monaco.editor.defineTheme("myTheme", theme);
  monaco.editor.setTheme("myTheme");
  const grammers = new Map();
  grammers.set("typescript", "source.tsx");
  grammers.set("javascript", "source.jsx");
  grammers.set("css", "source.css");
  grammers.set("html", "text.html.basic");
  grammers.set("json", "source.json");
  await wireTmGrammars(monaco, registry, grammers);
};
```

`onigasm`というライブラリは`monaco-textmate`の依存ライブラリで、`monaco-textmate`は`onigasm`を使って正規表現を処理しています。`onigasm`は`WebAssembly`で実装されているので、`onigasm.wasm`を別途読み込む必要があります。
上記のコードではちょうど、`esm.sh`に`onigasm.wasm`があるので、それを読み込んでいます。

`monaco-editor-textmate`は`monaco-textmate`を使って`monaco-editor`に`tmLanguage`を登録するためのライブラリです。これを使うことでようやく`jsx`のハイライトが実現できます。

### 4. コードを実行する

前回の[記事](https://zenn.dev/ryuu/articles/browser-bundler) が参考になるのですが、大体以下のようにimportされたファイルの依存関係を解決しています。

#### importが相対パスなしで書かれている場合

importが相対パスなしで書かれている場合は、`esm.sh`を使ってCDNからモジュールを取得します。

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

#### importが相対パスで書かれている場合

importが相対パスで書かれている場合は、importされるファイルを`Blob URL`に変換して、`import`文を書き換えます。


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

### 4. iframeへの遅延反映処理

コードが変更されるたびにiframeを更新していると処理が重くなるので、`debounce`を使って遅延反映させています。
詳しくはコードをご覧ください。

## まとめ

今回はブラウザー上でVS Codeのような開発体験を実現する方法を紹介しました。`esm.sh`と`monaco-editor`を使えばかなり体験的に`VS Code`に近い開発体験を実現できることがわかりました。
さらに`ES Lint`や`Emmet`も`monaco-editor`に導入することができるそうなのでこれからもっとブラウザー上での開発体験が向上していくと思います。

環境構築不要でReactやTypeScriptのコードを書いて実現できる世界を目指しているので今後とも応援よろしくお願いします！

今回作ったツールは以下のリポジトリで公開しています。

https://github.com/steelydylan/monaco-browser-bundler
