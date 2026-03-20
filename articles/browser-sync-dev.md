---
title: "BrowserSyncで快適なローカル開発環境を構築する"
emoji: "🔄"
type: "tech"
topics: ["browsersync", "gulp", "フロントエンド", "開発環境"]
published: false
---

[ToolShare Lab](https://webatives.com/) の開発環境は Gulp + BrowserSync で構築している。EJSを編集すると即座にブラウザが更新される環境で、150個のツールページを日々開発している。BrowserSync の設定ノウハウをまとめる。

## BrowserSync とは

BrowserSync は静的サイトの開発用ローカルサーバーで、以下の機能を持つ:

- **ライブリロード**: ファイルを保存するとブラウザが自動更新
- **CSSインジェクション**: CSSの変更はリロードなしで即時反映（ページ状態を維持したまま）
- **デバイス同期**: 複数のブラウザ/デバイスを同時に同期操作
- **UI**: `localhost:3001` でコントロールUIにアクセス

Gulp 5（ESM）での注意点: BrowserSync 本体はまだCJS形式のため `createRequire` でimportする。

## 基本設定

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

const { src, dest, watch, series, parallel } = gulp;

// BrowserSync サーバー起動
function serve(done) {
  browserSync.init({
    server: {
      baseDir: './dist',
    },
    port: 3000,
    ui: {
      port: 3001,  // BrowserSync コントロールUI
    },
    open: false,      // 起動時にブラウザを自動オープンしない
    notify: false,    // ブラウザ上の通知バナーを非表示
    ghostMode: false, // デバイス同期（クリック・スクロール）を無効に
  });
  done();
}

// ブラウザリロード
function reload(done) {
  browserSync.reload();
  done();
}

// EJS ビルド
function buildEjs() {
  return src(['src/ejs/**/*.ejs', '!src/ejs/**/_*.ejs'])
    .pipe(ejs({}, { root: './src/ejs' }))
    .pipe(rename({ extname: '.html' }))
    .pipe(dest('dist'));
}

// SCSS ビルド（CSSインジェクション）
function buildScss() {
  return src('src/scss/style.scss')
    .pipe(sass().on('error', sass.logError))
    .pipe(postcss([tailwindcss(), autoprefixer()]))
    .pipe(dest('dist/css'))
    .pipe(browserSync.stream()); // ← stream()でCSSインジェクション
}

// ファイル監視
function watchFiles() {
  // EJS変更 → ビルド → リロード
  watch('src/ejs/**/*.ejs', series(buildEjs, reload));
  // SCSS変更 → ビルド → CSSインジェクション（リロードなし）
  watch('src/scss/**/*.scss', buildScss);
  // JS変更 → ビルド → リロード
  watch('src/js/**/*.js', series(buildJs, reload));
}

export const dev = series(
  parallel(buildEjs, buildScss, buildJs),
  serve,
  watchFiles,
);
export default dev;
```

## CSSインジェクション vs ページリロード

BrowserSync の重要な機能が **CSS インジェクション**。

```javascript
// CSSインジェクション（リロードなし）
.pipe(browserSync.stream())

// ページリロード（状態がリセットされる）
browserSync.reload()
```

**CSSインジェクション**のメリット:
- フォームに入力中でも値が消えない
- スクロール位置が維持される
- アニメーションが途切れない

[ToolShare Lab](https://webatives.com/) のツールページは「フォームに値を入力して計算する」UIが多い。SCSSを調整しながらフォームの見た目を確認するとき、リロードされると入力値が消えてしまう。CSSインジェクションなら入力値を保ったままスタイルの変化を確認できる。

## 複数のURLパターンへの対応

150ページのサイトではURLのディレクトリ構造を正しく扱う必要がある。

```javascript
// NG: index.html が見えるURL
// http://localhost:3000/tools/income-tax/index.html

// OK: きれいなURL
// http://localhost:3000/tools/income-tax/
```

BrowserSync の `serveStaticOptions` で対応:

```javascript
function serve(done) {
  browserSync.init({
    server: {
      baseDir: './dist',
      serveStaticOptions: {
        extensions: ['html'],  // /tools/income-tax/ → /tools/income-tax/index.html を探す
      },
    },
    port: 3000,
  });
  done();
}
```

## プロキシモードの使い方

PHPやWordPressのローカルサーバーにBrowserSyncをプロキシとして重ねる使い方もある。

```javascript
// Xampや MAMPで動いているWordPressへのプロキシ
function serve(done) {
  browserSync.init({
    proxy: 'http://localhost:8080',  // 既存のサーバー
    port: 3000,
    open: false,
  });
  done();
}
```

[ToolShare Lab](https://webatives.com/) は静的サイトなので `server` モードを使っているが、WordPressサイトを並行開発するときはプロキシモードが役立つ。

## ミドルウェアで404カスタマイズ

```javascript
const require = createRequire(import.meta.url);
const connectHistory = require('connect-history-api-fallback');

function serve(done) {
  browserSync.init({
    server: {
      baseDir: './dist',
      middleware: [
        // SPA風の404ハンドリング
        connectHistory({
          rewrites: [
            // /tools/xxx/ → dist/tools/xxx/index.html
            { from: /^\/tools\/.*\/$/, to: '/404.html' },
          ],
        }),
      ],
    },
    port: 3000,
  });
  done();
}
```

## 複数デバイスでの確認

BrowserSync の強みのひとつが複数デバイス同期。

```javascript
function serve(done) {
  browserSync.init({
    server: { baseDir: './dist' },
    port: 3000,
    // ローカルネットワークのIPを表示（スマホからアクセス用）
    // 起動時に "External: http://192.168.x.x:3000" が表示される
  });
  done();
}
```

起動時に表示されるIPアドレス（例: `http://192.168.1.5:3000`）をスマホのブラウザで開くと、PC側でのスクロール・クリックがスマホにも同期される。

ただし `ghostMode` はツールサイトの場合は混乱の元になるので無効にしている:

```javascript
ghostMode: {
  clicks: false,   // クリック同期なし
  scroll: true,    // スクロール同期あり（レスポンシブ確認に便利）
  forms: false,    // フォーム入力同期なし
},
```

## パフォーマンス: watchの絞り込み

ファイル数が多いと `watch` が重くなる。監視対象を絞る。

```javascript
function watchFiles() {
  // NG: node_modules や dist まで監視してしまう
  watch('**/*.ejs', series(buildEjs, reload));

  // OK: src 以下だけを監視
  watch('src/ejs/**/*.ejs', series(buildEjs, reload));
  watch('src/scss/**/*.scss', buildScss);
  watch('src/js/**/*.js', series(buildJs, reload));
  // JSONデータが変わったときはEJSもリビルド
  watch('src/data/**/*.json', series(buildEjs, reload));
}
```

gulp の `watch` は `chokidar` を使っており、`usePolling` オプションで Docker 内でも使える:

```javascript
watch('src/ejs/**/*.ejs', { usePolling: true }, series(buildEjs, reload));
```

## 起動時の出力確認

BrowserSync が正しく起動していると以下が表示される:

```
[Browsersync] Access URLs:
 -------------------------------------------
        Local: http://localhost:3000
     External: http://192.168.1.5:3000
 -------------------------------------------
          UI: http://localhost:3001
 UI External: http://192.168.1.5:3001
 -------------------------------------------
[Browsersync] Serving files from: ./dist
```

## まとめ

BrowserSync 設定のポイント:

1. **Gulp 5（ESM）では `createRequire`** でBrowserSyncをimport
2. **SCSS変更は `browserSync.stream()`** でCSSインジェクション → フォームの入力値が消えない
3. **EJS/JS変更は `reload()`** でページリロード
4. **`serveStaticOptions: { extensions: ['html'] }`** できれいなURLを実現
5. **`ghostMode`** はツールサイトでは無効にする
6. **watch対象を `src/` 以下に絞る** → パフォーマンス改善

[ToolShare Lab](https://webatives.com/) では BrowserSync のおかげで、SCSSを調整しながらフォームの動作を同時に確認できる。リロードのたびに入力値を再入力する手間がなくなり、開発効率が大幅に上がった。
