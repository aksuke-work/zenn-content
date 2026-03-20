---
title: "コピペで使えるbox-shadowパターン20選 — 2026年トレンド対応"
emoji: "🎨"
type: "tech"
topics: ["css", "webdesign", "frontend", "ui"]
published: false
---

フリーランス向けの無料ツールサイト [ToolShare Lab](https://webatives.com/) を運営している。CSS関連のツールを作る中で、box-shadowは「地味だけど奥が深い」プロパティだと実感した。単なるドロップシャドウにとどまらず、ニューモーフィズムやグロー効果まで、使いこなせば表現の幅が大きく広がる。

この記事では、2026年のWebデザイントレンドに合わせた `box-shadow` パターン20選をコードごとまとめた。全部コピペで使える。

---

## box-shadowの基本構文

まず基本を押さえる。

```css
box-shadow: offset-x offset-y blur-radius spread-radius color;

/* 複数指定（カンマ区切り） */
box-shadow:
  0 2px 4px rgba(0,0,0,0.1),
  0 8px 24px rgba(0,0,0,0.15);
```

| 値 | 説明 |
|---|---|
| `offset-x` | 水平方向のオフセット（正で右、負で左） |
| `offset-y` | 垂直方向のオフセット（正で下、負で上） |
| `blur-radius` | ぼかし半径（大きいほどぼける） |
| `spread-radius` | 広がり半径（正で広がる、負で縮む） |
| `color` | 影の色 |
| `inset` | キーワードを先頭に付けると内側の影になる |

---

## パターン1〜5：定番系

### 1. シンプルなドロップシャドウ

最も基本的な影。カード系UIに幅広く使える。

```css
.card {
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.12);
}
```

### 2. 二重シャドウ（マテリアルデザイン風）

近距離の鮮明な影と遠距離のソフトな影を重ねる。

```css
.card-material {
  box-shadow:
    0 1px 3px rgba(0, 0, 0, 0.12),
    0 4px 16px rgba(0, 0, 0, 0.08);
}
```

### 3. ホバー時に浮き上がるエフェクト

```css
.card-hover {
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  transition: box-shadow 0.3s ease, transform 0.3s ease;
}

.card-hover:hover {
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.15);
  transform: translateY(-4px);
}
```

### 4. 底面シャドウ（フローティングボタン風）

```css
.fab {
  box-shadow:
    0 4px 8px rgba(0, 0, 0, 0.2),
    0 2px 4px rgba(0, 0, 0, 0.12);
}
```

### 5. インナーシャドウ（押し込まれた入力欄）

```css
input {
  box-shadow: inset 0 2px 4px rgba(0, 0, 0, 0.1);
  border: 1px solid #e0e0e0;
}

input:focus {
  box-shadow:
    inset 0 2px 4px rgba(0, 0, 0, 0.06),
    0 0 0 3px rgba(66, 153, 225, 0.2);
  outline: none;
}
```

---

## パターン6〜10：カラーシャドウ系

### 6. カラーシャドウ（ブランドカラーで）

ブランドカラーをそのまま影に使う。ボタンに効果的。

```css
.btn-primary {
  background: #4f46e5;
  box-shadow: 0 4px 14px rgba(79, 70, 229, 0.4);
}

.btn-primary:hover {
  box-shadow: 0 6px 20px rgba(79, 70, 229, 0.5);
}
```

### 7. グロー効果（ネオン系）

ダークテーマで特に映える。

```css
.neon-blue {
  box-shadow:
    0 0 8px rgba(59, 130, 246, 0.6),
    0 0 24px rgba(59, 130, 246, 0.4),
    0 0 48px rgba(59, 130, 246, 0.2);
}

.neon-purple {
  box-shadow:
    0 0 8px rgba(168, 85, 247, 0.6),
    0 0 24px rgba(168, 85, 247, 0.4),
    0 0 48px rgba(168, 85, 247, 0.2);
}
```

### 8. グリーン成功状態

```css
.status-success {
  box-shadow: 0 4px 16px rgba(34, 197, 94, 0.3);
  border: 1px solid rgba(34, 197, 94, 0.4);
}
```

### 9. レッドエラー状態

```css
.status-error {
  box-shadow: 0 4px 16px rgba(239, 68, 68, 0.3);
  border: 1px solid rgba(239, 68, 68, 0.4);
}
```

### 10. アンバー警告状態

```css
.status-warning {
  box-shadow: 0 4px 16px rgba(245, 158, 11, 0.3);
  border: 1px solid rgba(245, 158, 11, 0.4);
}
```

---

## パターン11〜15：ニューモーフィズム系

ニューモーフィズム（ニューモーフィック）は、要素が背景から飛び出しているように見せるデザイン手法。明るい影と暗い影を両側に付けることで立体感を生む。

### 11. 凸（押し出し）ニューモーフィズム

```css
.neuro-raised {
  background: #e0e0e0;
  box-shadow:
    6px 6px 12px #bebebe,
    -6px -6px 12px #ffffff;
  border-radius: 12px;
}
```

### 12. 凹（押し込み）ニューモーフィズム

```css
.neuro-inset {
  background: #e0e0e0;
  box-shadow:
    inset 6px 6px 12px #bebebe,
    inset -6px -6px 12px #ffffff;
  border-radius: 12px;
}
```

### 13. ダークニューモーフィズム

```css
.neuro-dark {
  background: #1e1e2e;
  box-shadow:
    6px 6px 12px #16161e,
    -6px -6px 12px #26263e;
  border-radius: 12px;
}
```

### 14. ソフトニューモーフィズム（控えめ版）

```css
.neuro-soft {
  background: #f5f5f5;
  box-shadow:
    4px 4px 8px rgba(0, 0, 0, 0.08),
    -4px -4px 8px rgba(255, 255, 255, 0.9);
  border-radius: 8px;
}
```

### 15. ニューモーフィズムボタン（押下エフェクト付き）

```css
.neuro-btn {
  background: #e0e0e0;
  box-shadow:
    6px 6px 12px #bebebe,
    -6px -6px 12px #ffffff;
  border-radius: 8px;
  transition: box-shadow 0.15s ease;
  cursor: pointer;
  border: none;
}

.neuro-btn:active {
  box-shadow:
    inset 4px 4px 8px #bebebe,
    inset -4px -4px 8px #ffffff;
}
```

---

## パターン16〜20：2026年トレンド対応

### 16. レイヤードシャドウ（立体感重視）

複数のレイヤーを重ねて自然な影を作る。

```css
.layered-shadow {
  box-shadow:
    0 1px 2px rgba(0, 0, 0, 0.07),
    0 2px 4px rgba(0, 0, 0, 0.07),
    0 4px 8px rgba(0, 0, 0, 0.07),
    0 8px 16px rgba(0, 0, 0, 0.07),
    0 16px 32px rgba(0, 0, 0, 0.07);
}
```

### 17. シャープシャドウ（フラットデザイン系）

ぼかしなしのシャープな影。コミック・ポップ系デザインに。

```css
.sharp-shadow {
  box-shadow: 4px 4px 0 #000;
}

.sharp-shadow:hover {
  box-shadow: 6px 6px 0 #000;
  transform: translate(-2px, -2px);
  transition: all 0.2s ease;
}
```

### 18. グラデーションシャドウ（疑似実装）

CSSの `box-shadow` 自体はグラデーションをサポートしていないが、疑似要素を使って近似できる。

```css
.gradient-shadow-wrap {
  position: relative;
}

.gradient-shadow-wrap::after {
  content: '';
  position: absolute;
  inset: 0;
  border-radius: inherit;
  background: linear-gradient(135deg, #667eea, #764ba2);
  filter: blur(16px);
  opacity: 0.5;
  z-index: -1;
  transform: translateY(8px) scale(0.95);
}
```

### 19. フォーカスリング（アクセシビリティ対応）

アクセシビリティのためのフォーカスインジケーター。`outline` の代わりに `box-shadow` を使うと角丸に追従する。

```css
.btn:focus-visible {
  outline: none;
  box-shadow:
    0 0 0 2px #fff,
    0 0 0 4px #4f46e5;
}
```

### 20. アニメーションシャドウ（パルス効果）

```css
@keyframes shadow-pulse {
  0% {
    box-shadow: 0 0 0 0 rgba(79, 70, 229, 0.4);
  }
  70% {
    box-shadow: 0 0 0 16px rgba(79, 70, 229, 0);
  }
  100% {
    box-shadow: 0 0 0 0 rgba(79, 70, 229, 0);
  }
}

.pulse-btn {
  animation: shadow-pulse 2s infinite;
}
```

---

## CSS変数で管理するとメンテナンスが楽

案件ごとに影のトーンを統一したい場合、CSS変数（カスタムプロパティ）でシャドウセットを定義するとよい。

```css
:root {
  --shadow-sm: 0 1px 3px rgba(0, 0, 0, 0.1);
  --shadow-md: 0 4px 12px rgba(0, 0, 0, 0.1);
  --shadow-lg: 0 8px 32px rgba(0, 0, 0, 0.12);
  --shadow-xl: 0 16px 48px rgba(0, 0, 0, 0.15);

  --shadow-brand: 0 4px 14px rgba(79, 70, 229, 0.35);
  --shadow-focus: 0 0 0 2px #fff, 0 0 0 4px #4f46e5;
}

.card { box-shadow: var(--shadow-md); }
.btn:focus-visible { box-shadow: var(--shadow-focus); }
```

---

## よくあるミスと注意点

**1. `rgba` を使わずに `opacity` で色を薄めない**

`opacity` をかけると子要素ごと透明になる。影には必ず `rgba()` または `hsla()` を使う。

**2. 影のオーバーキル**

レイヤードシャドウは多用しすぎるとページ全体が重くなる。特にGPUアクセラレーションが効かないブラウザ環境で注意。

**3. ダークモードで影が見えなくなる**

ダークモード時は背景が暗いため、明るい影も有効活用する。

```css
@media (prefers-color-scheme: dark) {
  .card {
    box-shadow:
      0 4px 16px rgba(0, 0, 0, 0.4),
      0 0 0 1px rgba(255, 255, 255, 0.05);
  }
}
```

---

## まとめ

`box-shadow` は1行で書けるシンプルなプロパティだが、複数重ねたり `inset` を活用したりすることで表現の幅が一気に広がる。

実際のデザインで迷ったときは、まずこの20パターンから選んで使ってほしい。[ToolShare Lab](https://webatives.com/) では、こういった「コピペで即使えるツール・リファレンス」を無料で公開している。また、フリーランス向けの各種計算ツールは [AND TOOLS](https://and-tools.net/) でまとめて使える。

box-shadowのさらなる可能性は、MDN Web Docsの仕様を読むとより深く理解できる。特に `spread-radius` の負値と `inset` の組み合わせは、まだまだ使われていないテクニックが眠っている。
