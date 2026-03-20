---
title: "コンテナクエリ入門 — メディアクエリの限界を超える"
emoji: "📦"
type: "tech"
topics: ["css", "responsive", "frontend", "webdesign"]
published: false
---

フリーランス向け無料ツール集 [AND TOOLS](https://and-tools.net/) のカードコンポーネントを設計しているとき、「このカードはサイドバーに置いたときと、メインカラムに置いたときで異なるレイアウトにしたい」という要件があった。

メディアクエリはビューポートの幅に基づくため、「コンポーネント自身の幅」に基づいてスタイルを変えることができない。これを解決するのが **CSS Container Queries（コンテナクエリ）** だ。

---

## コンテナクエリとは

コンテナクエリは、**親要素（コンテナ）のサイズに基づいてスタイルを変える**CSSの仕組み。メディアクエリはビューポート基準だが、コンテナクエリはコンポーネントの配置されたコンテキストに基づく。

2023年に主要ブラウザ（Chrome 105+, Firefox 110+, Safari 16+）でサポートされ、2026年現在は積極的に使えるようになった。

---

## 基本構文

```css
/* Step 1: コンテナを定義する */
.card-container {
  container-type: inline-size; /* 横幅に基づくコンテナ */
  container-name: card;        /* 名前（任意） */
}

/* 短縮形 */
.card-container {
  container: card / inline-size;
}

/* Step 2: コンテナクエリを書く */
@container card (min-width: 400px) {
  .card {
    flex-direction: row; /* コンテナが400px以上のとき */
  }
}
```

### container-type の値

| 値 | 説明 |
|---|---|
| `inline-size` | 横幅に基づくコンテナ（最もよく使う） |
| `size` | 横幅・縦幅両方に基づくコンテナ |
| `normal` | スタイルクエリのみ（サイズクエリ不可） |

---

## メディアクエリとの比較

### メディアクエリの問題

```css
/* ビューポートが大きいときにカードを横並びにしたい */
@media (min-width: 768px) {
  .card {
    flex-direction: row;
  }
}
```

問題：カードがサイドバー（幅300px）に置かれていても `flex-direction: row` になってしまう。

### コンテナクエリの解決

```css
/* カードのコンテナを定義 */
.card-wrapper {
  container-type: inline-size;
}

/* コンテナが400px以上のとき横並びに */
@container (min-width: 400px) {
  .card {
    flex-direction: row;
  }
}
```

カードをサイドバーに置けばコンテナ幅が狭いため縦並び、メインカラムに置けば横並びになる。**コンポーネントが自律的にレイアウトを決定できる。**

---

## 実装例：ニュースカードコンポーネント

### HTML

```html
<!-- メインカラム（広い） -->
<div class="card-container main-col">
  <div class="card">
    <img class="card__img" src="/images/news.jpg" alt="ニュース画像">
    <div class="card__body">
      <h3 class="card__title">見出しテキスト</h3>
      <p class="card__description">説明テキストが入ります。</p>
      <a class="card__link" href="#">続きを読む</a>
    </div>
  </div>
</div>

<!-- サイドバー（狭い） -->
<aside class="card-container sidebar">
  <div class="card">
    <img class="card__img" src="/images/news.jpg" alt="ニュース画像">
    <div class="card__body">
      <h3 class="card__title">見出しテキスト</h3>
      <p class="card__description">説明テキストが入ります。</p>
      <a class="card__link" href="#">続きを読む</a>
    </div>
  </div>
</aside>
```

### CSS

```css
/* コンテナ定義 */
.card-container {
  container-type: inline-size;
  container-name: news-card;
}

/* デフォルト：縦並び（狭いコンテナ用） */
.card {
  display: flex;
  flex-direction: column;
  gap: 16px;
  border-radius: 12px;
  overflow: hidden;
  border: 1px solid #e2e8f0;
}

.card__img {
  width: 100%;
  aspect-ratio: 16 / 9;
  object-fit: cover;
}

.card__body {
  padding: 16px;
}

.card__title {
  font-size: 16px;
  font-weight: 600;
  margin-bottom: 8px;
}

.card__description {
  font-size: 14px;
  color: #64748b;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

/* コンテナが400px以上：横並びレイアウト */
@container news-card (min-width: 400px) {
  .card {
    flex-direction: row;
    align-items: stretch;
  }

  .card__img {
    width: 200px;
    flex-shrink: 0;
    aspect-ratio: unset;
    height: auto;
  }

  .card__title {
    font-size: 18px;
  }
}

/* コンテナが600px以上：さらに大きなレイアウト */
@container news-card (min-width: 600px) {
  .card__img {
    width: 280px;
  }

  .card__title {
    font-size: 22px;
  }

  .card__description {
    -webkit-line-clamp: 3;
  }
}
```

---

## コンテナクエリ単位（cqi, cqb, cqw, cqh）

コンテナサイズに相対的な単位も使える。

| 単位 | 説明 |
|---|---|
| `cqw` | コンテナの幅の1% |
| `cqh` | コンテナの高さの1% |
| `cqi` | コンテナのインライン軸の1%（横書きでは幅） |
| `cqb` | コンテナのブロック軸の1%（横書きでは高さ） |
| `cqmin` | `cqi` と `cqb` の小さい方 |
| `cqmax` | `cqi` と `cqb` の大きい方 |

```css
@container (min-width: 300px) {
  .card__title {
    font-size: clamp(16px, 4cqi, 24px);
    /* コンテナ幅の4%、最小16px、最大24px */
  }
}
```

---

## スタイルクエリ（Style Queries）

コンテナのカスタムプロパティ（CSS変数）の値に基づいてスタイルを変える。2024〜2025年からChrome・Safariでサポート。

```css
.card-container {
  container-type: normal;
}

/* カスタムプロパティで状態を管理 */
.card-container_theme_dark {
  --card-theme: dark;
}

/* スタイルクエリ */
@container style(--card-theme: dark) {
  .card {
    background: #1e1e30;
    color: #e2e8f0;
    border-color: #2d3748;
  }
}
```

---

## ネストしたコンテナクエリ

コンテナはネストできる。内側のコンテナは直近の `container-type` を持つ祖先を参照する。

```css
.outer-container {
  container: outer / inline-size;
}

.inner-container {
  container: inner / inline-size;
}

/* innerコンテナを参照 */
@container inner (min-width: 300px) {
  .item { color: red; }
}

/* outerコンテナを参照 */
@container outer (min-width: 800px) {
  .item { font-size: 18px; }
}
```

---

## メディアクエリとコンテナクエリの使い分け

| ケース | 使うべきもの |
|---|---|
| ページ全体のレイアウト切り替え | メディアクエリ |
| カラム数の変更（1カラム↔2カラム） | メディアクエリ |
| コンポーネントの内部レイアウト | コンテナクエリ |
| 再利用するコンポーネント | コンテナクエリ |
| フォントサイズのスケール | `clamp()` |

実際のプロジェクトではメディアクエリとコンテナクエリを組み合わせる。

---

## まとめ

コンテナクエリは「コンポーネントが自分のコンテキストを知る」という考え方の転換をもたらす。再利用性の高いコンポーネント設計において、コンテナクエリはメディアクエリの限界を超える強力なツールだ。

導入ステップ：
1. コンポーネントのラッパーに `container-type: inline-size` を付ける
2. `@container (min-width: ...)` でスタイルを書く
3. `cqi` 単位でフォントサイズをスケールさせる

[ToolShare Lab](https://and-and.net/) では、レスポンシブデザインに関するコンポーネント実装ガイドも公開している。フリーランスの業務効率化ツールは [AND TOOLS](https://and-tools.net/) でまとめて利用できる。
