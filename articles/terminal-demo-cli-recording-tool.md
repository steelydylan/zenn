---
title: "CLIのデモ動画をテキストで書いてSVG/GIF録画できるツールを作りました"
emoji: "🎬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["cli", "terminal", "react", "typescript", "npm"]
published: true
---

## はじめに

僕はCLIツールをたまに作るんですが、README用のデモ動画を作成しようと思うと毎回骨が折れてました。

ターミナルでタイポしないように慎重にコマンドを打って、いざ録画しようとしたら録画ボタン押し忘れてた...みたいになります！
地味にめんどくさいんですよね。

そこで、テキストベースでターミナルのデモ動画を作成できるツール「terminal-demo」を作ってみました！

こんな感じのデモが実際のCLIを動かさなくてもテキストベースで楽に作れます！

![terminal-demoのデモ](https://raw.githubusercontent.com/steelydylan/terminal-demo/main/demo.svg)

https://github.com/steelydylan/terminal-demo

## 最初はWebツールとして公開しました

当初は「ブラウザ上でサクッとデモが作れたら便利だな」と思い、Webツールとして公開しました。

https://jsers.dev/service/terminal-demo

このWebツールでは、テキストでシナリオを書くと、その場でターミナルのアニメーションをプレビューできます。macOS風のターミナルウィンドウで、実際にタイピングしているようなアニメーションが流れるので、見栄えも良い感じです。

ちなみに、このWebツールにはClaude Code向けのプロンプトが用意されています。「Copy Prompt」ボタンでプロンプトをコピーして、自分のCLIツールのコードベースを読み込んでもらったうえでClaude Codeに渡せば、シナリオを自動生成してくれます。自分でシナリオを書かなくても結構いい感じに作ってくれるので、ぜひ試してみてください。

このツールを作ったときにXでポストしたら、思った以上に反響がありました。

https://x.com/steelydylan/status/2024680758192066985

僕しか得しないんじゃないかと思ってたので正直これはびっくりしました笑

Webツールとしてはこれで十分便利だったのですが、よくよく考えると「ターミナルでそのまま録画できた方が便利なのでは？」と思うようになりました。

## CLIでの録画機能を追加

そこで、マークダウン形式のシナリオファイルを読み込んで、asciinema形式（`.cast`）で録画できる機能を追加しました。

:::message
asciinemaはターミナルの操作を録画・再生できるツールです。`.cast`ファイルはそのフォーマットで、SVGやGIFに変換するツールも充実しています。
https://asciinema.org/
:::

下のようにシナリオが書かれたファイルを指定して、--recordで出力したいファイル名を決めます！

```bash
# シナリオファイルを再生
npx terminal-demo play demo.md

# 録画してcastファイルに出力
npx terminal-demo play demo.md --record output.cast
```

録画モードでは、コマンドを実行するといきなり録画が始まるのではなく、「Press Enter to start recording...」というメッセージが表示されます。Enterを押すと録画がスタートするので、準備ができてから始められます。

シナリオファイルはこんな感じで書けます。

```markdown
# install
Install the CLI

---
$ npm install my-cli
[spinner:1500] Installing...
> [green]✓ Installed successfully[/green]

# setup
Interactive setup

---
$ my-cli init
[select:1500] Choose framework: | React, Vue, Angular | 0
[progress:2000:100] Setting up...
> [green]✓ Done![/green]
```

`$`で始まる行がコマンド入力、`>`で始まる行が出力、`[spinner:1500]`でローディングアニメーション、といった具合にシンプルな記法で書けます。色付けも`[green]`のようなタグで指定できます。

これで、シナリオを一度書いておけば何度でも同じデモを再現できるようになりました。タイポの心配もないし、録画ボタンの押し忘れもありません。

## SVGで出力するのがおすすめ

録画したデモは、GIFよりもSVGで出力するのがおすすめです。

SVGはGIFと比べてファイルサイズがかなり軽量で、同じ内容でも1/10以下になることもあります。GitHubのリポジトリ容量を圧迫しないのは地味に嬉しいポイントですね。

さらに、SVGはベクター形式なのでどんな解像度でもくっきり表示されます。Retinaディスプレイでもぼやけないし、GitHubのREADMEに貼っても綺麗に見えるんですよね。

svg-term-cliを使えば簡単に変換できます。

```bash
# svg-term-cliをインストール
npm install -g svg-term-cli

# castファイルからSVGに変換
svg-term --in output.cast --out demo.svg --window
```

READMEには`![demo](./demo.svg)`と書くだけでOKです。

## LPやドキュメントサイトにも埋め込める

terminal-demoはReactコンポーネントとしても提供しているので、自社サービスのLPやドキュメントサイトに埋め込むこともできます。

```tsx
import { TerminalDemoComponent } from 'terminal-demo/react'
import 'terminal-demo/style.css'

function LandingPage() {
  const scenarios = [
    {
      name: 'demo',
      steps: [
        { type: 'prompt' },
        { type: 'command', text: 'npx create-my-app', delay: 60 },
        { type: 'spinner', text: 'Creating app...', duration: 2000 },
        { type: 'output', text: '[green]✓ Success![/green]' },
      ]
    }
  ]

  return (
    <div className="hero-section">
      <h1>My Awesome CLI</h1>
      <TerminalDemoComponent
        title="my-cli — zsh"
        scenarios={scenarios}
        autoPlay
        loop
        theme="dark"
      />
    </div>
  )
}
```

プロダクトのLPにこれを埋め込むと、静的なスクリーンショットより動きがあって良い感じです。実際にコマンドを打っているようなアニメーションが流れるので、見てる人も「お、なんか良さそう」って思ってくれるんじゃないかなと。

CDNから直接読み込んで使うこともできるので、既存のHTMLページにサクッと追加することも可能です。

```html
<link rel="stylesheet" href="https://esm.sh/terminal-demo/dist/style.css">
<script type="module">
  import { TerminalDemo } from 'https://esm.sh/terminal-demo'

  const demo = new TerminalDemo(document.getElementById('demo'), {
    title: 'my-cli — zsh',
    scenarios: [...],
    autoPlay: true,
    loop: true,
  })
</script>
```

## シナリオ記法について

シナリオの記法をもう少し詳しく説明しておきます。

`# name`で新しいシナリオを開始し、`---`でプロンプトを表示します。`$ command`でコマンド入力のアニメーション、`> text`で出力行を表示します。

ローディングを表現したいときは`[spinner:1500] Loading...`のように書きます。数字はミリ秒です。選択メニューは`[select:1500] Choose: | A, B, C | 0`で、最後の数字が選択されるインデックスです。プログレスバーは`[progress:2000:100] Installing...`で表現できます。

色は`[green]`, `[red]`, `[cyan]`, `[yellow]`, `[purple]`, `[gray]`, `[bold]`が使えます。`[green]Success![/green]`のように囲んで使います。

実際のCLIツールのデモを想定したサンプルを載せておきます。

```markdown
# create-app
Create a new project with create-my-app

---
$ npx create-my-app my-project
[spinner:2000] Downloading template...
> [green]✓[/green] Template downloaded
[progress:3000:100] Installing dependencies...
> [green]✓[/green] Dependencies installed
>
> [bold]Success![/bold] Created my-project at /Users/you/my-project
>
> [cyan]Next steps:[/cyan]
>   cd my-project
>   npm run dev

# interactive-setup
Interactive configuration wizard

---
$ my-cli config
? [cyan]What is your project name?[/cyan]
: my-awesome-project
[select:1500] Choose a framework: | React, Vue, Svelte, Angular | 0
[multiselect:2000] Select features: | TypeScript, ESLint, Prettier, Testing | 0,1,2
[spinner:1500] Generating config...
> [green]✓[/green] Configuration saved to [cyan]my-cli.config.js[/cyan]

# deploy
Deploy to production

---
$ my-cli deploy --production
[spinner:1000] Connecting to server...
> [green]✓[/green] Connected
[progress:4000:100] Uploading files...
> [green]✓[/green] Upload complete
[spinner:2000] Building...
> [green]✓[/green] Build successful
>
> [bold][green]Deployed![/green][/bold]
> [cyan]https://my-project.example.com[/cyan]
```

ところで、この記法にはまだ正式な名前がありません。個人的には「TermScript」あたりが候補かなと思っていますが、もっと良い名前があればぜひこの記事にコメントください！

## 技術的な工夫

ライブラリとして配布するにあたって、いくつか工夫した点があります。

まず、package.jsonの`exports`フィールドでエントリーポイントを分けています。`terminal-demo`でバニラJS版、`terminal-demo/react`でReactコンポーネント、`terminal-demo/node`でNode.js向けのAPIをそれぞれ提供しています。これにより、Reactを使わないプロジェクトでは不要なReact関連のコードがバンドルに含まれません。

```json
{
  "exports": {
    ".": { "import": "./dist/index.js" },
    "./react": { "import": "./dist/react.js" },
    "./node": { "import": "./dist/node.js" },
    "./style.css": "./dist/style.css"
  }
}
```

ReactはpeerDependenciesに入れてoptionalにしています。これにより、Reactを使わないプロジェクトでも警告が出ません。

```json
{
  "peerDependencies": {
    "react": ">=18.0.0"
  },
  "peerDependenciesMeta": {
    "react": { "optional": true }
  }
}
```

また、dependenciesを空にして、開発に必要なものはすべてdevDependenciesに入れています。ライブラリとして配布する場合、dependenciesに入れたパッケージはインストール時に一緒にダウンロードされてしまうので、本当に必要なものだけに絞るのが大事ですね。このライブラリは依存なしで動作します。

asciinemaの`.cast`形式への変換も、既存のライブラリを使わずに自作しました。castファイルの仕様はシンプルで、ヘッダー行とイベント行（タイムスタンプ、イベントタイプ、データ）のJSON配列なので、自前で実装してもそこまで大変ではありませんでした。外部ライブラリへの依存を増やさずに済んだのは良かったなと思います。

## まとめ

terminal-demoを使えば、CLIのデモ動画作成が格段に楽になります。テキストでシナリオを書くだけでタイポの心配なく録画でき、SVG出力で軽量かつ高品質なデモが作れます。LPに埋め込めばプロダクトの見栄えも良くなるので、ぜひ試してみてください！

https://github.com/steelydylan/terminal-demo
