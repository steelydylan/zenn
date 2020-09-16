---
title: OpenAPIをexpressで型として使用する
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---

## Open APIとは
Open APIとはRESTful APIを記述するためのフォーマットです。yaml形式などで記述でき、Swagger Editor などを利用すると以下のように定義したAPIをWeb上で閲覧することができます。


最近では、このyamlファイルから、TypeScriptの型を生成する仕組みもあり、自分は https://github.com/horiuchi/dtsgenerator を使わせてもらっています。

これは、OpenAPIの仕様にそって、yamlファイルからd.tsを吐き出してくれるライブラリで非常に便利なのですが、expressをtsで書こうと思った時に、expressにどうやって生成した型を繋こもうかとても悩みました。

そこで今回、さらに、dtsgeneratorで生成したコードを使って、express用のTypeScriptファイルを生成するジェネレーターを作りました。
express-ts-generatorです。

## 使い方
### まずはインストール

```sh
$ npm install dtsgenerator express-ts-generator --save-dev
```

openapi.ymlを指定してファイル生成

```sh
$ npx dtsgen openapi/openapi.yaml -o ./src/@types/openapi.d.ts && npx apigen -s ./src/@types/openapi.d.ts -d ./src/@types/api.ts
```

実行すると以下のファイルのようにexpressのController側のTypeScriptファイルが生成されます。

```js
import { Controller } from 'express-ts-generator';

export namespace Music$MusicIdController {
  export type Get = Controller<{
    response: Paths.Music$MusicId.Get.Responses.$200;
  }>;
}
export namespace MusicsController {
  export type Get = Controller<{
    response: Paths.Musics.Get.Responses.$200;
  }>;
  export type Post = Controller<{
    body: Paths.Musics.Post.RequestBody;
  }>;
}
```
express側で型を利用

```js
import { SomeController } from './types/api';
export const Post: SomeController.Post = async (
  request,
  response
): Promise<void> => {
  // request and response will be typed automatically
};

export const Get: SomeController.Get = async (
  request,
  response
): Promise<void> => {
  // request and response will be typed automatically
};
```
使うとこんな感じでrequestやresponseの型を強力に補完してくれます！


よかったらgithubにスターいただけると嬉しいです！

https://github.com/steelydylan/express-ts-generator
expressのrequestにもresponseにも型がついていて非常にプログラミングが捗ります。