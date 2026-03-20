---
title: "TailwindとSCSSのハイブリッド構成 — ユーティリティとBEMを両立"
emoji: "🎭"
type: "tech"
topics: ["tailwind", "scss", "bem", "gulp"]
published: false
---

[ToolShare Lab](https://webatives.com/) は Tailwind CSS と SCSS の両方を使っている。「どちらか一方でいいのでは」と思われがちだが、役割を分けると非常に相性がいい。150個のツールページで実際に運用しているハイブリッド構成を解説する。

## なぜ両方使うのか

### Tailwindだけでは辛い場面

```html
<!-- NG: 複雑なコンポーネントをTailwindだけで書くと... -->
<div class="bg-white rounded-lg shadow-md p-6 hover:shadow-lg transition-shadow duration-200 border border-gray-100 relative overflow-hidden before:absolute before:top-0 before:left-0 before:w-full before:h-1 before:bg-gradient-to-r before:from-blue-500 before:to-indigo-500">
  <label class="block text-sm font-medium text-gray-700 mb-1 font-feature-palt">
    年収（万円）
  </label>
  <input type="number" class="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent text-base transition-colors duration-150">
</div>
```

- クラスが長くなりすぎて読めない
- EJSのforループ内でこれが続くと地獄
- コンポーネントの「意味」が見えない

### SCSSだけでは辛い場面

```scss
// レイアウト調整のためだけにクラスを量産...
.mt-4 { margin-top: 16px; }
.mt-8 { margin-top: 32px; }
.flex { display: flex; }
.flex-col { flex-direction: column; }
.items-center { align-items: center; }
.gap-4 { gap: 16px; }
// → Tailwindを手書きしているだけ
```

### ハイブリッドにすると

| 役割 | 担当 |
|------|------|
| レイアウト（flex, grid, gap） | Tailwind |
| 余白（m-*, p-*, space-*） | Tailwind |
| レスポンシブ（sm:, md:, lg:） | Tailwind |
| テキスト（text-*, font-*, leading-*） | Tailwind |
| カスタムコンポーネント（toolForm, resultTable） | SCSS + BEM |
| 状態（hover, focus, active） | どちらでも |
| アニメーション（複雑なもの） | SCSS |

## Tailwind v4 + Gulp の設定

Tailwind CSS v4 は `@tailwindcss/postcss` 経由で使う。

```bash
npm install tailwindcss @tailwindcss/postcss
```

```javascript
// gulpfile.mjs
import sass from 'gulp-dart-sass';
import postcss from 'gulp-postcss';
import tailwindcss from '@tailwindcss/postcss';
import autoprefixer from 'autoprefixer';
import cssnano from 'cssnano';

function buildScss() {
  return src('src/scss/style.scss')
    .pipe(sass().on('error', sass.logError))
    .pipe(postcss([
      tailwindcss(),     // Tailwindを処理
      autoprefixer(),    // ベンダープレフィクス
      cssnano(),         // 圧縮
    ]))
    .pipe(dest('dist/css'));
}
```

```scss
// src/scss/style.scss

// Tailwind（全ユーティリティクラスを読み込む）
@import "tailwindcss";

// カスタムコンポーネント（BEM）
@import "components/toolForm";
@import "components/resultTable";
@import "components/toolCard";
@import "components/header";
@import "components/footer";
@import "components/breadcrumb";
```

## Tailwind v4のCSS設定

v4 では JavaScript の `tailwind.config.js` ではなく、CSSで設定する:

```css
/* src/scss/style.scss のTailwindセクション */
@import "tailwindcss";

@theme {
  /* カラーパレット */
  --color-primary: #3B82F6;
  --color-primary-50: #EFF6FF;
  --color-primary-700: #1D4ED8;
  --color-secondary: #10B981;
  --color-gray-50: #F9FAFB;
  --color-gray-900: #111827;

  /* フォントサイズ */
  --font-size-xs: 0.75rem;
  --font-size-sm: 0.875rem;
  --font-size-base: 1rem;
  --font-size-lg: 1.125rem;

  /* スペーシング */
  --spacing-18: 4.5rem;
  --spacing-22: 5.5rem;

  /* ブレークポイント */
  --breakpoint-sm: 640px;
  --breakpoint-md: 768px;
  --breakpoint-lg: 1024px;
  --breakpoint-xl: 1280px;
}
```

## EJSでの使い方

### レイアウトにはTailwind

```ejs
<!-- ツール一覧ページのレイアウト -->
<section class="py-12 md:py-20">
  <div class="max-w-6xl mx-auto px-4 sm:px-6">
    <h2 class="text-2xl font-bold text-gray-900 mb-8">
      フリーランス向け無料ツール
    </h2>

    <!-- グリッドはTailwindで -->
    <div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-6">
      <% tools.forEach(function(tool) { %>
        <!-- カードコンポーネントはBEMで -->
        <article class="toolCard">
          <a href="<%= tool.href %>" class="toolCard__link">
            <span class="toolCard__icon"><%= tool.icon %></span>
            <h3 class="toolCard__title"><%= tool.title %></h3>
            <p class="toolCard__description"><%= tool.description %></p>
          </a>
        </article>
      <% }) %>
    </div>
  </div>
</section>
```

### カスタムコンポーネントにはBEM

```ejs
<!-- ツールフォーム（複雑なコンポーネント）はBEMクラスで -->
<div class="toolForm">
  <div class="toolForm__inputGroup">
    <label for="income" class="toolForm__inputLabel">
      年収（万円）
    </label>
    <input
      type="number"
      id="income"
      class="toolForm__input"
      placeholder="例: 500"
    >
  </div>

  <div class="toolForm__inputGroup">
    <label class="toolForm__inputLabel">控除</label>
    <div class="toolForm__checkboxGroup">
      <label class="toolForm__checkboxLabel">
        <input type="checkbox" class="toolForm__checkbox js-deduction" value="hoken">
        社会保険料控除
      </label>
    </div>
  </div>

  <!-- ボタンのサイズ調整はTailwindで -->
  <button class="toolForm__submitButton w-full mt-6 js-calculate">
    計算する
  </button>
</div>
```

```scss
// BEM コンポーネント（SCSSで定義）
.toolForm {
  @apply bg-white rounded-xl shadow-sm border border-gray-100;
  padding: 24px;

  &__inputGroup {
    @apply flex flex-col gap-2 mb-5;
  }

  &__inputLabel {
    @apply text-sm font-medium text-gray-700;
    font-feature-settings: 'palt';
  }

  &__input {
    @apply w-full px-3 py-2.5 border border-gray-300 rounded-md text-base;
    @apply focus:outline-none focus:border-primary focus:ring-2 focus:ring-primary/20;
    @apply transition-colors duration-150;
  }

  &__submitButton {
    @apply bg-primary text-white font-medium rounded-lg py-3;
    @apply hover:bg-primary-700 transition-colors duration-150;
    @apply focus:outline-none focus:ring-2 focus:ring-primary/50;
  }

  &__submitButton_state_disabled {
    @apply opacity-50 cursor-not-allowed pointer-events-none;
  }
}
```

`@apply` を使うと Tailwind のユーティリティをSCSSコンポーネント内で使える。

## @apply の使いどころと注意点

**使う場面**:
- BEMコンポーネント内でTailwindのユーティリティを使いたいとき
- ホバー/フォーカス状態にTailwindの `transition`, `ring` を使うとき

**使わない場面**:
- EJSテンプレートで直接書いたほうが明確な場合
- PurgeCSS（Tailwindの未使用クラス削除）を妨げる形になるとき

```scss
// OK: コンポーネントの「構造」として意味があるもの
.resultTable {
  @apply w-full border-collapse;

  &__th {
    @apply bg-gray-50 text-sm font-medium text-gray-600 px-4 py-3 text-left;
  }

  &__td {
    @apply px-4 py-3 border-b border-gray-100 text-gray-900;
  }

  // 金額セルは右揃え + モノスペース
  &__td_type_amount {
    @apply text-right font-mono font-semibold text-primary;
  }
}

// NG: @applyで単一プロパティを繰り返す（直接書いたほうが早い）
.something {
  @apply flex;    // display: flex; と同じ
  @apply mt-4;   // margin-top: 16px; と同じ
}
```

## Tailwindだけにするタイミング、SCSSだけにするタイミング

**Tailwindだけ**（SCSSファイルを作らない）:
- レイアウト調整（余白、グリッド、flex）
- テキストスタイル（サイズ、ウェイト、カラー）
- ページ固有の1回しか使わない装飾

**SCSSコンポーネント**（BEMで管理）:
- 3ページ以上で使い回すコンポーネント
- ホバー・フォーカスなどのインタラクション定義
- `::before` / `::after` を使う複雑なスタイル
- カスタムアニメーション（`@keyframes`）

## まとめ

[ToolShare Lab](https://webatives.com/) でTailwind + SCSS ハイブリッドを1年以上運用した所感:

1. **レイアウト・余白・タイポグラフィは Tailwind** → ユーティリティの恩恵を最大に活用
2. **カスタムコンポーネントは SCSS + BEM** → 複雑なインタラクション・状態管理が明確
3. **`@apply` はコンポーネント内の「構造」に限定** → 乱用すると Tailwind の意味がなくなる
4. **BEM Modifier の状態制御は JS で className 切り替え** → Tailwind のダイナミッククラスより明確

「TailwindかSCSSか」という二択ではなく、役割分担が決まると両者の良いところを使い分けられる。150ツールのサイトでもこの設計で混乱なく管理できている。
