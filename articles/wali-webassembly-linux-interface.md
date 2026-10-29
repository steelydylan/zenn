---
title: "WASIじゃない、もう一つのインターフェース「WALI」という提案"
emoji: "🐧"
type: "tech" # tech: 技術記事 / idea: アイデア
publication_name: "progate"
topics: ["webassembly", "wasi", "linux", "llvm", "wasm"]
published: false
---

## はじめに

最近、仕事でいろんな言語やツールをWebAssembly（wasm）にして、ブラウザーの中で動かすことをやっています。RubyやPython、vimのような、もともとLinuxやMacで動いているプログラムをソースからwasmにビルドする作業です。

このとき毎回悩むのが、ファイルを開く、プロセスを増やす、といった機能をどうするかです。wasmそのものはこうした機能を持っていないので、外から用意してあげる必要があります。よく使われるのは、JavaScriptで用意した関数に任せるEmscriptenと、WASIという標準の関数のセットを使う方法の2つです。

僕はWASIを使うことが多いのですが、`fork`が無い、シグナルが無い、`mmap`が無い、と毎回のように引っかかります。CLIをひとつwasmにしたいだけなのに、ソースのあちこちに`#ifdef __wasi__`を足すことになるんですよね。そんな中で知ったのが、WALI（WebAssembly Linux Interface）という研究です。

一言でいうと「WASIのような新しいAPIを作るのはやめて、Linuxのシステムコールをそのままwasmに見せてしまおう」という提案です。最初に読んだときは「それアリなの？」と思ったのですが、調べていくと思った以上にちゃんと考えられていて、しかもLLVMやRustにもすでにターゲットとして入っていました。

この記事では、WALIがどんなものなのかを論文と実物のバイナリの両方から覗いてみます。

## 先に言葉を整理しておきます

この記事は、wasmにしたプログラムがファイルやプロセスを使いたいときの話です。普段Webを書いていると見慣れない単語がたくさん出てくるので、先にざっくり整理しておきます。

| 言葉 | ざっくりした意味 |
| --- | --- |
| システムコール | プログラムがLinuxのカーネルに「ファイルを開いて」「画面に書いて」とお願いするための入口です。`read`、`write`、`fork`など、Linuxには数百個あります |
| libc | C言語の標準ライブラリです。`printf`や`fopen`の中で、最終的にシステムコールを呼んでいます。muslはその実装のひとつで、小さくて読みやすいのが特徴です |
| import | wasmのモジュールが「この関数は外からもらいます」と宣言しているものです。wasmは自分だけではファイルも画面も触れないので、外とのやり取りは全部importした関数を通します |
| リニアメモリ | wasmのプログラムが使える1本の大きなバイト列です。wasmの中のポインタは、このバイト列の先頭から何バイト目か、という数字です |
| WASI | wasmをブラウザーの外で動かすために標準化が進められている、ファイルや時計などを使うための関数のセットです。preview1、0.2、0.3と版を重ねています |
| Emscripten | CやC++をwasmにするコンパイラーです。ファイルなどの機能はJavaScriptで用意したものを使うのが基本で、ブラウザーで動かすときによく使われます |
| wasm32 | ポインタが32ビットのwasmのことです。普段見かけるwasmはほぼこれです |
| ABI | コンパイル済みのプログラム同士が合わせておく約束ごとです。`long`が何バイトか、構造体をどう並べるか、などが含まれます。ABIが違うものは混ぜて使えません |
| ターゲットトリプル | コンパイラーに「どこ向けに作るか」を伝える名前です。`x86_64-linux-gnu`や`wasm32-wasip1`のように、CPUの種類や動かす環境の名前を並べて書きます |
| WAMR | WebAssembly Micro Runtimeの略で、Bytecode Allianceが開発している小さなwasmの実行エンジンです |

## Linuxでforkを呼ぶと何が起きているか

WALIの話をする前に、普通のLinuxでCのプログラムがファイルやプロセスを使うときの流れを確認しておきます。ここがわかると、WASIやWALIが「どこを差し替えているのか」がすっと入ってくると思います。

たとえば、こんなCのプログラムがあったとします。

```c
#include <stdio.h>
#include <unistd.h>

int main(void) {
  pid_t pid = fork();
  if (pid == 0) {
    printf("子プロセスです\n");
  } else {
    printf("親プロセスです（子のpidは%d）\n", pid);
  }
  return 0;
}
```

`fork()`も`printf()`も、自分で書いた関数ではないですよね。これはlibcが用意している標準の関数です。Cのプログラムは、ファイルを開くのもプロセスを増やすのも、基本的にlibcの関数を呼ぶだけで済むようになっています。

ではlibcの`fork()`の中で何をしているかというと、カーネルに「プロセスを複製してください」とお願いするシステムコールを発行しています。`printf()`も、文字列を組み立てたあと最後は`write`というシステムコールで標準出力に書き込んでいます。図にするとこんな感じです。

![自分のプログラムがlibcの関数を呼び、libcがシステムコールでLinuxカーネルにお願いする3段の図](/images/wali-webassembly-linux-interface/libc-syscall-layers.png)

つまり、自分のプログラムはlibcの関数を呼んでいるだけで、カーネルとやり取りしているのはlibcのほうです。この「libcとカーネルの境目」がシステムコールなんですね。

wasmの場合、一番下のシステムコールの命令がありません。wasmは自分だけではカーネルに何もお願いできないので、どこかで「外からもらった関数」、つまりimportに置き換える必要があります。

Emscriptenでは、この部分をJavaScriptで用意した関数が受け持ちます。WASIの場合は、wasi-libcという専用のlibcが、`open()`や`write()`のようなlibcの関数をWASIの関数に読み替えます。ただ、WASIに対応する関数が無いものは読み替えようがないので、wasi-libcには`fork()`そのものがありません。冒頭の「`fork`が無い」というのはこういうことなんです。

## WASIの何がつらいのか

WALIの話に入る前に、WALIの論文がWASIのどこを問題にしているのかを見ておきます。

論文はカーネギーメロン大学のArjun Rameshさんたちによるもので、EuroSys '25に通っています。最初にarXivに出たときのタイトルが「Stop Hiding The Sharp Knives: The WebAssembly Linux Interface」で、なかなか攻めた名前なんですよね。いまは「Empowering WebAssembly with Thin Kernel Interfaces」に変わっています。

https://arxiv.org/abs/2312.03858

論文がまず指摘しているのは、WASIがいろいろな環境をまたぐために作られた新しいAPIだという点です。新しいAPIなので、実行エンジンごと・環境ごとに実装し直す必要があります。しかもPOSIXと形が違うので、既存のソフトウェアはそのままでは動きません。メモリマップやプロセス、シグナルも無いので、そこを使っているプログラムは書き換えるしかないんですよね。エンジン側の実装も大きくなりがちで、libuvwasiのpreview1の実装はエンジン側のコードだけで6,000行を超えると書かれていました。

もうひとつが、仕様がまだ動いている最中で、長く使うものを載せるには不安定だという点です。これはここ数年WASIを追いかけてきた人なら実感があると思います。preview1から、wasmのモジュール同士を型付きでつなぐComponent Modelという仕組みを前提にしたWASI 0.2に移り、2026年6月にはasyncを入れたWASI 0.3.0が出ました。

https://wasi.dev/releases/wasi-p3

どれも良い方向の進化なのですが、そのたびにツールチェーンやランタイムが追いかけることになるのは地味にしんどいところです。

## WALIのアイデア

そこでWALIは発想を逆にします。さっきの図でいうと「libcとカーネルの境目」、つまりすでに何十年も安定しているLinuxのシステムコールを、そのままwasmのimportとして出してしまおう、というものです。

WALIはLinuxのシステムコールの上に薄く1枚かぶせるだけの層です。WALIの関数はLinuxのシステムコールとほぼ1対1で対応していて、やることは主にwasmのリニアメモリ上のポインタとホストのアドレスの変換くらいです。

こうすると、libcはmuslをほぼそのまま使えます。muslはもともとシステムコールを発行する部分が1か所にまとまっているので、CPUの命令でカーネルを呼んでいたところをimportした関数の呼び出しに差し替えるだけで済むんです。`fork()`もlibcの関数としてちゃんと残ります。libcがLinuxのつもりで動くので、その上に載っているbashやvim、sqliteも、ほとんどソースを変えずにwasmになります。エンジン側も小さくて、論文ではWAMRへの実装が約2,000行のC、そのうちプラットフォーム固有の部分は100行未満だそうです。

理想のAPIを作るより、もう世の中で一番使われているAPIに合わせてしまう。この割り切りがWALIのおもしろいところなんですよね。

## 実物を覗いてみる

論文を読むだけだとイメージしにくいので、WALIのリポジトリに入っているビルド済みのバイナリを見てみます。

https://github.com/arjunr2/WALI

`examples/precompiled`の下に、lua・bash・vim・sqliteなどがwasmになって置いてあります。WALIのランタイムはLinuxが前提なので、Macだと動かすにはDockerが要るのですが、中身を覗くだけなら`wasm-tools`があれば十分です。

```sh
git clone --depth 1 https://github.com/arjunr2/WALI.git
cd WALI/examples/precompiled
wasm-tools print lua/lua.wasm | grep '(import' | head
```

すると、こんなimportが並んでいます。見やすいように、関数名と型番号は省きました。

```wasm
(import "wali" "SYS_read"  (func (param i32 i32 i32) (result i64)))
(import "wali" "SYS_write" (func (param i32 i32 i32) (result i64)))
(import "wali" "SYS_open"  (func (param i32 i32 i32) (result i64)))
(import "wali" "SYS_fstat" (func (param i32 i32) (result i64)))
(import "wali" "SYS_mmap"  (func (param i32 i32 i32 i32 i32 i64) (result i64)))
```

importの名前空間は`wali`ひとつだけで、関数名が`SYS_read`、`SYS_write`、`SYS_mmap`…と、Linuxのシステムコールの名前そのものになっています。WASIだと`fd_write`に「どこから何バイト」を並べた配列を渡して、書けた長さをポインタで受け取る、という独特の形でしたが、こちらは`write(fd, buf, count)`がそのままの形で出ています。戻り値も、成功なら値、失敗ならエラーの番号をマイナスにしたもの（`-errno`）を返す、というLinuxのシステムコールの流儀そのままです。

同じようにbash.wasmのimportも眺めていたら、驚くものが並んでいました。

```wasm
(import "wali" "SYS_fork"   (func ...))
(import "wali" "SYS_execve" (func ...))
(import "wali" "SYS_wait4"  (func ...))
(import "wali" "SYS_kill"   (func ...))
(import "wali" "SYS_pipe2"  (func ...))
```

wasmが`fork`をimportしているんですよね！WASIだとそもそもプロセスという概念が薄いので、bashをwasmにしようとしたら`fork`と`exec`まわりを全部書き換えることになります。WALIではそれをそのままホストのLinuxに投げてしまいます。論文でも、`fork`を含むプロセス・スレッド系のシステムコールのほとんどは素通しで実装できる、と書かれています。

ちなみにスレッドも使えます。WALIでは、スレッドを1本増やすたびに同じwasmをもう1つ起動して、メモリだけを共有させる、という方法で動かしているそうです。

## 名前でimportするのがうまい

もうひとつ「なるほど」と思ったのが、importをシステムコールの番号ではなく名前で結んでいるところです。

Linuxのシステムコール番号はCPUのアーキテクチャごとに違います。たとえば`write`はx86_64だと1番、arm64とriscv64だと64番です。さらに`open`や`stat`のような古いシステムコールは、arm64やriscv64にはそもそもなくて、`openat`や`newfstatat`だけがあります。

WALIの仕様はリポジトリの`spec/`にPythonで書かれていて、そこを見るとこの対応表がそのまま載っています。

```python
impl("read",  ["int fd", "void* buf", "size_t count"], ArchNrs(x86_64=0, arm64=63, rv64=63)),
impl("write", ["int fd", "void* buf", "size_t count"], ArchNrs(x86_64=1, arm64=64, rv64=64)),
impl("open",  ["char* pathname", "int flags", "mode_t mode"], ArchNrs(x86_64=2)),
```

wasmのバイナリは`SYS_open`という名前でimportしておいて、エンジンがホストのアーキテクチャに合わせて番号に変えたり、無いものは他のシステムコールで代わりにやったりします。これで同じwasmがx86_64でもarm64でもriscv64でも動くわけです。

仕様にはLinuxのシステムコールがひと通り並んでいて、よく使われるものから実装が進められています。システムコール以外には、コマンドライン引数を受け取る`__cl_get_argc`のような、wasmならではの関数が少しだけ足されています。

## wasm32なのに`long`が64ビット

ここが今回いちばん「細かいけど驚いた」ところです。

普通、wasm32ではポインタも`long`も32ビットです。WASIのターゲットでもEmscriptenでもそうなっています。ところがWALIでは、wasm32なのに`long`が64ビットなんです！

RustのWALIターゲットのドキュメントには、普通のwasm32との一番の違いは`long`が64ビットであること、そしてそれは64ビットのホストへソースをなるべく変えずに持っていくためだ、と書かれています。

https://doc.rust-lang.org/rustc/platform-support/wasm32-wali-linux.html

ホストが64ビットのLinuxなので、カーネルとやり取りする構造体のレイアウトを64ビットのLinuxに寄せておけば、変換せずにそのまま渡せる場面が増える、という理屈です。実際、WALIの仕様の型定義はCの型の大きさを64ビットのLinuxに合わせてあって、`struct stat`は144バイト、つまりx86_64のLinuxと同じ大きさで定義されています。

一方でポインタは4バイトのままで、`size_t`も32ビットです。LLVMのclangのソースを見ると、WALIのときだけこう分岐しています。

```cpp
if (T.isWALI()) {
  // The WALI ABI is documented here:
  // https://doc.rust-lang.org/rustc/platform-support/wasm32-wali-linux.html
  // Currently, this ABI only applies to wasm32 targets and notably requires
  // 64-bit longs
  LongAlign = LongWidth = 64;
  SizeType = UnsignedInt;
  PtrDiffType = SignedInt;
  IntPtrType = SignedInt;
}
```

「ポインタは32ビット、`long`は64ビット」というのは、普通のCの世界だとなかなか見ない組み合わせです。`long`にポインタを入れているような古いCのコードはこれで壊れるので、そこは割り切っているんだなと思います。

## LLVMとRustにも入っている

研究のプロトタイプで終わっているのかと思いきや、上流のツールチェーンにもちゃんと入っています。

Rustでは`wasm32-wali-linux-musl`がTier 3のターゲットとして入っています。`cfg`は`target_os = "linux"`かつ`target_family`に`wasm`と`unix`の両方が立つ、というおもしろい形です。

LLVMでは2025年9月にRFCが出て、

https://discourse.llvm.org/t/rfc-new-wasm-linux-target-support/88203

ターゲットの名前を何にするかでけっこう議論がありました。最初はRustに合わせて、vendorの位置に`wali`を入れた`wasm32-wali-linux-musl`が提案されたのですが、「`long`の大きさが違うABIなのだから、トリプルで区別できるべき」「LLVMでは普通ABIの違いをvendorでは表さない」という意見が出て、最終的に環境の位置に入れた`wasm32-linux-muslwali`に落ち着いています。この名前でのPRが2025年10月にマージされました。

https://github.com/llvm/llvm-project/pull/162581

clangでこのターゲットを指定すると、`__linux__`と`__unix__`に加えて`__wali__`が定義されます。つまりCのプログラムから見ると「Linuxでコンパイルされている」ように見えるので、`#ifdef __linux__`で分岐しているコードがそのままLinuxのほうに入ってくれます。WASIだと`__linux__`は立たないので、ここも地味に大きい違いだなと思います。

## WASIはWALIの上に載せればいい

ここまで読むと「じゃあWASIはいらないの？」となりそうですが、論文の主張はそうではありません。

WALIは「カーネルとの境目をできるだけ薄く・低くする」ためのもので、WASIのように「このディレクトリだけ触ってよい」と権限を渡して使うAPIは、WALIの上にwasmのモジュールとして載せればいいと言っています。実際に論文では、WASIの実装であるlibuvwasiをソースを変えずにWALIの上でビルドし、テストが全部通ることを確かめています。

こうすると、WASIの実装が信頼しないといけないエンジンの中から外に出ていくので、エンジンは小さく、安全になる、という理屈です。WASIを置き換えるというより、WASIより下にもう一段の層を作る提案なんですよね。

## 気になるところ

もちろん良いことばかりではありません。

まず、ホストがLinux前提です。WALIのエンジンはLinuxのシステムコールを素通しする作りなので、MacやWindows、ブラウザーではそのままは動きません。リポジトリでも、Linux以外ではDockerで動かす手順になっています。

次に、WASIのバイナリと混ぜられません。`long`の大きさが違うので、wasm32-wasip1でビルドしたライブラリとWALIでビルドしたものはABIが合わないんですよね。既存のwasmの資産とは別の世界になってしまいます。

安全面の考え方も違います。wasmのプログラムが自分のメモリの外を壊せない、という性質はそのまま残ります。ただ、ファイルやネットワークには、wasmを動かしているプロセスが触れるものなら何でも触れてしまいます。「このフォルダだけ読める」のように権限を絞りたいなら、WASIのような層を上に載せることになります。

なので、「サーバーやエッジの上で、既存のLinuxのソフトウェアをそのままwasmで動かしたい」という用途にはかなり刺さる一方、「信頼できないコードを厳しく閉じ込めたい」「ブラウザーで動かしたい」という用途では、WALIそのものよりも考え方のほうが参考になるかなと思います。

## Progateでも考え方は近い

ここで少しだけProgateの話をさせてください。

Progateでは、学習者が環境構築をしなくてもコードを書いて動かせるように、いろんな言語やツールをwasmにしてブラウザーの中で動かす取り組みを進めています。以前、Node.jsの学習環境をブラウザーの中で再現した話も書きました。

https://zenn.dev/progate/articles/browser-node-without-webcontainers

言語が増えてくると、冒頭に書いた「WASIに無いもの」に何度もぶつかります。たとえばWASIのpreview1には作業ディレクトリを変える`chdir`が無いので、相対パスを解決するところを言語ごとに手当てすることになりました。全画面で動くvimのようなツールは、端末の大きさを取ったり入力の受け取り方を切り替えたりする機能がpreview1に無くて、そこも足す必要がありました。1つずつは小さな話なのですが、言語やツールの数だけ積み重なっていくんですよね。

そこで僕も、WASI独自の形に合わせて言語ごとに手当てを重ねるより、Linuxのシステムコールの形に寄せていく方向で考えています。考え方はWALIとかなり近いです。

ただ、ブラウザーの中には素通しする先のLinuxがないので、WALIをそのまま使うことはできません。また、すでにwasm32-wasip1でビルドした資産がたくさんあるので、`long`が64ビットになるWALIのABIには合わせていません。WALIの論文を読んだときは「同じことを考えている人がいて、しかも論文と上流のツールチェーンまで持っていっている」と知って、すごく励まされました。

## まとめ

WALIは、Linuxのシステムコールというすでにある境界をそのままwasmに出す、という割り切った提案でした。importには`SYS_write`のようなシステムコールの名前がそのまま並び、bashのwasmは`fork`までimportしています。番号ではなく名前で結ぶことでCPUの違いを吸収していたり、wasm32なのに`long`が64ビットだったりと、細かいところを見ていくとかなり考えて作られているのがわかります。研究で終わらず、LLVMとRustにもターゲットとして入っているのもすごいなと思います。

個人的には「新しいAPIを一から設計するより、Linuxに合わせたほうが結局みんな幸せなのでは」という問いかけがすごく刺さりました。WASIの話になるとComponent Modelやasyncの話題が多いですが、こういう「もっと下の層をどうするか」という方向の議論もあるんだなと、しみじみ思います。

Progateでも、ブラウザーの中で既存のプログラムをどこまでそのまま動かせるかは、まだまだ試行錯誤している途中です。また進んだら書きたいと思います！

## 参考

- Arjun Ramesh, Tianshu Huang, Ben L. Titzer, Anthony Rowe. "Empowering WebAssembly with Thin Kernel Interfaces." EuroSys '25. https://dl.acm.org/doi/10.1145/3689031.3717470
- arXiv版: https://arxiv.org/abs/2312.03858
- WALI のリポジトリ: https://github.com/arjunr2/WALI
- WALI の仕様: https://wasm-thin-kernel-interfaces.github.io/WALI/specification/
- Rust の `wasm32-wali-linux-*`: https://doc.rust-lang.org/rustc/platform-support/wasm32-wali-linux.html
- LLVM の RFC: https://discourse.llvm.org/t/rfc-new-wasm-linux-target-support/88203
- LLVM の PR #162581: https://github.com/llvm/llvm-project/pull/162581
- WASI 0.3.0: https://wasi.dev/releases/wasi-p3
