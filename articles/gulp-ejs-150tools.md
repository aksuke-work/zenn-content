---
title: "Gulp5 + EJS + Tailwind + Webpackで150個のツールサイトを構築した技術スタック解説"
emoji: "🛠️"
type: "tech"
topics: ["gulp", "ejs", "tailwind", "個人開発"]
published: false
---

[ToolShare Lab](https://webatives.com/) というフリーランス向けの無料ツール集サイトを運営している。現在150個以上のツールが稼働中。技術スタックは Gulp 5 + EJS + Tailwind CSS + Webpack。なぜこの構成を選んだのか、どう設計しているのかを解説する。

## なぜNext.jsではなくGulp + EJSなのか

結論: **静的サイトにSPAフレームワークは不要**。

| 観点 | Next.js | Gulp + EJS |
|------|---------|-----------|
| 初回表示速度 | SSR/SSG可だがランタイムJSがある | 純粋なHTML。JSは必要な分だけ |
| Core Web Vitals | LCP/CLSに注意が必要 | ほぼ満点が出やすい |
| ビルド時間 | 150ページだと数分 | 数秒 |
| デプロイ | Vercel等のプラットフォーム依存 | rsyncでどこにでも置ける |
| 学習コスト | React + Next.jsの理解が必要 | HTML/CSS/JSの基礎だけ |
| ランニングコスト | 無料枠 or 月額課金 | Xserver等の安い共用サーバーでOK |

ツールサイトの各ページは「フォーム + 計算ロジック + 結果表示」というシンプルな構造。Reactの状態管理もサーバーサイドレンダリングも不要。**最もシンプルな技術で最も速いサイト**を作れるのがGulp + EJSの強み。

## Gulp 5のタスク設計

`gulpfile.mjs` の全体像:

```javascript
import gulp from 'gulp';
import ejs from 'gulp-ejs';
import sass from 'gulp-dart-sass';
import postcss from 'gulp-postcss';
import tailwindcss from '@tailwindcss/postcss';
import autoprefixer from 'autoprefixer';
import cssnano from 'cssnano';
import webpack from 'webpack-stream';
import browserSync from 'browser-sync';
import rename from 'gulp-rename';

// EJS → HTML
function buildEjs() {
  return gulp.src(['src/ejs/**/*.ejs', '!src/ejs/**/_*.ejs'])
    .pipe(ejs({}, {}, { ext: '.html' }))
    .pipe(rename({ extname: '.html' }))
    .pipe(gulp.dest('dist'));
}

// SCSS → CSS（Tailwindを含む）
function buildScss() {
  return gulp.src('src/scss/style.scss')
    .pipe(sass().on('error', sass.logError))
    .pipe(postcss([tailwindcss(), autoprefixer(), cssnano()]))
    .pipe(gulp.dest('dist/css'))
    .pipe(browserSync.stream());
}

// JS → バンドル（Webpack）
function buildJs() {
  return gulp.src('src/js/main.js')
    .pipe(webpack({
      mode: 'production',
      output: { filename: 'bundle.js' },
      module: {
        rules: [{
          test: /\.js$/,
          use: { loader: 'babel-loader' },
        }],
      },
    }))
    .pipe(gulp.dest('dist/js'));
}

// 監視 + BrowserSync
function watch() {
  browserSync.init({ server: './dist', port: 3000 });
  gulp.watch('src/ejs/**/*.ejs', buildEjs);
  gulp.watch('src/scss/**/*.scss', buildScss);
  gulp.watch('src/js/**/*.js', buildJs);
  gulp.watch('dist/**/*.html').on('change', browserSync.reload);
}

export const build = gulp.parallel(buildEjs, buildScss, buildJs);
export const dev = gulp.series(build, watch);
export default build;
```

Gulp 5のポイント:
- **ESM対応**（`gulpfile.mjs`）。`import/export` が使える
- `gulp-dart-sass` で Dart Sass を使う（Node Sassは非推奨）
- Tailwind CSSは `@tailwindcss/postcss` 経由でPostCSSパイプラインに統合

## EJSテンプレートのコンポーネント設計

150個のツールページで共通部分を使い回すため、EJSのパーシャル（`_*.ejs`）でコンポーネント分割している。

```
src/ejs/
├── _components/
│   ├── _header.ejs          # グローバルヘッダー
│   ├── _footer.ejs          # フッター
│   ├── _breadcrumb.ejs      # パンくずリスト
│   ├── _tool-form.ejs       # ツール共通のフォームレイアウト
│   ├── _result-table.ejs    # 計算結果テーブル
│   ├── _related-tools.ejs   # 関連ツール表示
│   └── _seo-head.ejs        # OGP/構造化データ
├── _layouts/
│   └── _base.ejs            # ベースレイアウト
├── tools/
│   ├── tax-calculator/
│   │   └── index.ejs
│   ├── loan-calculator/
│   │   └── index.ejs
│   └── ... (150個)
├── guides/
│   └── ... (ガイド記事)
└── index.ejs                 # トップページ
```

ベースレイアウトの使い方:

```ejs
<!-- src/ejs/tools/tax-calculator/index.ejs -->
<% const meta = {
  title: '所得税計算ツール',
  description: '年収から所得税を計算。累進課税の仕組みを...',
  path: '/tools/tax-calculator/',
  ogImage: '/images/og/tax-calculator.png',
}; %>
<%- include('../../_layouts/_base', { meta }) %>
```

```ejs
<!-- src/ejs/_layouts/_base.ejs -->
<!DOCTYPE html>
<html lang="ja">
<head>
  <%- include('../_components/_seo-head', { meta }) %>
  <link rel="stylesheet" href="/css/style.css">
</head>
<body>
  <%- include('../_components/_header') %>
  <%- include('../_components/_breadcrumb', { meta }) %>
  <main>
    <%- body %>
  </main>
  <%- include('../_components/_related-tools', { current: meta.path }) %>
  <%- include('../_components/_footer') %>
  <script src="/js/bundle.js"></script>
</body>
</html>
```

これでヘッダー/フッター/パンくず/関連ツールの修正は1ファイルで済む。150ページ分を一括で更新できる。

## Tailwind CSSとSCSSの共存

「TailwindとSCSSの両方を使う」のは一見冗長に見えるが、役割を分けている。

```scss
// src/scss/style.scss

// Tailwind（ユーティリティクラス）
@import "tailwindcss";

// BEM設計のカスタムスタイル
@import "components/header";
@import "components/footer";
@import "components/toolForm";
@import "components/resultTable";
@import "components/breadcrumb";
```

使い分けルール:
- **Tailwind**: レイアウト、余白、レスポンシブ、テキストサイズなどのユーティリティ
- **SCSS + BEM**: ツールフォーム、計算結果テーブルなど、サイト固有のコンポーネント

```scss
// src/scss/components/_toolForm.scss
.toolForm {
  &__inputGroup {
    // BEMでコンポーネント定義
  }

  &__label {
    font-feature-settings: 'palt';
  }

  &__input {
    // ...
  }

  &__submitButton {
    // ...
  }

  &__submitButton_state_disabled {
    opacity: 0.5;
    pointer-events: none;
  }
}
```

BEM命名規則:
- Element: `block__element`（サブ要素はキャメルケース `block__elementSub`）
- Modifier: `block__element_key_value` 形式
- `--` 記法は使わない

## 150個のツールの管理

### ディレクトリ設計

各ツールを独立したディレクトリにしている。

```
src/
├── ejs/tools/
│   ├── take-home-pay/index.ejs
│   ├── income-tax/index.ejs
│   ├── loan-calculator/index.ejs
│   └── ... (150個)
├── js/tools/
│   ├── take-home-pay.js
│   ├── income-tax.js
│   ├── loan-calculator.js
│   └── ... (各ツールのロジック)
└── scss/tools/
    ├── _take-home-pay.scss  (ツール固有のスタイルがある場合のみ)
    └── ...
```

### ツール一覧の自動生成

ツールのメタ情報をJSONで管理し、一覧ページとサイトマップを自動生成する。

```javascript
// tools-meta.json
[
  {
    "slug": "take-home-pay",
    "title": "手取り計算シミュレーター",
    "category": "税金",
    "description": "年収から手取り額を計算",
    "keywords": ["手取り", "年収", "税金"]
  },
  // ... 150個
]
```

## ビルド→rsyncデプロイの自動化

`deploy.sh` でビルドからデプロイまでワンコマンド:

```bash
#!/bin/bash
set -e

echo "Building..."
npx gulp build

echo "Deploying..."
rsync -avz --delete \
  --exclude '.git' \
  --exclude 'node_modules' \
  dist/ user@server:/home/user/public_html/

echo "Done!"
```

Xserverのような共用サーバーでも `rsync` でデプロイできる。CI/CDは使っていない。150ページのビルド+転送で30秒もかからない。

## Core Web Vitalsのスコア

Gulp + EJSの静的サイトはCore Web Vitalsで高スコアが出やすい。

| 指標 | スコア | 閾値 |
|------|--------|------|
| LCP（Largest Contentful Paint） | 0.8s | < 2.5s |
| FID（First Input Delay） | < 10ms | < 100ms |
| CLS（Cumulative Layout Shift） | 0.01 | < 0.1 |
| PageSpeed Insights | 98/100 | - |

高スコアの理由:
- **ランタイムJSがほぼない**。計算ツールのJSだけ
- **CSSが1ファイル**（cssnanoで圧縮済み）
- **画像が少ない**（ツールサイトなので）
- **サーバーサイド処理なし**（純粋な静的HTML）

## まとめ

Gulp 5 + EJS + Tailwind + Webpack で150個のツールサイトを運営して1年以上経つが、この構成に不満はない。

1. **静的サイトにReactフレームワークは不要**。Gulp + EJSで十分
2. **EJSのパーシャル**で150ページの共通部分を効率管理
3. **Tailwind + SCSS/BEM**は役割を分ければ共存できる
4. **ビルドが数秒**。開発体験が良い
5. **Core Web Vitals 98点**。SEOにも有利
6. **rsyncデプロイ**。プラットフォーム依存なし

「個人開発で大量のページを管理するならNext.js一択」みたいな風潮があるが、用途によっては古典的なGulpベースの構成のほうがシンプルで速い。

---

ToolShare Labのツール一覧は [webatives.com/tools/](https://webatives.com/tools/) で確認できる。使い方ガイドは [webatives.com/guides/](https://webatives.com/guides/) にまとめている。姉妹サイトの [AND TOOLS](https://and-tools.net/) はPHP + JSの構成で、こちらは税金計算に特化している。
