---
title: "SVGアイコンシステムの構築 — sprite vs inline vs component"
emoji: "🔷"
type: "tech"
topics: ["svg", "css", "frontend", "webdesign"]
published: false
---

フリーランス向けの無料ツール集 [AND TOOLS](https://and-tools.net/) や [ToolShare Lab](https://webatives.com/) のUIには、多数のアイコンを使っている。「アイコンをどう管理・実装するか」は、プロジェクトの技術スタックや規模によって最適解が変わる。

この記事では、SVGアイコンの実装方法の代表的な3パターン（sprite / inline / component）を比較し、それぞれの実装コードをまとめる。

---

## 3パターンの比較

| 方式 | メリット | デメリット | 向くケース |
|---|---|---|---|
| **SVG Sprite** | HTTPリクエストを削減、CSSでスタイル変更可 | ビルド設定が必要、初期学習コスト | 静的サイト・多数のアイコン |
| **Inline SVG** | CSSでフル制御可、SEO・a11y◎ | HTMLが冗長になる、管理しにくい | 少数のアイコン・特殊なアニメーション |
| **img / CSS mask** | シンプル | CSSカスタマイズ制限、1アイコン1リクエスト | 小規模・動的な色変更不要な場合 |

---

## パターン1：SVG Sprite

複数のSVGアイコンを1つのSVGファイル（スプライト）にまとめ、`<use>` 要素で参照する方法。

### スプライトファイルの作成

```xml
<!-- /images/icons.svg -->
<svg xmlns="http://www.w3.org/2000/svg" style="display: none;">

  <symbol id="icon-home" viewBox="0 0 24 24">
    <path d="M10 20v-6h4v6h5v-8h3L12 3 2 12h3v8z"/>
  </symbol>

  <symbol id="icon-search" viewBox="0 0 24 24">
    <path d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z"
      stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
  </symbol>

  <symbol id="icon-close" viewBox="0 0 24 24">
    <path d="M6 18L18 6M6 6l12 12"
      stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
  </symbol>

  <symbol id="icon-menu" viewBox="0 0 24 24">
    <path d="M4 6h16M4 12h16M4 18h16"
      stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
  </symbol>

</svg>
```

### HTML での使い方

```html
<!-- スプライトファイルを読み込む（bodyの先頭） -->
<div aria-hidden="true" style="display: none;">
  <!-- インラインで埋め込む場合 -->
</div>

<!-- 外部ファイルを参照する場合 -->
<svg class="icon" aria-hidden="true">
  <use href="/images/icons.svg#icon-home"></use>
</svg>

<!-- ボタン内でのアクセシブルな使い方 -->
<button class="btn-icon" aria-label="ホームに戻る">
  <svg class="icon" aria-hidden="true">
    <use href="/images/icons.svg#icon-home"></use>
  </svg>
</button>

<!-- テキストと並べる場合 -->
<a href="/" class="nav-link">
  <svg class="icon" aria-hidden="true">
    <use href="/images/icons.svg#icon-home"></use>
  </svg>
  <span>ホーム</span>
</a>
```

### CSS スタイリング

```css
.icon {
  width: 24px;
  height: 24px;
  fill: currentColor; /* 親要素のcolorを継承 */
  flex-shrink: 0;     /* Flexコンテナ内で潰れない */
}

/* サイズバリエーション */
.icon_size_sm { width: 16px; height: 16px; }
.icon_size_md { width: 24px; height: 24px; }
.icon_size_lg { width: 32px; height: 32px; }
.icon_size_xl { width: 48px; height: 48px; }

/* カラーバリエーション */
.icon_color_primary { color: #4f46e5; }
.icon_color_muted   { color: #94a3b8; }
.icon_color_danger  { color: #ef4444; }
.icon_color_success { color: #10b981; }
```

### Gulpでスプライトを自動生成する

```javascript
// gulpfile.js
const { src, dest } = require('gulp');
const svgSprite = require('gulp-svg-sprite');

function buildSvgSprite() {
  return src('src/icons/*.svg')
    .pipe(svgSprite({
      mode: {
        symbol: {
          sprite: '../icons.svg',
          dest: '.',
          render: false
        }
      }
    }))
    .pipe(dest('dist/images'));
}
```

---

## パターン2：Inline SVG

SVGをHTMLに直接埋め込む方法。アニメーションや複雑なCSSスタイリングが必要なケースに向く。

### 基本的な書き方

```html
<svg
  class="icon"
  viewBox="0 0 24 24"
  fill="none"
  stroke="currentColor"
  stroke-width="2"
  stroke-linecap="round"
  stroke-linejoin="round"
  aria-hidden="true"
>
  <path d="M3 9l9-7 9 7v11a2 2 0 01-2 2H5a2 2 0 01-2-2z"/>
  <polyline points="9,22 9,12 15,12 15,22"/>
</svg>
```

### アニメーション付きインラインSVG

```html
<!-- ハンバーガー → ×へのアニメーション -->
<button class="menu-btn" aria-expanded="false" aria-label="メニュー">
  <svg class="hamburger-icon" viewBox="0 0 24 24" aria-hidden="true">
    <line class="hamburger-line hamburger-line_top"
      x1="3" y1="6" x2="21" y2="6"/>
    <line class="hamburger-line hamburger-line_mid"
      x1="3" y1="12" x2="21" y2="12"/>
    <line class="hamburger-line hamburger-line_bot"
      x1="3" y1="18" x2="21" y2="18"/>
  </svg>
</button>
```

```css
.hamburger-icon {
  width: 24px;
  height: 24px;
}

.hamburger-line {
  stroke: currentColor;
  stroke-width: 2;
  stroke-linecap: round;
  transform-origin: center;
  transition: transform 0.3s ease, opacity 0.3s ease;
}

/* 開いた状態 */
.menu-btn[aria-expanded="true"] .hamburger-line_top {
  transform: rotate(45deg) translate(3px, 3px);
}

.menu-btn[aria-expanded="true"] .hamburger-line_mid {
  opacity: 0;
  transform: scaleX(0);
}

.menu-btn[aria-expanded="true"] .hamburger-line_bot {
  transform: rotate(-45deg) translate(3px, -3px);
}
```

---

## パターン3：CSS mask を使ったアイコン

外部SVGをCSSのマスクで使う方法。特にフレームワーク非依存のシンプルな実装に向く。

```css
/* CSS mask でSVGアイコンを表示 */
.icon-home {
  display: inline-block;
  width: 24px;
  height: 24px;
  background-color: currentColor;
  mask-image: url('/icons/home.svg');
  mask-size: contain;
  mask-repeat: no-repeat;
  mask-position: center;
  -webkit-mask-image: url('/icons/home.svg');
  -webkit-mask-size: contain;
  -webkit-mask-repeat: no-repeat;
  -webkit-mask-position: center;
}
```

```html
<!-- 使い方 -->
<span class="icon-home" aria-hidden="true"></span>
```

この方法の利点は `background-color` を変えるだけでアイコンの色が変わること。

---

## SVGの最適化

SVGファイルはデザインツールから書き出すと不要な属性が多く含まれる。SVGO で最適化する。

```bash
npm install -D svgo

# 単体ファイルの最適化
svgo input.svg -o output.svg

# ディレクトリ一括
svgo -f src/icons -o dist/icons
```

### gulpfileに組み込む

```javascript
const svgo = require('gulp-svgo');

function optimizeSvg() {
  return src('src/icons/*.svg')
    .pipe(svgo({
      plugins: [
        { removeViewBox: false },   // viewBoxは残す
        { removeUselessStrokeAndFill: false },
        { cleanupIds: false }
      ]
    }))
    .pipe(dest('dist/icons'));
}
```

---

## アクセシビリティのまとめ

```html
<!-- 装飾的なアイコン（スクリーンリーダーにスキップさせる） -->
<svg aria-hidden="true" focusable="false">...</svg>

<!-- 意味を持つアイコン（テキストなし） -->
<button aria-label="検索">
  <svg aria-hidden="true">...</svg>
</button>

<!-- ツールチップ代わりに title を使う -->
<svg role="img" aria-labelledby="icon-title">
  <title id="icon-title">ホームページに移動</title>
  <path d="..."/>
</svg>
```

---

## EJS テンプレートでの管理（静的サイト向け）

```ejs
<!-- /_includes/icon.ejs -->
<svg
  class="icon <%- className || '' %>"
  aria-hidden="true"
  focusable="false"
>
  <use href="/images/icons.svg#icon-<%- name %>"></use>
</svg>
```

```ejs
<!-- 使い方 -->
<%- include('/_includes/icon', { name: 'home', className: 'icon_size_lg' }) %>
<%- include('/_includes/icon', { name: 'search' }) %>
```

---

## まとめ

| 状況 | 推奨方式 |
|---|---|
| 静的サイト（Gulp + EJS） | SVG Sprite |
| アイコンが5個以下 | Inline SVG |
| 色変更が必要 | Sprite か Inline |
| 外部SVGをシンプルに | CSS mask |

アイコン管理で最もよく使うのはSVG Spriteだ。Gulpと組み合わせてビルド時に自動生成すると管理が楽になる。

[ToolShare Lab](https://webatives.com/) ではSVGアイコン関連のWeb制作ツールを提供している。フリーランス向けの各種計算ツールは [AND TOOLS](https://and-tools.net/) でまとめて利用できる。
