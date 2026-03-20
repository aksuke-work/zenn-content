---
title: "静的サイトの最適化 — HTML/CSS/JS minifyとGzip圧縮"
emoji: "⚡"
type: "tech"
topics: ["gulp", "パフォーマンス", "最適化", "静的サイト"]
published: false
---

[ToolShare Lab](https://and-and.net/) は PageSpeed Insights で 98/100 を維持している。Gulp のビルドパイプラインに HTML/CSS/JS のminifyとGzip圧縮を組み込んだ結果だ。静的サイトの最適化パイプラインをまとめる。

## 最適化の全体像

```
src/ → [Gulp ビルド] → dist/（minify済み）→ [rsync] → サーバー
                                                        ↓
                                               Xserver の gzip 圧縮設定
```

最適化は「ビルド時」と「サーバー設定」の2段階:

1. **ビルド時**: HTML/CSS/JS をminify → ファイルサイズを削減
2. **サーバー設定**: Gzip/Brotli 圧縮 → 転送サイズをさらに削減

## CSS の最適化（cssnano）

```bash
npm install cssnano
```

```javascript
// gulpfile.mjs
import postcss from 'gulp-postcss';
import cssnano from 'cssnano';
import autoprefixer from 'autoprefixer';

function buildScss() {
  const isProduction = process.env.NODE_ENV === 'production';

  const postcssPlugins = [autoprefixer()];
  if (isProduction) {
    postcssPlugins.push(cssnano({
      preset: ['default', {
        discardComments: { removeAll: true },   // コメント削除
        normalizeWhitespace: true,               // 空白最適化
        minifyFontValues: true,                  // font値を最適化
        colormin: true,                          // #ffffff → #fff
      }],
    }));
  }

  return src('src/scss/style.scss')
    .pipe(sass().on('error', sass.logError))
    .pipe(postcss(postcssPlugins))
    .pipe(dest('dist/css'));
}
```

[ToolShare Lab](https://and-and.net/) での効果:

| 状態 | ファイルサイズ |
|------|-------------|
| SCSS → CSS（minifyなし） | 210KB |
| cssnano 適用後 | 68KB |
| gzip 後 | 14KB |

Tailwind を使っている場合、`cssnano` に加えて Tailwind の PurgeCSS 機能（未使用クラス削除）が重要。

```css
/* src/scss/style.scss */
@import "tailwindcss";

/* Tailwind v4 は EJSファイルを自動スキャンして未使用クラスを除去 */
/* コンテンツのスキャン対象を設定 */
@source "../../src/ejs/**/*.ejs";
@source "../../src/js/**/*.js";
```

## JS の最適化（Webpack）

Webpack の `mode: 'production'` で自動的に Terser が動く。

```javascript
// webpack.config.cjs
const TerserPlugin = require('terser-webpack-plugin');

module.exports = {
  mode: process.env.NODE_ENV === 'production' ? 'production' : 'development',
  devtool: process.env.NODE_ENV === 'production' ? false : 'source-map',

  optimization: {
    minimize: true,
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          compress: {
            drop_console: true,       // console.log を本番では削除
            drop_debugger: true,
            pure_funcs: ['console.info', 'console.debug'],
          },
          output: {
            comments: false,          // コメント削除
          },
          mangle: {
            safari10: true,           // Safari 10のバグ対応
          },
        },
        extractComments: false,       // LICENSE ファイルを生成しない
      }),
    ],
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendor',
          chunks: 'all',
        },
      },
    },
  },
};
```

**`drop_console: true`** は本番で `console.log` を削除する。デバッグ用ログが残ったまま本番デプロイするミスを防ぐ。

## HTML の最適化（gulp-htmlmin）

```bash
npm install gulp-htmlmin
```

```javascript
// gulpfile.mjs
import htmlmin from 'gulp-htmlmin';

function buildEjs() {
  const isProduction = process.env.NODE_ENV === 'production';

  let stream = src(['src/ejs/**/*.ejs', '!src/ejs/**/_*.ejs'])
    .pipe(ejs({}, { root: './src/ejs' }))
    .pipe(rename({ extname: '.html' }));

  if (isProduction) {
    stream = stream.pipe(htmlmin({
      collapseWhitespace: true,       // 空白を削除
      removeComments: true,           // HTMLコメントを削除
      removeAttributeQuotes: false,   // 属性の引用符は残す（安全のため）
      minifyCSS: true,                // <style>タグ内のCSSも圧縮
      minifyJS: true,                 // <script>タグ内のJSも圧縮
      removeEmptyAttributes: false,   // 空属性は残す（aria-*など）
    }));
  }

  return stream.pipe(dest('dist'));
}
```

HTMLminのbefore/after比較（1ページあたり）:

| 状態 | サイズ |
|------|--------|
| EJS → HTML（生） | 28KB |
| htmlmin 適用後 | 18KB |

150ページ × 10KB削減 = **1.5MB削減**（転送量の節約）

## 画像の最適化（gulp-imagemin）

```bash
npm install gulp-imagemin
```

```javascript
// gulpfile.mjs
import { createRequire } from 'module';
const require = createRequire(import.meta.url);
const imagemin = require('gulp-imagemin');
const imageminMozjpeg = require('imagemin-mozjpeg');
const imageminPngquant = require('imagemin-pngquant');
const imageminSvgo = require('imagemin-svgo');
const imageminWebp = require('imagemin-webp');

function buildImages() {
  return src('src/images/**/*.{jpg,jpeg,png,gif,svg}')
    .pipe(imagemin([
      imageminMozjpeg({ quality: 85, progressive: true }),
      imageminPngquant({ quality: [0.65, 0.80] }),
      imageminSvgo({
        plugins: [
          { name: 'removeViewBox', active: false },
        ],
      }),
    ]))
    .pipe(dest('dist/images'));
}

// WebP変換タスク
function buildWebp() {
  return src('src/images/**/*.{jpg,jpeg,png}')
    .pipe(imagemin([imageminWebp({ quality: 85 })]))
    .pipe(rename({ extname: '.webp' }))
    .pipe(dest('dist/images'));
}
```

EJSで `<picture>` タグを使ってWebPを提供:

```ejs
<!-- _components/_image.ejs -->
<picture>
  <source srcset="/images/<%= src %>.webp" type="image/webp">
  <img
    src="/images/<%= src %>.jpg"
    alt="<%= alt %>"
    width="<%= width %>"
    height="<%= height %>"
    loading="<%= lazy ? 'lazy' : 'eager' %>"
  >
</picture>
```

## サーバー側のGzip設定（Xserver）

Xserverの `.htaccess` でGzip圧縮を設定:

```apache
# .htaccess（dist/ にこのファイルを置く）

# Gzip圧縮
<IfModule mod_deflate.c>
  AddOutputFilterByType DEFLATE text/html
  AddOutputFilterByType DEFLATE text/css
  AddOutputFilterByType DEFLATE text/javascript
  AddOutputFilterByType DEFLATE application/javascript
  AddOutputFilterByType DEFLATE application/json
  AddOutputFilterByType DEFLATE image/svg+xml
  AddOutputFilterByType DEFLATE application/xml
  AddOutputFilterByType DEFLATE text/xml
</IfModule>

# ブラウザキャッシュ設定
<IfModule mod_expires.c>
  ExpiresActive On

  # HTML: キャッシュなし（常に最新を取得）
  ExpiresByType text/html "access plus 0 seconds"

  # CSS/JS: 1年キャッシュ（ファイル名にハッシュを使う場合）
  ExpiresByType text/css "access plus 1 year"
  ExpiresByType application/javascript "access plus 1 year"

  # 画像: 30日キャッシュ
  ExpiresByType image/jpeg "access plus 30 days"
  ExpiresByType image/png "access plus 30 days"
  ExpiresByType image/webp "access plus 30 days"
  ExpiresByType image/svg+xml "access plus 30 days"
</IfModule>

# WebP サポートチェック
<IfModule mod_rewrite.c>
  RewriteEngine On
  RewriteCond %{HTTP_ACCEPT} image/webp
  RewriteCond %{REQUEST_FILENAME} \.(jpe?g|png)$
  RewriteCond %{REQUEST_FILENAME}.webp -f
  RewriteRule ^(.+)\.(jpe?g|png)$ $1.webp [T=image/webp,L]
</IfModule>
```

## ビルドサイズの可視化

```bash
npm install -D gulp-size
```

```javascript
// gulpfile.mjs
import size from 'gulp-size';

function buildScss() {
  return src('src/scss/style.scss')
    .pipe(sass().on('error', sass.logError))
    .pipe(postcss([tailwindcss(), autoprefixer(), cssnano()]))
    .pipe(size({ showFiles: true, showTotal: false, title: 'CSS' }))
    .pipe(dest('dist/css'));
}
```

ビルド時に出力されるサイズ情報:

```
[14:23:01] CSS style.css 68.2 kB
[14:23:01] JS bundle.js 142 kB
[14:23:01] JS vendor.js 80.4 kB
```

## PageSpeed Insights の改善前後

[ToolShare Lab](https://and-and.net/) での最適化前後:

| 指標 | 最適化前 | 最適化後 |
|------|---------|---------|
| PageSpeed (Mobile) | 72 | 98 |
| PageSpeed (Desktop) | 89 | 100 |
| LCP | 3.2s | 0.8s |
| CLS | 0.05 | 0.01 |
| TBT | 180ms | 20ms |

最も効果があった対策:
1. **Tailwind PurgeCSS** → CSS 800KB → 68KB（92%削減）
2. **画像の WebP 変換** → 画像転送量 60%削減
3. **`drop_console: true`** → JS 15%削減
4. **`loading="lazy"`** → 画像の遅延読み込み

## まとめ

静的サイトの最適化パイプライン:

1. **CSS**: cssnano + Tailwind PurgeCSS → 90%以上の削減も可能
2. **JS**: Webpack production mode + Terser → Tree-shaking + minify
3. **HTML**: gulp-htmlmin → 30〜40%削減
4. **画像**: imagemin + WebP変換 → 50〜60%削減
5. **サーバー**: Gzip + ブラウザキャッシュ → 転送量をさらに削減

開発環境（`gulp dev`）ではminifyしない、本番ビルド（`NODE_ENV=production gulp build`）でだけ最適化が走るように切り替えるのがポイント。minifyしない環境でデバッグして、本番は最適化済みの成果物をデプロイするワークフローが快適だ。
