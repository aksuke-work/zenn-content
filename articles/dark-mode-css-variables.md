---
title: "CSS変数だけでダークモードを実装する — JSなしの軽量版"
emoji: "🌙"
type: "tech"
topics: ["css", "darkmode", "frontend", "webdesign"]
published: false
---

フリーランス向けのツールサイト [AND TOOLS](https://and-tools.net/) にダークモード対応を追加するとき、最初はJavaScriptで `classList.toggle('dark')` を使っていた。しかし調べると、CSS変数と `prefers-color-scheme` メディアクエリだけで、OSの設定に自動追従するダークモードを実装できることがわかった。

JavaScriptなしの場合、以下のメリットがある。

- JavaScript読み込み前に正しい配色が適用される（フラッシュなし）
- JavaScript無効環境でも動作する
- コードがシンプルになる

この記事では、CSS変数を使ったダークモードの実装方法を、段階的に解説する。

---

## 基本：prefers-color-scheme

`prefers-color-scheme` はOSのカラーモード設定を検知するCSSメディアクエリ。

```css
@media (prefers-color-scheme: dark) {
  /* OSがダークモードのときに適用 */
}

@media (prefers-color-scheme: light) {
  /* OSがライトモードのときに適用（通常はデフォルト） */
}
```

---

## Step 1：カラートークンをCSS変数で定義する

ライトモードを `:root` に定義し、ダークモードをメディアクエリでオーバーライドする。

```css
/* ライトモード（デフォルト） */
:root {
  --color-bg:           #ffffff;
  --color-bg-muted:     #f8fafc;
  --color-bg-subtle:    #f1f5f9;
  --color-surface:      #ffffff;

  --color-text:         #1e293b;
  --color-text-muted:   #64748b;
  --color-text-subtle:  #94a3b8;

  --color-border:       #e2e8f0;
  --color-border-muted: #f1f5f9;

  --color-primary:      #4f46e5;
  --color-primary-hover:#4338ca;
  --color-primary-light:rgba(79, 70, 229, 0.1);

  --color-success:      #10b981;
  --color-warning:      #f59e0b;
  --color-error:        #ef4444;
}

/* ダークモード */
@media (prefers-color-scheme: dark) {
  :root {
    --color-bg:           #0f0f1a;
    --color-bg-muted:     #1a1a2e;
    --color-bg-subtle:    #16213e;
    --color-surface:      #1e1e30;

    --color-text:         #e2e8f0;
    --color-text-muted:   #94a3b8;
    --color-text-subtle:  #64748b;

    --color-border:       #2d3748;
    --color-border-muted: #1e293b;

    --color-primary:      #818cf8;
    --color-primary-hover:#6366f1;
    --color-primary-light:rgba(129, 140, 248, 0.15);

    --color-success:      #34d399;
    --color-warning:      #fbbf24;
    --color-error:        #f87171;
  }
}
```

---

## Step 2：コンポーネントでCSS変数を使う

色をハードコードせず、すべてCSS変数を参照する。

```css
/* 基本のスタイル */
body {
  background-color: var(--color-bg);
  color: var(--color-text);
  transition:
    background-color 0.3s ease,
    color 0.3s ease;
}

/* カード */
.card {
  background: var(--color-surface);
  border: 1px solid var(--color-border);
  border-radius: 12px;
  padding: 24px;
}

/* ボタン（プライマリー） */
.btn-primary {
  background: var(--color-primary);
  color: #fff;
  border: none;
  padding: 10px 24px;
  border-radius: 8px;
  cursor: pointer;
  transition: background 0.2s ease;
}

.btn-primary:hover {
  background: var(--color-primary-hover);
}

/* テキスト */
.text-muted { color: var(--color-text-muted); }
.text-subtle { color: var(--color-text-subtle); }

/* ボーダー */
hr { border-color: var(--color-border); }

/* 入力欄 */
input, textarea, select {
  background: var(--color-bg-subtle);
  border: 1px solid var(--color-border);
  color: var(--color-text);
  border-radius: 6px;
  padding: 10px 14px;
}

input::placeholder {
  color: var(--color-text-subtle);
}

input:focus {
  outline: none;
  border-color: var(--color-primary);
  box-shadow: 0 0 0 3px var(--color-primary-light);
}
```

---

## Step 3：コードブロックの配色

コードブロックはダークモードで特に差が出やすい。

```css
pre, code {
  background: var(--color-bg-muted);
  color: var(--color-text);
  border: 1px solid var(--color-border);
  border-radius: 8px;
}

pre {
  padding: 20px;
  overflow-x: auto;
}

code {
  padding: 2px 6px;
  font-size: 0.875em;
  border-radius: 4px;
}
```

---

## Step 4：画像のダークモード対応

ロゴやイラストが白背景前提で作られている場合、ダークモードでそのまま表示すると浮いて見えることがある。

```css
/* オプション1：フィルターで暗くする */
@media (prefers-color-scheme: dark) {
  .logo {
    filter: brightness(0.8);
  }
}

/* オプション2：ダークモード用の画像に差し替える */
.logo-light { display: block; }
.logo-dark  { display: none; }

@media (prefers-color-scheme: dark) {
  .logo-light { display: none; }
  .logo-dark  { display: block; }
}

/* オプション3：mix-blend-modeで馴染ませる */
@media (prefers-color-scheme: dark) {
  .logo {
    mix-blend-mode: luminosity;
  }
}
```

HTMLで `<picture>` を使う方法もある：

```html
<picture>
  <source
    srcset="/images/logo-dark.png"
    media="(prefers-color-scheme: dark)"
  />
  <img src="/images/logo-light.png" alt="ロゴ" />
</picture>
```

---

## Step 5：SVGのダークモード対応

インラインSVGは `fill: currentColor` と CSS変数の組み合わせで対応できる。

```css
.icon {
  fill: var(--color-text-muted);
  transition: fill 0.3s ease;
}

.icon:hover {
  fill: var(--color-primary);
}
```

```html
<svg class="icon" viewBox="0 0 24 24">
  <path d="..." />
</svg>
```

外部SVGファイルの場合は `color: currentColor` と CSS maskを使う：

```css
.icon-mask {
  --icon-url: url('/icons/star.svg');
  width: 24px;
  height: 24px;
  background-color: var(--color-text);
  mask-image: var(--icon-url);
  mask-size: contain;
  mask-repeat: no-repeat;
  mask-position: center;
  -webkit-mask-image: var(--icon-url);
  -webkit-mask-size: contain;
  -webkit-mask-repeat: no-repeat;
  -webkit-mask-position: center;
}
```

---

## トランジションをスムーズにする

ページロード時のカラーモード切り替えは、メディアクエリが適用されるタイミングで発生するのでフラッシュが起きない。ただし開発中に手動でダークモードをテストする際に見た目が突然変わると確認しにくい。

`transition` をbodyに設定すると滑らかになる：

```css
body {
  transition:
    background-color 0.4s ease,
    color 0.2s ease;
}

*,
*::before,
*::after {
  transition:
    background-color 0.4s ease,
    border-color 0.4s ease,
    box-shadow 0.4s ease;
}
```

ただし、トランジションを全要素に適用するとアニメーション性能に影響が出ることがある。必要最低限の要素に絞るほうが安全だ。

---

## ユーザーが手動で切り替えたい場合

「OSの設定に関係なく手動でダークモードを切り替えたい」という要件は多い。その場合はJavaScriptとHTMLの `data-theme` 属性を組み合わせる。

```css
/* CSS側 */
:root[data-theme="light"] {
  --color-bg:   #ffffff;
  --color-text: #1e293b;
}

:root[data-theme="dark"] {
  --color-bg:   #0f0f1a;
  --color-text: #e2e8f0;
}

/* OSの設定をデフォルトにしつつ、手動上書きを可能にする */
@media (prefers-color-scheme: dark) {
  :root:not([data-theme="light"]) {
    --color-bg:   #0f0f1a;
    --color-text: #e2e8f0;
  }
}
```

```javascript
const toggle = document.getElementById('theme-toggle');
const root = document.documentElement;

toggle.addEventListener('click', () => {
  const current = root.getAttribute('data-theme');
  const next = current === 'dark' ? 'light' : 'dark';
  root.setAttribute('data-theme', next);
  localStorage.setItem('theme', next);
});

// 初期化（ページ読み込み時）
const saved = localStorage.getItem('theme');
if (saved) root.setAttribute('data-theme', saved);
```

---

## チェックリスト

- [ ] すべての色をCSS変数で管理している
- [ ] コントラスト比（テキスト vs 背景）を確認した
- [ ] 画像・SVGのダークモード対応をした
- [ ] `placeholder`、`border`、`shadow` もCSS変数にした
- [ ] サードパーティのUI（Prism.js等）にダークテーマを適用した

---

## まとめ

CSS変数と `prefers-color-scheme` の組み合わせは、最もシンプルかつ信頼性の高いダークモード実装方法だ。JavaScriptを使わないので、スクリプトの読み込みより先にスタイルが適用される利点がある。

[ToolShare Lab](https://webatives.com/) では、カラートークンジェネレーターなどのWeb制作補助ツールを公開している。フリーランス業務に使える計算ツール全般は [AND TOOLS](https://and-tools.net/) にまとまっている。

ダークモード対応は「後からやる」より「最初からCSS変数で設計する」方が、後の工数を大幅に削減できる。
