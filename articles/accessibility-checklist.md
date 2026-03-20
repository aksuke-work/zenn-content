---
title: "Webアクセシビリティチェックリスト — 最低限やるべき10項目"
emoji: "♿"
type: "tech"
topics: ["accessibility", "html", "css", "frontend"]
published: false
---

フリーランス向けの無料ツールサイト [AND TOOLS](https://and-tools.net/) を運営している。「アクセシビリティ（a11y）対応」は後回しにされやすいが、実は最低限の対応であれば実装コストは低く、SEO・UXの両方にプラスになる。

この記事では、Web制作者が今日からできる「最低限やるべきアクセシビリティ対応」10項目をまとめた。コードと確認方法もセットで解説する。

---

## なぜアクセシビリティが重要か

- 日本の障害者人口は約1,160万人（厚生労働省、2022年）
- 高齢者の増加によるニーズ拡大
- 音声読み上げや一時的な怪我（腕の骨折など）でも使うケースがある
- Googleはアクセシブルなサイトをポジティブに評価する傾向がある
- 法的リスクの低減（障害者差別解消法）

---

## チェック項目10選

### 1. コントラスト比を確保する

テキストと背景のコントラスト比を確認する。

| テキスト | 最低基準（AA） | 推奨（AAA） |
|---|---|---|
| 通常テキスト（18pt未満） | 4.5:1 | 7:1 |
| 大きなテキスト（18pt以上、または14pt太字） | 3:1 | 4.5:1 |

**確認方法**：[WebAIM Contrast Checker](https://webaim.org/resources/contrastchecker/) にカラーコードを入力。

**よくあるNG例**：

```css
/* NG: グレーテキストのコントラストが低い */
.muted { color: #aaaaaa; } /* 背景白のとき約2.3:1 → 不合格 */

/* OK: 少し濃いグレーに */
.muted { color: #767676; } /* 約4.5:1 → AA合格 */
```

### 2. キーボード操作できるようにする

すべてのインタラクティブ要素がキーボードだけで操作できることを確認する。

**確認方法**：Tabキーで全要素を順番に移動できるか確認。Enterキーでクリックできるか。

**よくあるNG例**：

```html
<!-- NG: divをクリックイベントだけで操作するボタン -->
<div onclick="doSomething()" class="fake-button">クリック</div>

<!-- OK: 本物のボタンを使う -->
<button onclick="doSomething()" class="btn">クリック</button>

<!-- OK: divをボタンとして使いたい場合 -->
<div
  role="button"
  tabindex="0"
  onclick="doSomething()"
  onkeydown="if(event.key==='Enter')doSomething()"
  class="custom-button"
>クリック</div>
```

### 3. フォーカスインジケーターを削除しない

多くのサイトで `outline: none` を全体に適用してフォーカスリングを消してしまっているが、これはキーボードユーザーにとって致命的。

```css
/* NG: フォーカスリングを完全に消す */
* { outline: none; }

/* OK: マウスユーザーには非表示、キーボードユーザーには表示 */
*:focus:not(:focus-visible) {
  outline: none;
}

*:focus-visible {
  outline: 2px solid #4f46e5;
  outline-offset: 2px;
  border-radius: 2px;
}

/* または box-shadow を使う（角丸に追従する） */
*:focus-visible {
  outline: none;
  box-shadow:
    0 0 0 2px #fff,
    0 0 0 4px #4f46e5;
}
```

### 4. imgに alt属性を付ける

```html
<!-- NG: alt属性なし -->
<img src="/images/logo.png">

<!-- OK: 説明的なaltテキスト -->
<img src="/images/logo.png" alt="AND TOOLS ロゴ">

<!-- OK: 装飾画像は空のaltで読み上げをスキップ -->
<img src="/images/decoration.png" alt="">

<!-- OK: アイコン（SVG）のアクセシビリティ -->
<svg aria-hidden="true" focusable="false">
  <!-- decorative icon -->
</svg>

<button>
  <svg aria-hidden="true">...</svg>
  <span>メニューを開く</span>
</button>
```

### 5. フォームのラベルを正しく設定する

```html
<!-- NG: プレースホルダーだけでラベルなし -->
<input type="email" placeholder="メールアドレス">

<!-- OK: label要素でラベルを関連付ける -->
<label for="email">メールアドレス</label>
<input id="email" type="email" placeholder="user@example.com">

<!-- OK: aria-labelを使う -->
<input
  type="search"
  aria-label="サイト内検索"
  placeholder="キーワードを入力"
>

<!-- OK: aria-labelledby を使う -->
<h2 id="contact-heading">お問い合わせ</h2>
<form aria-labelledby="contact-heading">
  <!-- フォーム内容 -->
</form>
```

### 6. エラーメッセージをスクリーンリーダーに伝える

```html
<!-- NG: エラーが視覚的にしか伝わらない -->
<input type="email" class="input-error">
<p class="error-text">メールアドレスが正しくありません</p>

<!-- OK: aria-describedby + aria-invalid で関連付ける -->
<input
  type="email"
  id="email"
  aria-invalid="true"
  aria-describedby="email-error"
>
<p id="email-error" role="alert">
  メールアドレスが正しくありません
</p>
```

```javascript
// フォームバリデーション時にaria属性を更新
function showError(inputEl, errorEl, message) {
  inputEl.setAttribute('aria-invalid', 'true');
  errorEl.textContent = message;
  errorEl.setAttribute('role', 'alert'); // スクリーンリーダーに即通知
}

function clearError(inputEl, errorEl) {
  inputEl.removeAttribute('aria-invalid');
  errorEl.textContent = '';
  errorEl.removeAttribute('role');
}
```

### 7. 見出し構造を正しくする

```html
<!-- NG: 見た目のためにh3をh1より前に使う -->
<h3>サブタイトル</h3>
<h1>メインタイトル</h1>

<!-- NG: 見出しをデザイン目的で使う -->
<h2 class="text-sm text-muted">キャプション</h2>

<!-- OK: 論理的な階層構造 -->
<h1>ページタイトル</h1>
  <h2>セクション1</h2>
    <h3>サブセクション1-1</h3>
  <h2>セクション2</h2>
```

見出しのスタイルは CSS で変えればよい。レベルは文書の論理構造で決める。

### 8. ランドマークを使う

```html
<!-- NG: divだけで構成 -->
<div class="header">...</div>
<div class="nav">...</div>
<div class="main">...</div>
<div class="footer">...</div>

<!-- OK: セマンティックな要素を使う -->
<header>
  <nav aria-label="グローバルナビゲーション">...</nav>
</header>
<main>
  <article>...</article>
  <aside aria-label="関連リンク">...</aside>
</main>
<footer>...</footer>
```

よく使うランドマーク要素：

| 要素 | ARIAロール | 用途 |
|---|---|---|
| `<header>` | `banner` | サイトのヘッダー |
| `<nav>` | `navigation` | ナビゲーション |
| `<main>` | `main` | メインコンテンツ |
| `<aside>` | `complementary` | サイドバー・補足 |
| `<footer>` | `contentinfo` | フッター |
| `<section>` | — | セクション（見出し必要） |

### 9. モーションアニメーションをオプトアウトできるようにする

```css
/* 通常のアニメーション */
.animated {
  animation: slideIn 0.5s ease;
  transition: transform 0.3s ease;
}

/* モーション低減設定のユーザーにはアニメーションを無効 */
@media (prefers-reduced-motion: reduce) {
  .animated {
    animation: none;
    transition: none;
  }

  /* 完全に無効にしたい場合 */
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

### 10. モーダル・ダイアログのフォーカストラップ

モーダルを開いたとき、フォーカスがモーダル外に出ないようにする。

```javascript
function trapFocus(modalEl) {
  const focusable = modalEl.querySelectorAll(
    'a, button, input, select, textarea, [tabindex]:not([tabindex="-1"])'
  );
  const firstEl = focusable[0];
  const lastEl  = focusable[focusable.length - 1];

  modalEl.addEventListener('keydown', (e) => {
    if (e.key !== 'Tab') return;

    if (e.shiftKey) {
      // Shift+Tab: 最初の要素のとき最後にジャンプ
      if (document.activeElement === firstEl) {
        e.preventDefault();
        lastEl.focus();
      }
    } else {
      // Tab: 最後の要素のとき最初にジャンプ
      if (document.activeElement === lastEl) {
        e.preventDefault();
        firstEl.focus();
      }
    }
  });

  // モーダルを開いたとき最初の要素にフォーカス
  firstEl.focus();
}

// ESCキーでモーダルを閉じる
document.addEventListener('keydown', (e) => {
  if (e.key === 'Escape' && modalIsOpen) {
    closeModal();
  }
});
```

---

## 確認ツール

| ツール | 用途 |
|---|---|
| axe DevTools（Chrome拡張） | 自動でアクセシビリティ問題を検出 |
| WAVE（Web Accessibility Evaluation Tool） | ビジュアルなアクセシビリティ確認 |
| Chrome DevTools > Accessibility | アクセシビリティツリーの確認 |
| VoiceOver（macOS） | 実際のスクリーンリーダーでテスト |

---

## まとめ

| 項目 | 作業量 | 効果 |
|---|---|---|
| コントラスト比の確認 | 小 | 大 |
| altテキスト | 小 | 大 |
| フォームラベル | 小 | 大 |
| フォーカスインジケーター | 小 | 大 |
| 見出し構造 | 小 | 中 |
| セマンティックHTML | 中 | 大 |
| キーボード操作 | 中〜大 | 大 |

まずは「コントラスト比の確認」「altテキスト」「フォームラベル」の3つから始めるとコスパが高い。

[ToolShare Lab](https://and-and.net/) ではアクセシビリティ対応のUIコンポーネントを公開している。フリーランスの業務効率化ツールは [AND TOOLS](https://and-tools.net/) でまとめて使える。

アクセシビリティは「特別な対応」ではなく「正しいHTMLを書くこと」から始まる。
