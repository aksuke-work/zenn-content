---
title: "CSS aspect-ratioで画像・動画のレイアウトを崩さない方法"
emoji: "📷"
type: "tech"
topics: ["css", "responsive", "frontend", "webdesign"]
published: false
---

フリーランス向けの無料ツールサイト [AND TOOLS](https://and-tools.net/) のUIを作るとき、画像のアスペクト比を維持しながらレスポンシブ対応する処理で詰まることがあった。以前は `padding-top` ハックで対応していたが、CSS の `aspect-ratio` プロパティが主要ブラウザでサポートされた今、よりシンプルに書ける。

この記事では、`aspect-ratio` の使い方と実際のユースケースをまとめる。

---

## aspect-ratioの基本

```css
.element {
  aspect-ratio: width / height;
}

/* 例 */
.square   { aspect-ratio: 1 / 1; }    /* 正方形 */
.widescreen { aspect-ratio: 16 / 9; } /* 横長 */
.portrait { aspect-ratio: 3 / 4; }    /* 縦長 */
.golden   { aspect-ratio: 1.618 / 1; } /* 黄金比 */
```

`aspect-ratio` は**幅が決まった状態で高さを自動計算**する（またはその逆）。

---

## 以前の方法との比較

### 従来：padding-top ハック

```css
/* 16:9の要素を作る（旧来の方法） */
.video-wrap {
  position: relative;
  width: 100%;
  padding-top: 56.25%; /* 9/16 × 100% */
}

.video-wrap iframe {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
}
```

### 現在：aspect-ratio を使う

```css
.video-wrap {
  aspect-ratio: 16 / 9;
  width: 100%;
}

.video-wrap iframe {
  width: 100%;
  height: 100%;
}
```

---

## 画像のアスペクト比を維持する

### object-fitと組み合わせる

`aspect-ratio` と `object-fit` を組み合わせると、どんなサイズの画像でも指定のアスペクト比で表示できる。

```css
.card-img {
  width: 100%;
  aspect-ratio: 16 / 9;
  object-fit: cover;     /* 比率を保ちながら切り抜き */
  border-radius: 8px;
}
```

```html
<img class="card-img" src="/images/photo.jpg" alt="説明">
```

### object-fitの値の違い

| 値 | 挙動 |
|---|---|
| `cover` | アスペクト比を保ちながら領域を埋める（切り抜き） |
| `contain` | アスペクト比を保ちながら全体を表示（余白が出る） |
| `fill` | 領域に引き伸ばす（歪む） |
| `none` | 原寸を維持 |
| `scale-down` | `none` か `contain` の小さい方を適用 |

---

## ユースケース別実装

### ブログカードのサムネイル（3:2）

```css
.blog-card__img {
  width: 100%;
  aspect-ratio: 3 / 2;
  object-fit: cover;
  border-radius: 8px 8px 0 0;
}
```

### サービス紹介カード（1:1 正方形）

```css
.service-card__icon {
  width: 64px;
  aspect-ratio: 1 / 1;
  border-radius: 12px;
  background: var(--color-primary-light);
  display: flex;
  align-items: center;
  justify-content: center;
}
```

### 商品画像（4:3）

```css
.product-img {
  width: 100%;
  aspect-ratio: 4 / 3;
  object-fit: contain;
  background: #f8fafc;
  border-radius: 8px;
  padding: 8px;
}
```

### ポートフォリオグリッド（混在サイズ対応）

```css
.portfolio-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 16px;
}

.portfolio-item img {
  width: 100%;
  aspect-ratio: 4 / 3;
  object-fit: cover;
  border-radius: 8px;
  transition: transform 0.3s ease;
}

.portfolio-item:hover img {
  transform: scale(1.03);
}
```

---

## 動画・iframeのレスポンシブ対応

YouTubeやVimeoの埋め込みは `aspect-ratio` を使うと簡潔に書ける。

```css
.video-container {
  width: 100%;
  aspect-ratio: 16 / 9;
}

.video-container iframe {
  width: 100%;
  height: 100%;
  border: none;
  border-radius: 12px;
}
```

```html
<div class="video-container">
  <iframe
    src="https://www.youtube.com/embed/xxxxx"
    allowfullscreen
    loading="lazy"
    title="動画タイトル"
  ></iframe>
</div>
```

---

## スケルトンローディングに使う

画像の読み込み前にプレースホルダーを表示する。`aspect-ratio` でサイズを確保してからシマーアニメーションを付ける。

```css
@keyframes shimmer {
  0%   { background-position: -200% 0; }
  100% { background-position: 200% 0; }
}

.skeleton-img {
  width: 100%;
  aspect-ratio: 16 / 9;
  background: linear-gradient(
    90deg,
    #f0f0f0 25%,
    #e0e0e0 50%,
    #f0f0f0 75%
  );
  background-size: 200% 100%;
  animation: shimmer 1.5s infinite;
  border-radius: 8px;
}
```

---

## 高さから幅を決める（逆算）

通常は幅から高さを決めるが、高さが先に決まっている場合は `height` を固定して幅を自動計算できる。

```css
.avatar {
  height: 48px;
  aspect-ratio: 1 / 1; /* 高さ48px → 幅も48px */
  border-radius: 50%;
  object-fit: cover;
}
```

---

## グリッドレイアウトとの組み合わせ

```css
.masonry-like {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-template-rows: auto;
  gap: 16px;
}

/* 特定のアイテムだけ縦長に */
.masonry-like .item_size_tall {
  grid-row: span 2;
}

.masonry-like img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}
```

---

## aspect-ratio に min-height / max-height を合わせる

`aspect-ratio` だけでは、コンテンツが少なすぎると領域が小さくなりすぎることがある。

```css
.hero-image {
  width: 100%;
  aspect-ratio: 16 / 9;
  min-height: 300px;
  max-height: 600px;
  object-fit: cover;
}
```

`aspect-ratio` と `min-height` が競合する場合、`min-height` が優先される（仕様通りの挙動）。

---

## CLSへの対応

画像のサイズをHTMLに明示することで、ブラウザがレイアウトを事前に確保できる。`aspect-ratio` プロパティは `width` と `height` 属性があれば自動的に計算されるため、HTML側の指定が重要。

```html
<!-- width/height属性を必ず指定する -->
<img
  src="/images/photo.jpg"
  alt="説明"
  width="1200"
  height="800"
  loading="lazy"
>
```

```css
/* CSS側でレスポンシブ対応 */
img {
  max-width: 100%;
  height: auto;
  /* ブラウザはwidth/height属性からaspect-ratioを自動計算する */
}
```

---

## ブラウザサポート

`aspect-ratio` は2021年以降の主要ブラウザでサポート済み（Chrome 88+, Firefox 89+, Safari 15+）。2026年現在、IE11のサポートが必要なケースを除き積極的に使える。

---

## まとめ

| 用途 | コード例 |
|---|---|
| 画像のアスペクト比維持 | `aspect-ratio: 16/9; object-fit: cover;` |
| YouTube埋め込み | `aspect-ratio: 16/9;` をコンテナに |
| 正方形のアイコン枠 | `aspect-ratio: 1/1;` |
| スケルトンUI | `aspect-ratio: 4/3;` + shimmerアニメーション |

`aspect-ratio` はシンプルな記述で堅牢なレイアウトを実現できる。`padding-top` ハックはもう使わないでいい。

[ToolShare Lab](https://and-and.net/) では、レイアウト設計に関するWeb制作ツールを公開している。フリーランスの業務に使える各種ツールは [AND TOOLS](https://and-tools.net/) でまとめて確認できる。
