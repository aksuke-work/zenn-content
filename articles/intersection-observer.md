---
title: "Intersection Observerでスクロールアニメーションを実装する"
emoji: "🎬"
type: "tech"
topics: ["javascript", "css", "animation", "frontend"]
published: false
---

フリーランス向けの無料ツール集 [AND TOOLS](https://and-tools.net/) のLPやランディング的なページでは、スクロールに連動したアニメーションを使っている。以前は `window.addEventListener('scroll', ...)` とスクロール位置の計算で実装していたが、Intersection Observer APIを使うと、パフォーマンスが良くかつコードがシンプルになる。

この記事では Intersection Observer API の基本から実践的なアニメーション実装まで解説する。

---

## Intersection Observerとは

**Intersection Observer API** は、要素がビューポート（または指定した要素）と交差しているかを監視するブラウザのAPIだ。

従来の `scroll` イベントと違い：
- メインスレッドをブロックしない（非同期で動作）
- スロットリングが不要
- 複数の要素を効率よく監視できる

---

## 基本的な使い方

```javascript
// Observer の作成
const observer = new IntersectionObserver(
  (entries, observer) => {
    entries.forEach(entry => {
      if (entry.isIntersecting) {
        // 要素がビューポートに入ったとき
        console.log('表示された:', entry.target);
      }
    });
  },
  {
    root: null,          // null = ビューポートを基準
    rootMargin: '0px',   // 交差判定の余白
    threshold: 0.1       // 10%見えたら発火
  }
);

// 要素を監視開始
const target = document.querySelector('.animate-target');
observer.observe(target);

// 監視停止
observer.unobserve(target);

// 全監視停止
observer.disconnect();
```

---

## フェードインアニメーション（基本）

最も基本的な使い方。スクロールで要素が見えたらフェードインする。

```css
/* 初期状態（非表示） */
.reveal {
  opacity: 0;
  transform: translateY(32px);
  transition:
    opacity 0.6s ease,
    transform 0.6s ease;
}

/* 表示状態 */
.reveal.is-visible {
  opacity: 1;
  transform: translateY(0);
}
```

```javascript
const revealObserver = new IntersectionObserver(
  (entries) => {
    entries.forEach(entry => {
      if (entry.isIntersecting) {
        entry.target.classList.add('is-visible');
        revealObserver.unobserve(entry.target); // 1回だけ発火
      }
    });
  },
  { threshold: 0.15 }
);

document.querySelectorAll('.reveal').forEach(el => {
  revealObserver.observe(el);
});
```

---

## 遅延フェードイン（ずれて表示される）

子要素にインデックスを使って遅延させる。

```css
.reveal-stagger {
  opacity: 0;
  transform: translateY(24px);
  transition:
    opacity 0.5s ease,
    transform 0.5s ease;
  transition-delay: var(--delay, 0s);
}

.reveal-stagger.is-visible {
  opacity: 1;
  transform: translateY(0);
}
```

```javascript
// CSS変数で遅延を付ける
document.querySelectorAll('.stagger-parent').forEach(parent => {
  const children = parent.querySelectorAll('.reveal-stagger');

  children.forEach((child, index) => {
    child.style.setProperty('--delay', `${index * 0.1}s`);
  });
});

// Observer
const staggerObserver = new IntersectionObserver(
  (entries) => {
    entries.forEach(entry => {
      if (entry.isIntersecting) {
        entry.target.querySelectorAll('.reveal-stagger').forEach(el => {
          el.classList.add('is-visible');
        });
        staggerObserver.unobserve(entry.target);
      }
    });
  },
  { threshold: 0.1 }
);

document.querySelectorAll('.stagger-parent').forEach(el => {
  staggerObserver.observe(el);
});
```

---

## カウントアップアニメーション

数値が見えたときにカウントアップする。

```html
<span class="count-up" data-target="1234567" data-duration="2000">0</span>
```

```javascript
function countUp(el) {
  const target   = parseInt(el.dataset.target, 10);
  const duration = parseInt(el.dataset.duration, 10) || 2000;
  const start    = performance.now();

  function update(now) {
    const elapsed  = now - start;
    const progress = Math.min(elapsed / duration, 1);
    // easeOutQuart
    const ease     = 1 - Math.pow(1 - progress, 4);
    const current  = Math.floor(ease * target);

    el.textContent = current.toLocaleString('ja-JP');

    if (progress < 1) {
      requestAnimationFrame(update);
    } else {
      el.textContent = target.toLocaleString('ja-JP');
    }
  }

  requestAnimationFrame(update);
}

const countObserver = new IntersectionObserver(
  (entries) => {
    entries.forEach(entry => {
      if (entry.isIntersecting) {
        countUp(entry.target);
        countObserver.unobserve(entry.target);
      }
    });
  },
  { threshold: 0.5 }
);

document.querySelectorAll('.count-up').forEach(el => {
  countObserver.observe(el);
});
```

---

## 画像の遅延読み込み（Lazy Load）

```html
<img
  class="lazy-img"
  data-src="/images/photo.jpg"
  src="/images/placeholder.jpg"
  alt="説明"
  width="800"
  height="600"
>
```

```javascript
const lazyObserver = new IntersectionObserver(
  (entries) => {
    entries.forEach(entry => {
      if (entry.isIntersecting) {
        const img = entry.target;
        const src = img.dataset.src;

        if (src) {
          img.src = src;
          img.removeAttribute('data-src');
          img.classList.add('loaded');
        }

        lazyObserver.unobserve(img);
      }
    });
  },
  {
    rootMargin: '200px 0px', // 200px手前から読み込み開始
    threshold: 0
  }
);

document.querySelectorAll('img[data-src]').forEach(img => {
  lazyObserver.observe(img);
});
```

```css
.lazy-img {
  opacity: 0;
  transition: opacity 0.4s ease;
}

.lazy-img.loaded {
  opacity: 1;
}
```

---

## スクロール進捗バー

```html
<div class="scroll-progress-bar"></div>
```

```css
.scroll-progress-bar {
  position: fixed;
  top: 0;
  left: 0;
  height: 3px;
  background: #4f46e5;
  z-index: 9999;
  width: 0%;
  transition: width 0.1s linear;
}
```

これはIntersection Observerより `scroll` イベントが向いているケースだが、ドキュメント全体の進捗は `scroll` + `document.documentElement.scrollHeight` で計算する。

```javascript
const progressBar = document.querySelector('.scroll-progress-bar');

window.addEventListener('scroll', () => {
  const scrollTop    = window.scrollY;
  const docHeight    = document.documentElement.scrollHeight - window.innerHeight;
  const scrollPercent = (scrollTop / docHeight) * 100;

  progressBar.style.width = `${scrollPercent}%`;
}, { passive: true });
```

---

## セクションの切り替え（ナビのハイライト）

スクロール位置に応じてナビメニューのアクティブ項目を変える。

```javascript
const sections = document.querySelectorAll('section[id]');
const navLinks = document.querySelectorAll('nav a[href^="#"]');

const sectionObserver = new IntersectionObserver(
  (entries) => {
    entries.forEach(entry => {
      if (entry.isIntersecting) {
        const id = entry.target.id;

        navLinks.forEach(link => {
          link.classList.toggle(
            'is-active',
            link.getAttribute('href') === `#${id}`
          );
        });
      }
    });
  },
  {
    rootMargin: '-40% 0px -40% 0px', // セクションの中央付近で発火
    threshold: 0
  }
);

sections.forEach(section => {
  sectionObserver.observe(section);
});
```

---

## アクセシビリティ：モーションを減らす設定への対応

`prefers-reduced-motion` を確認して、アニメーションを無効にする。

```javascript
function createRevealObserver() {
  const prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;

  // モーション低減設定の場合はアニメーションをスキップ
  if (prefersReducedMotion) {
    document.querySelectorAll('.reveal').forEach(el => {
      el.classList.add('is-visible'); // 即座に表示
    });
    return;
  }

  const observer = new IntersectionObserver(
    (entries) => {
      entries.forEach(entry => {
        if (entry.isIntersecting) {
          entry.target.classList.add('is-visible');
          observer.unobserve(entry.target);
        }
      });
    },
    { threshold: 0.1 }
  );

  document.querySelectorAll('.reveal').forEach(el => observer.observe(el));
}

createRevealObserver();
```

---

## options の threshold の活用

`threshold` は0〜1の値（または配列）で、「何%見えたら発火するか」を設定する。

```javascript
// 0%でも発火（要素の端がビューポートに入った瞬間）
threshold: 0

// 50%見えたら発火
threshold: 0.5

// 0%, 25%, 50%, 75%, 100% それぞれで発火
threshold: [0, 0.25, 0.5, 0.75, 1]
```

複数の閾値を設定すると `entry.intersectionRatio` で現在の比率を取得できる。

---

## まとめ

| 用途 | 設定 |
|---|---|
| フェードイン | `threshold: 0.1`, 1回のみ発火 |
| 遅延読み込み | `rootMargin: '200px 0px'` |
| ナビハイライト | `rootMargin: '-40% 0px -40% 0px'` |
| カウントアップ | `threshold: 0.5` |

Intersection Observerは `scroll` イベントより効率的で、実装もシンプルになる。スクロールアニメーションが必要なすべてのプロジェクトで活用できる。

[ToolShare Lab](https://and-and.net/) ではJavaScriptのUI実装に役立つサンプルを公開している。フリーランス向けの各種計算ツールは [AND TOOLS](https://and-tools.net/) でまとめて確認できる。
