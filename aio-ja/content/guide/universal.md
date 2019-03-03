# Angular Universal: サーバーサイドレンダリング

このガイドでは、サーバー上でAngularアプリケーションを実行する技術である**Angular Universal**について説明します。

通常のAngularアプリケーションは _ブラウザ_ 上で実行され、ユーザーアクションに応じてDOMのページをレンダリングします。Angular Universalは_サーバーサイドレンダリング（SSR）_と呼ばれるプロセスを通じて、_サーバー_ 上に _静的な_ アプリケーションページを生成します。

Universalがアプリケーションに統合されると、ブラウザからのリクエストに応じて、それらのページを生成して配信できます。また、後に配信するHTMLファイルとしてページを事前に生成することもできます。

あなたは[Angular CLI](guide/glossary#cli)を使用して、サーバーサイドレンダリング用アプリケーションを簡単に準備できます。CLIの`@nguniversal/express-engine` Schematicsは、後述するSSRを実現するために必要な手順を代行します。

このガイドでは、サーバーレンダリングされたページとしてすぐに起動するUniversalアプリケーションのサンプルについて説明します。SSRの過程において、ブラウザは完全なクライアントバージョンをダウンロードし、コードがロードされた後に自動でそれに切り替わります。

<div class="alert is-helpful">

  **Note:** [Node.js® Express](https://expressjs.com/) サーバーで動作する[サンプルコードの完成形をダウンロード](generated/zips/universal/universal.zip),
  
</div>

{@a why-do-it}

## なぜサーバーサイドレンダリングを利用するのか

アプリケーションのユニバーサルバージョンを作成する主な理由は3つあります。

1. Webクローラーを支援する（SEO）
1. モバイルおよび低スペックデバイスのパフォーマンスを向上させる
1. 最初のページを瞬時に表示する

{@a seo}
{@a web-crawlers}
### Webクローラーを支援する

Google、Bing、Facebook、Twitter、その他のソーシャルメディアサイトは、Webクローラーに依存してアプリケーションコンテンツのインデックスを作成し、そのコンテンツをWeb上で検索可能にします。これらのWebクローラーは、高度にインタラクティブなAngularアプリケーションをユーザーと同様に操作してインデックス化できない場合があります。

Angular Universalは、JavaScriptなしで簡単に検索、リンク、ナビゲートできる静的バージョンのアプリケーションを生成できます。また、各URLは完全にレンダリングされたページを返すためサイトのプレビューも可能とします。

Webクローラーの有効化は、よく[Search Engine Optimization（SEO）](https://static.googleusercontent.com/media/www.google.com/en//webmasters/docs/search-engine-optimization-starter-guide.pdf)と呼ばれます。

{@a no-javascript}

### モバイルおよび低スペックデバイスのパフォーマンスを改善する

一部の端末ではJavaScriptをサポートしていないか、ユーザー体験が容認できないほどにJavaScriptの実行が不完全です。このような場合には、サーバーレンダリングされたJavaScript未使用バージョンのアプリケーションが必要になることがあります。そのバージョンを使用するケースはそう多くありませんが、そのアプリケーションをまったく利用できない人々のための唯一の実用的な代替手段かもしれません。

{@a startup-performance}

### 最初のページを瞬時に表示する

最初のページを瞬時に表示することは、ユーザーエンゲージメントの面で非常に重要です。ページの表示に3秒以上かかる場合、[モバイルサイト訪問者の53%が離脱しました](https://www.doubleclickbygoogle.com/articles/mobile-speed-matters/)。あなたのアプリケーションは、ユーザーの気が散るより早く注意を引きつけるために、より早く立ち上げる必要があります。

Angular Universalを使用すると、完全なアプリケーションのようなランディングページを生成できます。このページは純粋なHTMLであり、JavaScriptが無効になっていても表示できます。このページはブラウザイベントを処理しませんが、[routerLink](guide/router#router-link)を使用して、サイト経由のナビゲーションをサポートします。

実際は、ランディングページの静的バージョンを配信しユーザーの注意を引きつけます。それと同時に、バックグラウンドで完全なAngularアプリケーションを読み込みます。ユーザーはランディングページからほぼ瞬間的と言ってよいパフォーマンスを認識し、アプリケーションが完全に読み込まれた後にインタラクティブな体験を得ることができます。

{@a how-does-it-work}

## Universal webサーバー

Universal webサーバーは、アプリケーションのページリクエストに対して[Universalテンプレートエンジン](#universal-engine)によってレンダリングされた静的なHTMLを返します。そのサーバーは、クライアント（通常はブラウザ）からのHTTPリクエストに対して受信応答し、スクリプト、CSS、そして画像等の静的アセットを配信します。それは直接または別のデータサーバーのプロキシとしてデータリクエストに応答するかもしれません。

このガイドのサンプルWebサーバーは、よく知られている[Express](https://expressjs.com/)フレームワークをベースとしています。

<div class="alert is-helpful">

  **Note:** Universalの`renderModuleFactory()`関数を実行できるのであれば、_どんな_ Webサーバー技術でもUniversalアプリケーションを提供できます。ここで説明する原則と意思決定のポイントは、あらゆるWebサーバー技術に当てはまります。

</div>

Universalアプリケーションを作成するには、DOM、`XMLHttpRequest`、そしてブラウザに依存しない低レベル機能のサーバー実装を提供する`platform-server`パッケージをインストールします。クライアントアプリケーションを`platform-server`モジュールでコンパイルし、（`platform-browser`モジュールの代わりに）結果として得られるUniversalアプリケーションをWebサーバーで実行します。

そのサーバー（このガイドの例では[Node Express](https://expressjs.com/)）は、アプリケーションページのクライアントリクエストをUniversalの`renderModuleFactory()`関数に渡します。

その`renderModuleFactory()`関数は入力としてHTMLの *テンプレート* ページ（通常は`index.html`）、コンポーネントを含むAngular *モジュール* や表示するコンポーネントを決定する *ルート* を受け取ります。そのルートは、クライアントのサーバーへのリクエストから発生します。

各リクエストは、要求されたルートの適切なビューをもたらします。`renderModuleFactory()`はテンプレートの`<app>`タグ内でビューをレンダリングし、クライアント用の最終的なHTMLページを作成します。


最後に、サーバーはレンダリングされたページをクライアントに返します。

{@a summary}
## サーバーサイドレンダリングの準備

アプリケーショをサーバー上でレンダリングする前に、アプリ自体に変更を加え、サーバーも設定する必要があります。

1. 依存関係をインストールします。
1. アプリケーションコードとその設定の両方を変更してアプリを準備します。
1. ビルドターゲットを追加し、CLIを使用して`@nguniversal/express-engine` Schematicsと共にユニバーサルバンドルをビルドします。
1. ユニバーサルバンドルを実行するためのサーバーを設定します。
1. サーバーにアプリを設置して実行します。

次のセクションでは、これらの主な手順のそれぞれについてより詳細に説明します。

<div class="alert is-helpful">

  **注意：**以下の[Universalのチュートリアル](#the-example)では、Tour of Heroesのサンプルアプリケーションを通じて順を追って手順を説明し、その結果実現できること及びそれをする理由について詳しく説明します。
   
  サーバーサイドレンダリングを備えたアプリケーションの動作するバージョンを確認するには、[Angular Universal starter](https://github.com/angular/universal-starter)を複製してください。

</div>

<div class="callout is-critical">

<header>サーバーリクエスト用のセキュリティー</header>

ブラウザアプリから発行されたHTTPリクエストは、サーバー上のユニバーサルアプリによって発行されたものと同じではありません。ユニバーサルHTTPリクエストには異なるセキュリティ要件があります。

ブラウザがHTTPリクエストを行う際、サーバーはCookieやXSRFヘッダーなどについて推測できます。たとえば、ブラウザは現在のユーザーの認証Cookieを自動的に送信します。Angular Universalはこれらの認証情報を別のデータサーバーに転送できません。サーバーがHTTPリクエストを処理する場合は、独自のセキュリティ機能を追加する必要があります。

</div>

## Step1: 依存関係のインストール

プロジェクトに`@angular/platform-server`をインストールしてください。 プロジェクト内の他の`@angular`パッケージと同じバージョンを利用しましょう。webpackのビルドには`ts-loader`、サーバーレンダリング内で遅延読み込みを扱うには`@nguniversal/module-map-ngfactory-loader`も必要です。

<code-example format="." language="bash" linenums="false">
$ npm install --save @angular/platform-server @nguniversal/module-map-ngfactory-loader ts-loader
</code-example>

## Step2: アプリケーションの準備

ユニバーサルレンダリング用のアプリケーションを準備するには、次の手順に従います:

* アプリケーションにユニバーサルサポートを追加します。

* サーバーのルートモジュールを作成します。

* サーバーのルートモジュールをエクスポートするためのメインファイルを作成します。

* サーバーのルートモジュールを設定します。

### 2a. アプリケーションにユニバーサルサポートを追加する

`.withServerTransition()`及びアプリケーションIDを`src/app/app.module.ts`にある`BrowserModule`のインポートに追加することで`AppModule`をUniversalと互換性があるようにしてください。

<code-example format="." language="typescript" linenums="false">
@NgModule({
  bootstrap: [AppComponent],
  imports: [
    // ユニバーサルレンダリングをサポートするために.withServerTransition()を追加します。
    // アプリケーションIDは、ページ上で一意の識別子にすることができます。
    BrowserModule.withServerTransition({appId: 'my-app'}),
    ...
  ],

})
export class AppModule {}
</code-example>

### 2b. サーバーのルートモジュールを作成する

サーバー上で実行している際にルートモジュールとして振る舞うための`AppServerModule`という名前のモジュールを作成します。この例では、`app.server.module.ts`という名称のファイルで`app.module.ts`と並んで配置しています。その新しいモジュールはルートの`AppModule`からすべてをインポートし、`ServerModule`を追加します。また、Angular CLIを使用したサーバーサイドレンダリングでルートの遅延み込みを可能にするために、`ModuleMapLoaderModule`を追加しています。

これが`src/app/app.server.module.ts`の例です。

<code-example format="." language="typescript" linenums="false">
import {NgModule} from '@angular/core';
import {ServerModule} from '@angular/platform-server';
import {ModuleMapLoaderModule} from '@nguniversal/module-map-ngfactory-loader';

import {AppModule} from './app.module';
import {AppComponent} from './app.component';

@NgModule({
  imports: [
    // AppServerModuleは、@angular/platform-serverからAppModule、
    // 続いてServerModuleをインポートします。
    AppModule,
    ServerModule,
    ModuleMapLoaderModule // <-- *重要* ルートの遅延読み込みを動作させるため
  ],
  // ブートストラップされたコンポーネントはインポートされたAppModuleから
  // 継承されていないため、ここで繰り返す必要があります。
  bootstrap: [AppComponent],
})
export class AppServerModule {}
</code-example>

### 2c. AppServerModuleへエクスポートするためのメインファイルを作成する

あなたの`AppServerModule`インスタンスをエクスポートするために、あなたのユニバーサルバンドルのメインファイルをアプリケーションの`src/`フォルダに作成してください。この例は`main.server.ts`を呼び出します。

<code-example format="." language="typescript" linenums="false">
export { AppServerModule } from './app/app.server.module';
</code-example>

### 2d. AppServerModuleの設定ファイルを作成する

`tsconfig.app.json`を`tsconfig.server.json`にコピーして次のように修正してください:

* `"compilerOptions"`では、`"module"`のターゲットを`"commonjs"`に設定してください。
* `"angularCompilerOptions"`のセクションを追加し、あなたの`AppServerModule`インスタンスを指すように`"entryModule"`を設定しましょう。`importPath#symbolName`のフォーマットを使用してください。この例では、エントリーモジュールは`app/app.server.module#AppServerModule`です。

<code-example format="." language="none" linenums="false">
{
  "extends": "../tsconfig.json",
  "compilerOptions": {
    "outDir": "../out-tsc/app",
    "baseUrl": "./",
    // モジュールのフォーマットを"commonjs"に設定します:
    "module": "commonjs",
    "types": []
  },
  "exclude": [
    "test.ts",
    "**/*.spec.ts"
  ],
  // "entryModule"として記述したAppServerModuleと共に、
  // "angularCompilerOptions"を追加します。
  "angularCompilerOptions": {
    "entryModule": "app/app.server.module#AppServerModule"
  }
}
</code-example>

## Step3: 新しいビルドターゲットを作成してバンドルをビルドする

プロジェクトのAngular設定ファイルである`angular.json`を開き、サーバービルド用の`"architect"`セクションに新しいターゲットを追加してください。次の例では、新しいターゲットに`"server"`と名前を付けます。

<code-example format="." language="none" linenums="false">
"architect": {
  "build": { ... }
  "server": {
    "builder": "@angular-devkit/build-angular:server",
    "options": {
      "outputPath": "dist/my-project-server",
      "main": "src/main.server.ts",
      "tsConfig": "src/tsconfig.server.json"
    }
  }
}
</code-example>

アプリケーションのサーバーバンドルを構築するには、`projectName#serverTarget`の形式で`ng run`コマンドを使用します。この例では、`"build"`と`"server"`の2つのターゲットが設定されています。

<code-example format="." language="none" linenums="false">
# This builds your project using the server target, and places the output
# in dist/my-project-server/
$ ng run my-project:server

Date: 2017-07-24T22:42:09.739Z
Hash: 9cac7d8e9434007fd8da
Time: 4933ms
chunk {0} main.js (main) 9.49 kB [entry] [rendered]
chunk {1} styles.css (styles) 0 bytes [entry] [rendered]
</code-example>

## Step4: ユニバーサルバンドルを実行するためのサーバーを設定する

ユニバーサルバンドルを実行するには、それをサーバーに送るする必要があります。

次の例は、`AppServerModule`（AoTコンパイル）を`PlatformServer`メソッドの`renderModuleFactory()`に渡します。これはアプリをシリアライズし、その結果をブラウザに返します。

<code-example format="." language="typescript" linenums="false">
app.engine('html', (_, options, callback) => {
  renderModuleFactory(AppServerModuleNgFactory, {
    // index.html
    document: template,
    url: options.req.url,
    // 異なる動作の遅延読み込みを行うためにDIを設定する
    // （瞬時にビューをレンダリングする必要がある）
    extraProviders: [
      provideModuleMap(LAZY_MODULE_MAP)
    ]
  }).then(html => {
    callback(null, html);
  });
});
</code-example>

この技術はあなたに完全な柔軟性を与えます。効率のために、いくつかの組み込み機能を持つ`@nguniversal/express-engine`ツールを使うことも可能です。

<code-example format="." language="typescript" linenums="false">
import { ngExpressEngine } from '@nguniversal/express-engine';

app.engine('html', ngExpressEngine({
  bootstrap: AppServerModuleNgFactory,
  providers: [
    provideModuleMap(LAZY_MODULE_MAP)
  ]
}));
</code-example>

次の簡単な例では、全体を起動するための最小構成でNode Expressサーバーを実装しています。
（これはデモ用です。実際の運用環境では、追加の認証とセキュリティを設定する必要があります。）

プロジェクトのルートレベルで、`package.json`の隣に、`server.ts`という名前のファイルを作成して次の内容を追加してください。

<code-example format="." language="typescript" linenums="false">
// これらは重要であり、最初に記述する必要があります
import 'zone.js/dist/zone-node';
import 'reflect-metadata';

import { renderModuleFactory } from '@angular/platform-server';
import { enableProdMode } from '@angular/core';

import * as express from 'express';
import { join } from 'path';
import { readFileSync } from 'fs';

// 本番用のより高速なサーバーレンダリング（開発用は必要ありません）
enableProdMode();

// Expressサーバー
const app = express();

const PORT = process.env.PORT || 4000;
const DIST_FOLDER = join(process.cwd(), 'dist');

// テンプレートとして使用するindex.html
const template = readFileSync(join(DIST_FOLDER, 'browser', 'index.html')).toString();

// * 注意 :: このファイルはwebpackから動的に生成されるため、require()のままにしておきます
const { AppServerModuleNgFactory, LAZY_MODULE_MAP } = require('./dist/server/main.bundle');

const { provideModuleMap } = require('@nguniversal/module-map-ngfactory-loader');

app.engine('html', (_, options, callback) => {
  renderModuleFactory(AppServerModuleNgFactory, {
    // index.html
    document: template,
    url: options.req.url,
    // 遅延ロードを別の方法で実行できるようにするためのDI（瞬時にレンダリングするために必要）
    extraProviders: [
      provideModuleMap(LAZY_MODULE_MAP)
    ]
  }).then(html => {
    callback(null, html);
  });
});

app.set('view engine', 'html');
app.set('views', join(DIST_FOLDER, 'browser'));

// /browser配下の静的ファイル
app.get('*.*', express.static(join(DIST_FOLDER, 'browser')));

// すべての通常ルートはUniversalエンジンを使用します
app.get('*', (req, res) => {
  res.render(join(DIST_FOLDER, 'browser', 'index.html'), { req });
});

// Nodeサーバーを起動します
app.listen(PORT, () => {
  console.log(`Node server listening on http://localhost:${PORT}`);
});
</code-example>

## Step5: サーバーにアプリを設置して実行しする

Node Expressの`server.ts`ファイルを処理するようにwebpack設定を構成し、アプリケーションを提供しましょう。

アプリケーションのルートディレクトリに、`server.ts`ファイルとその依存関係を`dist/server.js`へコンパイルするwebpack設定ファイル（`webpack.server.config.js`）を作成します。

<code-example format="." language="typescript" linenums="false">
@NgModule({
const path = require('path');
const webpack = require('webpack');

module.exports = {
  entry: {  server: './server.ts' },
  resolve: { extensions: ['.js', '.ts'] },
  target: 'node',
  // これにより、node_modulesや他のサードパーティライブラリを確実に含めることができます
  externals: [/(node_modules|main\..*\.js)/],
  output: {
    path: path.join(__dirname, 'dist'),
    filename: '[name].js'
  },
  module: {
    rules: [
      { test: /\.ts$/, loader: 'ts-loader' }
    ]
  },
  plugins: [
    // issue のための一時修正: https://github.com/angular/angular/issues/11580
    // for "WARNING Critical dependency: the request of a dependency is an expression"
    new webpack.ContextReplacementPlugin(
      /(.+)?angular(\\|\/)core(.+)?/,
      path.join(__dirname, 'src'), // location of your src
      {} // a map of your routes
    ),
    new webpack.ContextReplacementPlugin(
      /(.+)?express(\\|\/)(.+)?/,
      path.join(__dirname, 'src'),
      {}
    )
  ]
}
</code-example>

プロジェクトの`dist/`フォルダはブラウザとサーバーの両方のフォルダを含みます。

<code-example format="." language="none" linenums="false">
dist/
   browser/
   server/
</code-example>

サーバー上でアプリケーションを実行するには、コマンドシェルで次のように入力します。

<code-example format="." language="bash" linenums="false">
node dist/server.js
</code-example>

### スクリプトを作成する

それでは、今後これらすべてを行っていくために、いくつかの便利なスクリプトを作成しましょう。これらはAngular設定ファイルである`angular.json`の`"server"`セクションに追加することができます。

<code-example format="." language="none" linenums="false">
"architect": {
  "build": { ... }
  "server": {
    ...
     "scripts": {
      // 汎用スクリプト
      "build:ssr": "npm run build:client-and-server-bundles && npm run webpack:server",
      "serve:ssr": "node dist/server.js",

      // スクリプト用のヘルパー
      "build:client-and-server-bundles": "ng build --prod && ng build --prod --app 1 --output-hashing=false",
      "webpack:server": "webpack --config webpack.server.config.js --progress --colors"
    }
   ...
</code-example>

ローカル環境でユニバーサル化したアプリケーションの本番ビルドを実行するには、次のコマンドを使用します。

<code-example format="." language="bash" linenums="false">
npm run build:ssr && npm run serve:ssr
</code-example>

### ブラウザAPIを回避する

Universalの`platform-server`アプリケーションはブラウザ上で実行されないため、サーバー上に存在しないブラウザAPIと機能の一部を回避する必要があるかもしれません。

例えば、`window`、`document`、`navigator`や`location`のようなブラウザ専用のネイティブオブジェクトを参照することはできません。サーバーレンダリングされたページでそれらを必要としない場合は、条件付きロジックを使用して回避することができます。あるいは`Location`や`Document`のような、必要なオブジェクトに対して注入可能なAngularの抽象化を探します。あなたが使用している特定のAPIを適切に置き換えることができるかもしれません。Angularがそれを提供していない場合、ブラウザ上ではブラウザAPIに、サーバー上では十分な代替実装に委譲する独自の抽象処理を書くことができます。

同様に、マウスやキーボードのイベントがなければ、サーバーサイドのアプリケーションはコンポーネントを表示するためにユーザーのボタンクリックに頼ることはできません。アプリケーションは受信したクライアントリクエストのみに基づき、何をレンダリングするか決定する必要があります。これは、アプリケーションを[ルーティング可能](guide/router)とするために十分な情報です。

サーバーレンダリングされたページのユーザーはリンクをクリックする以上のことはできないため、適切なインタラクティブエクスペリエンスを実現するには、できるだけ早く実際のクライアントアプリケーションに切り替える必要があります。

{@a the-example}

## Universalのチュートリアル

このチュートリアルは[Tour of Heroes tutorial](tutorial)をベースにしています。

アプリケーションのコアファイルはほとんど変更されていませんが、以下に示すいくつかの例外があります。Universalでのビルドおよび配信をサポートするファイルを追加しましょう。

この例では、Angular CLIがUniversalバージョンのアプリケーションを[AOT（Ahead-of-Time）コンパイラー](guide/aot-compiler)でコンパイルしてひとまとめにします。Node Express Webサーバーは、クライアントリクエストをUniversalによってレンダリングされたHTMLページへ変換します。

サーバーサイドのアプリケーションモジュールである`app.server.module.ts`を作成するために、次のCLIコマンドを実行しましょう。

<code-example format="." language="bash">

ng add @nguniversal/express-engine --clientProject angular.io-example

</code-example>

The command creates the following folder structure.

<code-example format="." language="none" linenums="false">
src/
  index.html                 <i>アプリケーションのwebページ</i>
  main.ts                    <i>クライアント用の初期化処理</i>
  main.server.ts             <i>* サーバー用の初期化処理</i>
  tsconfig.app.json          <i>クライアント用のTypeScript設定</i>
  tsconfig.server.json       <i>* サーバー用のTypeScript設定</i>
  tsconfig.spec.json         <i>テスト用のTypeScript設定</i>
  style.css                  <i>アプリケーションのスタイル</i>
  app/ ...                   <i>アプリケーションコード</i>
    app.server.module.ts     <i>* サーバー用のアプリケーションモジュール</i>
server.ts                    <i>* express webサーバー</i>
tsconfig.json                <i>クライアント用のTypeScript設定</i>
package.json                 <i>npmの設定</i>
webpack.server.config.js     <i>* サーバー用のWebpack設定</i>
</code-example>

`*`マークがついているファイルは新規に作成するもので、元のチュートリアルサンプルには含まれていません。
このガイドでは以下のセクションでそれらを説明します。

{@a http-urls}

### サーバーリクエストに絶対URLを使用する

このチュートリアルの`HeroService`と`HeroSearchService`は、アプリケーションデータの取得をAngularの`HttpClient`モジュールに委譲します。これらのサービスは、`api/heroes`のような _相対_ URLにリクエストを送信します。Universalアプリケーションにおいて、HTTPのURLはUniversal webサーバーが相対的なリクエストを扱える場合でも、（たとえば`https://my-server.com/api/heroes`のような） _絶対_ パスでなければなりません。つまり、サーバーで実行している際は絶対URLで、ブラウザで実行している場合は相対URLをリクエストするように、それらのサービスを変更する必要があります。

解決策の1つは、Angularの[`APP_BASE_HREF`](api/common/APP_BASE_HREF)トークンを介してサーバー実行時のオリジンを提供することです。それをサービスに注入し、リクエストURLの前にオリジンを追加します。

まず、オプションで注入された`APP_BASE_HREF`トークンを介して、第2引数である`origin`パラメーターを取得するように`HeroService`のコンストラクターを変更します。

<code-example path="universal/src/app/hero.service.ts" region="ctor" header="src/app/hero.service.ts (constructor with optional origin)">
</code-example>

コンストラクタでは _もしオリジンが存在すれば_ それを`heroesUrl`の前に追加するために`@Optional()`ディレクティブを使います。ブラウザバージョンでは`APP_BASE_HREF`を提供しないので、`heroesUrl`は相対的なままです。

<div class="alert is-helpful">

  **Note:** チュートリアルのサンプルで行うように、ルーターのベースアドレスを決定するために`index.html`で`<base href="/">`を指定している場合、ブラウザで`APP_BASE_HREF`を無視することができます。

</div>

{@a universal-engine}
### Universalテンプレートエンジン

`server.ts`ファイルで重要な部分は`ngExpressEngine()`関数です。

<code-example path="universal/server.ts" header="server.ts" region="ngExpressEngine">
</code-example>

`ngExpressEngine()`関数は、Universalにおける`renderModuleFactory()`関数のラッパーであり、クライアントのリクエストをサーバーレンダリングされたHTMLページへ変換します。サーバーサイド処理に適した _テンプレートエンジン_ 内でこの関数を呼び出しましょう。

* 最初のパラメータは`AppServerModule`です。これは、Universalのサーバーサイドレンダラーとアプリケーション間のブリッジです。

* 2番目のパラメーターは任意で指定する`extraProviders`です。これには、サーバー側で実行する際にのみ適用される依存プロバイダーを指定します。現在実行中のサーバーインスタンスでしか判断できない情報がアプリケーション側で必要な際にこれを行います。今回のケースで必要な情報は、アプリが[絶対パスの HTTP URL を判断](#http-urls)できるように`APP_BASE_HREF`トークンを介して提供される、実行中のサーバーの *origin* です。

`ngExpressEngine()`関数はレンダリングされたページを解決する`Promise`コールバックを返します。
そのページで何をするかはあなたのエンジン次第です。このエンジンの`Promise`コールバックはレンダリングされたページをWebサーバーに返し、その後HTTPレスポンスでそれをクライアントに転送します。

<div class="alert is-helpful">

  **Note:**  これらのラッパーは`renderModuleFactory()`関数の複雑さを隠蔽するのに役立ちます。この他にも、[Universalリポジトリー](https://github.com/angular/universal)には異なるバックエンド技術のためのラッパーが用意されています。

</div>

### リクエストURLのフィルタリング

webサーバーは _アプリケーションページのリクエスト_ と他の種類のリクエストを区別する必要があります。

それはルートアドレス`/`へのリクエストを傍受するほど簡単ではありません。ブラウザは、`/dashboard`、`/heroes`、`/detail:12`といったアプリケーションルートの1つをリクエストできます。実際に、アプリケーションがサーバーによってのみレンダリングされた場合、クリックされた _すべての_ アプリケーションリンクはルーター用のナビゲーションURLとしてサーバーに到達します。

幸いアプリケーションのルーティングURLには、ファイル拡張子が存在しないといった共通点があります。（データリクエストにも拡張子は存在しませんが、常に`/api`で始まるため簡単に判別できます。）すべての静的アセットのリクエストにはファイル拡張子が存在します。（たとえば、`main.js`または`/node_modules/zone.js/dist/zone.js`）

私達はルーティングを使用しているため、3タイプのリクエストを簡単に判別し、それらを別々に処理することができます。

1. データリクエスト - `/api`から始まるリクエストURL
2. アプリケーションのナビゲーション - ファイル拡張子のないリクエストURL
3. 静的アセット - 他すべてのリクエスト

Node Expressサーバーは、URLリクエストを順次フィルタリングして処理するミドルウェアのパイプラインです。
データリクエストを行うため、このように`app.get()`を呼び出してNode Expressサーバーのパイプラインを設定しましょう。

<code-example path="universal/server.ts" header="server.ts (data URL)" region="data-request" linenums="false">
</code-example>

<div class="alert is-helpful">

  **Note:** このサンプルサーバーはデータリクエストを処理しません。

  このチュートリアルのデモおよび開発ツールである"in-memory web api"モジュールは、すべてのHTTPリクエストに割り込み、リモートデータサーバーの動作をシミュレートします。
  実際はそのモジュールを削除し、サーバーのweb APIミドルウェアをここに登録します。

</div>

次のコードは拡張子のないリクエストURLをフィルタリングし、それらをナビゲーションリクエストとして扱います。

<code-example path="universal/server.ts" header="server.ts (navigation)" region="navigation-request" linenums="false">
</code-example>

### 静的ファイルを安全に配信する

単一の`app.use()`は、他すべてのJavaScript、画像、およびstyleファイルの静的アセット用リクエストURLを扱います。

クライアントに閲覧許可があるファイルのみをダウンロードさせるには、すべてのクライアント向けアセットファイルを`/dist`フォルダに置き、`/dist`フォルダへのリクエストのみを許可してください。

次のNode Expressコードは、残りすべてのリクエストを`/dist`にルーティングし、ファイルが見つからない場合は`404 - NOT FOUND`を返します。

<code-example path="universal/server.ts" header="server.ts (static files)" region="static" linenums="false">
</code-example>


### Universalの実践

http://localhost:4000/ でブラウザを開きましょう。お馴染みのTour of Heroesダッシュボードページが表示されるはずです。

`routerLinks`経由のナビゲーションは正しく動作します。ダッシュボードからヒーローリストのページに遷移し、戻ってくることができます。ダッシュボードページのヒーローをクリックすると、詳細ページが表示されます。

ただし、クリック、マウスの移動、そしてキーボード入力に対しては反応しません。

* ヒーローリストページのヒーローをクリックしても何も起こりません。
* ヒーローを追加、または削除することはできません。
* ダッシュボードページの検索ボックスは無反応です。
* 詳細ページの *戻る* および *保存* ボタンは機能しません。

`routerLink`のクリック以外のユーザーイベントはサポートされていません。完全なクライアントアプリケーションが使用可能となるまで待機する必要があります。それはクライアントアプリケーションをコンパイルして、出力結果を`dist/`フォルダに移動するまで利用できません。

サーバーレンダリングされたアプリケーションからクライアントアプリケーションへの移行は、開発マシン上では迅速に行われます。より遅いネットワークをシミュレートすると、その移行をより明確に確認できます。また、低スペックで接続が芳しくないデバイス上におけるUniversalアプリケーションの起動速度における優位性がより分かりやすいでしょう。

Chrome Dev Toolsを開き、ネットワークタブに移動します。メニューバーの右端にある[Network Throttling](https://developers.google.com/web/tools/chrome-devtools/network-performance/reference#throttling)ドロップダウンリストを探します。

"3G"速度の1つを試してみてください。サーバーレンダリングされたアプリケーションは依然として高速に起動しますが、完全なクライアントアプリケーションでは読み込みに数秒かかる場合があります。