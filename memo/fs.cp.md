# Node.jsのfs.cpでファイルのコピーを試みる

　ファイルはコピーできたが、ディレクトリはコピーできなかった。原因不明。

<!-- more -->

# ブツ

* [][]

[]:

# [fs.cp][]

　Node.js APIである[fs.cp][]を使えばファイルやディレクトリをコピーできるっぽいので使ってみた。

　`v16.7.0`から追加されたらしいので注意。私の環境は`v16.15.1`だったのでセーフ。

[fs.cp]:https://nodejs.org/api/fs.html#fscpsrc-dest-options-callback

```sh
$ node --version
v16.15.1
```

　こんな基本的なコマンドも割と最近できたみたい。Node.jsって結構昔からあったような気がするけど。[wikipedia][]によると2009年が初版らしい。なのに最近まで[fs.cp][]コマンドがなかったのが不思議。まあいいや。藪蛇になる前に進めよう。

[wikipedia]:https://ja.wikipedia.org/wiki/Node.js

# コード抜粋（成功）

## main.js

```javascript
const fs = require('fs')
ipcMain.handle('cp', async(event, src, dst) => {
    fs.cp(src, dst, ()=>{})
})
```

位置|名前|意味
----|----|----
1|`src`|コピー元のパス
2|`dst`|コピー先のパス
3|`callback`|コールバック関数

　コールバック関数は一体何に使うのか不明。引数さえ何も受け取れないみたいだし。なので空っぽのメソッドをセットした。

　コピー先に同名ファイルがあってもエラーにならずスルーされるらしい。`fs.cp(src, dest[, options], callback)`のうち`options`引数に`errorOnExist:true`プロパティを付与すれば、エラー発生するようだ。

　`options`も任意で付与できるようにするなら以下。

```javascript
ipcMain.handle('cp', async(event, src, dst, options=null) => {
    if (options) { fs.cp(src, dst, options, ()=>{}) }
    else { fs.cp(src, dst, ()=>{}) }
})
```

　`options`はオブジェクトであり、以下のようなプロパティを許容するようだ。

`options`プロパティ名|型|デフォルト値|概要
---------------------|--|-----------|----
`dereference`|`boolean`|`false`|シンボリックリンクを逆参照する
`errorOnExist`|`boolean`|`false`|コピー先が既存ならエラー
`filter`|`function`|-|コピーされたファイル/ディレクトリをフィルタリングする。`true`の項目をコピーし`false`を無視する
`force`|`boolean`|`true`|既存のファイルまたはディレクトリを上書き
`preserveTimestamps`|`boolean`|`false`|タイムスタンプ保持
`recursive`|`boolean`|`false`|ディレクトリ再帰的コピー
`verbatimSymlinks`|`boolean`|`false`|`true`ならシンボリックリンクのパス解決をスキップする

　`recursive`は是非ほしい。

　あと、`recursive:true`でなくとも、出力先ディレクトリの階層が深ければ必要な分だけ作成してくれるらしい。`fs.mkdirSync`なら第二引数に`{recursive:true}`が必要だった。けれど[fs.cp][]なら不要であり自動作成してくれるようだ。そうした違いもあるので若干混乱しそう。

　ディレクトリコピーの場合はglobも使えないらしいので、もうふつうにLinuxの`cp`コマンドを使いたいなぁと思ってしまう。でもOS間差異を吸収してもらうために仕方なく[fs.cp][]を使う。

　というか、なぜかディレクトリはコピーできなかった。原因不明。

## preload.js

```javascript
const {remote,contextBridge,ipcRenderer} =  require('electron');
contextBridge.exposeInMainWorld('myApi', {
    cp:async(src, dst, options=null)=>await ipcRenderer.invoke('cp', src, dst, options=null),
})
```

## renderer.js

```javascript
const maker = new SiteMaker(setting)
await maker.make()
```

## renderer.js

```javascript
class SiteMaker {
    constructor(setting) {
        this.setting = setting
    }
    async make() { // 初回にリモートリポジトリを作成するとき一緒に作成する
        await this.#make(`src/lib/toastify/1.11.2/min.js`)
        await this.#make(`src/lib/toastify/1.11.2/min.css`)
    }
    async #make(path) { await window.myApi.cp(path, `dst/${this.setting.github.repo}/${path}`, {'recursive':true, 'preserveTimestamps':true}) }
}
```

　もし存在しないパスを`src`に渡すと以下のようなエラーが出る。

```sh
[Error: ENOENT: no such file or directory, lstat 'lib/toastify/1.11.2/min.css'] {
  errno: -2,
  code: 'ENOENT',
  syscall: 'lstat',
  path: 'lib/toastify/1.11.2/min.css'
}
```

　コピー対象がディレクトリではなくファイル単体であっても`'recursive':true`があっても問題ないらしい。なら、もうこれは何も考えずにつけておけばいいかな。

　そう思って以下のようにディレクトリをコピーしようとしたら、できなかった。エラーも表示されず、コピーもされない状態。なぜ？　原因不明。

```javascript
await this.#make(`memo/`)
```

# 第三引数は必須

## main.js

　以下のように[fs.cp][]の第三引数をセットしなかったらエラーになった。

```javascript
ipcMain.handle('cp', async(event, src, dst) => {
    fs.cp(src, dst)
})
```

　アプリの開発者ツールのコンソールでは以下エラー。

```
Uncaught (in promise) Error: Error invoking remote method 'cp': TypeError [ERR_INVALID_CALLBACK]: Callback must be a function. Received undefined
```

　端末では以下エラー。

```sh
Error occurred in handler for 'cp': TypeError [ERR_INVALID_CALLBACK]: Callback must be a function. Received undefined
    at makeCallback (node:fs:186:3)
    at Object.cp (node:fs:2834:14)
    at /tmp/work/Electron.MyLog.20220829121957/src/js/main.js:98:8
    at node:electron/js2c/browser_init:189:579
    at EventEmitter.<anonymous> (node:electron/js2c/browser_init:161:11327)
    at EventEmitter.emit (node:events:527:28) {
  code: 'ERR_INVALID_CALLBACK'
}
```

　第三引数の使い方がさっぱりわからないが、それでも必須らしい。

　仮に使い方がわかったとしても、IPC通信のシリアライズ制約により関数を引数として受け取れない。なので第三引数のコールバック関数は引数として受け取れない。

　結局、[fs.cp][]は最低でも以下の3つ引数が必要だとわかった。

```javascript
const fs = require('fs')
ipcMain.handle('cp', async(event, src, dst) => {
    fs.cp(src, dst, ()=>{})
})
```















# Node.jsのfs.cpでディレクトリのコピーを試みる

# ブツ

* [リポジトリ][]

[リポジトリ]:

## 前回

　[Node.jsのfs.cpでファイルのコピーを試みる][]のとき、ディレクトリがコピーできなかった。

[Node.jsのfs.cpでファイルのコピーを試みる]:

## 今回

　色々試した結果、どうやらIPC通信の引数にデフォルトオプションをつけていたせいらしい。それを必須にしたら成功した。何を言っているかわからないと思うので順に説明する。

# バグ調査

　main.jsの以下`cp`ハンドルに`console.log`を仕込んだ。すると`if`側に入ってほしいのに、実際は`else`側に入っていたことが判明した。

```javascript
ipcMain.handle('cp', async(event, src, dst, options=null) => {
    if (options) { console.log('aaaaaaaaa', src, dst, options); fs.cp(src, dst, options, ()=>{}); }
    else { console.log('bbbbbbbbb', src, dst); fs.cp(src, dst, ()=>{}); }
})
```

　`if`の条件式を`null!=options`に変えても同様だった。

　つまり、`options`は必ず`null`になっていたことになる。

　だが、呼出元ではちゃんと`options`を渡している。`{'recursive':true, 'preserveTimestamps':true}`というオプションをしっかり渡している。

```javascript
class SiteMaker {
    constructor(setting) {
        this.setting = setting
    }
    async make() {
        await this.#make(`memo/`)
    }
    async #make(path) { await window.myApi.cp(path, `dst/${this.setting.github.repo}/${path}`, {'recursive':true, 'preserveTimestamps':true}) }
```

　なのになぜ`null`になるのか？　コードをみるかぎりmain.jsの以下しかない。メソッド定義の仮引数のところで`options=null`にしている。これがそのまま入ってしまったのでは？

```javascript
ipcMain.handle('cp', async(event, src, dst, options=null) => {
```

　そんなバカな。JavaScriptは[デフォルト引数][]にできるはず。MDNにも書いてある。

[デフォルト引数]:https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Functions/Default_parameters

　ああ、もしかしてこれもElectronのIPC通信における制約なのか？　うわー、それっぽい予感。確認するためにコードを書いてみる。

# ディレクトリをコピーするコード

　以下が成功したコード。思ったとおり[デフォルト引数][]をやめたらディレクトリのコピーができた。うわー……

## main.js

```javascript
ipcMain.handle('cpdir', async(event, src, dst, options) => {
    if (null!=options) { console.log('aaaaaaaaa', src, dst, options); fs.cp(src, dst, options, ()=>{}); }
    else { console.log('bbbbbbbbb', src, dst); fs.cp(src, dst, ()=>{}); }
})
```

　ようするに[fs.cp][]の引数`options`を必須にした。ただし`cpdir`という名前で新しくICPハンドルを作った。ファイルをコピーしたいなら先述の`cp`ハンドルを使う。

## preload.js

```javascript
contextBridge.exposeInMainWorld('myApi', {
    cpdir:async(src, dst, options)=>await ipcRenderer.invoke('cpdir', src, dst, options),
})
```

## renderer.js

```javascript
class SiteMaker {
    constructor(setting) {
        this.setting = setting
    }
    async make() {
        await this.#cpdir(`memo/`)
    }
    async #cpdir(path) { await window.myApi.cpdir(path, `dst/${this.setting.github.repo}/${path}`, {'recursive':true, 'preserveTimestamps':true}) }
}
```

# 結論

　IPC通信の制約によりIPCハンドラの引数には[デフォルト引数][]が使えない。仮に使ったとしたら、必ずそのデフォルト値がセットされてしまう。

　これはひどい。ちゃんと教えてほしかった。

