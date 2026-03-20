---
title: "コピペで使えるCSSアニメーション20選 — @keyframesパターン集"
emoji: "✨"
type: "tech"
topics: ["css", "animation", "frontend", "webdesign"]
published: false
---

フリーランス向け無料ツール集 [AND TOOLS](https://and-tools.net/) のUIに、さまざまなCSSアニメーションを実装してきた。JavaScriptなしで動くアニメーションはページ重量を増やさず、実装コストも低い。

この記事では、`@keyframes` を使ったCSSアニメーションパターン20選をまとめた。すべてコピペで使える。

---

## CSSアニメーションの基本

```css
@keyframes アニメーション名 {
  from { /* 開始状態 */ }
  to   { /* 終了状態 */ }
}

/* または */
@keyframes アニメーション名 {
  0%   { /* 開始 */ }
  50%  { /* 中間 */ }
  100% { /* 終了 */ }
}

.element {
  animation: アニメーション名 時間 タイミング関数 遅延 繰り返し 方向 fill-mode;
}
```

| プロパティ | 値の例 | 説明 |
|---|---|---|
| `animation-duration` | `0.3s`, `1s` | 1回の再生時間 |
| `animation-timing-function` | `ease`, `linear`, `cubic-bezier(...)` | 速度曲線 |
| `animation-delay` | `0.2s` | 開始遅延 |
| `animation-iteration-count` | `1`, `infinite` | 繰り返し回数 |
| `animation-direction` | `normal`, `alternate`, `reverse` | 再生方向 |
| `animation-fill-mode` | `forwards`, `backwards`, `both` | 開始前・終了後の状態 |

---

## フェード系

### 1. フェードイン

```css
@keyframes fadeIn {
  from { opacity: 0; }
  to   { opacity: 1; }
}

.fade-in {
  animation: fadeIn 0.4s ease forwards;
}
```

### 2. フェードインアップ（下から上へ）

```css
@keyframes fadeInUp {
  from {
    opacity: 0;
    transform: translateY(24px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.fade-in-up {
  animation: fadeInUp 0.5s ease forwards;
}
```

### 3. フェードインスケール

```css
@keyframes fadeInScale {
  from {
    opacity: 0;
    transform: scale(0.9);
  }
  to {
    opacity: 1;
    transform: scale(1);
  }
}

.fade-in-scale {
  animation: fadeInScale 0.4s cubic-bezier(0.34, 1.56, 0.64, 1) forwards;
}
```

### 4. ストークアップ（文字系のフェードイン）

複数要素に遅延を付けると文字が1つずつ現れる。

```css
@keyframes fadeInUp {
  from { opacity: 0; transform: translateY(16px); }
  to   { opacity: 1; transform: translateY(0); }
}

.stagger > * {
  opacity: 0;
  animation: fadeInUp 0.4s ease forwards;
}

.stagger > *:nth-child(1) { animation-delay: 0s; }
.stagger > *:nth-child(2) { animation-delay: 0.1s; }
.stagger > *:nth-child(3) { animation-delay: 0.2s; }
.stagger > *:nth-child(4) { animation-delay: 0.3s; }
```

---

## ローディング系

### 5. スピナー（回転）

```css
@keyframes spin {
  to { transform: rotate(360deg); }
}

.spinner {
  width: 24px;
  height: 24px;
  border: 2px solid rgba(0, 0, 0, 0.1);
  border-top-color: #4f46e5;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}
```

### 6. ドットローディング（3点）

```css
@keyframes dotBounce {
  0%, 80%, 100% { transform: scale(0.6); opacity: 0.5; }
  40% { transform: scale(1); opacity: 1; }
}

.dots {
  display: flex;
  gap: 6px;
}

.dots span {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: #4f46e5;
  animation: dotBounce 1.2s infinite;
}

.dots span:nth-child(2) { animation-delay: 0.2s; }
.dots span:nth-child(3) { animation-delay: 0.4s; }
```

### 7. プログレスバー（無限ローディング）

```css
@keyframes progressSlide {
  from { transform: translateX(-100%); }
  to   { transform: translateX(400%); }
}

.progress-bar {
  position: relative;
  height: 3px;
  background: #e5e7eb;
  overflow: hidden;
}

.progress-bar::after {
  content: '';
  position: absolute;
  height: 100%;
  width: 25%;
  background: #4f46e5;
  animation: progressSlide 1.5s ease infinite;
}
```

### 8. スケルトンシマー

```css
@keyframes shimmer {
  0%   { background-position: -200% 0; }
  100% { background-position: 200% 0; }
}

.skeleton {
  background: linear-gradient(
    90deg,
    #f0f0f0 25%,
    #e0e0e0 50%,
    #f0f0f0 75%
  );
  background-size: 200% 100%;
  animation: shimmer 1.5s infinite;
  border-radius: 4px;
}
```

---

## ボタン・インタラクション系

### 9. パルス（注目を促す）

```css
@keyframes pulse {
  0%   { box-shadow: 0 0 0 0 rgba(79, 70, 229, 0.4); }
  70%  { box-shadow: 0 0 0 12px rgba(79, 70, 229, 0); }
  100% { box-shadow: 0 0 0 0 rgba(79, 70, 229, 0); }
}

.pulse-btn {
  animation: pulse 2s infinite;
}
```

### 10. シェイク（エラー時）

```css
@keyframes shake {
  0%, 100% { transform: translateX(0); }
  10%, 50%, 90% { transform: translateX(-6px); }
  30%, 70% { transform: translateX(6px); }
}

.shake {
  animation: shake 0.5s ease;
}
```

### 11. バウンス

```css
@keyframes bounce {
  0%, 100% {
    transform: translateY(0);
    animation-timing-function: ease-out;
  }
  50% {
    transform: translateY(-16px);
    animation-timing-function: ease-in;
  }
}

.bounce {
  animation: bounce 1s infinite;
}
```

### 12. ウォブル（揺れる）

```css
@keyframes wobble {
  0%   { transform: rotate(0deg); }
  15%  { transform: rotate(-6deg); }
  30%  { transform: rotate(5deg); }
  45%  { transform: rotate(-4deg); }
  60%  { transform: rotate(3deg); }
  75%  { transform: rotate(-2deg); }
  100% { transform: rotate(0deg); }
}

.wobble:hover {
  animation: wobble 0.6s ease;
}
```

---

## 背景・装飾系

### 13. グラデーション流れ

```css
@keyframes gradientFlow {
  0%   { background-position: 0% 50%; }
  50%  { background-position: 100% 50%; }
  100% { background-position: 0% 50%; }
}

.gradient-flow {
  background: linear-gradient(
    -45deg,
    #667eea, #764ba2, #f093fb, #4facfe
  );
  background-size: 400% 400%;
  animation: gradientFlow 6s ease infinite;
}
```

### 14. フロート（浮かぶ）

```css
@keyframes float {
  0%, 100% { transform: translateY(0); }
  50%      { transform: translateY(-12px); }
}

.floating {
  animation: float 3s ease-in-out infinite;
}
```

### 15. 回転する円（デコレーション）

```css
@keyframes rotateSlow {
  to { transform: rotate(360deg); }
}

.rotating-circle {
  width: 200px;
  height: 200px;
  border-radius: 50%;
  border: 2px dashed rgba(79, 70, 229, 0.3);
  animation: rotateSlow 10s linear infinite;
}
```

### 16. タイピングカーソル

```css
@keyframes blink {
  0%, 100% { opacity: 1; }
  50%      { opacity: 0; }
}

.cursor::after {
  content: '|';
  animation: blink 1s step-end infinite;
  color: #4f46e5;
}
```

---

## 通知・フィードバック系

### 17. トースト（スライドイン）

```css
@keyframes slideInRight {
  from {
    transform: translateX(100%);
    opacity: 0;
  }
  to {
    transform: translateX(0);
    opacity: 1;
  }
}

@keyframes slideOutRight {
  from {
    transform: translateX(0);
    opacity: 1;
  }
  to {
    transform: translateX(100%);
    opacity: 0;
  }
}

.toast {
  animation: slideInRight 0.3s ease forwards;
}

.toast.removing {
  animation: slideOutRight 0.3s ease forwards;
}
```

### 18. モーダル開閉

```css
@keyframes modalOpen {
  from {
    opacity: 0;
    transform: scale(0.9) translateY(-20px);
  }
  to {
    opacity: 1;
    transform: scale(1) translateY(0);
  }
}

.modal {
  animation: modalOpen 0.3s cubic-bezier(0.34, 1.56, 0.64, 1) forwards;
}
```

### 19. チェックマーク描画

```css
@keyframes checkDraw {
  to { stroke-dashoffset: 0; }
}

.check-icon path {
  stroke-dasharray: 100;
  stroke-dashoffset: 100;
  animation: checkDraw 0.4s ease forwards 0.2s;
}
```

---

## スクロールアニメーション系

### 20. スクロール連動フェードイン（Intersection Observer と組み合わせ）

CSSアニメーションとJavaScriptのIntersection Observerを組み合わせると、スクロール時にアニメーションを発火できる。

```css
/* 初期状態 */
.reveal {
  opacity: 0;
  transform: translateY(32px);
  transition: opacity 0.6s ease, transform 0.6s ease;
}

/* クラスが付いたら発火 */
.reveal.is-visible {
  opacity: 1;
  transform: translateY(0);
}
```

```javascript
const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      entry.target.classList.add('is-visible');
      observer.unobserve(entry.target);
    }
  });
}, { threshold: 0.1 });

document.querySelectorAll('.reveal').forEach(el => observer.observe(el));
```

---

## パフォーマンスの鉄則

CSSアニメーションで重くなる主な原因と対策：

**1. `transform` と `opacity` だけアニメーションする**

`width`、`height`、`top`、`margin` などはレイアウトの再計算を引き起こす。

```css
/* NG */
.bad { transition: width 0.3s; }

/* OK */
.good { transition: transform 0.3s; }
```

**2. `will-change` を使いすぎない**

`will-change: transform` は別レイヤーに分離するため、メモリを消費する。本当に必要な要素だけに使う。

**3. `animation-fill-mode: backwards` を活用**

遅延付きアニメーションで初期状態がちらつく場合に使う。

```css
.fade-in {
  animation: fadeIn 0.4s ease 0.3s both;
  /* `both` = backwards + forwards */
}
```

---

## まとめ

CSSアニメーションは `@keyframes` と `animation` プロパティの組み合わせで作られる。`transform` と `opacity` のみを動かすことでパフォーマンスを保てる。

[ToolShare Lab](https://and-and.net/) ではCSSジェネレーター系のツールも提供している。また [AND TOOLS](https://and-tools.net/) では、フリーランスの実務に役立つ計算ツールを多数公開しているので、ブックマークしておくと便利だ。
