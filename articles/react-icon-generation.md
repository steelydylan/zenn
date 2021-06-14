---
title: "DesignOps推進の一環としてFigma上のコンポーネントを自動でnpmにpublishしてみる"
emoji: "📦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["github", "figma", "react", "typescript", "npm"]
published: true
---

## DesignOpsとは
DesignOpsとは質の高いデザインのアウトプットの維持を支えるための仕組みづくりのことです。
この仕組みを整えることでデザイナーもエンジニアもお互いストレスなくそれぞれの仕事に専念できます。


今回は、デザイナーがデザインツールのFigma上で作成した**アイコン**をReactコンポーネントに変換してnpmに定期的にpublishするところまでを自動化しました。この自動化により、わざわざFigmaにみに行って、デザインされたsvgを切り出しReactコンポーネントに変換して利用するという作業が不要になりました。Figama上でアイコンが更新されたら、新たに`npm install`するだけでアイコンを取り込んで使うことができるようになります。
js側では、以下のような雰囲気で`npm`からアイコンコンポーネントを利用可能です。

```tsx
import { MenuClose } from '@karakuri-ai/icons' // <- 現在非公開のパッケージです

export const Main = () => (
  <div> 
    <MenuClose style={{ color: '#efefef', width: '20px', height: '20px' }} />
  </div>
)
```

使い勝手的には `@material-ui/icons` に似ていると思います。

https://material-ui.com/components/material-icons/

この記事ではどのような手順で自動化に成功したのかメモがてら紹介したいと思います。

## 自動化の流れ
自動化の流れはざっくり以下のような感じです。

1. Figma上で書き出し対象のアイコンは全てコンポーネント化してもらう
2. FigmaAPIを利用してコンポーネント一覧をsvgとしてダウンロードする
3. svgに設定されているカラーを`currentColor`に変換する
4. `svgr`を利用して、svgをReactコンポーネントに変換する
5. 2~5の一連の流れをGitHub Actionsで定期的に回す

### 1. Figma上で書き出し対象のアイコンは全てコンポーネント化してもらう

Figmaではデザインを共通で使い回すために作ったパーツをコンポーネント化することができます。
コンポーネント化したアイコンは一覧でこのように確認することができます。

![](https://storage.googleapis.com/zenn-user-upload/d10bb6abea2d1de23c8fee0d.png =200x)

同時にFigmaAPIからもコンポーネント化した部品は取得しやすいのでデザイナーにコンポーネント管理してもらいましょう。

Figma上でコンポーネント化する方法はこちらの記事では触れませんがよろしければこちらの動画をご覧ください。

@[youtube](k74IrUNaJVk)

### 2. FigmaAPIを利用してコンポーネント一覧を取得できるようにする

Figmaはデザイナーだけではなくエンジニアにもフレンドリーなサービスです。FigmaAPIはこちらの公開されていて、思っている以上にたくさんのことがAPIで実現できます。

https://www.figma.com/developers/api

コンポーネント一覧を取得するにはComponent APIを利用します。以下のような形式でAPIを叩くとcomponent一覧を取得することができます。

`https://api.figma.com/v1/files/${FIGMA_FILE_KEY}/components`

ブラウザーでFigmaのファイルを開いていただくと、`https://www.figma.com/file/xxxxxxx/******` のようなURLになっています。xxxxxxxの部分が`FIGMA_FILE_KEY`にあたります。


また、リクエストヘッダーに`X-FIGMA-TOKEN`をつけて送る必要があり、このトークンは管理画面のアカウントより取得できます。

![](https://storage.googleapis.com/zenn-user-upload/ee9d4a4ee1de0dc1b8077d8c.png =200x)

取得できたコンポーネント一覧から、さらにそれぞれのコンポーネントのSVGのURLを取得します。以下のAPIを叩いていただくことで取得できます。`ids`の部分には、先ほどのComponent APIから取得できたnode_id一覧をカンマ区切りにした文字列が入ります。

`https://api.figma.com/v1/images/${FIGMA_FILE_KEY}?ids=${ids}&format=svg`


最終的なソースコードは以下のようになります。
かなり雑ですが。。。

```js:figma.js
const got = require('got');
const request = require('request');
const { resolve } = require('path');
const fs = require('fs');
const TOKEN = process.env.FIGMA_TOKEN;
const FIGMA_FILE_KEY = process.env.FIGMA_FILE_KEY;

const download = (url, path, callback) => {
  request.head(url, (err, res, body) => {
    request(url)
      .pipe(fs.createWriteStream(path))
      .on('close', callback)
  })
}

async function main() {
  const { body } = await got(`https://api.figma.com/v1/files/${FIGMA_FILE_KEY}/components`, {
    headers: {
      'X-FIGMA-TOKEN': TOKEN,
    },
    responseType: 'json',
  })
  const results = body.meta.components
  const ids = results.map(r => r.node_id).join(',')
  const { 
    body: {
      images
    } 
  } = await got(`https://api.figma.com/v1/images/${FIGMA_FILE_KEY}?ids=${ids}&format=svg`, {
    headers: {
      'X-FIGMA-TOKEN': TOKEN,
    },
    responseType: 'json',
  })
  const nodeIds = Object.keys(images);
  for (const nodeId of nodeIds) {
    const url = images[nodeId]
    const result = results.find(r => r.node_id === nodeId)
    const name = result.name
    const path = resolve(__dirname, `../assets/${name}.svg`);
    download(url, path, () => {
      console.log('done!')
    })
  }
}

main();
```

自動化のために、このスクリプトを実行するコマンドを`package.json`に登録しておきましょう。

```json:package.json
"scripts": {
  "figma": "node ./tools/figma.js",
}
```

### 3. svgに設定されているカラーを`currentColor`に変換する

2の手順で、無事にsvgを手元にダウンロードすることができましたが、そのままsvgを利用するとアイコンに背景色が設定されているため、単色でしかアイコンを利用できません。そこでsvgの色を親要素から定義できるように設定されているカラーを`currentColor`に変換します。
こうしておくことでsvgの親要素に対して、`color`を設定するだけで`svg`の色を変更することができます。
以下はassets配下に設置されているsvgのfill属性を`currentColor`に変換するコマンドになります。

```shell
$ sed -i 's/ fill=\"none\"//g; s/fill=\"[^\"]+\"/fill=\"currentColor\"/' ./assets/*.svg
```

自動化のために、`package.json`にコマンドを登録しておきましょう

```json:package.json
"scripts": {
  "figma": "node ./tools/figma.js",
  "currentColor": "sed -i 's/ fill=\"none\"//g; s/fill=\"[^\"]+\"/fill=\"currentColor\"/' ./assets/*.svg"
}
```

### 4. svgrを利用して、svgをReactコンポーネントに変換する

次にsvgrを利用して、svgをReactコンポーネントに変換します。
svgrを使えば少しjsを書く必要はありますが、アイコンを`tsx`として変換することもできます。
以下はsvgをReactコンポーネントに変換するコマンドと、tsxに変換するためのコードになります。

```shell
$ svgr -d ./src ./assets/**/*.svg --template ./template.js --ext tsx --no-svgo
```

```js:template.js
function svgBuild({ template }, _opts, { componentName, jsx }) {
  const typeScriptTpl = template.smart({ plugins: ['typescript'] })
  componentName.name = componentName.name.replace('Svg', '')
  return typeScriptTpl.ast`
    import React from 'react'
    export function ${componentName}(props: React.SVGProps<SVGSVGElement>) {
        return ${jsx}
    }
  `
}
module.exports = svgBuild
```

これを実行するとsrc配下に以下のような`tsx`ファイルが書き出されました。

```tsx:MenuClose.ts
import React from "react";
export function MenuClose(props: React.SVGProps<SVGSVGElement>) {
  return (
    <svg
      width={24}
      height={24}
      viewBox="0 0 24 24"
      xmlns="http://www.w3.org/2000/svg"
      {...props}
    >
      <path
        d="M15.7321 6.00928H0V7.57207H15.7321V6.00928Z"
        fill="currentColor"
      />
      <path
        d="M22.373 8.90796L19.1192 12.1619L22.373 15.4158L22.373 8.90796Z"
        fill="currentColor"
      />
      <path d="M15.7321 16.428H0V17.9908H15.7321V16.428Z" fill="currentColor" />
      <path
        d="M15.7321 11.2185H0V12.7813H15.7321V11.2185Z"
        fill="currentColor"
      />
    </svg>
  );
}
```

`npm install`した際、生成されたファイルが一つのindexからexportされていると便利です。
生成したファイルを全て`export`するための`index.ts`を生成するツールを作りましょう。

```js:export.js
const fs = require('fs')
const files = fs.readdirSync('./src');
let source = '';

files.forEach(f => {
  if (f.indexOf('index.ts') === -1) {
    source += `export * from "./${f.replace('.tsx', '')}"\n`
  }
})

fs.writeFileSync('./src/index.ts', source, 'utf-8')
```

さらに、TypeScriptをインストールし`tsc`コマンドで、srcからdistにアイコンを吐き出します。
一連の流れを`npm scripts`としてpackage.jsonに登録します。

```json:package.json
"scripts": {
  "figma": "node ./tools/figma.js",
  "currentColor": "sed -i 's/ fill=\"none\"//g; s/fill=\"[^\"]+\"/fill=\"currentColor\"/' ./assets/*.svg",
  "build:svg": "svgr -d ./src ./assets/**/*.svg --template ./template.js --ext tsx --no-svgo",
  "build:index": "node ./tools/export.js",
  "build:ts": "tsc",
  "build": "yarn currentColor && yarn build:svg && yarn build:index && yarn build:ts"
}
```

ライブラリをインストールした際にどこにある型定義ファイルとjsを読み込めばいいか`Webpack`などのツールが判断できるように、エントリーポイントも`package.json`に忘れないように書いておきましょう。

```json:package.json
"main": "dist/index.js",
"types": "dist/index.d.ts",
```

### 5. 2~5の一連の流れをGitHub Actionsで定期的に回す

最後にGitHub Actionsを定期的に実行してこれまでの流れを全てCIに実行してもらいましょう。

`.github/workflows/publish.yml`を作成して一連の流れをCIに登録します。GitHub Actionsはcronで定期的に命令を実行することができます。以下の書き方で30分に一度CIを動かすことができます。

```yml
on:
  schedule:
    - cron:  '*/30 * * * *'
```

`git status --porcelain`を使って、Figamaから新たに取得したsvgファイル一式によってgitに差分が発生したかチェックし、差分が発生していない場合は`exit 1`でGitHub Actionsを終了しています。
差分があった場合は、`yarn version --patch`でpackage.jsonのバージョンを更新し、commit後、`yarn publish`でアイコン一式をnpmにpublishします。

```yml:publish.yml
name: fetch
env:
  CI: true
on:
  schedule:
    - cron:  '*/30 * * * *'
  push:
    branches:
      - 'master'  

jobs:
  release:
    name: fetch
    runs-on: ubuntu-20.04
    steps:
      - name: checkout
        uses: actions/checkout@v2
        with:
          ref: 'refs/heads/master'
          fetch-depth: 0 # All history
      - name: setup node
        uses: actions/setup-node@v2
        with:
          node-version: 14.x
          always-auth: true
          registry-url: 'https://npm.pkg.github.com/'
          scope: 'もしあればここにスコープ名'
          token: ${{ secrets.GPR_TOKEN }}
      - uses: actions/cache@v2
        id: yarn-cache
        with:
          path: '**/node_modules'
          key: ${{ runner.os }}-${{ hashFiles('**/yarn.lock') }}
      - name: install
        run: yarn --frozen-lockfile
        env:
          NODE_AUTH_TOKEN: ${{ secrets.GPR_TOKEN }}
      - name: figma
        run: yarn run figma 
        env:
          FIGMA_TOKEN: ${{ secrets.FIGMA_TOKEN }}
          FIGMA_FILE_KEY: ${{ secrets.FIGMA_FILE_KEY }}
      - name: commit if changed
        run: |
          if [ -z "$(git status --porcelain)" ]; then
            echo "Clean!"
            exit 1
          else
            git config --global user.email "push@no-reply.github.com"
            git config --global user.name "GitHub Push Action"
            yarn version --patch
            git add --all
            git commit -m "GitHub Push"
            git push origin master
          fi
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      - name: build
        run: yarn run build 
      - name: publish
        run: yarn publish
        env:
          NODE_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## まとめ

以上のような手続きで無事、デザイナーもエンジニアもストレスなくアイコンの共有ができるようになりました。エンジニアもわざわざ苦手なFigmaを開く必要もないですし、デザイナーもエンジニアに余計な注文をされなくてすみます。
こういったDesignOpsは今後積極的に進めたいです。
次はアイコンだけではなく、ヘッダーやボタンなど基本的なコンポーネントもDesignOpsの力で解決できないか色々調査をしてみたいと思います。