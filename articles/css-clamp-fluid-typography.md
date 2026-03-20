---
title: "CSS clamp()でレスポンシブデザインを劇的に楽にする"
emoji: "📐"
type: "tech"
topics: ["css", "responsive", "frontend", "webdesign"]
published: false
---

フリーランス向けの無料ツール集 [AND TOOLS](https://and-tools.net/) のUIを作るとき、フォントサイズや余白をレスポンシブ対応させるのに毎回メディアクエリを書いていた。しかし `clamp()` 関数を使い始めてから、そのほとんどが不要になった。

この記事では `clamp()` の基本から実際のデザインへの応用まで、コード中心で解説する。

---

## clamp()の基本構文

```css
property: clamp(最小値, 推奨値, 最大値);
```

| 引数 | 説明 |
|---|---|
| 最小値 | この値より小さくならない |
| 推奨値 | ビューポートに応じて変動する値（vw等を使う） |
| 最大値 | この値より大きくならない |

### 動作イメージ

```
ビューポート狭い   →  最小値が適用
ビューポート中間   →  推奨値が適用（滑らかにスケール）
ビューポート広い   →  最大値が適用
```

---

## フォントサイズに使う

### 従来のメディアクエリ方式（非効率）

```css
.title {
  font-size: 24px;
}

@media (min-width: 768px) {
  .title { font-size: 32px; }
}

@media (min-width: 1200px) {
  .title { font-size: 48px; }
}
```

### clamp()方式（1行で完結）

```css
.title {
  font-size: clamp(24px, 4vw, 48px);
}
```

320px幅では 24px、1200px幅では 48px になり、その間は滑らかにスケールする。

---

## clamp()の計算式：vwから px への変換

「375px で 16px、1440px で 24px にしたい」という要件をよく受ける。この計算を公式で出せる。

### 計算式

```
slope      = (max-px - min-px) / (max-viewport - min-viewport)
intercept  = min-px - slope * min-viewport

推奨値 = slope * 100vw + intercept
```

### 例：375pxで16px / 1440pxで24px

```
slope     = (24 - 16) / (1440 - 375) = 8 / 1065 ≈ 0.0075
intercept = 16 - 0.0075 * 375 ≈ 13.19

推奨値 = 0.75vw + 13.19px
```

```css
font-size: clamp(16px, 0.75vw + 13.19px, 24px);
```

この計算を自動化するSCSSミックスインも作れる。

```scss
@function fluid($min-px, $max-px, $min-vp: 375, $max-vp: 1440) {
  $slope: ($max-px - $min-px) / ($max-vp - $min-vp);
  $intercept: $min-px - $slope * $min-vp;
  $vw-part: $slope * 100;

  @return clamp(
    #{$min-px}px,
    #{$vw-part}vw + #{$intercept}px,
    #{$max-px}px
  );
}

/* 使い方 */
.title { font-size: fluid(24, 48); }
.body  { font-size: fluid(14, 18); }
```

---

## タイポグラフィスケール：実用的な設定例

```css
:root {
  /* フォントスケール（375px〜1440pxに対応） */
  --text-xs:   clamp(11px, 0.56vw + 8.9px, 12px);
  --text-sm:   clamp(13px, 0.56vw + 10.9px, 14px);
  --text-base: clamp(15px, 0.75vw + 12.2px, 16px);
  --text-lg:   clamp(17px, 1.03vw + 13.1px, 18px);
  --text-xl:   clamp(19px, 1.5vw + 13.4px, 24px);
  --text-2xl:  clamp(22px, 2.26vw + 13.5px, 30px);
  --text-3xl:  clamp(26px, 3.38vw + 13.3px, 40px);
  --text-4xl:  clamp(30px, 5.07vw + 11px,   56px);
  --text-5xl:  clamp(36px, 7.51vw + 7.8px,  72px);
}
```

---

## スペーシング（余白）に使う

メディアクエリを使わずにセクション余白を流動的にできる。

```css
:root {
  /* スペーシングスケール */
  --space-xs:  clamp(4px,  0.5vw + 2px,  8px);
  --space-sm:  clamp(8px,  1vw + 4px,   16px);
  --space-md:  clamp(16px, 2vw + 8px,   32px);
  --space-lg:  clamp(24px, 4vw + 8px,   56px);
  --space-xl:  clamp(40px, 6vw + 16px,  80px);
  --space-2xl: clamp(60px, 8vw + 20px, 120px);
}

/* 実際の使い方 */
.section     { padding-block: var(--space-2xl); }
.container   { padding-inline: var(--space-md); }
.card        { padding: var(--space-md); gap: var(--space-sm); }
```

---

## コンテナ幅に使う

```css
.container {
  width: clamp(320px, 90%, 1200px);
  margin-inline: auto;
}
```

`min()` / `max()` と組み合わせるパターンも：

```css
/* padding考慮版 */
.container {
  width: min(90%, 1200px);
  margin-inline: auto;
}
```

---

## グリッドカラム数を流動的に

`clamp()` を `grid-template-columns` の `repeat()` と組み合わせると、メディアクエリなしで列数が変わるグリッドを作れる。

```css
/* カードが280px以上を保ちながら自動で折り返す */
.card-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(clamp(200px, 30%, 280px), 1fr));
  gap: var(--space-md);
}
```

---

## ヒーローセクションの実用例

```css
.hero {
  padding-block: clamp(60px, 10vw, 140px);
}

.hero__title {
  font-size: clamp(32px, 6vw, 72px);
  line-height: 1.1;
  letter-spacing: clamp(-0.02em, -0.01vw, 0.01em);
}

.hero__subtitle {
  font-size: clamp(16px, 2vw, 22px);
  margin-top: clamp(16px, 2vw, 32px);
}

.hero__cta {
  margin-top: clamp(24px, 4vw, 48px);
  padding: clamp(12px, 1.5vw, 18px) clamp(24px, 3vw, 40px);
  font-size: clamp(14px, 1.5vw, 18px);
}
```

---

## line-height にも応用できる

画面が大きくなるほど `line-height` を少し広げると読みやすくなる。

```css
.body-text {
  font-size: clamp(15px, 1vw + 12px, 17px);
  line-height: clamp(1.6, 1.5 + 0.05vw, 1.75);
}
```

---

## border-radius をスケールさせる

```css
.card {
  border-radius: clamp(8px, 1.5vw, 16px);
}
```

小さい画面では角丸を小さく、大きい画面では少し丸みを持たせる。

---

## clamp()を使う際の注意点

### 1. 最小値と最大値を固定単位で書く

```css
/* OK：px（固定） */
font-size: clamp(16px, 2vw, 24px);

/* NG：remだと基準が変わる場合に混乱しやすい（意図的な場合を除く） */
font-size: clamp(1rem, 2vw, 1.5rem);
```

`rem` を使う場合は `html` の `font-size` が固定されている前提を明記しておく。

### 2. 推奨値が最小値と最大値の間に収まるか確認する

推奨値の `vw` 単位が大きすぎると最小値が適用される範囲が広くなりすぎる。

### 3. アクセシビリティ：ユーザーのフォントサイズ設定を尊重する

`px` で最小値を固定すると、ブラウザのフォントサイズ設定（例：大きくする）が無視される場合がある。アクセシビリティを重視する場合は `rem` を使う。

```css
/* アクセシビリティ重視版 */
font-size: clamp(1rem, 2vw, 1.5rem);
```

---

## ツールを使って計算を楽にする

`clamp()` の推奨値の計算は毎回手でやるのは非効率。[ToolShare Lab](https://webatives.com/) では、こういった計算補助ツールを提供している。フリーランスの業務効率化ツールは [AND TOOLS](https://and-tools.net/) でも複数公開しているので合わせて確認してほしい。

---

## まとめ

| 用途 | clamp()の使い方 |
|---|---|
| フォントサイズ | `clamp(最小px, vw + intercept, 最大px)` |
| 余白・パディング | `clamp(最小, 中間%, 最大)` |
| コンテナ幅 | `clamp(320px, 90%, 1200px)` |
| border-radius | `clamp(8px, 1.5vw, 16px)` |

`clamp()` を使いこなすと、メディアクエリの数を大幅に削減できる。「いくつかのブレークポイントで段階的に変える」より「なめらかに変化する」方が、ユーザー体験的にも優れている場面が多い。まずはフォントサイズと余白から導入してみることをおすすめする。
