TypescriptプロジェクトにWebAssemblyを導入してみたのですが、何か所かハマったポイントがあったのでメモしておきます。OSはWindows10です。

#プロジェクト構成例
`src/ts/main.ts` を webpack + ts-loaderを用いて `build/js/main.js` にbuildし、`index.html`から読み込む構成を想定します。`tsconfig.json` と `webpack.config.js` については、自分で作って配置します。

<pre>
[Project Home]
|_  build
|    |_  js
|         |_  main.js
|
|_  src
|    |_  ts
|         |_  main.ts
|
|_  index.html
|_  package.json
|_　tsconfig.json
|_  webpack.config.js
</pre>


# npm各ライブラリのインストール

 webpackを、グローバルインストール

 ```bash
  npm i -g webpack
 ```

typescript ほか必要なライブラリをインストール

```bash
 npm i -D path ts-loader typescript
```

package.jsonのdevDependenciesは以下のようになります。

```js:package.json
  "devDependencies": {
    "path": "^0.12.7",
    "ts-loader": "^5.4.5",
    "typescript": "^3.4.5",
  },
```

# Cargoのインストール

RustのパッケージマネージャであるCargoをインストールします。
>　https://www.rust-lang.org/tools/install

から、RUSTUP-INIT.EXEをダウンロードし、管理者権限で実行します。 インストールが終わったら，以下のコマンドでPATHが通っていることを確認します。

```bash
rustup -V
cargo -V
```


# wasm-packのインストール

今回は[wasm-pack](https://github.com/rustwasm/wasm-pack)を使ってWebAssemblyの環境を構築します。次のコマンドでインストールします。

```bash
 cargo install wasm-pack
```

# crateの作成
WebAssemblyのプロジェクトのことをcrateと呼んだりするようです。（厳密な解釈ではないかもですが）
次のコマンドでsrcフォルダ内にcrateを作成します。今回はcrateをモジュールとして使いたいので、--libオプションをつけました。crate名は適当につけます。今回はwasmとしました。（ちなみにcrate名を「crate」とすると怒られました。）

```bash
cargo new src/wasm --lib 
```

現時点で、プロジェクト構成は以下のようになります。
<pre>
[Project Home]
|_  build
|    |_  js
|         |_  main.js
|
|_  src
|    |_  ts
|    |    |_  main.ts
|    |
|    |_  wasm
|         |_  src
|         |    |_  lib.rs  *
|         |
|         |_  Cargo.toml  *
|   
|_  index.html
|_  package.json
|_　tsconfig.json
|_  webpack.config.js
</pre>


# Cargo.tomlとlib.rsの編集

cargo newで生成された2ファイルを以下のように書き換えます。Cargo.tomlは設定ファイルで、lib.rsには実際の処理を記述します。

```js:Cargo.toml
[package]
name = "wasm"
version = "0.1.0"
authors = ["ユーザー名<メールアドレス>"]
edition = "2018"

[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2"
```

```rust:lib.rs
extern crate wasm_bindgen;
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
extern {
    pub fn alert(s: &str);
}

#[wasm_bindgen]
pub fn greet(name: &str) {
    alert(&format!("Hello, {}!", name));
}
```

今回の設定例は、[MDNのチュートリアル](https://developer.mozilla.org/ja/docs/WebAssembly/Rust_to_wasm)をそのまま引用しています。
javascript側から関数alertをインポートし、関数greetでHello,の後に引数で指定した文字列をalertするという処理です。


# Rustのコンパイル

`wasm-pack build`コマンドでコンパイルします。
`Cargo.toml`の存在するパスを指定する必要があるため、今回は`src/wasm`を指定します。

```bash
wasm-pack build src/wasm
```

コンパイルに成功すると、以下のようにpkgフォルダとtargetフォルダと`Cargo.lock`というファイルが生成されます。
<pre>
[Project Home]
|_  build
|    |_  js
|         |_  main.js
|
|_  src
|    |_  ts
|    |    |_  main.ts
|    |
|    |_  wasm
|         |_  pkg *
|         |     
|         |_  src
|         |    |_  lib.rs  
|         |   
|         |_  target *
|         |   
|         |_  Cargo.lock  *
|         |_  Cargo.toml  
|   
|_  index.html
|_  package.json
|_　tsconfig.json
|_  webpack.config.js
</pre>


# wasmの読み込み

pkgフォルダ内にwasm.jsというファイルが作成されているので、これをmain.tsからDynamic importして使用します。

```ts:main.ts
 const wasm = import('../wasm/pkg/wasm.js');
 wasm.then(mod => {
     mod.greet('WebAssembly');
 });
```


# webpack.config.jsとtsconfig.jsonの設定

WebpackでTypescriptとWebAssemblyが正しくトランスパイルできるように設定します。

```js:webpack.config.js
const path = require('path');

module.exports = {
    // mode: 'production',
    mode: 'development',
    entry: './src/ts/main.ts',
    output: {
      path: path.resolve(__dirname, "build/js"),
      publicPath: "./build/js/", 
      filename: 'main.js',
    },
    resolve: {
      // Add `.ts` and `.tsx` as a resolvable extension.
      extensions: ['.ts', '.tsx', '.js','.wasm']
    },
    module: {
      rules: [
        {
        // all files with a `.ts` or `.tsx` extension will be handled by `ts-loader`
          test: /\.ts$/,
          use: 'ts-loader',
        },
        {
          test: /\.wasm$/,
          type: "webassembly/experimental"
        }
      ]
    }
  }
```

```js:tsconfig.json
{
    "compilerOptions": {
        "sourceMap": false,
        "target": "es5",
        "lib": ["dom","es6","es2015.core"],
        "outDir": "build/js",
        "module": "esNext"
    },
    "include": [
        "src/ts/**.ts"
    ],
    "exclude": [
        "node_modules",
        "**/*.spec.ts"
    ]
}
```

上記の設定をして、後は`webpack`コマンドを実行するだけなのですが、ここで何度かハマりました。

## 1. Module not found: Error: Can't resolve './wasm_bg' ...

wasmモジュールが見つからないというエラーです。`webpack.config.js`の resolve-> extensionsに'.wasm' を追加することで解消しました。

```js:webpack.config.js
    resolve: {
      // Add `.ts` and `.tsx` as a resolvable extension.
      extensions: ['.ts', '.tsx', '.js', '.wasm'] // <-- 重要！
    },
```

## 2. WebAssembly module is included in initial chunk. This is not allowed, because WebAssembly download and compilation must happen asynchronous. ...

WebAssemblyは非同期読み込みする必要があるらしく、同期読み込みした場合に出るエラーです。`tsconfig.json`のコンパイラオプションのモジュールに`esNext`が設定されていない場合も発生します。


```js:tsconfig.json
{
    "compilerOptions": {
        "sourceMap": false,
        "target": "es5",
        "lib": ["dom","es6","es2015.core"],
        "outDir": "build/ts",
        "module": "esNext"  // <-- 重要！
    },
}
```


## 3. Uncaught (in promise) TypeError: Failed to execute 'compile' on 'WebAssembly': HTTP status code is not ok

ローカルサーバで`index.html`を開くと、ブラウザコンソールに表示されるエラーです。
これは、今回のプロジェクト構成例のように`index.html`と`main.js`のパスが異なる場合に表示されるエラーのようです。

```js:webpack.config.js
    output: {
      path: path.resolve(__dirname, "build/js"),
      publicPath: "./build/js/",   //  <--重要！
      filename: 'main.js',
    },
```
これを解決するには、`webpack.config.js`で、webpackの公開パスを、ビルドされたファイルの出力先パスとは別に、明示的に設定する必要があります。


# 結果
![Hello,WebAssembly.JPG](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/264338/9346c023-fb28-6b0b-82df-dd1ef8ee95f1.jpeg)

ちゃんとalertが表示されました。めでたしめでたし。

後はTypescript側からlib.rsに重そうな処理をどんどん引っ越ししていきたいと思います。

DOM操作やレンダリングもWebAssemblyでできたら一番いいのになあ。


#参考にさせていただいたサイト
https://developer.mozilla.org/ja/docs/WebAssembly/Rust_to_wasm
https://qiita.com/mizchi/items/dc089c28e4d3afa78207
https://qiita.com/ito-hiroki/items/aaaf66e082f917baacc9
https://github.com/rustwasm/rust-webpack-template/issues/43#issuecomment-426597176
