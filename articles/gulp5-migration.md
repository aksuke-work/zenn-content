---
title: "Gulp 4からGulp 5への移行ガイド — ESM対応のハマりポイント"
emoji: "⚡"
type: "tech"
topics: ["gulp", "esm", "nodejs", "javascript"]
published: false
---

[ToolShare Lab](https://and-and.net/) を Gulp 4 から Gulp 5 に移行したとき、思ったより詰まるポイントが多かった。150個以上のツールページを抱えるサイトでの移行体験をもとに、ハマりポイントと対処法をまとめる。

## Gulp 5の何が変わったのか

Gulp 5 の最大の変更点は **ESM（ES Modules）ネイティブ対応**。`gulpfile.js` を `gulpfile.mjs` に変えてESM構文を使えるようになった。

主な変更点:

| 項目 | Gulp 4 | Gulp 5 |
|------|--------|--------|
| モジュール形式 | CommonJS (`require`) | ESM (`import/export`) |
| エクスポート方法 | `exports.build = ...` | `export const build = ...` |
| gulpfileの拡張子 | `gulpfile.js` | `gulpfile.mjs` または `gulpfile.js`（type:module） |
| Node.js要件 | v10.13以上 | v18以上推奨 |

## ハマりポイント1: CommonJSプラグインが動かない

移行して最初にぶつかる壁がこれ。

```bash
Error [ERR_REQUIRE_ESM]: require() of ES Module ...
```

Gulp 5 自体はESMだが、古いプラグインがCJS形式のままだと競合する。

**対処法**: `createRequire` でCJSプラグインをESM環境からimportする。

```javascript
// gulpfile.mjs
import { createRequire } from 'module';
const require = createRequire(import.meta.url);

// CJS形式のプラグインをimport
const gulpSomeLegacyPlugin = require('gulp-some-legacy-plugin');
```

ただし、これは暫定対応。できる限りESM対応済みのプラグインに切り替えるほうが長期的にはよい。

[ToolShare Lab](https://and-and.net/) では移行時に以下のプラグインをアップデートした:

| プラグイン | 旧バージョン | 新バージョン | 理由 |
|-----------|------------|------------|------|
| gulp-sass | 5.x（Node Sass） | gulp-dart-sass | Dart Sassに移行 |
| gulp-uglify | 3.x | webpack-streamに置き換え | ESMバンドラーへ |
| gulp-imagemin | 7.x | 8.x（ESM化済み） | ESM対応 |

## ハマりポイント2: `gulpfile.mjs` のexport構文

Gulp 4 では `exports.default` や `exports.build` を使っていたが、Gulp 5 では ESM の `export` 構文に変わる。

```javascript
// Gulp 4（gulpfile.js）
const { src, dest, watch, series, parallel } = require('gulp');

function buildEjs() { /* ... */ }
function buildScss() { /* ... */ }

exports.build = parallel(buildEjs, buildScss);
exports.default = exports.build;
```

```javascript
// Gulp 5（gulpfile.mjs）
import gulp from 'gulp';
const { src, dest, watch, series, parallel } = gulp;

function buildEjs() { /* ... */ }
function buildScss() { /* ... */ }

export const build = parallel(buildEjs, buildScss);
export default build;
```

**`export default`** が `gulp` コマンドを引数なしで実行したときのデフォルトタスクになる。

## ハマりポイント3: package.jsonの`"type": "module"`設定

`gulpfile.mjs` を使う場合は問題ないが、`gulpfile.js` のまま ESM にしたい場合は `package.json` に設定が必要。

```json
{
  "type": "module",
  "scripts": {
    "build": "gulp build",
    "dev": "gulp dev"
  }
}
```

ただし、`"type": "module"` にすると **プロジェクト内のすべての `.js` ファイルがESMとして扱われる**。CJS形式のconfigファイル（webpack.config.js など）があると壊れる。

[ToolShare Lab](https://and-and.net/) では拡張子で明示的に分けるアプローチを選んだ:
- `gulpfile.mjs` → ESM（Gulp タスク）
- `webpack.config.cjs` → CJS（Webpack設定）

```javascript
// webpack.config.cjs（CJS形式のまま維持）
module.exports = {
  mode: 'production',
  entry: './src/js/main.js',
  output: {
    filename: 'bundle.js',
  },
};
```

## ハマりポイント4: `gulp-dart-sass` への切り替え

Node Sass（`node-sass`）は非推奨になっており、Gulp 5 環境では動作しないケースが多い。Dart Sass（`sass`パッケージ）を使う `gulp-dart-sass` に切り替える。

```bash
npm uninstall gulp-sass node-sass
npm install gulp-dart-sass sass
```

```javascript
// gulpfile.mjs
import sass from 'gulp-dart-sass';

function buildScss() {
  return src('src/scss/style.scss')
    .pipe(sass({ outputStyle: 'expanded' }).on('error', sass.logError))
    .pipe(postcss([autoprefixer(), cssnano()]))
    .pipe(dest('dist/css'));
}
```

Dart Sass への切り替えで `/deep/` や `::slotted` などの非推奨CSS記法の警告が出ることがある。古いSCSSファイルを棚卸しする機会にもなった。

## ハマりポイント5: BrowserSyncのESM対応

BrowserSync は現時点でESMに完全対応していない。`createRequire` で回避する。

```javascript
// gulpfile.mjs
import { createRequire } from 'module';
const require = createRequire(import.meta.url);
const browserSync = require('browser-sync').create();

function serve(done) {
  browserSync.init({
    server: {
      baseDir: './dist',
    },
    port: 3000,
    open: false,
  });
  done();
}

function reload(done) {
  browserSync.reload();
  done();
}
```

## 移行後の完成形gulpfile.mjs

[ToolShare Lab](https://and-and.net/) の実際の `gulpfile.mjs` に近い形:

```javascript
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
const webpackStream = require('webpack-stream');
const webpackConfig = require('./webpack.config.cjs');

const { src, dest, watch, series, parallel } = gulp;

// EJS → HTML
function buildEjs() {
  return src(['src/ejs/**/*.ejs', '!src/ejs/**/_*.ejs'])
    .pipe(ejs({}, {}, { ext: '.html' }))
    .pipe(rename({ extname: '.html' }))
    .pipe(dest('dist'));
}

// SCSS → CSS
function buildScss() {
  return src('src/scss/style.scss')
    .pipe(sass({ outputStyle: 'expanded' }).on('error', sass.logError))
    .pipe(postcss([tailwindcss(), autoprefixer(), cssnano()]))
    .pipe(dest('dist/css'))
    .pipe(browserSync.stream());
}

// JS → Webpack バンドル
function buildJs() {
  return src('src/js/main.js')
    .pipe(webpackStream(webpackConfig))
    .pipe(dest('dist/js'));
}

// BrowserSync サーバー起動
function serve(done) {
  browserSync.init({ server: './dist', port: 3000 });
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

## Gulp 4 → 5 移行チェックリスト

```
[ ] gulpfile.js → gulpfile.mjs にリネーム
[ ] require() → import に書き換え
[ ] exports.xxx → export const xxx に書き換え
[ ] exports.default → export default に書き換え
[ ] node-sass → sass + gulp-dart-sass に切り替え
[ ] ESM非対応プラグインを createRequire でラップ
[ ] webpack.config.js → webpack.config.cjs にリネーム
[ ] Node.js バージョンを v18以上に
[ ] npm install で依存関係を再インストール
[ ] gulp build でエラーが出ないか確認
[ ] gulp dev でBrowserSyncが起動するか確認
```

## まとめ

Gulp 4 から Gulp 5 への移行は、**ESMへの対応**が主な作業になる。移行でぶつかるポイントは概ね:

1. CJSプラグインとESMの競合 → `createRequire` で回避
2. `gulpfile.mjs` の `export` 構文への書き換え
3. Dart Sass への切り替え（Node Sass は廃止）
4. BrowserSync の ESM 非対応問題

[ToolShare Lab](https://and-and.net/) の150ページ規模のサイトでも、移行作業自体は半日で完了した。Gulp 5 に移行するとESMの`import/export`が使えてコードがすっきりするので、Gulp 4 を使い続けているプロジェクトは早めに移行を検討したい。
