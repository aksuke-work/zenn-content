---
title: "アクセシブルなアコーディオンFAQの実装 — WAI-ARIA対応"
emoji: "♿"
type: "tech"
topics: ["javascript", "accessibility", "aria", "webdev"]
published: false
---

FAQ セクションのアコーディオンUIは多くのサイトで使われるが、アクセシビリティが疎かになりがち。[AND TOOLS](https://and-tools.net/) と [ToolShare Lab](https://webatives.com/) のFAQセクションで実装した、WAI-ARIA 準拠のアコーディオンと JSON-LD FAQPage の連携を解説する。

## 2つの実装方針 — ネイティブ vs カスタム

### 方針1: details/summary（ネイティブ）

HTML 標準の `<details>` / `<summary>` はアコーディオンの最もシンプルな実装。

```html
<details class="faq__item">
  <summary class="faq__question">確定申告は必要ですか？</summary>
  <div class="faq__answer">
    <p>副業の所得が年間20万円を超える場合は確定申告が必要です。</p>
  </div>
</details>
```

メリット:
- **JSゼロ**で動作する
- ブラウザが `aria-expanded` 相当の状態管理を自動で行う
- キーボード操作（Enter / Space）がデフォルトで動作

デメリット:
- **開閉アニメーションが付けにくい**（`open` 属性のトグルはCSSトランジションと相性が悪い）
- ブラウザ間でデフォルトスタイルが異なる
- 古いブラウザ（IE）では非対応

### 方針2: カスタム実装（WAI-ARIA）

アニメーションやスタイルの制御が必要な場合はカスタム実装が現実的。その場合、WAI-ARIA を正しく設定する。

```html
<div class="faq" role="list">
  <div class="faq__item" role="listitem">
    <button class="faq__question"
            id="faq-q1"
            aria-expanded="false"
            aria-controls="faq-a1">
      確定申告は必要ですか？
    </button>
    <div class="faq__answer"
         id="faq-a1"
         role="region"
         aria-labelledby="faq-q1"
         hidden>
      <p>副業の所得が年間20万円を超える場合は確定申告が必要です。</p>
    </div>
  </div>
</div>
```

ARIA属性の役割:

| 属性 | 要素 | 目的 |
|------|------|------|
| `aria-expanded` | button | 開閉状態をスクリーンリーダーに伝達 |
| `aria-controls` | button | この要素が制御する対象のID |
| `role="region"` | 回答div | セクションとしてランドマークに登録 |
| `aria-labelledby` | 回答div | ラベルとなる質問要素のID |
| `hidden` | 回答div | 閉じている状態をアクセシビリティツリーから除外 |

## JavaScript実装

```javascript
class Accordion {
  constructor(containerEl) {
    this.container = containerEl;
    this.items = [...containerEl.querySelectorAll('.faq__item')];
    this.allowMultiple = containerEl.dataset.allowMultiple === 'true';

    this.init();
  }

  init() {
    this.items.forEach(item => {
      const trigger = item.querySelector('.faq__question');
      const content = item.querySelector('.faq__answer');

      trigger.addEventListener('click', () => this.toggle(item));
      trigger.addEventListener('keydown', (e) => this.handleKeydown(e, item));
    });
  }

  toggle(item) {
    const trigger = item.querySelector('.faq__question');
    const content = item.querySelector('.faq__answer');
    const isOpen = trigger.getAttribute('aria-expanded') === 'true';

    if (!this.allowMultiple && !isOpen) {
      // 他のアイテムを閉じる（シングルモード）
      this.items.forEach(other => {
        if (other !== item) this.close(other);
      });
    }

    if (isOpen) {
      this.close(item);
    } else {
      this.open(item);
    }
  }

  open(item) {
    const trigger = item.querySelector('.faq__question');
    const content = item.querySelector('.faq__answer');

    trigger.setAttribute('aria-expanded', 'true');
    content.removeAttribute('hidden');

    // アニメーション: 高さを0からautoへ
    content.style.height = '0';
    content.style.overflow = 'hidden';

    requestAnimationFrame(() => {
      const targetHeight = content.scrollHeight;
      content.style.height = `${targetHeight}px`;

      content.addEventListener('transitionend', function handler() {
        content.style.height = '';
        content.style.overflow = '';
        content.removeEventListener('transitionend', handler);
      });
    });
  }

  close(item) {
    const trigger = item.querySelector('.faq__question');
    const content = item.querySelector('.faq__answer');

    trigger.setAttribute('aria-expanded', 'false');

    // アニメーション: 現在の高さから0へ
    content.style.height = `${content.scrollHeight}px`;
    content.style.overflow = 'hidden';

    requestAnimationFrame(() => {
      content.style.height = '0';

      content.addEventListener('transitionend', function handler() {
        content.setAttribute('hidden', '');
        content.style.height = '';
        content.style.overflow = '';
        content.removeEventListener('transitionend', handler);
      });
    });
  }

  handleKeydown(e, item) {
    const triggers = this.items.map(i => i.querySelector('.faq__question'));
    const currentIndex = triggers.indexOf(e.target);

    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault();
        const nextIndex = (currentIndex + 1) % triggers.length;
        triggers[nextIndex].focus();
        break;

      case 'ArrowUp':
        e.preventDefault();
        const prevIndex = (currentIndex - 1 + triggers.length) % triggers.length;
        triggers[prevIndex].focus();
        break;

      case 'Home':
        e.preventDefault();
        triggers[0].focus();
        break;

      case 'End':
        e.preventDefault();
        triggers[triggers.length - 1].focus();
        break;
    }
  }
}
```

### キーボード操作の仕様

WAI-ARIA Authoring Practices に準拠したキーボード操作:

| キー | 動作 |
|------|------|
| Enter / Space | アコーディオンの開閉 |
| ↓ | 次の質問にフォーカス移動 |
| ↑ | 前の質問にフォーカス移動 |
| Home | 最初の質問にフォーカス |
| End | 最後の質問にフォーカス |

## CSS — スムーズな開閉アニメーション

```css
.faq__question {
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 100%;
  padding: 16px 20px;
  border: none;
  background: #f8f9fa;
  font-size: 16px;
  font-weight: 600;
  font-feature-settings: 'palt';
  text-align: left;
  cursor: pointer;
  transition: background-color 0.2s ease;
}

.faq__question:hover {
  background: #e9ecef;
}

.faq__question:focus-visible {
  outline: 2px solid #2563eb;
  outline-offset: -2px;
}

/* 開閉アイコン */
.faq__question::after {
  content: '';
  width: 12px;
  height: 12px;
  border-right: 2px solid #666;
  border-bottom: 2px solid #666;
  transform: rotate(45deg);
  transition: transform 0.3s ease;
  flex-shrink: 0;
  margin-left: 16px;
}

.faq__question[aria-expanded="true"]::after {
  transform: rotate(-135deg);
}

/* 回答エリア */
.faq__answer {
  transition: height 0.3s ease;
}

.faq__answer > * {
  padding: 16px 20px;
}
```

`height: auto` へのCSS transitionは直接動かないため、JS側で `scrollHeight` を取得して明示的な高さを設定している。

## JSON-LD FAQPage との連携

Google検索でFAQリッチリザルトを表示するには、`FAQPage` 構造化データが必要。HTML上のFAQ要素から自動生成する。

```javascript
function generateFaqJsonLd(containerEl) {
  const items = containerEl.querySelectorAll('.faq__item');
  const faqEntries = [];

  items.forEach(item => {
    const question = item.querySelector('.faq__question')?.textContent?.trim();
    const answer = item.querySelector('.faq__answer')?.textContent?.trim();

    if (question && answer) {
      faqEntries.push({
        '@type': 'Question',
        name: question,
        acceptedAnswer: {
          '@type': 'Answer',
          text: answer,
        },
      });
    }
  });

  if (faqEntries.length === 0) return;

  const jsonLd = {
    '@context': 'https://schema.org',
    '@type': 'FAQPage',
    mainEntity: faqEntries,
  };

  const script = document.createElement('script');
  script.type = 'application/ld+json';
  script.textContent = JSON.stringify(jsonLd);
  document.head.appendChild(script);
}

// DOMContentLoadedで実行
document.addEventListener('DOMContentLoaded', () => {
  const faqContainer = document.querySelector('.faq');
  if (faqContainer) {
    new Accordion(faqContainer);
    generateFaqJsonLd(faqContainer);
  }
});
```

これにより、HTMLのFAQ要素を更新するだけでJSON-LDも自動的に更新される。構造化データの二重管理が不要になる。

## details/summary にアニメーションを付ける方法

ネイティブの `details/summary` でもアニメーションを付けたい場合の実装。

```javascript
function animateDetails(detailsEl) {
  const summary = detailsEl.querySelector('summary');
  const content = detailsEl.querySelector('.faq__answer');

  summary.addEventListener('click', (e) => {
    e.preventDefault();

    if (detailsEl.open) {
      // 閉じるアニメーション
      const startHeight = content.scrollHeight;
      content.style.height = `${startHeight}px`;
      content.style.overflow = 'hidden';

      requestAnimationFrame(() => {
        content.style.height = '0';
        content.addEventListener('transitionend', function handler() {
          detailsEl.open = false;
          content.style.height = '';
          content.style.overflow = '';
          content.removeEventListener('transitionend', handler);
        });
      });
    } else {
      // 開くアニメーション
      detailsEl.open = true;
      const targetHeight = content.scrollHeight;
      content.style.height = '0';
      content.style.overflow = 'hidden';

      requestAnimationFrame(() => {
        content.style.height = `${targetHeight}px`;
        content.addEventListener('transitionend', function handler() {
          content.style.height = '';
          content.style.overflow = '';
          content.removeEventListener('transitionend', handler);
        });
      });
    }
  });
}
```

ただし `details/summary` に JS でアニメーションを付けるなら、最初からカスタム実装にしたほうがコードはシンプルになる。

## まとめ

1. **シンプルなFAQなら `details/summary`** で十分。JSゼロで動作する
2. **アニメーションが必要ならカスタム実装**。WAI-ARIA を正しく設定すること
3. **`aria-expanded`、`aria-controls`、`role="region"`** の3点セットでアクセシビリティ確保
4. **キーボード操作**（矢印キー、Home/End）を忘れずに実装
5. **JSON-LD FAQPage** をDOMから自動生成すれば、構造化データの二重管理を回避

---

## 関連サイト

アコーディオンFAQの実装例はこちらで確認できます。

- [AND TOOLS](https://and-tools.net/) — 各ツールページのFAQセクションでアクセシブルなアコーディオンを使用
- [ToolShare Lab ガイド](https://webatives.com/guides/) — ツールの使い方ガイドにFAQセクションを設置
- [ToolShare Lab ツール一覧](https://webatives.com/tools/) — 150個以上の無料ツールを公開中
