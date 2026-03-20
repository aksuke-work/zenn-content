---
title: "CSS filterで画像エフェクト — blur/grayscale/hue-rotateの実用例"
emoji: "🖼️"
type: "tech"
topics: ["css", "frontend", "webdesign", "ui"]
published: false
---

フリーランス向けの無料ツールサイト [ToolShare Lab](https://webatives.com/) を運営している。サイトのUIで画像を使う機会が多く、CSSの `filter` プロパティを使いこなすと画像関連の表現が格段に増えることを実感した。

この記事では、`filter` プロパティの各値を実際の使いどころと合わせて解説する。コードはすべてコピペで使える。

---

## filter の基本

```css
.element {
  filter: 関数名(値) 関数名(値) ...;
}
```

複数の関数はスペース区切りで並べられ、左から順に適用される。

| 関数 | 効果 | 値の範囲 |
|---|---|---|
| `blur(px)` | ぼかし | 0以上 |
| `brightness(%)` | 明るさ | 0〜（1が元の明るさ） |
| `contrast(%)` | コントラスト | 0〜（1が元） |
| `grayscale(%)` | グレースケール | 0〜1（または0〜100%） |
| `sepia(%)` | セピア | 0〜1 |
| `saturate(%)` | 彩度 | 0〜（1が元） |
| `hue-rotate(deg)` | 色相回転 | 0〜360deg |
| `invert(%)` | 反転 | 0〜1 |
| `opacity(%)` | 透明度 | 0〜1 |
| `drop-shadow()` | 影 | box-shadowと同形式 |

---

## blur（ぼかし）

### 1. 背景ブラー（ガラスモーフィズム）

最もよく使われるのは `backdrop-filter` と組み合わせたガラス効果。

```css
.glass-card {
  background: rgba(255, 255, 255, 0.15);
  backdrop-filter: blur(12px);
  -webkit-backdrop-filter: blur(12px); /* Safari対応 */
  border: 1px solid rgba(255, 255, 255, 0.3);
  border-radius: 16px;
}
```

### 2. ホバー時のぼかし解除（フォーカス効果）

```css
.gallery-item img {
  filter: blur(4px);
  transition: filter 0.4s ease;
}

.gallery-item:hover img {
  filter: blur(0);
}
```

### 3. ローディング時のぼかし（Progressive Enhancement）

画像の読み込み中はぼかしを掛け、読み込み完了でシャープにする。

```css
.img-loading {
  filter: blur(20px);
  transition: filter 0.8s ease;
}

.img-loaded {
  filter: blur(0);
}
```

```javascript
const images = document.querySelectorAll('img');
images.forEach(img => {
  img.addEventListener('load', () => {
    img.classList.remove('img-loading');
    img.classList.add('img-loaded');
  });
  img.classList.add('img-loading');
});
```

### 4. ヒーロー画像のソフトフォーカス背景

```css
.hero {
  position: relative;
  overflow: hidden;
}

.hero__bg {
  position: absolute;
  inset: -20px; /* blur端のにじみ対策 */
  background: url('/images/hero.jpg') center / cover;
  filter: blur(8px) brightness(0.6);
  z-index: 0;
}

.hero__content {
  position: relative;
  z-index: 1;
}
```

---

## grayscale（グレースケール）

### 5. ホバーで色付き（グレー→カラー）

```css
.team-photo {
  filter: grayscale(100%);
  transition: filter 0.4s ease;
}

.team-photo:hover {
  filter: grayscale(0%);
}
```

### 6. 非アクティブ状態の表現

```css
.card_state_disabled {
  filter: grayscale(80%);
  opacity: 0.6;
  pointer-events: none;
}
```

### 7. 印刷時のグレースケール

```css
@media print {
  img {
    filter: grayscale(100%);
  }
}
```

---

## brightness / contrast（明るさ・コントラスト）

### 8. ホバー時のハイライト

```css
.card {
  transition: filter 0.2s ease;
}

.card:hover {
  filter: brightness(1.05);
}
```

### 9. テキストのオーバーレイを読みやすくする

暗い画像の上に白テキストを置く場合、画像を暗くすることで可読性を確保する。

```css
.hero__img {
  filter: brightness(0.5) contrast(1.1);
}
```

### 10. 夜間モード（画像のみ暗くする）

```css
@media (prefers-color-scheme: dark) {
  img:not([class*="no-dark"]) {
    filter: brightness(0.85) contrast(1.05);
  }
}
```

---

## sepia（セピア）

### 11. ヴィンテージ写真風

```css
.vintage {
  filter: sepia(0.8) contrast(1.1) brightness(0.9);
}
```

### 12. セピア → カラーへのホバー遷移

```css
.vintage-hover {
  filter: sepia(100%);
  transition: filter 0.5s ease;
}

.vintage-hover:hover {
  filter: sepia(0%);
}
```

---

## hue-rotate（色相回転）

色相を指定した角度だけ回転させる。同じ画像でも印象を大きく変えられる。

### 13. ブランドカラーに合わせたアイコン調整

アイコン素材が青色の場合、`hue-rotate` でブランドカラーに寄せられる。

```css
/* 青い素材を緑に */
.icon-green { filter: hue-rotate(120deg); }

/* 青い素材を赤に */
.icon-red   { filter: hue-rotate(240deg); }
```

### 14. アニメーティングカラーサイクル

```css
@keyframes colorCycle {
  0%   { filter: hue-rotate(0deg); }
  100% { filter: hue-rotate(360deg); }
}

.color-cycle {
  animation: colorCycle 4s linear infinite;
}
```

---

## saturate（彩度）

### 15. ホバーで彩度アップ

```css
.product-img {
  filter: saturate(0.8);
  transition: filter 0.3s ease;
}

.product-img:hover {
  filter: saturate(1.3);
}
```

---

## drop-shadow

`box-shadow` と違い、要素の実際の形状（透明部分を除く）に沿ってシャドウを付ける。PNGのアイコンやSVGに特に効果的。

### 16. PNGアイコンへの影

```css
/* box-shadowは四角形の影しか付けられない */
.icon-png {
  filter: drop-shadow(2px 4px 8px rgba(0, 0, 0, 0.3));
}

/* 透明部分を除いた形状に沿った影になる */
```

### 17. ホバー時のカラーグロー

```css
.icon-primary {
  filter: drop-shadow(0 0 0 transparent);
  transition: filter 0.3s ease;
}

.icon-primary:hover {
  filter: drop-shadow(0 0 8px rgba(79, 70, 229, 0.6));
}
```

---

## invert（反転）

### 18. ダークモードでの白黒反転

ダークモード時にロゴを反転させる方法。

```css
@media (prefers-color-scheme: dark) {
  .logo-mono {
    filter: invert(1);
  }
}
```

---

## 複数のフィルターを組み合わせる

### 19. フィルムカメラ風

```css
.film-look {
  filter:
    sepia(0.3)
    contrast(1.15)
    brightness(0.95)
    saturate(1.2);
}
```

### 20. グランジ・ノイズ効果（SVGフィルター連携）

CSSの `filter` プロパティからSVGフィルターも参照できる。

```html
<!-- SVGフィルター定義（非表示） -->
<svg style="display: none;">
  <defs>
    <filter id="noise">
      <feTurbulence
        type="fractalNoise"
        baseFrequency="0.65"
        numOctaves="3"
        stitchTiles="stitch"
      />
      <feColorMatrix type="saturate" values="0" />
      <feBlend in="SourceGraphic" mode="multiply" />
    </filter>
  </defs>
</svg>
```

```css
.noise-overlay {
  filter: url(#noise);
  opacity: 0.15;
}
```

---

## パフォーマンスの注意点

`filter` は描画コストが高い。特に `blur()` はGPU負荷が大きく、スクロール時にカクつく原因になりやすい。

```css
/* will-changeでGPUに分離 */
.glass-card {
  backdrop-filter: blur(12px);
  will-change: backdrop-filter; /* 必要な場合のみ */
}
```

アニメーションと組み合わせる場合は `transform` と `opacity` を優先し、`filter` のアニメーションは最小限に抑える。

---

## まとめ

| 用途 | filter 関数 |
|---|---|
| ガラスモーフィズム | `backdrop-filter: blur()` |
| グレー→カラーホバー | `grayscale(100%)` → `grayscale(0%)` |
| 画像を暗くして文字を読みやすく | `brightness(0.5)` |
| PNG/SVGへのシャドウ | `drop-shadow()` |
| ブランドカラー合わせ | `hue-rotate()` |

CSS `filter` は画像表現の可能性を大きく広げるプロパティだ。特に `grayscale` と `blur` のホバーエフェクトは実装コストが低くインパクトが大きい。

[ToolShare Lab](https://webatives.com/) ではCSSツールを各種無料公開している。フリーランス向けの計算ツールは [AND TOOLS](https://and-tools.net/) で確認できる。
