---
title: "WebpackとGulpを共存させる設計 — JSバンドルとタスクランナーの役割分担"
emoji: "🔧"
type: "tech"
topics: ["webpack", "gulp", "javascript", "フロントエンド"]
published: false
---

[ToolShare Lab](https://webatives.com/) のビルド構成は Gulp + Webpack の組み合わせだ。「Webpack だけで完結させない理由」「Gulp だけではできない理由」があって、この2つを共存させている。設計の考え方と実装パターンを解説する。

## なぜ両方使うのか

Gulp と Webpack はそれぞれ得意なことが違う。

| ツール | 得意なこと | 苦手なこと |
|--------|-----------|-----------|
| Gulp | ファイル操作、EJS/SCSS変換、BrowserSync、デプロイ | JSのモジュール解決、Tree-shaking |
| Webpack | JSバンドル、モジュール解決、コード分割、Tree-shaking | EJSテンプレート処理、任意タスクの実行 |

[ToolShare Lab](https://webatives.com/) は150個のツールページを持ち、各ページに専用のJS計算ロジックがある。このロジックを：

- **共通JS**（ヘッダーのUI、モーダル制御など）→ Webpack でバンドル
- **ページ固有JS**（所得税計算ロジックなど）→ Webpack でバンドル（複数エントリ）
- **EJS → HTML変換**、**SCSS → CSS変換**、**BrowserSync** → Gulp

という役割分担にしている。

## ディレクトリ設計

```
project/
├── src/
│   ├── ejs/           # EJSテンプレート（Gulpが処理）
│   ├── scss/          # SCSS（Gulpが処理）
│   └── js/
│       ├── main.js    # 共通JS（Webpackエントリ）
│       ├── tools/
│       │   ├── income-tax.js      # 所得税計算（ツール固有）
│       │   ├── loan-calculator.js # ローン計算（ツール固有）
│       │   └── ... (150個)
│       └── lib/
│           ├── utils.js
│           ├── formatter.js
│           └── validator.js
├── dist/              # ビルド成果物
│   ├── *.html
│   ├── css/
│   │   └── style.css
│   └── js/
│       ├── bundle.js          # 共通JS
│       └── tools/
│           ├── income-tax.js  # ツール固有JS
│           └── ...
├── gulpfile.mjs
├── webpack.config.cjs         # CJS形式（GulpがESMのため）
└── package.json
```

## webpack.config.cjs の設計

Gulp 5 が ESM（`gulpfile.mjs`）なので、Webpack 設定は `.cjs` 拡張子でCJS形式に保つ。

```javascript
// webpack.config.cjs
const path = require('path');
const fs = require('fs');

// src/js/tools/ 以下のJSを全部エントリとして登録
function buildToolEntries() {
  const toolsDir = path.resolve(__dirname, 'src/js/tools');
  const entries = {};

  if (fs.existsSync(toolsDir)) {
    fs.readdirSync(toolsDir).forEach((file) => {
      if (file.endsWith('.js')) {
        const name = file.replace('.js', '');
        entries[`tools/${name}`] = `./src/js/tools/${file}`;
      }
    });
  }

  return entries;
}

module.exports = {
  mode: process.env.NODE_ENV === 'production' ? 'production' : 'development',
  devtool: process.env.NODE_ENV === 'production' ? false : 'source-map',

  // 複数エントリ: 共通JS + ツール固有JS
  entry: {
    bundle: './src/js/main.js',
    ...buildToolEntries(),
  },

  output: {
    path: path.resolve(__dirname, 'dist/js'),
    filename: '[name].js',
    clean: false, // Gulp が dist を管理するので false
  },

  module: {
    rules: [
      {
        test: /\.js$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader',
          options: {
            presets: ['@babel/preset-env'],
          },
        },
      },
    ],
  },

  optimization: {
    // 共通モジュールを vendor.js に分割
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendor',
          chunks: 'all',
        },
        lib: {
          test: /[\\/]src\/js\/lib[\\/]/,
          name: 'lib',
          chunks: 'all',
          minChunks: 2, // 2ツール以上が使うlibは分割
        },
      },
    },
  },
};
```

## gulpfile.mjs でWebpackを呼ぶ

```javascript
// gulpfile.mjs
import gulp from 'gulp';
import ejs from 'gulp-ejs';
import sass from 'gulp-dart-sass';
import postcss from 'gulp-postcss';
import tailwindcss from '@tailwindcss/postcss';
import autoprefixer from 'autoprefixer';
import cssnano from 'cssnano';
import rename from 'gulp-rename';
import { createRequire } from 'module';

const require = createRequire(import.meta.url);
const browserSync = require('browser-sync').create();
const webpack = require('webpack');
const webpackConfig = require('./webpack.config.cjs');

const { src, dest, watch, series, parallel } = gulp;

// EJS → HTML
function buildEjs() {
  return src(['src/ejs/**/*.ejs', '!src/ejs/**/_*.ejs'])
    .pipe(ejs({}, { root: './src/ejs' }))
    .pipe(rename({ extname: '.html' }))
    .pipe(dest('dist'));
}

// SCSS → CSS
function buildScss() {
  return src('src/scss/style.scss')
    .pipe(sass().on('error', sass.logError))
    .pipe(postcss([tailwindcss(), autoprefixer(), cssnano()]))
    .pipe(dest('dist/css'))
    .pipe(browserSync.stream());
}

// JS → Webpack（Promise でラップして Gulp のタスクとして扱う）
function buildJs() {
  return new Promise((resolve, reject) => {
    const config = {
      ...webpackConfig,
      mode: process.env.NODE_ENV === 'production' ? 'production' : 'development',
    };

    webpack(config, (err, stats) => {
      if (err) {
        console.error(err.stack || err);
        reject(err);
        return;
      }

      if (stats.hasErrors()) {
        const info = stats.toJson();
        console.error(info.errors);
        reject(new Error('Webpack build failed'));
        return;
      }

      console.log(stats.toString({ colors: true, chunks: false }));
      resolve();
    });
  });
}

// BrowserSync
function serve(done) {
  browserSync.init({
    server: './dist',
    port: 3000,
    open: false,
  });
  done();
}

function reload(done) {
  browserSync.reload();
  done();
}

// ファイル監視
function watchFiles() {
  watch('src/ejs/**/*.ejs', series(buildEjs, reload));
  watch('src/scss/**/*.scss', buildScss);
  watch('src/js/**/*.js', series(buildJs, reload));
}

export const build = parallel(buildEjs, buildScss, buildJs);
export const dev = series(build, serve, watchFiles);
export default build;
```

## webpack-stream vs webpack直接呼び出し

`webpack-stream` を使う方法もある。比較:

| 方法 | メリット | デメリット |
|------|---------|-----------|
| `webpack-stream` | Gulp パイプラインに馴染む | 複数エントリの扱いが難しい |
| `webpack` 直接 | 複数エントリ、splitChunksが自由 | Gulp の stream と分離する |

[ToolShare Lab](https://webatives.com/) は150個のツール固有JSがあるため、複数エントリを動的生成する必要がある。`webpack` 直接呼び出し + Promise ラップの方法が適していた。

## 開発・本番の切り替え

```json
// package.json
{
  "scripts": {
    "dev": "gulp dev",
    "build": "NODE_ENV=production gulp build",
    "deploy": "npm run build && bash deploy.sh"
  }
}
```

`NODE_ENV` で Webpack の `mode` を切り替える:

```javascript
// webpack.config.cjs
module.exports = {
  mode: process.env.NODE_ENV === 'production' ? 'production' : 'development',
  devtool: process.env.NODE_ENV === 'production' ? false : 'source-map',
  // ...
};
```

- **development**: ソースマップあり、minifyなし → デバッグしやすい
- **production**: ソースマップなし、minify + Tree-shaking → 最小サイズ

## ビルド結果の確認

```bash
# 本番ビルド
npm run build

# JS のバンドルサイズを確認
ls -lh dist/js/

# Webpack の出力詳細
NODE_ENV=production npx webpack --config webpack.config.cjs --stats verbose
```

[ToolShare Lab](https://webatives.com/) での実際のビルド結果:

```
dist/js/
├── bundle.js        150KB → gzip 42KB
├── vendor.js        80KB  → gzip 28KB
├── lib.js           20KB  → gzip 6KB
└── tools/
    ├── income-tax.js    8KB
    ├── loan-calculator.js  12KB
    └── ... (150個、各2〜15KB)
```

各ツールページは `<script src="/js/bundle.js">` + `<script src="/js/tools/income-tax.js">` の2本だけ読み込む。不要なツールのJSはロードしない設計。

## まとめ

Gulp + Webpack 共存設計の要点:

1. **役割分担を明確に** → Gulp: ファイル変換・タスク、Webpack: JSバンドル
2. **webpack.config.cjs を CJS 形式で** → Gulp 5 の ESM と共存できる
3. **複数エントリを動的生成** → ツール数が増えても webpack.config を変更しない
4. **splitChunks で共通モジュールを分離** → 各ページの JS 読み込みを最小化
5. **webpack を Promise でラップ** → Gulp タスクとして扱える

この設計で150ページ規模のサイトでも管理コストを抑えながら、各ページのJS最適化ができている。
