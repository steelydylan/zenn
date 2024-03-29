---
title: "GitHub Actionsでリリースを自動化しテーマの品質を高める施策"
emoji: "🛠️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["githubactions", "github", "wordpress"]
published: true
---

この記事でいうテーマとはWordPressやGatsbyにおけるデザインテンプレートを指しています。

## 開発に携わっているテーマ

私は趣味や仕事でNext.jsやWordPress、Reactなど有償、無償テーマの開発をする機会が多くあります。
こういったテーマを開発するにあたって、品質を高めるためのGitHub Actionsの効果的な設定が少しずつ見えてきたので、今回はそれを記事にしたいと思います。

以下は私が開発に携わっているテーマです！

### SANGO

WordPressテーマ、装飾が豊富でWordPressのブロックエディターにも完全対応。
カスタマイズのしやすさや表示スピードが高速なことが自慢です！

https://saruwakakun.design/


###  Awesome （Free）

https://theme-awesome.vercel.app/

Next.js製のmdxで動作するブログテーマ。ContentfulやStrapiなどの外部サービスを必要とせずマークダウンだけで動作するブログテーマ。
Vercelにデプロイするだけで簡単にブログが始められます。

### Notion Portfolio Site （Free） 

https://github.com/steelydylan/notion-portfolio-site

Notion Public APIと連携したポートフォリオ用テーマ.
詳しい記事はこちらに書いています。

https://zenn.dev/steelydylan/articles/portfolio-with-notion-api

## GitHub Actionsの設定場面

趣味で作っている無償テーマならまだ自由に開発しても許されるのですが、
1万円以上する有償テーマSANGOの場合、利用者も多くいらっしゃることもあり責任重大です。不具合があると使ってくださっているたくさんのユーザーにご迷惑をかけてしまうことになります。
また万が一不具合があった場合でもどこでその不具合を作り込んでしまったのかある程度目処をつけられるようにしておく必要があります。

そこでちょっとでも心の負担を減らすために以下の場面でGitHub Actionsを利用して特にリリース時の作業を効率化しています。

1. バージョンごとにどのようなリリースを行なったのか分かるようにする
2. バージョンごとの完成系をリリースノートからダウンロードできるようにする
3. サポートしているPHPの一番低いバージョンで問題が起こらないか構文チェックを行う

### 1. バージョンごとにどのようなリリースを行なったのか分かるようにする

以下はGitHubのリリースノートのページなのですが、バージョンごとにどのようなリリースを行なったのか分かるようにGitHub Actionsで自動出力しています。

![](https://storage.googleapis.com/zenn-user-upload/bff4553d5638-20211120.png)

これによりユーザーさんから質問があった際にどのバージョンで該当の機能がリリースされているか把握しやすくなります。
これによりユーザーさんに利用いただいているテーマのバージョンを尋ねるだけで問題解決の糸口が見つかるケースがもあります。

リリースノートは以下のツールを利用して生成しています。

https://github.com/marketplace/actions/release-drafter

#### release-drafter.ymlの設定
このツールを使ってリリースノートを生成するためにはrelease-drafter.ymlの設定が必要です。
自分は以下のような設定を行なっています。

:::details .github/release-drafter.yml
```yml
name-template: 'v$RESOLVED_VERSION 🌈'
tag-template: 'v$RESOLVED_VERSION'
exclude-labels:
  - 'release'
  - 'auto'
categories:
  - title: '🚀 Features'
    labels:
      - 'feature'
      - 'enhancement'
  - title: '🐛 Bug Fixes'
    labels:
      - 'fix'
      - 'bugfix'
      - 'bug'
  - title: '🧰 Maintenance'
    label: 'chore'
change-template: '- $TITLE @$AUTHOR (#$NUMBER)'
change-title-escapes: '\<*_&' # You can add # and @ to disable mentions, and add ` to disable code blocks.
template: |
  ## Changes
  $CHANGES
```
:::


この設定では、GitHubのPull Requestに`fix`、`bug` `bugfix`などのラベルがついていると以下のようにリリースノートにBug Fixesというタイトルの下にPull Request時のタイトルが並ぶようになります。

```md
## 🐛 Bug Fixes

- テーマの有効化時にはレガシーウィジェットが有効化されるように調整 @steelydylan (#327)
```

また、`feature`や`enhancement`などのラベルがついていると以下のようにFeaturesというタイトルの下にPull Request時のタイトルが並びます。

```md
## 🚀 Features

- カスタマイザーにインフィード広告の設定を追加 @steelydylan (#339)
```

ただこれを実現するには必ずPull Requestごとに正しいラベルが設定されている必要があります。

![](https://storage.googleapis.com/zenn-user-upload/8c00f95cff4e-20211120.png)

ただし、そこは人なので忘れてしまうケースが多いです。
そのためPull Request時にラベルが設定されていないとGitHub Actionsに怒ってもらうように以下のように設定しています。

:::details .github/workflows/pr.yml

```yml
name: PR
env:
  CI: true
on:
  pull_request:
    branches:
      - master
      - development
    types:
      - opened
      - synchronize
      - closed
      - labeled
      - unlabeled
    tags:
      - "!*"
jobs:
  release:
    name: Setup
    runs-on: ubuntu-latest
    steps:
      - name: check label
        if: |
          github.event_name == 'pull_request' && 
          !contains(github.event.pull_request.labels.*.name, 'fix') && 
          !contains(github.event.pull_request.labels.*.name, 'bugfix')  && 
          !contains(github.event.pull_request.labels.*.name, 'enhancement')  && 
          !contains(github.event.pull_request.labels.*.name, 'chore')  && 
          !contains(github.event.pull_request.labels.*.name, 'feature')  && 
          !contains(github.event.pull_request.labels.*.name, 'release') &&
          !contains(github.event.pull_request.labels.*.name, 'auto')
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          URL: ${{ github.event.pull_request.comments_url }}
        run: |
          echo "Please add one of the following labels: fix, bugfix, enhancement, chore, feature, release" >> comments
          sed -i -z 's/\n/\\n/g' comments
          curl -X POST \
              -H "Authorization: token ${GITHUB_TOKEN}" \
              -d "{\"body\": \"$(cat comments)\"}" \
              ${URL}
          exit 1
```
:::

これにより下の画像のようにラベルがついていない場合はPR上で注意してくれます。

![](https://storage.googleapis.com/zenn-user-upload/a1591e664459-20211120.png)

### 2. バージョンごとの完成系をリリースノートからダウンロードできるようにする

先程のリリースノートからテーマを完成系でダウンロードできるようにしています。
それまでは手作業でローカルのテーマをzipしたりしていたのですが、それだとヒューマンエラーでテーマのバージョン間違いや、意図しないファイルが混在してしまうケース、zipするフォルダを間違えてしまうといったケースが発生してしまいます。

バージョンごとのリリースノートで完成系のファイルをダウンロードできるようにしておけばそういった誤りを防ぐことができます。

![](https://storage.googleapis.com/zenn-user-upload/5c8196179e81-20211120.png)

そこでGit Flowの考え方を一部採用し、以下のフローでリリースノートが完成し、そのリリースノートから成果物をダウンロードできるようにしました。

1. `develop`ブランチから`feature/***`ブランチを切る
2. `enhancement`などのラベルをつけて`develop`にマージする
3. リリース時は`develop`から`release/2.11.0`などのブランチ名を作る
4. `release/2.11.0`から`master`に向けてPRを作る
5. `master`にマージされるとリリースノートが生成される



4のステップではPR時に以下のようなワークフローを作っています。

:::details version-check

```yml
name: Version Check
on:
  pull_request:
    branches:
      - master
    types:
      - opened
      - synchronize

jobs:
  auto-bumping:
    runs-on: ubuntu-latest
    steps:
    - name: checkout
      uses: actions/checkout@v1
    - name: setup Node
      uses: actions/setup-node@v1
      with:
        node-version: 12.x
        registry-url: 'https://npm.pkg.github.com'
    - name: install
      run: yarn --frozen-lockfile
    - name: version check #ここでブランチ名とpackage.jsonのバージョンが一致しているかをチェック
      run: BRANCH_NAME=$HEAD_BRANCH node ./tools/version-check.js
      env:
        HEAD_BRANCH: ${{ github.head_ref }}
```

```js:version-check.yml
const pkg = require("../package.json");

const branch = process.env.BRANCH_NAME;
 // 例えば、release/2.11.0というブランチ名の場合、version変数に2.11.0を格納する
const [type, version] = branch.split("/");

console.log(pkg.version, version);

if (pkg.version !== version) {
  throw new Error("version not matched");
}
```

:::

ここでは`release/***`ブランチの`***`に当たるバージョンと`package.json`のバージョンが一致しているかを確認しています。


5のステップにおいてはreleaseブランチのPRがmasterにマージされた際のワークフローを以下のように設定しています。

:::details .github/workflows/publish.yml
```yml
name: Publish
env:
  CI: true
# masterブランチにpushした時のみ実行するワークフロー
on:
  push:
    branches:
      - master
    tags:
      - "!*"

jobs:
  release:
    name: Release
    runs-on: ubuntu-latest
    steps:
      - name: checkout
        uses: actions/checkout@v1
      - name: setup Node
        uses: actions/setup-node@v1
        with:
          node-version: 12.x
          registry-url: 'https://registry.npmjs.org'
      - name: install
        run: yarn --frozen-lockfile
      - name: package-version #ここで環境変数にpackage.jsonのバージョンをセット
        run: node -p -e '`PACKAGE_VERSION=${require("./package.json").version}`' >> $GITHUB_ENV
      - name: package-version-to-git-tag #ここで先程の環境変数をgitのtagに指定
        uses: pkgdeps/action-package-version-to-git-tag@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          github_repo: ${{ github.repository }}
          git_commit_sha: ${{ github.sha }}
          git_tag_prefix: ""
          version: ${{ env.PACKAGE_VERSION }}
      - name: get-npm-version
        id: package-version
        uses: martinbeentjes/npm-get-version-action@master
      - name: create release draft #ここでリリースノートを作成
        id: create-draft
        uses: release-drafter/release-drafter@v5.12.1
        with:
          version: ${{ steps.package-version.outputs.current-version }}
          name: ${{ steps.package-version.outputs.current-version }}
          tag: ${{ steps.package-version.outputs.current-version }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      - name: Upload Release Asset #ここで成果物をリリースノートにアップロード
        id: upload-release-asset 
        uses: actions/upload-release-asset@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          upload_url: ${{ steps.create-draft.outputs.upload_url }} 
          asset_path: ./sango-theme.${{ steps.package-version.outputs.current-version }}.zip
          asset_name: sango-theme.${{ steps.package-version.outputs.current-version }}.zip
          asset_content_type: application/zip
```
:::

package.jsonに記述されているテーマのバージョンが自動でgitのタグに設定され、そのgitのタグがリリースノートに自動で記載されるのがポイントです！

また、`upload-release-asset` というGitHub Actionを使うことで簡単にRelease Drafterから成果物を受け取ってアセットをリリースノートにアップロードすることができました。


### 3. サポートしているPHPの一番低いバージョンで問題が起こらないか構文チェックを行う

またWordPressテーマの開発などではレンタルサーバーごとのPHPのバージョンの違いが不具合につながったりするケースが多いです。例えば開発者が最新のPHP8系で開発しているのに対し一般のユーザーの方は5系のレンタルサーバーを使っていたりしてバージョンの違いからエラーになることがあります。

そこでテーマがサポートする最低限のバージョンに合わせてPHPの構文チェックをPull Request時のワークフローの中で取り入れています。

phplintというパッケージが構文チェックにとても便利です！

https://packagist.org/packages/overtrue/phplint

GitHub Actionsで`composer install`後にこのphplintを実行し意図しないPHPの構文が存在していないかチェックをします。

## まとめ

テーマの開発は大変なことが多いです。特にレンタルサーバーによって色々なPHPのバージョンが存在するWordPressのテーマ開発はとても大変です。WordPress自体の仕様変更に翻弄されるケースも多いです。
またリリースを焦っている時などはテーマの梱包に失敗したりするなどのミスを起こしがちです。
GitHub ActionsやCIの仕組みを利用すればそういったトラブルやストレスを軽減できます。
もしテーマを開発する機会があった際には今回の施策を思い出していただけると幸いです！

