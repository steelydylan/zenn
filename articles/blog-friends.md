---
title: "ブログを継続するためのサービス Blog Friends を個人開発した話"
emoji: "👨‍💻"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "react", "typescript", "tailwind", "supabase"]
published: true
---

ブログを継続するためのサービス Blog Friendsを個人開発しました。

https://blog-friends.com/

## どんなサービスか

自分のブログのRSSを登録すると下のようにブログの投稿履歴が確認できます。

![](https://blog-friends.com/images/activity.png)

ちょうどGitHubの草のようなイメージです。

そして、ブログの継続率などに応じて称号が付与されます。

![](https://blog-friends.com/images/reward.png)

同じようにブログの継続に励むユーザーをフォローできます。フォローするとタイムラインでそのユーザーの新着記事を確認できます。

![](https://storage.googleapis.com/zenn-user-upload/83518726384d2484a38e9c8c.png)

タイムライン上でRSSのフィードに対していいねを送ることもできます。いいねを送るとその記事の作成者にいいねが通知され継続の励みになります。またいいねを押した記事はストックできるため後で読み返すこともできます。

また、ユーザーランキング機能によってブログを頑張って継続しているユーザーがカテゴリーごとに上位表示されます。

![](https://storage.googleapis.com/zenn-user-upload/625256d0f15e329662cb0a69.png)

## なぜ作ったか

以前私はブログを継続するためのコミュニティに属していました。そのコミュニティではブログのRSSをGoogle Spreadシートに登録しておくとそれぞれのブログの継続日数がシート上に表示される仕組みがありました。
もし継続できていない人がいたら叱ってくれる運営者の方がいたのですが、そのモチベーション維持の仕組みをサービスとして開発したいという思いを3年間くらい眠らせていました。

また、最近ブログを始めたのですがそのブログを書くモチベーションを維持したいという思いで今回のサービスを作りました。

継続すればするほど他のユーザーに見つけてもらいやすくなる仕組みや表彰される仕組みがあるのでこれで私自身もこのサービスを使ってモチベも維持したいと思ってます。
また他のブロガーとこのサービスを通じて交流できるきっかけになればと思います。

また既存のブロガーを集めたサービスは人気記事ばかりが注目を浴び、なかなか頑張ってブログを継続している人にはスポットライトが当たりづらいので、そういった方々が報われるサービスを作りたいという思いもありました。

## 仕様技術

このサービスを作るにあたって以下の技術を使ってます。

- Next.js
- TypeScript
- TailwindCSS
- Supabase
- Playwright
- GitHub Actions
- SWR
- Recoil


Next.jsとTypeScriptはご存知の方多いと思うので割愛して、今回利用したTailwindCSSとSupabase、Playwrightについて紹介します。

### Supabase

https://supabase.io/

[Supabase](https://supabase.io/)はいわゆるBaasでサイトではThe Open Source Firebase Alternativeと謳っています。

ただし、Firebaseとは違いデータベースはNoSQLではなくPostgreSQLです。SQLに慣れている自分としてはこちらの方がありがたかったです。

#### RLS

ログインユーザーごとにテーブルへのアクセス権を以下のように細かく設定することができます。

- insert
- update
- delete
- read


例えば以下のようにRow Level Securityを設定すればログインしているユーザーのみが閲覧可能なコンテンツや、特定のユーザーのみが更新可能なコンテンツを設定することができます。

```
(role() = 'authenticated':: text)
```

```
uid() = profile_id
```

RLSをしっかり設定しておけばセキュリティを担保しつつ、`@supabase/supabase-js`を使ってブラウザ側から以下のようなコードでSQLから値を取得できるのでとても便利です。


```js
const { data: feeds } = await supabase
  .from('feeds')
  .select('link')
  .eq('user', user.id)
```

#### Foreign Key

PostgreSQLなのでテーブルからテーブルへの参照にForeign Keyを設定しておくことができます。
Blog Friendsではフィード情報を格納したfeedsテーブルにuser_idを設定しておき、フィード取得時にusersテーブルの情報も引っ張って表示するといった処理を行なっています。

```js
const { data: feedItems } = await supabase
  .from('feeds')
  .select(`
    link, 
    title, 
    date, 
    id,
    user (username, website, id, avatar_url)
  `)
  .order('date', { ascending: false })
```

#### Storageや認証機能

またStorageや認証機能もかなり高機能で用意されています。

今回のサービスではTwitterログインのために認証機能を利用しているのとStorageをアバター画像登録のために利用しています。

### Playwright

https://playwright.dev/

Microsoftが開発しているNode.jsでブラウザーを操作するためのツールです。E2Eテストなどに使えるのですが、今回はこのツールを使ってユーザーごとのOGPを生成しています。
ReactDOMでサーバーサイドでReactコンポーネントをHTMLに変換しその結果をPlaywrightにスクリーンショットを撮ってもらう形で実装しています。

zennのこの記事を参考にしました

https://zenn.dev/tdkn/articles/c52a0cc7bea561


おかげで以下のようにユーザーごとのブログの成績をOGPで出力することに成功しました

![](https://storage.googleapis.com/zenn-user-upload/d39825f7de68f7a257b439bb.png)


### TailwindCSS

https://tailwindcss.jp/

今まではCSSをたくさん指定するTailwindの書き方に違和感を感じており、毛嫌いしていて使っていなかったのですが新しいReactのドキュメントサイトもTailwindCSSで作られていたり、近年需要が高まりつつあるので、ちゃんと触ってみないとなと感じて今回全面的にTailwindCSSを採用しました。

TailwindCSSは周辺ツールやコミュニティが充実しているのでこの場合Tailwindでどうやって書くんだろうといった悩みもすぐに解決することができました。

TailwindCSSにはVSCodeの拡張機能もあるのでクラスのサジェストや似た機能のクラスを指定すると叱ってくれたりして開発時のUXがとてもいいです。

#### Flowrift

特にデザイン力がない自分にとっては、TailwindCSSで作られたUIコンポーネント集があるこのサイトはサービスのデザインを考える上でとても重宝しました。

https://flowrift.com/

#### Headless UI

また、JavaScriptで挙動の調整を必要とするコンポーネントにはHeadless UIを利用しました。
Headless UIはCSSが全くあたってないコンポーネントで、TailwindCSSなどでクラス名を指定することでコンポーネントの美しい表現を実現可能です。

TailwindCSSの開発元が作っているので非常にTailwindと親和性がよく細かいコンポーネントの作り込みをしなくてすみました。

Blog Friendsではダイアログやポップオーバー、メニューなどの少し挙動制御が難しいコンポーネントはHeadless UIを利用して実現しています。

https://headlessui.dev/

### GitHub Actions

GitHub ActionsはソースコードのPush時やPRのマージの際だけではなく、例えば1時間に1度処理を実行したりなどCronとしても十分役に立ちます。今回はCronとしてGitHub Actionsを利用しています。

以下のコードでAPIを実行し、ユーザーの投稿に対して押されたいいね数のカウントと新規フィード情報取得を行なっています。

```yml
name: hourly-cron-job
on:
  schedule:
    - cron: '*/60 * * * *'
jobs:
  cron:
    runs-on: ubuntu-latest
    steps:
      - name: count-likes
        run: |
          curl --request POST \
            --url 'https://blog-friends.com/api/likes/count' \
            --header 'Authorization: Bearer ${{ secrets.CRON_KEY }}'
      - name: import-feeds
        run: |
          curl --request POST \
          --url 'https://blog-friends.com/api/feeds/import' \
          --header 'Authorization: Bearer ${{ secrets.CRON_KEY }}'
```

## 開発期間

10日くらいかかりました。ただ現在育休中で育児もあるのでなかなか個人開発に割ける時間はなく、1日、3 ~ 4時間程度の作業で、合計30 ~ 40時間くらいです。
Next.jsやTailwindCSS、Supabaseなどに本当に助けられました。。。


## 今後の課題

あまりユーザー登録されないだろうと思い、結構技術的には手抜き感があります。
以下のようなことはやりたいです。

- ユーザーが増えたらSupabaseやVercelのプランを見直したい
- Next.jsのAPIを利用しているがGCPなどにAPIを移したい

---

継続したいブログがあるという方はぜひBlog FriendsにRSS登録してみてください！

https://blog-friends.com/

