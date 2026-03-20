---
title: "CSSグラデーション実践テクニック — linear/radial/conicの使い分け"
emoji: "🌈"
type: "tech"
topics: ["css", "webdesign", "frontend", "ui"]
published: false
---

フリーランス向けの無料ツールサイト [AND TOOLS](https://and-tools.net/) のUIを作る中で、CSSグラデーションを使う機会が増えた。`linear-gradient` だけでなく、`radial-gradient` や `conic-gradient` まで使いこなせると、デザインの表現力がぐっと上がる。

この記事では、3種類のグラデーション関数を実際のユースケースと合わせて解説する。コードはすべてコピペで使える。

---

## 3種類のグラデーション関数

| 関数 | 方向 | 主な用途 |
|---|---|---|
| `linear-gradient` | 直線方向 | 背景、ボタン、テキスト |
| `radial-gradient` | 円・楕円状に広がる | 光源効果、背景装飾 |
| `conic-gradient` | 中心点から回転 | 円グラフ、カラーホイール |

---

## linear-gradient（直線グラデーション）

### 基本構文

```css
background: linear-gradient(方向, 色1, 色2);

/* 角度指定 */
background: linear-gradient(135deg, #667eea, #764ba2);

/* 方向キーワード */
background: linear-gradient(to right, #f093fb, #f5576c);

/* 複数色 */
background: linear-gradient(135deg, #667eea 0%, #764ba2 50%, #f093fb 100%);
```

### よく使うパターン

**1. ヒーローセクションの背景**

```css
.hero {
  background: linear-gradient(135deg, #1a1a2e 0%, #16213e 50%, #0f3460 100%);
}
```

**2. テキストグラデーション**

```css
.gradient-text {
  background: linear-gradient(135deg, #667eea, #764ba2);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}
```

**3. グラデーションボーダー（疑似要素版）**

```css
.gradient-border {
  position: relative;
  border-radius: 12px;
}

.gradient-border::before {
  content: '';
  position: absolute;
  inset: -2px;
  border-radius: 14px;
  background: linear-gradient(135deg, #667eea, #764ba2);
  z-index: -1;
}
```

**4. フェードアウト（画像の下部を溶かす）**

```css
.fade-bottom::after {
  content: '';
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  height: 120px;
  background: linear-gradient(to bottom, transparent, #fff);
}
```

**5. ストライプパターン**

```css
.stripe {
  background: repeating-linear-gradient(
    45deg,
    #f0f0f0,
    #f0f0f0 10px,
    #e0e0e0 10px,
    #e0e0e0 20px
  );
}
```

**6. スケルトンローディング**

```css
@keyframes skeleton {
  0% { background-position: -200% 0; }
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
  animation: skeleton 1.5s infinite;
}
```

---

## radial-gradient（放射グラデーション）

中心点から外側に向かって広がるグラデーション。光源効果やスポットライト表現に適している。

### 基本構文

```css
background: radial-gradient(形状 サイズ at 位置, 色1, 色2);

/* 基本（円形） */
background: radial-gradient(circle, #667eea, #764ba2);

/* 楕円形 */
background: radial-gradient(ellipse, #667eea, #764ba2);

/* 中心位置を変える */
background: radial-gradient(circle at 30% 70%, #667eea, #764ba2);
```

### よく使うパターン

**7. スポットライト効果**

```css
.spotlight {
  background:
    radial-gradient(
      ellipse 80% 60% at 50% 0%,
      rgba(102, 126, 234, 0.3),
      transparent
    ),
    #0f0f1a;
}
```

**8. 光の玉（Blob）装飾**

```css
.blob {
  position: absolute;
  width: 400px;
  height: 400px;
  background: radial-gradient(
    circle,
    rgba(168, 85, 247, 0.4) 0%,
    rgba(168, 85, 247, 0) 70%
  );
  filter: blur(40px);
}
```

**9. ドット背景パターン**

```css
.dot-pattern {
  background:
    radial-gradient(circle, #d0d0d0 1px, transparent 1px);
  background-size: 20px 20px;
}
```

**10. グローイングカード**

```css
.glow-card {
  position: relative;
  overflow: hidden;
}

.glow-card::before {
  content: '';
  position: absolute;
  top: -50%;
  left: -50%;
  width: 200%;
  height: 200%;
  background: radial-gradient(
    circle at center,
    rgba(99, 102, 241, 0.15),
    transparent 60%
  );
  opacity: 0;
  transition: opacity 0.3s ease;
}

.glow-card:hover::before {
  opacity: 1;
}
```

**11. 複数の光源**

```css
.multi-light {
  background:
    radial-gradient(circle at 20% 80%, rgba(120, 119, 198, 0.3), transparent 50%),
    radial-gradient(circle at 80% 20%, rgba(255, 119, 115, 0.3), transparent 50%),
    radial-gradient(circle at 50% 50%, rgba(74, 222, 128, 0.2), transparent 60%),
    #0f0f1a;
}
```

---

## conic-gradient（コニックグラデーション）

中心点を軸に、角度に沿って色が変化するグラデーション。円グラフやカラーホイール、ゲーム風のUI表現などに使う。

### 基本構文

```css
background: conic-gradient(from 角度 at 位置, 色1 範囲, 色2 範囲);

/* 基本 */
background: conic-gradient(red, yellow, green, blue, red);

/* 開始角度を指定 */
background: conic-gradient(from 90deg, red, yellow, green);

/* 中心位置を変える */
background: conic-gradient(from 0deg at 30% 50%, red, blue);
```

### よく使うパターン

**12. 円グラフ（2分割）**

```css
/* 60%がblue, 残り40%がorange */
.pie-chart {
  width: 120px;
  height: 120px;
  border-radius: 50%;
  background: conic-gradient(
    #4f46e5 0% 60%,
    #f59e0b 60% 100%
  );
}
```

**13. 円グラフ（複数分割）**

```css
.pie-multi {
  width: 120px;
  height: 120px;
  border-radius: 50%;
  background: conic-gradient(
    #4f46e5 0% 40%,
    #10b981 40% 65%,
    #f59e0b 65% 85%,
    #ef4444 85% 100%
  );
}
```

**14. プログレスリング（CSS変数で動的に）**

```css
.progress-ring {
  --progress: 70; /* 0〜100 */
  width: 80px;
  height: 80px;
  border-radius: 50%;
  background: conic-gradient(
    #4f46e5 calc(var(--progress) * 1%),
    #e5e7eb calc(var(--progress) * 1%)
  );
}

/* 中心を白く塗ってドーナツに */
.progress-ring-donut {
  --progress: 70;
  width: 80px;
  height: 80px;
  border-radius: 50%;
  background:
    radial-gradient(circle, #fff 55%, transparent 55%),
    conic-gradient(
      #4f46e5 calc(var(--progress) * 1%),
      #e5e7eb calc(var(--progress) * 1%)
    );
}
```

**15. カラーホイール**

```css
.color-wheel {
  width: 200px;
  height: 200px;
  border-radius: 50%;
  background: conic-gradient(
    hsl(0, 100%, 50%),
    hsl(60, 100%, 50%),
    hsl(120, 100%, 50%),
    hsl(180, 100%, 50%),
    hsl(240, 100%, 50%),
    hsl(300, 100%, 50%),
    hsl(360, 100%, 50%)
  );
}
```

**16. オーロラ・メッシュグラデーション背景**

```css
.aurora {
  background:
    conic-gradient(from 230deg at 50% 50%, #4f46e5 0%, transparent 40%),
    conic-gradient(from 130deg at 30% 80%, #7c3aed 0%, transparent 40%),
    conic-gradient(from 320deg at 70% 20%, #06b6d4 0%, transparent 40%),
    #0f0f1a;
  filter: blur(60px);
}
```

---

## 応用：グラデーションを組み合わせる

複数のグラデーション関数はカンマで重ねられる。レイヤー順は上が優先される。

**17. メッシュグラデーション背景**

```css
.mesh-bg {
  background:
    radial-gradient(at 40% 20%, rgba(102, 126, 234, 0.5) 0%, transparent 50%),
    radial-gradient(at 80% 0%, rgba(120, 119, 198, 0.4) 0%, transparent 40%),
    radial-gradient(at 0% 50%, rgba(168, 85, 247, 0.3) 0%, transparent 50%),
    radial-gradient(at 80% 50%, rgba(99, 102, 241, 0.3) 0%, transparent 50%),
    radial-gradient(at 0% 100%, rgba(66, 153, 225, 0.4) 0%, transparent 50%),
    #f8fafc;
}
```

**18. グラデーション＋ノイズテクスチャ**

```css
/* SVGフィルターでノイズ生成 */
.noisy-gradient {
  background: linear-gradient(135deg, #667eea, #764ba2);
  position: relative;
}

.noisy-gradient::after {
  content: '';
  position: absolute;
  inset: 0;
  background-image: url("data:image/svg+xml,%3Csvg viewBox='0 0 256 256' xmlns='http://www.w3.org/2000/svg'%3E%3Cfilter id='noise'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.9' numOctaves='4' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23noise)' opacity='0.08'/%3E%3C/svg%3E");
  opacity: 0.15;
}
```

---

## パフォーマンス注意点

**filter: blur() を大きくしすぎない**

Blob系の装飾で `filter: blur(100px)` などを多用すると、GPU負荷が上がりスクロールがカクつく原因になる。`will-change: transform` を追加するか、`blur` の値を下げて対応する。

```css
/* 重い */
.blob { filter: blur(100px); }

/* 軽量化 */
.blob {
  filter: blur(60px);
  will-change: transform;
  transform: translateZ(0); /* GPU強制 */
}
```

---

## CSS変数でダークモード切り替え

```css
:root {
  --grad-primary: linear-gradient(135deg, #667eea, #764ba2);
  --grad-hero: linear-gradient(135deg, #1a1a2e, #16213e);
}

@media (prefers-color-scheme: dark) {
  :root {
    --grad-hero: linear-gradient(135deg, #0a0a1a, #0f0f2e);
  }
}
```

---

## まとめ

- `linear-gradient`：汎用。テキスト・背景・フェードに使い倒す
- `radial-gradient`：光源・Blob装飾・ドットパターンに
- `conic-gradient`：円グラフ・プログレスリング・メッシュ背景に

グラデーションは組み合わせることで無限に表現が広がる。[ToolShare Lab](https://webatives.com/) では、グラデーション生成ツールなどのWeb制作支援ツールを無料公開している。各種計算ツールは [AND TOOLS](https://and-tools.net/) でも確認してほしい。

ブラウザサポートは `conic-gradient` が最も遅く追従したが、2026年現在は主要ブラウザすべてでサポート済みなので、積極的に使っていきたい。
