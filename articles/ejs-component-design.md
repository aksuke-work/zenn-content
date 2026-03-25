---
title: "EJSでコンポーネント指向設計 — includeとpartialの使い分け"
emoji: "🧩"
type: "tech"
topics: ["ejs", "gulp", "html", "フロントエンド"]
published: true
---

[ToolShare Lab](https://webatives.com/) は150個以上のツールページを EJS でテンプレート化して運営している。ページ数が増えるにつれ「どう分割するか」が重要になってくる。`include` を適切に使ったコンポーネント設計のノウハウをまとめる。

## EJSのincludeの基本

EJSの `include` は2種類の書き方がある。

```ejs
<%- include('./path/to/file') %>    // HTMLとしてレンダリング（エスケープなし）
<%= include('./path/to/file') %>    // 文字列としてエスケープ（通常は使わない）
```

コンポーネントの埋め込みには必ず `<%-` を使う。`<%=` にすると生HTMLが文字列として出力されてしまう。

## ディレクトリ構造の設計

[ToolShare Lab](https://webatives.com/) でのディレクトリ構造:

```
src/ejs/
├── _components/       # アンダースコアプレフィクスで「パーシャル」を表す
│   ├── _head.ejs      # <head>タグ内（meta, OGP, JSON-LD）
│   ├── _header.ejs    # グローバルナビ
│   ├── _footer.ejs    # フッター
│   ├── _breadcrumb.ejs
│   ├── _tool-card.ejs # ツール一覧カード
│   └── _related-tools.ejs
├── _layouts/
│   └── _base.ejs      # ベースレイアウト
├── tools/
│   ├── income-tax/
│   │   └── index.ejs
│   ├── loan-calculator/
│   │   └── index.ejs
│   └── ... (150個)
├── guides/
│   └── ... (ガイド記事)
└── index.ejs
```

**命名規則**:
- `_*.ejs` 形式（アンダースコア始まり）→ パーシャル（直接HTMLにならないファイル）
- `index.ejs` → ページファイル（HTMLに変換される）

Gulp の glob パターンでパーシャルをビルド対象から除外できる:

```javascript
// gulpfile.mjs
function buildEjs() {
  return src(['src/ejs/**/*.ejs', '!src/ejs/**/_*.ejs'])
    .pipe(ejs({}, {}, { ext: '.html' }))
    .pipe(rename({ extname: '.html' }))
    .pipe(dest('dist'));
}
```

## ベースレイアウトパターン

150ページで共通する「外枠」をベースレイアウトとして切り出す。各ページはメタ情報とコンテンツだけを定義すればよい。

```ejs
<!-- src/ejs/_layouts/_base.ejs -->
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title><%= meta.title %> | ToolShare Lab</title>
  <meta name="description" content="<%= meta.description %>">
  <%- include('../_components/_head', { meta }) %>
  <link rel="stylesheet" href="/css/style.css">
</head>
<body class="<%= meta.bodyClass || '' %>">
  <%- include('../_components/_header', { currentPath: meta.path }) %>
  <%- include('../_components/_breadcrumb', { breadcrumb: meta.breadcrumb }) %>
  <main class="main">
    <%- body %>
  </main>
  <%- include('../_components/_related-tools', { current: meta.path, category: meta.category }) %>
  <%- include('../_components/_footer') %>
  <script src="/js/bundle.js"></script>
  <% if (meta.pageJs) { %>
    <script src="/js/tools/<%= meta.pageJs %>.js"></script>
  <% } %>
</body>
</html>
```

各ツールページ:

```ejs
<!-- src/ejs/tools/income-tax/index.ejs -->
<%
const meta = {
  title: '所得税計算ツール',
  description: '年収から所得税を正確に計算。累進課税・各種控除に対応。',
  path: '/tools/income-tax/',
  category: 'tax',
  pageJs: 'income-tax',
  breadcrumb: [
    { label: 'ホーム', href: '/' },
    { label: '税金計算', href: '/tools/tax/' },
    { label: '所得税計算ツール', href: null },
  ],
};
%>

<%- include('../../_layouts/_base', { meta, body: contentHtml }) %>
```

`body` 変数の受け渡しは gulp-ejs の制約上そのままでは動かないため、実際には「include でページコンテンツを呼ぶ」パターンのほうが素直。

## 実践的なincludeパターン

### パターン1: データを渡してコンポーネントを動的生成

```ejs
<!-- _components/_tool-card.ejs -->
<article class="toolCard">
  <a href="<%= tool.href %>" class="toolCard__link">
    <div class="toolCard__icon"><%= tool.icon %></div>
    <h3 class="toolCard__title"><%= tool.title %></h3>
    <p class="toolCard__description"><%= tool.description %></p>
  </a>
</article>
```

```ejs
<!-- tools/index.ejs（一覧ページ） -->
<% tools.forEach(function(tool) { %>
  <%- include('../_components/_tool-card', { tool }) %>
<% }) %>
```

データは gulpfile.mjs 側から EJS に渡す:

```javascript
// gulpfile.mjs
import toolsMeta from './src/data/tools-meta.json' assert { type: 'json' };

function buildEjs() {
  return src(['src/ejs/**/*.ejs', '!src/ejs/**/_*.ejs'])
    .pipe(ejs({ tools: toolsMeta }))
    .pipe(rename({ extname: '.html' }))
    .pipe(dest('dist'));
}
```

### パターン2: 条件付きinclude

```ejs
<!-- ページタイプに応じてコンポーネントを切り替え -->
<% if (meta.type === 'tool') { %>
  <%- include('../_components/_tool-cta') %>
<% } else if (meta.type === 'guide') { %>
  <%- include('../_components/_guide-cta') %>
<% } %>
```

### パターン3: デフォルト値付きのinclude

```ejs
<!-- _components/_breadcrumb.ejs -->
<% const items = breadcrumb || [{ label: 'ホーム', href: '/' }]; %>
<nav class="breadcrumb" aria-label="パンくずリスト">
  <ol class="breadcrumb__list">
    <% items.forEach(function(item, index) { %>
      <li class="breadcrumb__item">
        <% if (item.href && index < items.length - 1) { %>
          <a href="<%= item.href %>" class="breadcrumb__link"><%= item.label %></a>
        <% } else { %>
          <span class="breadcrumb__current" aria-current="page"><%= item.label %></span>
        <% } %>
      </li>
    <% }) %>
  </ol>
</nav>
```

## includeのパスの扱いに注意

EJSの `include` はパスが**相対パス**で解決される。ネストが深くなると管理が難しい。

```ejs
<!-- src/ejs/tools/tax/income-tax/index.ejs -->
<!-- NG: 深いネストで相対パスが複雑になる -->
<%- include('../../../../_components/_header') %>

<!-- OK: gulp-ejs の root オプションで絶対パス起点を設定 -->
<%- include('/_components/_header') %>
```

gulp-ejs の `root` オプションを設定すると `/` がEJSディレクトリのルートになる:

```javascript
function buildEjs() {
  return src(['src/ejs/**/*.ejs', '!src/ejs/**/_*.ejs'])
    .pipe(ejs({}, { root: './src/ejs' }))
    .pipe(rename({ extname: '.html' }))
    .pipe(dest('dist'));
}
```

これで深いネストでも `/_components/_header` で呼べる。[ToolShare Lab](https://webatives.com/) は3〜4階層のディレクトリがあるため、この設定は必須だった。

## コンポーネントの粒度設計

**細かすぎる分割はNG**。実際の失敗例:

```ejs
<!-- NG: 1行のためだけにinclude（オーバーエンジニアリング）-->
<%- include('./_tool-title', { title: meta.title }) %>

<!-- OK: 意味のある単位でまとめる -->
<div class="toolHero">
  <h1 class="toolHero__title"><%= meta.title %></h1>
  <p class="toolHero__description"><%= meta.description %></p>
</div>
```

[ToolShare Lab](https://webatives.com/) での分割基準:
- **コンポーネント化する**: 3ページ以上で使い回す部分
- **インラインで書く**: そのページ固有のコンテンツ
- **データで動的化する**: ツールカード、関連ツール一覧など繰り返し要素

## まとめ

EJSのコンポーネント設計のポイント:

1. **アンダースコアプレフィクス**でパーシャルを明示 → Gulp glob で自動除外
2. **ベースレイアウト**で外枠を共通化 → 全ページを一括更新できる
3. **データを上から下へ渡す** → `ejs()` にオブジェクトを渡して全ページで参照
4. **rootオプション**でパスを絶対パス起点に → 深いネストでも迷わない
5. **コンポーネント粒度は「3ページ以上で使い回すか」**で判断

150ページ規模でもEJSのシンプルな `include` で十分管理できる。TemplateリテラルやJSXと違い、HTMLをそのまま書けるので非エンジニアのチームメンバーへの引き継ぎもしやすい。
