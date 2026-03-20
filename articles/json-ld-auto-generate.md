---
title: "JSON-LD構造化データをJSで動的生成する — FAQPage+WebApplication"
emoji: "🏗️"
type: "tech"
topics: ["javascript", "seo", "json-ld", "webdev"]
published: false
---

構造化データ（JSON-LD）をHTMLに手書きするのは面倒だし、コンテンツとの不整合が起きやすい。DOMから情報を自動抽出して JSON-LD を動的に生成する仕組みを解説する。[AND TOOLS](https://and-tools.net/) と [ToolShare Lab](https://webatives.com/) で実際に使っている手法。

## なぜ動的生成なのか

JSON-LD を HTML に手書きする場合の問題:

1. **コンテンツの二重管理** — HTMLのテキストと JSON-LD の値を両方更新する必要がある
2. **不整合リスク** — 本文を変更して JSON-LD を更新し忘れるとエラーになる
3. **150ページ分の管理コスト** — ツールサイトのように大量のページがあると手動管理は破綻する

動的生成ならDOMが唯一の情報源（Single Source of Truth）になる。HTML を更新すれば JSON-LD も自動的に追従する。

## FAQPage スキーマの自動生成

FAQセクションのDOMから `FAQPage` 構造化データを抽出する。

```javascript
function generateFaqSchema(containerSelector) {
  const container = document.querySelector(containerSelector);
  if (!container) return null;

  const items = container.querySelectorAll('.faq__item');
  if (items.length === 0) return null;

  const mainEntity = [];

  items.forEach(item => {
    const questionEl = item.querySelector('.faq__question');
    const answerEl = item.querySelector('.faq__answer');

    if (!questionEl || !answerEl) return;

    const question = questionEl.textContent.trim();
    // HTMLタグを保持（Googleは一部HTMLタグを許可）
    const answer = answerEl.innerHTML.trim();

    mainEntity.push({
      '@type': 'Question',
      name: question,
      acceptedAnswer: {
        '@type': 'Answer',
        text: answer,
      },
    });
  });

  if (mainEntity.length === 0) return null;

  return {
    '@context': 'https://schema.org',
    '@type': 'FAQPage',
    mainEntity,
  };
}
```

### 回答にHTMLを含める

Googleの FAQPage スキーマでは `acceptedAnswer.text` に一部のHTMLタグ（`<a>`, `<b>`, `<br>`, `<ol>`, `<ul>`, `<li>`, `<p>` 等）を含められる。ただし不要なタグは除去しておくほうが安全。

```javascript
function sanitizeAnswerHtml(html) {
  const allowed = ['a', 'b', 'strong', 'br', 'p', 'ol', 'ul', 'li', 'em', 'i'];
  const temp = document.createElement('div');
  temp.innerHTML = html;

  // 許可タグ以外を削除
  const walker = document.createTreeWalker(temp, NodeFilter.SHOW_ELEMENT);
  const toRemove = [];

  while (walker.nextNode()) {
    const node = walker.currentNode;
    if (!allowed.includes(node.tagName.toLowerCase())) {
      toRemove.push(node);
    }
  }

  toRemove.forEach(node => {
    // 中身は残してタグだけ除去
    while (node.firstChild) {
      node.parentNode.insertBefore(node.firstChild, node);
    }
    node.parentNode.removeChild(node);
  });

  return temp.innerHTML.trim();
}
```

## WebApplication スキーマの設計

Webツールには `WebApplication` スキーマが適している。ツールのメタ情報をDOMから抽出して生成する。

```javascript
function generateWebAppSchema() {
  // ページのメタ情報から抽出
  const title = document.querySelector('h1')?.textContent?.trim();
  const description = document.querySelector('meta[name="description"]')?.content;
  const url = window.location.href;

  if (!title) return null;

  return {
    '@context': 'https://schema.org',
    '@type': 'WebApplication',
    name: title,
    description: description || '',
    url: url,
    applicationCategory: 'FinanceApplication',
    operatingSystem: 'All',
    offers: {
      '@type': 'Offer',
      price: '0',
      priceCurrency: 'JPY',
    },
    author: {
      '@type': 'Organization',
      name: 'AND TOOLS',
      url: 'https://and-tools.net/',
    },
    browserRequirements: 'Requires JavaScript. Requires HTML5.',
  };
}
```

`applicationCategory` はツールの種別に応じて変更する:

```javascript
const CATEGORY_MAP = {
  tax: 'FinanceApplication',
  loan: 'FinanceApplication',
  health: 'HealthApplication',
  utility: 'UtilitiesApplication',
  business: 'BusinessApplication',
  education: 'EducationalApplication',
};

function getCategoryFromPath(path) {
  if (path.includes('/tools/tax') || path.includes('/tools/income'))
    return CATEGORY_MAP.tax;
  if (path.includes('/tools/loan') || path.includes('/tools/mortgage'))
    return CATEGORY_MAP.loan;
  return CATEGORY_MAP.utility; // デフォルト
}
```

## 複数スキーマの統合挿入

1ページに FAQPage と WebApplication の両方を入れる場合は、`@graph` で統合するか、別々の `script` タグにする。

```javascript
function injectJsonLd(schemas) {
  // nullを除外
  const validSchemas = schemas.filter(Boolean);
  if (validSchemas.length === 0) return;

  if (validSchemas.length === 1) {
    // 単一スキーマ
    insertScript(validSchemas[0]);
  } else {
    // 複数スキーマ → @graph で統合
    const combined = {
      '@context': 'https://schema.org',
      '@graph': validSchemas.map(s => {
        // @context は @graph 使用時は不要
        const { '@context': _, ...rest } = s;
        return rest;
      }),
    };
    insertScript(combined);
  }
}

function insertScript(data) {
  // 既存のJSON-LDを削除（二重挿入防止）
  const existing = document.querySelectorAll('script[type="application/ld+json"][data-auto]');
  existing.forEach(el => el.remove());

  const script = document.createElement('script');
  script.type = 'application/ld+json';
  script.dataset.auto = 'true';
  script.textContent = JSON.stringify(data, null, 0);
  document.head.appendChild(script);
}
```

`data-auto` 属性を付けておくことで、手書きの JSON-LD と自動生成の JSON-LD を区別できる。再実行時に自動生成分だけを差し替える。

## BreadcrumbList の自動生成

パンくずリストも DOM から自動生成する。

```javascript
function generateBreadcrumbSchema(selector) {
  const breadcrumb = document.querySelector(selector);
  if (!breadcrumb) return null;

  const links = breadcrumb.querySelectorAll('a');
  const items = [];

  links.forEach((link, index) => {
    items.push({
      '@type': 'ListItem',
      position: index + 1,
      name: link.textContent.trim(),
      item: link.href,
    });
  });

  // 現在のページ（リンクなし）
  const currentPage = breadcrumb.querySelector('.breadcrumb__current');
  if (currentPage) {
    items.push({
      '@type': 'ListItem',
      position: items.length + 1,
      name: currentPage.textContent.trim(),
    });
  }

  if (items.length === 0) return null;

  return {
    '@context': 'https://schema.org',
    '@type': 'BreadcrumbList',
    itemListElement: items,
  };
}
```

## 全体の初期化

ページ読み込み時にすべてのスキーマをまとめて生成・挿入する。

```javascript
document.addEventListener('DOMContentLoaded', () => {
  const schemas = [
    generateWebAppSchema(),
    generateFaqSchema('.faq'),
    generateBreadcrumbSchema('.breadcrumb'),
  ];

  injectJsonLd(schemas);
});
```

## Google Search Console での検証

構造化データが正しく認識されているかは、以下の方法で確認する。

1. **リッチリザルトテスト**: https://search.google.com/test/rich-results でURLを入力
2. **Search Console → 拡張**: FAQPage、WebApplication 等のレポートを確認
3. **Chrome DevTools**: Elements タブで `script[type="application/ld+json"]` の中身を確認

```javascript
// デバッグ用: 生成されたJSON-LDをコンソールに出力
function debugJsonLd() {
  const scripts = document.querySelectorAll('script[type="application/ld+json"]');
  scripts.forEach(script => {
    try {
      const data = JSON.parse(script.textContent);
      console.log('JSON-LD:', JSON.stringify(data, null, 2));
    } catch (e) {
      console.error('Invalid JSON-LD:', script.textContent);
    }
  });
}
```

### よくあるエラーと対処

| エラー | 原因 | 対処 |
|--------|------|------|
| `Missing field "name"` | Question の name が空 | DOMからの抽出失敗 → セレクタを確認 |
| `Invalid value for "price"` | price が数値文字列でない | `"0"` のように文字列で指定 |
| `Invalid URL` | 相対URLを指定 | `window.location.origin` を付けて絶対URLに |
| `Duplicate structured data` | 同じスキーマが2つ | `data-auto` で既存を削除してから挿入 |

## SSR/SSG環境での注意

この記事の手法は**クライアントサイドで JSON-LD を生成**している。Googlebotは JS を実行するので認識されるが、以下の点に注意。

- **レンダリングの遅延**: Googlebot の JS 実行は遅れることがある。`DOMContentLoaded` で即座に生成すれば問題ない
- **SSG環境ではビルド時に静的出力が望ましい**: Gulp + EJS なら EJS のテンプレート内で JSON-LD を出力できる
- **ハイブリッド**が実用的: ベースの JSON-LD は静的に出力し、FAQ等の動的部分だけ JS で補完する

## まとめ

1. **DOMをSingle Source of Truth**にすれば、コンテンツとJSON-LDの不整合を防げる
2. **FAQPage** はアコーディオンのDOMから自動抽出。`sanitizeAnswerHtml` で安全なHTMLに整形
3. **WebApplication** はツールページに最適。`offers.price: "0"` で無料ツールを明示
4. **`@graph`** で複数スキーマを1つの `script` タグに統合
5. **`data-auto` 属性**で自動生成分と手書き分を区別し、再実行時の二重挿入を防止

---

## 関連サイト

JSON-LD の動的生成が実際に稼働しているサイトはこちら。

- [AND TOOLS](https://and-tools.net/) — 34個以上のツールに WebApplication + FAQPage スキーマを自動設定
- [ToolShare Lab JSON-LD ガイド](https://webatives.com/guides/json-ld-guide/) — 構造化データの設計方針と実装パターンをまとめたガイド
- [ToolShare Lab スキーマ生成ツール](https://webatives.com/tools/schema/) — JSON-LD の生成・検証ツール
- [ToolShare Lab](https://webatives.com/tools/) — 150個以上の無料ツールに構造化データを設定
