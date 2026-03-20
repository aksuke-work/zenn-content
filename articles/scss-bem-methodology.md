---
title: "BEM+SCSSの実践ガイド — 150コンポーネントを破綻なく管理する"
emoji: "🎨"
type: "tech"
topics: ["scss", "bem", "css", "gulp"]
published: false
---

[ToolShare Lab](https://webatives.com/) は150個以上のツールページを持つ静的サイトで、SCSSとBEMでスタイルを管理している。コンポーネント数が増えると命名の一貫性が崩れてきやすい。1年以上運用して確立したBEM+SCSSの実践的な設計方針をまとめる。

## BEMの命名規則（自分のdialect）

BEMにはいくつかの流派がある。[ToolShare Lab](https://webatives.com/) では以下のルールを使っている。

| 分類 | 規則 | 例 |
|------|------|---|
| Block | キャメルケース | `toolForm`, `breadcrumb`, `toolCard` |
| Element | `block__element`（サブ要素はキャメルケース） | `toolForm__inputGroup`, `toolCard__titleText` |
| Modifier | `block_key_value` / `block__element_key_value` | `toolForm_type_compact`, `toolCard__button_state_disabled` |

**使わない記法**:
- `block--modifier`（BEM公式の `--` 記法は使わない）
- `block__element__subelement`（Elementのネストは禁止）
- `block__element-sub`（ハイフン混在は禁止）

### Elementのサブ要素命名

```scss
// NG: Elementをさらに __ でネストする
.toolForm__input__label { }

// OK: サブ要素はキャメルケースで繋ぐ
.toolForm__inputLabel { }
.toolForm__inputGroup { }
.toolForm__inputGroupHeader { }
```

### Modifierの命名

```scss
// NG: キーなし
.toolCard_primary { }
.toolCard_disabled { }

// OK: キー名_値 形式
.toolCard_type_primary { }
.toolCard_state_disabled { }
.toolCard_size_large { }
```

状態は `state_*`、見た目のバリエーションは `type_*`、サイズは `size_*` で統一している。

## SCSSファイル構造

```
src/scss/
├── style.scss              # エントリーポイント
├── _variables.scss         # デザイントークン
├── _reset.scss             # リセットCSS
├── _base.scss              # bodyなど基本スタイル
├── _utilities.scss         # ユーティリティクラス
├── components/
│   ├── _toolForm.scss      # ツールフォーム
│   ├── _toolCard.scss      # ツール一覧カード
│   ├── _resultTable.scss   # 計算結果テーブル
│   ├── _header.scss        # グローバルヘッダー
│   ├── _footer.scss        # フッター
│   ├── _breadcrumb.scss    # パンくずリスト
│   ├── _modal.scss         # モーダル
│   └── _badge.scss         # バッジ
├── pages/
│   ├── _top.scss           # トップページ固有
│   └── _category.scss      # カテゴリページ固有
└── tools/
    ├── _income-tax.scss    # 所得税ツール固有（必要な場合のみ）
    └── ...
```

`style.scss` でimportを管理:

```scss
// style.scss

// Tailwind（ユーティリティ）
@import "tailwindcss";

// 基盤
@import "variables";
@import "reset";
@import "base";

// コンポーネント
@import "components/toolForm";
@import "components/toolCard";
@import "components/resultTable";
@import "components/header";
@import "components/footer";
@import "components/breadcrumb";
@import "components/modal";
@import "components/badge";

// ページ固有
@import "pages/top";
@import "pages/category";
```

## コンポーネントSCSSの書き方

### 基本パターン

```scss
// components/_toolForm.scss
.toolForm {
  background: var(--color-surface);
  border-radius: 8px;
  padding: 24px;

  // Elements
  &__title {
    font-size: 1.25rem;
    font-weight: 700;
    font-feature-settings: 'palt';
    margin-bottom: 16px;
  }

  &__inputGroup {
    display: flex;
    flex-direction: column;
    gap: 8px;
    margin-bottom: 20px;
  }

  &__inputLabel {
    font-size: 0.875rem;
    font-weight: 500;
    color: var(--color-text-secondary);
  }

  &__input {
    width: 100%;
    padding: 10px 12px;
    border: 1px solid var(--color-border);
    border-radius: 4px;
    font-size: 1rem;
    transition: border-color 0.2s ease;

    &:focus {
      outline: none;
      border-color: var(--color-primary);
    }
  }

  // Modifier: type
  &_type_compact {
    padding: 16px;

    .toolForm__inputGroup {
      margin-bottom: 12px;
    }
  }

  // Modifier: state（エラー状態）
  &__input_state_error {
    border-color: var(--color-error);
    background: var(--color-error-bg);
  }

  &__errorMessage {
    font-size: 0.75rem;
    color: var(--color-error);
    display: none;
  }

  &__errorMessage_state_visible {
    display: block;
  }
}
```

### Modifierの書き方2パターン

**パターンA: ブロック内にネストする**（推奨）

```scss
.toolCard {
  // base styles...

  &_type_featured {
    border: 2px solid var(--color-primary);
    background: var(--color-primary-light);
  }

  &_state_disabled {
    opacity: 0.5;
    pointer-events: none;
  }
}
```

**パターンB: ブロック外に書く**（Modifierが多い場合）

```scss
.toolCard { /* base */ }

// Modifierは下にまとめる
.toolCard_type_featured { /* ... */ }
.toolCard_type_compact { /* ... */ }
.toolCard_state_disabled { /* ... */ }
.toolCard_state_loading { /* ... */ }
```

[ToolShare Lab](https://webatives.com/) ではパターンAを基本にしつつ、Modifierが5つ以上になるコンポーネントはパターンBで書いている。

## デザイントークン（CSS Custom Properties）

変数ファイルでデザイントークンを定義:

```scss
// _variables.scss
:root {
  // Colors
  --color-primary: #3B82F6;
  --color-primary-light: #EFF6FF;
  --color-secondary: #10B981;
  --color-text: #1F2937;
  --color-text-secondary: #6B7280;
  --color-surface: #FFFFFF;
  --color-surface-secondary: #F9FAFB;
  --color-border: #E5E7EB;
  --color-error: #EF4444;
  --color-error-bg: #FEF2F2;

  // Spacing
  --space-xs: 4px;
  --space-sm: 8px;
  --space-md: 16px;
  --space-lg: 24px;
  --space-xl: 32px;
  --space-2xl: 48px;

  // Typography
  --font-size-xs: 0.75rem;
  --font-size-sm: 0.875rem;
  --font-size-base: 1rem;
  --font-size-lg: 1.125rem;
  --font-size-xl: 1.25rem;
  --font-size-2xl: 1.5rem;

  // Border Radius
  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 12px;

  // Transition
  --transition-fast: 0.15s ease;
  --transition-base: 0.2s ease;
  --transition-slow: 0.3s ease;
}
```

SCSSの `$variable` ではなくCSS Custom Propertiesを使う理由:
- JavaScript からも参照・変更できる（ダークモード切り替えなど）
- ブラウザのDevToolsで確認・上書きしやすい
- 子コンポーネントへのスコープ制御が可能

## JSとの連携: 状態クラスの付け方

Modifier で状態を表現するが、JSでクラスを付与するときの命名ルールも統一する。

```javascript
// NG: JSからスタイルを直接書く（禁止）
element.style.opacity = '0.5';
element.style.pointerEvents = 'none';

// OK: Modifierクラスをtoggleする
element.classList.add('toolCard_state_disabled');
element.classList.remove('toolCard_state_disabled');
element.classList.toggle('toolCard_state_loading');
```

JSフックが必要な場合は `js-` プレフィクスの別クラスを使い、スタイルには触れない:

```html
<button
  class="toolForm__submitButton js-calculate"
  data-tool="income-tax"
>
  計算する
</button>
```

```javascript
// js-* クラスでDOMを取得（スタイルクラスを直接触らない）
document.querySelector('.js-calculate').addEventListener('click', () => {
  // 計算処理...
});
```

## 150コンポーネントで破綻しないための運用ルール

### 1. コンポーネントの追加ルール

新しいコンポーネントを追加するとき、既存コンポーネントとの重複チェックを先にやる:

```bash
# 既存のBEMブロック一覧を確認
grep -r '^\.' src/scss/components/ | grep -v '&' | cut -d: -f2 | sort | uniq
```

### 2. 深いネストを禁止する

```scss
// NG: 深いネストは可読性が下がり、詳細度が高くなりすぎる
.toolCard {
  .toolCard__inner {
    .toolCard__title {
      span { }
    }
  }
}

// OK: フラットに書く
.toolCard { }
.toolCard__inner { }
.toolCard__title { }
.toolCard__titleText { }  // サブ要素はキャメルケースで繋ぐ
```

### 3. コンポーネント内にコンポーネントをネストしない

```scss
// NG: コンポーネントが別コンポーネントのスタイルを上書きする
.toolCard {
  .badge { margin-left: 8px; }  // .badgeコンポーネントの外部制御はNG
}

// OK: コンテキスト用のElementを使う
.toolCard {
  &__badge { margin-left: 8px; }  // toolCardのElementとして定義
}
```

## まとめ

150コンポーネントを破綻なく管理するための要点:

1. **命名規則を文書化する** → チームが変わっても一貫性を保てる
2. **Modifier は必ず `key_value` 形式** → `_disabled` だけでなく `_state_disabled`
3. **CSS Custom Propertiesでデザイントークン管理** → JSとの連携が容易
4. **JSフックは `js-` プレフィクス** → スタイルとロジックを分離
5. **深いネスト禁止、フラット設計** → 詳細度の問題を防ぐ

[ToolShare Lab](https://webatives.com/) ではこのルールを1年以上運用して、新しいツールページを追加するときの迷いがほぼなくなった。「どのクラス名にするか」「どのファイルに書くか」が自明になると、開発速度が上がる。
