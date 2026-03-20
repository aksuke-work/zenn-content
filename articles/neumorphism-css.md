---
title: "ニューモーフィズムをCSSで実装する — 凸・凹・フラットの3パターン"
emoji: "⬜"
type: "tech"
topics: ["css", "webdesign", "ui", "frontend"]
published: false
---

ニューモーフィズム（Neumorphism、またはソフトUI）は、2020年頃にデザイン界で大きな注目を集めたスタイルだ。フラットデザインの流れを汲みながら、まるで素材から押し出されたような凹凸感を持つ。

[AND TOOLS](https://and-tools.net/) のUIにニューモーフィズムを部分的に採用したとき、実装の細かいコツが意外に多いことに気づいた。この記事では、凸・凹・フラットの3パターンを中心に、実用的なCSS実装をまとめる。

---

## ニューモーフィズムの原理

ニューモーフィズムは `box-shadow` を使って実現する。ポイントは「**明るい影**と**暗い影**を両方付ける」こと。

```
要素の左上 → 明るい影（光源方向）
要素の右下 → 暗い影（光源の反対側）
```

これにより、要素が背景から「押し出されている」ように見える。

---

## 前提：背景色を設定する

ニューモーフィズムは **要素の背景色と親要素の背景色が同じ**であることが前提。

```css
:root {
  --nm-bg: #e0e5ec;
  --nm-shadow-light: rgba(255, 255, 255, 0.8);
  --nm-shadow-dark:  rgba(163, 177, 198, 0.6);
}

body {
  background: var(--nm-bg);
}
```

背景色が違うとニューモーフィズムの効果が出ない。この点がフラットデザインと大きく異なる制約だ。

---

## パターン1：凸（Raised）— 押し出し

最も基本的なニューモーフィズム。要素が背景から浮き出ているように見える。

```css
.nm-raised {
  background: var(--nm-bg);
  border-radius: 16px;
  box-shadow:
    6px 6px 12px var(--nm-shadow-dark),
    -6px -6px 12px var(--nm-shadow-light);
}
```

影のオフセットと半径でソフトさを調整：

```css
/* ソフト（大きな影） */
.nm-raised-soft {
  background: #e0e5ec;
  box-shadow:
    12px 12px 24px rgba(163, 177, 198, 0.6),
    -12px -12px 24px rgba(255, 255, 255, 0.8);
}

/* シャープ（小さな影） */
.nm-raised-sharp {
  background: #e0e5ec;
  box-shadow:
    4px 4px 8px rgba(163, 177, 198, 0.7),
    -4px -4px 8px rgba(255, 255, 255, 0.9);
}
```

---

## パターン2：凹（Inset）— 押し込み

`inset` キーワードを使って内側の影にする。入力欄やトグルスイッチに使いやすい。

```css
.nm-inset {
  background: var(--nm-bg);
  border-radius: 16px;
  box-shadow:
    inset 6px 6px 12px var(--nm-shadow-dark),
    inset -6px -6px 12px var(--nm-shadow-light);
}
```

---

## パターン3：フラット（Flat）— 平坦

影を片方だけ付けると、要素が浮いているが凹凸のない「フラット」な印象になる。

```css
.nm-flat {
  background: var(--nm-bg);
  border-radius: 16px;
  box-shadow: 4px 4px 8px rgba(163, 177, 198, 0.5);
}
```

---

## インタラクション：押下エフェクト

ボタンでよく使われる「凸→凹」の切り替えアニメーション。

```css
.nm-button {
  background: var(--nm-bg);
  border-radius: 12px;
  padding: 16px 32px;
  border: none;
  cursor: pointer;
  font-size: 16px;
  color: #6b7a8d;
  box-shadow:
    6px 6px 12px rgba(163, 177, 198, 0.6),
    -6px -6px 12px rgba(255, 255, 255, 0.8);
  transition: box-shadow 0.15s ease, transform 0.15s ease;
}

.nm-button:hover {
  box-shadow:
    8px 8px 16px rgba(163, 177, 198, 0.7),
    -8px -8px 16px rgba(255, 255, 255, 0.9);
}

.nm-button:active {
  box-shadow:
    inset 4px 4px 8px rgba(163, 177, 198, 0.6),
    inset -4px -4px 8px rgba(255, 255, 255, 0.8);
  transform: scale(0.98);
}
```

---

## トグルスイッチの実装

トグルは「凸（OFF）→凹（ON）」で状態を表現するのがニューモーフィズムらしい表現だ。

```html
<label class="nm-toggle">
  <input type="checkbox" class="nm-toggle__input">
  <span class="nm-toggle__track">
    <span class="nm-toggle__thumb"></span>
  </span>
</label>
```

```css
.nm-toggle {
  display: inline-flex;
  align-items: center;
  cursor: pointer;
}

.nm-toggle__input {
  display: none;
}

.nm-toggle__track {
  position: relative;
  width: 56px;
  height: 28px;
  border-radius: 14px;
  background: var(--nm-bg);
  box-shadow:
    inset 3px 3px 6px rgba(163, 177, 198, 0.6),
    inset -3px -3px 6px rgba(255, 255, 255, 0.8);
  transition: background 0.3s ease;
}

.nm-toggle__thumb {
  position: absolute;
  top: 4px;
  left: 4px;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: var(--nm-bg);
  box-shadow:
    2px 2px 5px rgba(163, 177, 198, 0.7),
    -2px -2px 5px rgba(255, 255, 255, 0.9);
  transition: transform 0.3s ease;
}

.nm-toggle__input:checked + .nm-toggle__track {
  background: #d1d9e6;
}

.nm-toggle__input:checked + .nm-toggle__track .nm-toggle__thumb {
  transform: translateX(28px);
  background: #4f46e5;
  box-shadow:
    2px 2px 5px rgba(79, 70, 229, 0.4),
    -2px -2px 5px rgba(255, 255, 255, 0.6);
}
```

---

## スライダーの実装

```css
.nm-slider {
  -webkit-appearance: none;
  appearance: none;
  width: 100%;
  height: 8px;
  border-radius: 4px;
  background: var(--nm-bg);
  box-shadow:
    inset 2px 2px 5px rgba(163, 177, 198, 0.6),
    inset -2px -2px 5px rgba(255, 255, 255, 0.8);
  outline: none;
  cursor: pointer;
}

.nm-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 24px;
  height: 24px;
  border-radius: 50%;
  background: var(--nm-bg);
  box-shadow:
    3px 3px 6px rgba(163, 177, 198, 0.6),
    -3px -3px 6px rgba(255, 255, 255, 0.8);
  cursor: pointer;
  transition: box-shadow 0.2s ease;
}

.nm-slider::-webkit-slider-thumb:active {
  box-shadow:
    inset 2px 2px 5px rgba(163, 177, 198, 0.6),
    inset -2px -2px 5px rgba(255, 255, 255, 0.8);
}
```

---

## カード型コンポーネント

```css
.nm-card {
  background: var(--nm-bg);
  border-radius: 20px;
  padding: 24px;
  box-shadow:
    8px 8px 16px rgba(163, 177, 198, 0.5),
    -8px -8px 16px rgba(255, 255, 255, 0.8);
}

/* カード内のアクセント数値 */
.nm-card__number {
  font-size: 2rem;
  font-weight: 700;
  color: #4f46e5;
  text-shadow: none; /* ニューモーフィズム背景ではtext-shadowは使わない */
}

/* カード内のアイコン（凹エリア） */
.nm-card__icon-wrap {
  width: 48px;
  height: 48px;
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow:
    inset 4px 4px 8px rgba(163, 177, 198, 0.5),
    inset -4px -4px 8px rgba(255, 255, 255, 0.8);
}
```

---

## ダークニューモーフィズム

暗い背景色でも同じ原理で実装できる。

```css
:root {
  --nm-dark-bg: #2d2d3a;
  --nm-dark-shadow-dark:  rgba(0, 0, 0, 0.4);
  --nm-dark-shadow-light: rgba(255, 255, 255, 0.05);
}

.nm-dark-raised {
  background: var(--nm-dark-bg);
  border-radius: 16px;
  box-shadow:
    6px 6px 12px var(--nm-dark-shadow-dark),
    -6px -6px 12px var(--nm-dark-shadow-light);
}

.nm-dark-inset {
  background: var(--nm-dark-bg);
  border-radius: 16px;
  box-shadow:
    inset 6px 6px 12px var(--nm-dark-shadow-dark),
    inset -6px -6px 12px var(--nm-dark-shadow-light);
}
```

---

## ニューモーフィズムの設計上の注意点

### アクセシビリティ問題

ニューモーフィズムはコントラスト比が取りにくい。影のみで凹凸を表現するため、背景との明度差が小さくなりがちで、視覚障害のあるユーザーには判別しにくい。

対策：
- ボタン等のインタラクティブ要素には必ずラベルを付ける
- フォーカス時は明確な色付きのアウトラインを付ける

```css
.nm-button:focus-visible {
  outline: 2px solid #4f46e5;
  outline-offset: 4px;
}
```

### 全面適用は避ける

テキスト主体のコンテンツや一覧UIにニューモーフィズムを全面適用すると、可読性が落ちる。「ダッシュボードのウィジェット」「コントロールパネル」など、パーツ的に使うのが効果的だ。

---

## まとめ

| パターン | box-shadow | 用途 |
|---|---|---|
| 凸（Raised） | 通常の二重影 | カード、ボタン |
| 凹（Inset） | inset二重影 | 入力欄、スライダー |
| フラット | 片側の影のみ | サブ要素、装飾 |

ニューモーフィズムは全面採用よりも、ダッシュボードのウィジェットやコントロールUIなどに部分的に使うと効果的だ。

[ToolShare Lab](https://and-and.net/) ではCSSシャドウジェネレーターなどのツールを公開している。日常業務向けの計算ツールは [AND TOOLS](https://and-tools.net/) にまとまっているので、あわせて活用してほしい。
