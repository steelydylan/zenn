---
title: "Type Challengesに挑戦できるmosya<TC>をわずか4日でリリースできた理由"
emoji: "👨‍💻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "個人開発"]
published: true
---

今回、mosya&lt;TC&gt;という[Type Challenges](https://github.com/type-challenges/type-challenges)をブラウザーのエディターから挑戦して採点できるサービスをリリースしました。

Type ChallengesはTypeScriptの型に関する問題を解くことでGitHub上のプロジェクトで、MITでライセンスされているので、誰でも自由に利用することができます。
これを解くことで、かなりの型力が身につくので、以前から昼休みによく友人と解いていました。
今回、このことを思い出しType Challengesに挑戦しやすくするためにブラウザー上でエディターを提供し、問題を解いた後にすぐ採点できるサービスを作りました。

https://mosya.dev/type-challenges

使い方を動画にしてみたのでぜひご覧ください

https://www.youtube.com/watch?v=pRARV_-YJ1o

反響もよく4日でリリースしたサービスの割にたくさんの方に利用していただけています。
おかげ様でTwitterでのいいねも過去のサービスリリースの中で最高のいいね数を獲得しました。

@[tweet](https://twitter.com/steelydylan/status/1679636826368114690)

## 4日でリリースできた理由

4日でリリースできたというのは決して誇張ではなく、本当に4日で思いついてからリリースができました。
その理由は以下の2点が大きいと思います。

- 既存のサービス [mosya](https://mosya.dev/) をベースにしている
- Chat GPTの Code Interpreterを利用している
- 以前開発したOSSを利用

### 既存のサービスをベースにしている

今月の4/30にリリースしたばかりの[mosya](https://mosya.dev/)というサービスをベースにしています。

https://mosya.dev

mosyaはブラウザー上でHTML,CSS,JavaScript,PHPを学習するためのサービスで、見本のサイトを模写することでHTML,CSSを学習できます。
エディターはこの時点で完成されており、Type Challengesの問題を解くために必要な機能はほぼ実装されていました。
また、現在、**mosya React** という別サービスを開発していて、そこで型のテストを行う機能を実装していたので、それを流用することで、Type Challengesの問題を解くための機能を実装することができました。

このように新しくサービスを一から作るのではなく、既存のサービスをベースにして新しいサービスを作るのはかなり効率的だと今回の開発で感じました。

というのも以下の機能を実装する必要がないからです。

- ログイン機能
- ユーザー受講状況の管理
- エディターの開発
- 採点の仕組み

これらの機能は既存のサービスで実装されているので、新しく開発する必要がありません。

mosyaについての開発記録は以前記事にしていますのでこちらの記事をぜひご覧ください。

https://zenn.dev/steelydylan/articles/mosya-development

### ChatGPTの Code Interpreterを利用している

つい最近、リリースされたChatGPTのCode Interpreterを利用して、一気に問題を量産しました。
ChatGPT Code Interpreterとは、ユーザーのプロンプトから必要に応じてJupyter Notebookを実行して結果を返してくれる仕組みです。

@[tweet](https://twitter.com/steelydylan/status/1678001771913019394)

今回、この仕組みを利用して、Type Challengesの問題を解くためのコードを生成しました。
そのために、[Type Challenges](https://github.com/type-challenges/type-challenges)のレポジトリをzip化して、ChatGPTに渡しました。
その上で私が命令した内容の一部は以下です。

![](https://storage.googleapis.com/zenn-user-upload/a066759cd15f-20230716.png)

「最終的に生成したコードをzipで渡してくれ」とChatGPTに伝えると、そのコードをzipで返してくれました。

![](https://storage.googleapis.com/zenn-user-upload/2041bc2d7853-20230716.png)

![](https://storage.googleapis.com/zenn-user-upload/aa3a16da2a91-20230716.png)

もらったコードの手直しは多少必要でしたが、これを利用することで、一気に問題を量産することができ、だいぶ時間短縮につながりました。

### 以前開発したOSSを利用

ブラウザーでのTypeScriptの型チェックにこだわりました。というのもサーバーサイドで型チェックを動かすとレスポンスに時間がかかるのでなるべく早く結果を返すようにフロントエンドで型チェックのテストを動かすようにしています。

その上で、**Jest**風に型のテストが書けるツールをOSSとして以前開発していました。
これは今開発中の**mosya React**のために開発したもので、今回のmosya&lt;TC&gt;でも利用しています。

https://github.com/steelydylan/type-tester

これを使うと以下のような感じで型のテストを書くことができます。

```ts
import { TypeTester } from "browser-type-tester";

const code = `
type Concat<T extends any[], U extends any[]> = [...T, ...U];
type Result = Concat<[1], [2]>; // expected to be [1, 2]
`;
const typeTest = new TypeTester({ code });

typeTest.it("Concat<A, B>型が正しい", () => {
  typeTest.expect("Concat<['a', 'b'], ['c', 'd']>").toBe("['a', 'b', 'c', 'd']");
});

typeTest.it("Concat<A, B>はanyを返さない", () => {
  typeTest.expect("Concat<['a', 'b'], ['c', 'd']>").not.toBeAny();
});

const results = await typeTest.run();

console.log(results);
```

このOSSの開発で苦労したのが、テストのために`TypeScript`をブラウザーで動かすことでした。
まずブラウザーでは、TypeScriptの実行APIである、`ts.createProgram`が動作しませんでした。
これは`TypeScript`が実際に`fs`を使った`Node.js`前提の実装になっているためです。
そこで、`compilerHost`というオブジェクトを定義し、それを`ts.createProgram`にわたすことで、`fs`を使わずに`TypeScript`を動かすことができました。

以下のようなコードで、実際にそのファイルへのアクセスがあった場合は、`fs`を使わずにメモリ上のファイルを返すようにしています。

```ts
const compilerHost: ts.CompilerHost = {
  getSourceFile: (fileName: string) => {
    if (fileName === sourceFileName) {
      return ts.createSourceFile(fileName, code, options.target);
    }
    return undefined;
  },
  writeFile: () => {},
  getCurrentDirectory: () => "",
  getDirectories: () => [],
  getCanonicalFileName: (fileName: string) => fileName,
  useCaseSensitiveFileNames: () => false,
  getNewLine: () => "\n",
  fileExists: (fileName: string) =>
    fileName === sourceFileName
  readFile: () => "",
  getEnvironmentVariable: () => "",
};
```

また、`TypeScript`の標準ライブラリである`lib.d.ts`をブラウザーで解釈させるために、ライブラリーの型定義ファイルをブラウザーで読み込む必要もありました。

かなり重いファイルなのですが、`monaco-editor`の実装を参考にして、以下のリンク先のコードで、`lib.d.ts`をブラウザーで解釈させることができました。

https://github.com/steelydylan/type-tester/blob/master/src/expect/lib.ts

そして、最終的に`compilerHost`を以下のように書き直し、`lib.d.ts`などの標準ライブラリを読み込むことができました。

```ts
const compilerHost: ts.CompilerHost = {
  getSourceFile: (fileName: string) => {
    if (fileName === sourceFileName) {
      return ts.createSourceFile(fileName, code, options.target!);
    }
    if (libFileMap[fileName]) {
        return ts.createSourceFile(
          fileName,
        libFileMap[fileName],
        options.target!
      );
    }
    return undefined;
  },
  writeFile: () => {},
  getCurrentDirectory: () => "",
  getDirectories: () => [],
  getCanonicalFileName: (fileName: string) => fileName,
  useCaseSensitiveFileNames: () => false,
  getNewLine: () => "\n",
  fileExists: (fileName: string) =>
    fileName === sourceFileName || !!libFileMap[fileName],
  readFile: () => "",
  getDefaultLibFileName: () => libFileName,
  getEnvironmentVariable: () => "",
};
```

これにより例えば以下のような`reference`がある場合には依存する標準ライブラリを順次読み込むことができます。

```ts
/// <reference lib="es2015.iterable" />
```

`createProgram`により実行した型のチェックはレポートとして以下のように取得できます。

```ts
const program = ts.createProgram(
  [libFileName, sourceFileName],
  options,
  compilerHost
);
const diagnostics = program.emit().diagnostics
```

もし、`diagnostics`の中身が空であれば、テストが通っているとみなします。

## 今後の課題

mosya&lt;TC&gt;で「なぜその回答になるかわからない」というお声を多くいただいているので解説を掲載したいと思います。
また、ところどころ型のチェックが甘い課題があるので、そこを修正していきたいと思います。

## まとめ

今回、サービスを4日でリリースできた理由は、既存のサービスをベースにしていることと、ChatGPTのCode Interpreterを利用していること、以前開発したOSSを利用していることが大きいと思います。
このようにサービスを開発するときは以前開発したサービスの上に別サービスを載せることで大きく工数を削減できます。
また、同じドメイン内にサービスを新しくリリースすることになるので、そのドメインのSEOにも良い影響を与えると思います。
もし、今後サービスをリリースする方がいらっしゃれば、ぜひ既存の仕組みや過去リリースしたサービスを活用してみるというのも検討してみてください。

