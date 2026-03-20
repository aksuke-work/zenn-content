---
title: "無料ツールサイトを作る技術選定 — Next.js vs 静的HTML vs WordPress"
emoji: "🏗️"
type: "tech"
topics: ["技術選定", "Next.js", "静的サイト", "個人開発"]
published: false
---

無料ツールサイトを作りたいとき、技術スタックの選択は長期的な運用コストとSEOに直結する。[AND TOOLS](https://and-tools.net/) と [ToolShare Lab](https://and-and.net/) という2つのツールサイトを運営してきた経験から、主要な選択肢を比較する。

## 比較する選択肢

今回比較するのは以下の4つだ：

1. **Next.js（SSG/ISR）** — React ベースのフレームワーク
2. **静的HTML（Gulp + EJS + SCSS）** — 素に近い静的サイト
3. **WordPress + プラグイン** — 最も普及したCMS
4. **Astro** — 最近注目のSSGフレームワーク

## 比較表

| 観点 | Next.js | 静的HTML | WordPress | Astro |
|-----|---------|---------|-----------|-------|
| Core Web Vitals | △ | ◎ | × | ◎ |
| SEO | ○ | ○ | △ | ○ |
| 開発速度 | ○ | △ | ◎ | ○ |
| ホスティングコスト | △〜× | ◎ | ◎〜△ | ◎ |
| ツール追加のしやすさ | ○ | △ | ◎ | ○ |
| AdSense適合性 | ○ | ◎ | ○ | ◎ |
| 長期保守性 | △ | ◎ | △ | ◎ |
| エンジニア以外の管理 | × | × | ◎ | × |

## Next.js の特徴と向き不向き

### 強み

**ISR（Incremental Static Regeneration）** でツールのデータを定期更新できる。外部APIのデータをツールに組み込む場合に便利だ。

```javascript
// pages/tool/exchange-rate.js
export async function getStaticProps() {
  const res = await fetch('https://api.exchangerate.host/latest?base=JPY');
  const data = await res.json();

  return {
    props: { rates: data.rates },
    revalidate: 3600, // 1時間ごとに再生成
  };
}
```

TypeScriptのサポートが充実しているため、ツールのロジックを型安全に書けるのも強みだ。

### 弱み

**ホスティングコスト** がかかる。VercelのFreeプランには制限がある（商用利用はProプランが必要）。CloudflareのPagesは寛容だが、Serverless Functionsに依存するとコストが上がる。

**ビルド時間** が長くなる。ツールが100〜200ページを超えるとビルドに数分かかるようになる。AND TOOLSを一時期Next.jsで作ろうとしたが、ビルド時間の問題と Vercelの商用サイト利用条件を確認して断念した。

**依存関係の複雑さ** もある。Next.jsのバージョンアップに追従するコストが長期的にかかる。

### 向いているケース

- ユーザー認証が必要なツール（ログイン機能付き）
- データベースと連携するツール
- 外部APIをリアルタイムで使うツール
- Reactエコシステムを使い慣れているチーム

## 静的HTML（Gulp + EJS + SCSS）の特徴

AND TOOLSは現在この構成だ。ToolShare LabはGulp 5 + EJS + SCSS + Tailwind + Webpackで構築している。

### 強み

**Core Web Vitalsが最強** になりやすい。余計なJavaScriptランタイムがないため、ページが軽い。PageSpeed Insightsのスコアを100点近くに維持しやすい。

AND TOOLSのトップページのPageSpeed Insightsスコア（モバイル）:
- Performance: 94
- SEO: 100
- Best Practices: 100
- Accessibility: 95

**ホスティングが安い** というか、Xserverやロリポップのような格安共有サーバーで十分動く。AND TOOLSはXserverのスタンダードプラン（月々1,000円程度）で動いている。

**長期保守性が高い** 。フレームワークに依存しないため、5年後も同じコードが動く。Next.jsのバージョン追従問題がない。

### 弱み

**繰り返し処理が手作業になりがち** 。ツールを1つ追加するたびにHTMLファイルを1つ作る必要がある。EJSのテンプレートで共通化できるが、限界がある。

**JavaScriptの管理が煩雑** になる。複数のツールで共通のロジックを使い回す仕組みを自前で作る必要がある。

### Gulp + EJSの構成例

```
src/
├── ejs/
│   ├── _layouts/
│   │   └── default.ejs   # 共通レイアウト
│   ├── _partials/
│   │   ├── header.ejs
│   │   └── footer.ejs
│   └── tool/
│       ├── consumption-tax.ejs
│       └── income-tax.ejs
├── scss/
│   ├── _base.scss
│   ├── _tool.scss
│   └── main.scss
└── js/
    ├── common.js
    └── tools/
        ├── consumption-tax.js
        └── income-tax.js
```

```javascript
// gulpfile.js（抜粋）
const gulp = require('gulp');
const ejs = require('gulp-ejs');
const rename = require('gulp-rename');
const sass = require('gulp-sass')(require('sass'));

function html() {
  return gulp
    .src('./src/ejs/**/*.ejs')
    .pipe(ejs({}))
    .pipe(rename({ extname: '.html' }))
    .pipe(gulp.dest('./dist'));
}

function css() {
  return gulp
    .src('./src/scss/main.scss')
    .pipe(sass({ outputStyle: 'compressed' }).on('error', sass.logError))
    .pipe(gulp.dest('./dist/css'));
}

exports.build = gulp.parallel(html, css);
```

### 向いているケース

- SEOとCore Web Vitalsを最優先にしたい
- 安いホスティングで運用したい
- フレームワークに依存したくない
- ツール数が50以下のサイト

## WordPress の特徴

### 強み

**管理画面でコンテンツを更新できる** 。エンジニア以外がコンテンツを追加できる。ツールサイトでは「ツール + 解説記事」を組み合わせることが多く、記事管理はWordPressが楽だ。

**プラグインでツールを追加できる** 。Gravity FormsやContact Form 7のようなプラグインで計算フォームを作れる。コードを書かずにツールが作れるケースもある。

### 弱み

**Core Web Vitalsが悪化しやすい** 。プラグインを入れるたびにCSSとJavaScriptが増える。デフォルトのWordPressで静的HTMLサイトに匹敵するパフォーマンスを出すには、かなりのチューニングが必要だ。

**セキュリティ管理コスト** がかかる。WordPressは最も攻撃される対象の1つだ。プラグインを定期的に更新しないとセキュリティリスクになる。

**ホスティングが高くなりがち** 。PHPとMySQLが動く環境が必要なため、格安の静的ホスティングは使えない。

### 向いているケース

- ツール + ブログ記事を組み合わせたサイト
- エンジニア以外も管理する
- SEO記事を大量に書く予定がある

## Astro の特徴

Astroは2023年以降急速に普及したSSGフレームワークだ。「Islands Architecture」という考え方でJavaScriptを最小限にできる。

### 強み

**デフォルトでJavaScriptを送らない** 。React/Vue/Svelteのコンポーネントを使いながらも、インタラクションが不要な部分はHTMLとして出力する。Core Web Vitalsが良くなりやすい。

**Markdownとコンポーネントを混在できる** 。ツールの解説記事をMarkdownで書きながら、計算ツール部分はReactコンポーネントとして埋め込める。

```astro
---
// src/pages/tool/consumption-tax.astro
import ConsumptionTaxCalculator from '../components/ConsumptionTaxCalculator.tsx';
import Layout from '../layouts/Layout.astro';
---

<Layout title="消費税計算ツール">
  <!-- 計算ツール: クライアントサイドで動作 -->
  <ConsumptionTaxCalculator client:load />

  <!-- 解説テキスト: 静的HTML（JSなし） -->
  <section>
    <h2>消費税の計算方法</h2>
    <p>...</p>
  </section>
</Layout>
```

### 弱み

**まだ新しい** ためエコシステムが発展途上だ。長期運用で問題が出るかは未知数。

**学習コスト** がある。独自のコンポーネント構文（`.astro`ファイル）に慣れるまで時間がかかる。

### 向いているケース

- 将来性を重視したい
- TypeScriptを活用したい
- JavaScriptフレームワークの知識がある

## 結論：AND TOOLSが静的HTMLを選んだ理由

AND TOOLSを立ち上げるときの選定基準は：

1. Core Web Vitalsのスコアを高く保ちたい
2. ホスティングコストを最小化したい
3. 長期的なメンテナンスコストを下げたい
4. フリーランス一人で管理できる

これらの条件で静的HTML（Gulp + EJS）が最適解だった。

SEOの観点では静的HTMLは最強クラスだ。「ツールサイトでCore Web Vitalsのスコアが良い」はそれ自体がSEO上の強みになる。

一方、[ToolShare Lab](https://and-and.net/) ではGulp 5 + EJS + SCSS + Tailwind + Webpackという構成でさらに進化させた。Tailwindを導入することでスタイルの一貫性が保ちやすくなり、Webpackで複雑なJavaScriptモジュールを管理できるようになった。

## 2026年時点での推奨

| ケース | 推奨スタック |
|--------|-----------|
| 個人開発・初回 | Astro（SEOを意識しながら開発しやすい） |
| 既存の静的サイト | 静的HTML継続（移行コストのデメリットが大きい） |
| 認証・DB必要 | Next.js |
| 非エンジニアも管理 | WordPress |
| コスト最優先 | Cloudflare Pages + Astro（無料） |

Core Web VitalsとSEOを最優先にするなら、JavaScriptフレームワークのランタイムを最小化できる構成を選ぶのが正解だ。「動くことは同じでも、どれだけ速く動くか」がツールサイトの競争力になる。
