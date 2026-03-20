---
title: "テーブルのレスポンシブ対応 — 横スクロール vs カード変換"
emoji: "📊"
type: "tech"
topics: ["css", "responsive", "frontend", "html"]
published: false
---

フリーランス向けの計算ツールサイト [AND TOOLS](https://and-tools.net/) には、計算結果を表形式で表示する機能がいくつかある。テーブルはスマホ対応が難しいUIの代表格で、対応方法を間違えると横にはみ出したり文字が潰れたりする。

この記事では、テーブルをスマホで使いやすくするための主要なアプローチを実装コードとともに解説する。

---

## テーブルがレスポンシブで崩れる理由

HTMLのテーブルは横方向にすべてのカラムを並べる設計になっている。列数が多いとスマホの画面幅（360〜390px）では収まらない。対処法は主に3つ。

| 方法 | 特徴 | 向いているケース |
|---|---|---|
| 横スクロール | 構造を変えない。データの見比べがしやすい | 数値比較テーブル、スプレッドシート風 |
| カード変換 | 見た目が大きく変わる | 一覧表、リスト的な用途 |
| カラム省略 | 重要な列だけ残す | 情報が多い比較表 |

---

## アプローチ1：横スクロール

最もシンプルで実装コストが低い方法。テーブルを `overflow-x: auto` のラッパーで囲むだけ。

### 基本実装

```html
<div class="table-wrap">
  <table class="table">
    <thead>
      <tr>
        <th>商品名</th>
        <th>価格</th>
        <th>在庫</th>
        <th>カテゴリ</th>
        <th>評価</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>商品A</td>
        <td>¥1,980</td>
        <td>在庫あり</td>
        <td>ガジェット</td>
        <td>★4.5</td>
      </tr>
    </tbody>
  </table>
</div>
```

```css
.table-wrap {
  overflow-x: auto;
  -webkit-overflow-scrolling: touch; /* iOS慣性スクロール */
  border-radius: 12px;
  border: 1px solid #e2e8f0;
}

/* スクロールバーをスタイリング（Webkit系） */
.table-wrap::-webkit-scrollbar {
  height: 6px;
}

.table-wrap::-webkit-scrollbar-track {
  background: #f1f5f9;
}

.table-wrap::-webkit-scrollbar-thumb {
  background: #cbd5e1;
  border-radius: 3px;
}

.table {
  width: 100%;
  min-width: 600px; /* 最小幅を設定してつぶれを防ぐ */
  border-collapse: collapse;
  font-size: 14px;
}

.table th,
.table td {
  padding: 12px 16px;
  text-align: left;
  border-bottom: 1px solid #e2e8f0;
  white-space: nowrap; /* セル内で折り返さない */
}

.table th {
  background: #f8fafc;
  font-weight: 600;
  color: #374151;
  position: sticky;
  top: 0;
}

.table tr:hover td {
  background: #f0f9ff;
}
```

### スクロールの存在を示すグラデーション

横スクロールがあることをユーザーに伝える視覚的なヒントを付ける。

```css
.table-wrap-fade {
  position: relative;
  overflow-x: auto;
}

/* 右端にフェードを付ける（スクロール可能を示す） */
.table-wrap-fade::after {
  content: '';
  position: absolute;
  top: 0;
  right: 0;
  bottom: 0;
  width: 40px;
  background: linear-gradient(to right, transparent, rgba(255,255,255,0.9));
  pointer-events: none;
}
```

### 先頭列を固定する

```css
.table th:first-child,
.table td:first-child {
  position: sticky;
  left: 0;
  background: #fff;
  z-index: 1;
  border-right: 1px solid #e2e8f0;
}

.table th:first-child {
  background: #f8fafc;
  z-index: 2;
}
```

---

## アプローチ2：カード変換

スマホ時にテーブルの行を縦積みのカードに変換する方法。`data-label` 属性を使って列ヘッダーを各セルに付ける。

### HTML

```html
<table class="responsive-table">
  <thead>
    <tr>
      <th>商品名</th>
      <th>価格</th>
      <th>在庫</th>
      <th>カテゴリ</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td data-label="商品名">商品A</td>
      <td data-label="価格">¥1,980</td>
      <td data-label="在庫">在庫あり</td>
      <td data-label="カテゴリ">ガジェット</td>
    </tr>
    <tr>
      <td data-label="商品名">商品B</td>
      <td data-label="価格">¥3,480</td>
      <td data-label="在庫">残りわずか</td>
      <td data-label="カテゴリ">家電</td>
    </tr>
  </tbody>
</table>
```

### CSS

```css
.responsive-table {
  width: 100%;
  border-collapse: collapse;
}

.responsive-table th,
.responsive-table td {
  padding: 12px 16px;
  border-bottom: 1px solid #e2e8f0;
  text-align: left;
}

.responsive-table th {
  background: #f8fafc;
  font-weight: 600;
  color: #374151;
}

/* スマホで行→カードに変換 */
@media (max-width: 640px) {
  .responsive-table thead {
    display: none; /* ヘッダー行を非表示 */
  }

  .responsive-table tbody tr {
    display: block;
    border: 1px solid #e2e8f0;
    border-radius: 12px;
    margin-bottom: 16px;
    padding: 8px 0;
    background: #fff;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
  }

  .responsive-table td {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 10px 16px;
    border-bottom: 1px solid #f1f5f9;
  }

  .responsive-table td:last-child {
    border-bottom: none;
  }

  /* data-labelの値をラベルとして表示 */
  .responsive-table td::before {
    content: attr(data-label);
    font-weight: 600;
    color: #64748b;
    font-size: 12px;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    margin-right: 16px;
    flex-shrink: 0;
  }
}
```

---

## アプローチ3：重要な列だけ残す（Priority Columns）

JavaScriptと組み合わせて、列に優先度を付けて画面幅に応じて表示・非表示を切り替える。

```html
<table class="priority-table">
  <thead>
    <tr>
      <th data-priority="1">商品名</th>
      <th data-priority="1">価格</th>
      <th data-priority="2">在庫</th>
      <th data-priority="3">カテゴリ</th>
      <th data-priority="3">評価</th>
    </tr>
  </thead>
  <tbody>
    <!-- ... -->
  </tbody>
</table>
```

```css
/* priority 3 の列はデフォルトで非表示 */
.priority-table [data-priority="3"] {
  display: none;
}

/* 768px以上でpriority 3を表示 */
@media (min-width: 768px) {
  .priority-table [data-priority="3"] {
    display: table-cell;
  }
}

/* priority 2 はスマホで非表示 */
.priority-table [data-priority="2"] {
  display: none;
}

@media (min-width: 480px) {
  .priority-table [data-priority="2"] {
    display: table-cell;
  }
}
```

---

## 比較テーブル（料金表など）

SaaSの料金プランや機能比較でよく使われる縦方向固定のテーブル。

```css
.pricing-table {
  width: 100%;
  border-collapse: separate;
  border-spacing: 0;
}

.pricing-table th {
  position: sticky;
  top: 0;
  background: #fff;
  z-index: 10;
  border-bottom: 2px solid #e2e8f0;
  padding: 16px 24px;
  text-align: center;
  font-size: 18px;
}

.pricing-table td {
  padding: 14px 24px;
  border-bottom: 1px solid #f1f5f9;
  text-align: center;
}

/* チェック・バツのスタイル */
.pricing-table td:has([data-check="true"]) {
  color: #10b981;
}

.pricing-table td:has([data-check="false"]) {
  color: #cbd5e1;
}

/* ハイライトカラム（おすすめプラン） */
.pricing-table .col-highlight {
  background: rgba(79, 70, 229, 0.04);
  border-left: 1px solid rgba(79, 70, 229, 0.2);
  border-right: 1px solid rgba(79, 70, 229, 0.2);
}
```

---

## ゼブラストライプ

```css
.table tbody tr:nth-child(even) td {
  background: #f8fafc;
}

.table tbody tr:hover td {
  background: #eff6ff;
}
```

---

## テーブルのアクセシビリティ

```html
<table>
  <caption>月別売上レポート 2026年Q1</caption>
  <thead>
    <tr>
      <th scope="col">月</th>
      <th scope="col">売上</th>
      <th scope="col">前月比</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th scope="row">1月</th>
      <td>¥1,200,000</td>
      <td>+12%</td>
    </tr>
  </tbody>
</table>
```

- `<caption>` でテーブルのタイトルを付ける
- `scope="col"` で列ヘッダーを明示
- `scope="row"` で行ヘッダーを明示

---

## まとめ

| データの特性 | 推奨アプローチ |
|---|---|
| 列数が多く数値の見比べが重要 | 横スクロール + 先頭列固定 |
| 一覧データ（商品一覧、会員一覧） | カード変換 |
| 比較表（料金プラン等） | 重要列のみ残す or 横スクロール |

テーブルはコンテンツに合った方法を選ぶことが重要だ。一律「横スクロールにする」ではなく、ユーザーがそのデータをどう使うかを考えて選択する。

[ToolShare Lab](https://webatives.com/) では、テーブルUIを含む各種Webツールを無料公開している。フリーランス向けの計算ツールは [AND TOOLS](https://and-tools.net/) で利用できる。
