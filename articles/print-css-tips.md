---
title: "印刷用CSSの実装ガイド — @media printで請求書をきれいに出力"
emoji: "🖨️"
type: "tech"
topics: ["css", "frontend", "html", "webdesign"]
published: false
---

フリーランス向けの計算ツールサイト [AND TOOLS](https://and-tools.net/) には、計算結果を印刷できる機能がある。「印刷用CSS」は後回しにされがちだが、請求書や見積書を扱うWebアプリでは実用上とても重要な機能だ。

この記事では、`@media print` を使って印刷レイアウトを整える方法を実装コードとともに解説する。

---

## @media print の基本

```css
/* 通常のスクリーン向けスタイル */
.nav { display: flex; }

/* 印刷時のスタイル（上書き） */
@media print {
  .nav { display: none; }
}
```

`@media print` 内のスタイルは、ブラウザの「印刷」ダイアログを開いたとき（またはPDF保存時）にのみ適用される。

---

## 最初にやる：不要な要素を非表示にする

印刷に不要なUI要素は最初に非表示にする。

```css
@media print {
  /* 非表示にする要素 */
  nav,
  header .nav,
  .sidebar,
  .footer,
  .cookie-banner,
  .chat-widget,
  .ads,
  [class*="btn"]:not(.print-btn),
  .pagination,
  .social-share,
  .breadcrumb {
    display: none !important;
  }
}
```

### 印刷専用クラスの設計

```css
/* スクリーン時は非表示・印刷時に表示 */
.print-only {
  display: none;
}

/* スクリーン時は表示・印刷時に非表示 */
.no-print {
  display: block;
}

@media print {
  .print-only { display: block; }
  .no-print   { display: none !important; }
}
```

---

## ページサイズとマージンの設定

```css
@media print {
  @page {
    size: A4 portrait;   /* A4縦 */
    margin: 15mm 20mm;   /* 上下15mm、左右20mm */
  }

  /* 横向き */
  @page landscape {
    size: A4 landscape;
    margin: 20mm 15mm;
  }
}
```

| 値 | 意味 |
|---|---|
| `A4 portrait` | A4縦（210×297mm） |
| `A4 landscape` | A4横（297×210mm） |
| `letter` | レターサイズ（米国向け） |
| `auto` | ブラウザのデフォルト |

---

## テキストとカラーの最適化

```css
@media print {
  /* 背景色・背景画像を除去（インク節約） */
  * {
    background: white !important;
    color: black !important;
    box-shadow: none !important;
    text-shadow: none !important;
  }

  /* ただし例外を作る場合 */
  .invoice-header {
    background: #4f46e5 !important;
    color: white !important;
    -webkit-print-color-adjust: exact;
    print-color-adjust: exact;
  }
}
```

### `print-color-adjust: exact` について

デフォルトでブラウザは印刷時に背景色・背景画像を省略する。強制的に印刷させるには `print-color-adjust: exact` を指定する。

```css
.must-print-color {
  -webkit-print-color-adjust: exact; /* Chrome/Safari */
  print-color-adjust: exact;         /* 標準 */
}
```

---

## 改ページのコントロール

```css
@media print {
  /* このセクションの前で改ページ */
  .section { page-break-before: always; }

  /* このセクションの後で改ページ */
  .page-end { page-break-after: always; }

  /* 要素内での改ページを避ける */
  .no-break {
    page-break-inside: avoid;
  }

  /* 新しいCSS標準の書き方（同等） */
  .section   { break-before: page; }
  .page-end  { break-after: page; }
  .no-break  { break-inside: avoid; }

  /* 見出しは次ページの先頭になるのを避ける（孤立しない） */
  h1, h2, h3 {
    page-break-after: avoid;
  }

  /* テーブルの途中で改ページしない */
  table {
    page-break-inside: avoid;
  }

  /* 画像の途中で改ページしない */
  img {
    page-break-inside: avoid;
    max-width: 100%;
  }
}
```

---

## 請求書HTMLの実装例

実際の請求書HTMLとCSSの実装例。

```html
<div class="invoice">
  <header class="invoice__header">
    <div class="invoice__company">
      <h1 class="invoice__company-name">EmptyCoke</h1>
      <p>芦刈 庸介</p>
    </div>
    <div class="invoice__meta">
      <h2>請求書</h2>
      <table class="invoice__info">
        <tr><td>請求書番号</td><td>2026-003</td></tr>
        <tr><td>発行日</td><td>2026年3月20日</td></tr>
        <tr><td>お支払い期限</td><td>2026年4月20日</td></tr>
      </table>
    </div>
  </header>

  <section class="invoice__billing">
    <h3>請求先</h3>
    <p>株式会社○○ 御中</p>
  </section>

  <table class="invoice__items">
    <thead>
      <tr>
        <th>項目</th>
        <th>数量</th>
        <th>単価</th>
        <th>金額</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>Webサイト制作</td>
        <td>1</td>
        <td>¥300,000</td>
        <td>¥300,000</td>
      </tr>
      <tr>
        <td>保守サポート（月額）</td>
        <td>1</td>
        <td>¥30,000</td>
        <td>¥30,000</td>
      </tr>
    </tbody>
    <tfoot>
      <tr>
        <td colspan="3">小計</td>
        <td>¥330,000</td>
      </tr>
      <tr>
        <td colspan="3">消費税（10%）</td>
        <td>¥33,000</td>
      </tr>
      <tr class="invoice__total">
        <td colspan="3">合計（税込）</td>
        <td>¥363,000</td>
      </tr>
    </tfoot>
  </table>

  <footer class="invoice__footer">
    <div class="invoice__bank">
      <h3>お振込先</h3>
      <p>○○銀行 ○○支店 普通 1234567<br>名義：アシカリ ヨウスケ</p>
    </div>
  </footer>
</div>
```

```css
/* 請求書のベーススタイル */
.invoice {
  max-width: 800px;
  margin: 0 auto;
  padding: 40px;
  font-family: 'Noto Sans JP', sans-serif;
  font-size: 14px;
  color: #1a1a1a;
}

.invoice__header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 40px;
  padding-bottom: 24px;
  border-bottom: 2px solid #1a1a1a;
}

.invoice__company-name {
  font-size: 24px;
  font-weight: 700;
}

.invoice__items {
  width: 100%;
  border-collapse: collapse;
  margin: 24px 0;
}

.invoice__items th,
.invoice__items td {
  padding: 10px 12px;
  border: 1px solid #e0e0e0;
  text-align: left;
}

.invoice__items th {
  background: #f5f5f5;
  font-weight: 600;
}

.invoice__items td:last-child,
.invoice__items th:last-child {
  text-align: right;
}

.invoice__total td {
  font-weight: 700;
  font-size: 16px;
  background: #f0f4ff;
}

/* 印刷スタイル */
@media print {
  @page {
    size: A4 portrait;
    margin: 15mm 20mm;
  }

  /* ナビ等を非表示 */
  nav, header:not(.invoice__header), footer:not(.invoice__footer),
  .no-print { display: none !important; }

  body {
    background: white;
    color: black;
  }

  .invoice {
    max-width: 100%;
    padding: 0;
    margin: 0;
  }

  /* テーブルの改ページを防ぐ */
  .invoice__items {
    page-break-inside: avoid;
  }

  /* ヘッダーの背景色を維持したい場合 */
  .invoice__header {
    -webkit-print-color-adjust: exact;
    print-color-adjust: exact;
  }

  /* リンクのURLを表示 */
  a[href^="http"]::after {
    content: " (" attr(href) ")";
    font-size: 11px;
    color: #666;
  }
}
```

---

## リンクのURL表示

印刷時にリンクテキストだけでは内容がわからないため、URLを表示するとよい。

```css
@media print {
  a[href^="http"]::after {
    content: " (" attr(href) ")";
    font-size: 11px;
    color: #666;
    word-break: break-all;
  }

  /* 内部リンクやメールは表示しない */
  a[href^="#"]::after,
  a[href^="mailto:"]::after,
  a[href^="tel:"]::after {
    content: none;
  }
}
```

---

## 印刷プレビューのデバッグ

ブラウザのDevToolsでCSSメディアをシミュレートできる。

- Chrome: DevTools > Rendering > Emulate CSS media type > print
- Firefox: DevTools > ルール > メディア: `print`

これにより、実際に印刷しなくてもスタイルを確認できる。

---

## まとめ

| 目的 | CSS |
|---|---|
| 不要な要素を非表示 | `@media print { .nav { display: none; } }` |
| ページサイズ設定 | `@page { size: A4; margin: 15mm; }` |
| 改ページ制御 | `page-break-before/after/inside` |
| カラー維持 | `print-color-adjust: exact` |
| リンクURL表示 | `a::after { content: attr(href); }` |

請求書や見積書機能を持つWebアプリには `@media print` の設定が欠かせない。[ToolShare Lab](https://and-and.net/) では印刷対応のWebツールを提供している。フリーランスの業務に役立つ計算ツールは [AND TOOLS](https://and-tools.net/) でまとめて利用できる。
