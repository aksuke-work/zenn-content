---
title: "コピペで使えるJSON-LDテンプレート8種 — Article/FAQ/HowTo他"
emoji: "🧩"
type: "tech"
topics: ["seo", "html", "javascript", "frontend"]
published: false
---

フリーランス向け無料ツール集 [AND TOOLS](https://and-tools.net/) の各ページにJSON-LD構造化データを設定したところ、Googleの検索結果でリッチリザルト（FAQアコーディオン等）が表示されるようになった。

構造化データはSEOの観点でも重要で、特にFAQスキーマは実装コストが低い割に効果が出やすい。この記事では、よく使う8種類のJSON-LDテンプレートをコピペできる形でまとめる。

---

## JSON-LDの基本

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "スキーマタイプ",
  // ... プロパティ
}
</script>
```

`<head>` または `<body>` のどこに置いてもよい。一般的には `<head>` 内か `<body>` の末尾に置く。

---

## テンプレート1：WebSite（サイト全体）

サイトのトップページに設定する。検索ボックスをリッチリザルトに表示できる。

```json
{
  "@context": "https://schema.org",
  "@type": "WebSite",
  "name": "AND TOOLS",
  "url": "https://and-tools.net/",
  "description": "フリーランス・個人事業主向けの無料ツール集",
  "inLanguage": "ja",
  "potentialAction": {
    "@type": "SearchAction",
    "target": {
      "@type": "EntryPoint",
      "urlTemplate": "https://and-tools.net/search?q={search_term_string}"
    },
    "query-input": "required name=search_term_string"
  }
}
```

---

## テンプレート2：Organization（組織情報）

会社・個人事業のナレッジパネルに影響する。

```json
{
  "@context": "https://schema.org",
  "@type": "Organization",
  "name": "EmptyCoke",
  "url": "https://and-tools.net/",
  "logo": {
    "@type": "ImageObject",
    "url": "https://and-tools.net/images/logo.png",
    "width": 400,
    "height": 60
  },
  "description": "フリーランス向け無料ツール集「AND TOOLS」を運営",
  "sameAs": [
    "https://twitter.com/emptycoke",
    "https://zenn.dev/emptycoke"
  ],
  "contactPoint": {
    "@type": "ContactPoint",
    "contactType": "customer support",
    "availableLanguage": "Japanese"
  }
}
```

---

## テンプレート3：Article（記事）

ブログ記事・ニュース記事に設定する。

```json
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "フリーランスの税金計算方法 — 所得税・住民税・国民健康保険",
  "description": "フリーランスの税金の仕組みと計算方法を解説。具体的な数字を使って所得税・住民税・国保の計算手順を説明。",
  "url": "https://and-tools.net/blog/freelance-tax",
  "datePublished": "2026-03-01",
  "dateModified": "2026-03-20",
  "author": {
    "@type": "Person",
    "name": "芦刈庸介",
    "url": "https://and-tools.net/about"
  },
  "publisher": {
    "@type": "Organization",
    "name": "AND TOOLS",
    "logo": {
      "@type": "ImageObject",
      "url": "https://and-tools.net/images/logo.png"
    }
  },
  "image": {
    "@type": "ImageObject",
    "url": "https://and-tools.net/images/blog/freelance-tax.jpg",
    "width": 1200,
    "height": 630
  },
  "mainEntityOfPage": {
    "@type": "WebPage",
    "@id": "https://and-tools.net/blog/freelance-tax"
  }
}
```

---

## テンプレート4：FAQPage（よくある質問）

リッチリザルトで最も効果が出やすいスキーマ。検索結果でFAQがアコーディオン表示される。

```json
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "フリーランスの所得税はいつ払うのですか？",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "確定申告後に翌年3月15日までに一括納付、または振替納税。予定納税がある場合は7月・11月にも納税します。"
      }
    },
    {
      "@type": "Question",
      "name": "青色申告と白色申告の違いは何ですか？",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "青色申告は65万円（電子申告の場合）または10万円の特別控除を受けられる代わりに、複式簿記での記帳が必要です。白色申告は記帳がシンプルですが控除が少ない。"
      }
    },
    {
      "@type": "Question",
      "name": "国民健康保険はどのくらいかかりますか？",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "自治体や前年の所得によって大きく異なります。所得400万円の場合、年間50〜70万円程度が目安です。<a href='https://and-tools.net/tools/national-health-insurance'>国民健康保険計算ツール</a>で試算できます。"
      }
    }
  ]
}
```

---

## テンプレート5：HowTo（手順記事）

手順記事でリッチリザルトに手順が表示される。

```json
{
  "@context": "https://schema.org",
  "@type": "HowTo",
  "name": "フリーランスの確定申告の手順",
  "description": "フリーランス・個人事業主の確定申告を自分でやる手順を解説します。",
  "totalTime": "PT3H",
  "estimatedCost": {
    "@type": "MonetaryAmount",
    "currency": "JPY",
    "value": "0"
  },
  "step": [
    {
      "@type": "HowToStep",
      "name": "1年間の収支を集計する",
      "text": "領収書・請求書をもとに、収入と経費を月別に集計します。会計ソフトを使うと効率的です。",
      "position": 1
    },
    {
      "@type": "HowToStep",
      "name": "確定申告書を作成する",
      "text": "国税庁の「確定申告書作成コーナー」でオンライン作成するのが最も簡単です。",
      "url": "https://www.nta.go.jp/taxes/shiraberu/shinkoku/tokushu/",
      "position": 2
    },
    {
      "@type": "HowToStep",
      "name": "申告書を提出する",
      "text": "e-Taxで電子申告するか、税務署に持参・郵送します。提出期限は3月15日です。",
      "position": 3
    }
  ]
}
```

---

## テンプレート6：BreadcrumbList（パンくずリスト）

パンくずリストを構造化データで実装する。URLの階層構造が検索結果のURLの横に表示される。

```json
{
  "@context": "https://schema.org",
  "@type": "BreadcrumbList",
  "itemListElement": [
    {
      "@type": "ListItem",
      "position": 1,
      "name": "ホーム",
      "item": "https://and-tools.net/"
    },
    {
      "@type": "ListItem",
      "position": 2,
      "name": "ツール一覧",
      "item": "https://and-tools.net/tools/"
    },
    {
      "@type": "ListItem",
      "position": 3,
      "name": "税金計算ツール",
      "item": "https://and-tools.net/tools/tax-calculator"
    }
  ]
}
```

---

## テンプレート7：SoftwareApplication（ツール・アプリ）

Webアプリやオンラインツールに設定できる。

```json
{
  "@context": "https://schema.org",
  "@type": "SoftwareApplication",
  "name": "フリーランス税金計算ツール",
  "operatingSystem": "Web",
  "applicationCategory": "FinanceApplication",
  "description": "所得税・住民税・国民健康保険を一括計算できる無料のシミュレーターツール",
  "url": "https://and-tools.net/tools/tax-calculator",
  "offers": {
    "@type": "Offer",
    "price": "0",
    "priceCurrency": "JPY"
  },
  "aggregateRating": {
    "@type": "AggregateRating",
    "ratingValue": "4.8",
    "ratingCount": "245"
  },
  "author": {
    "@type": "Organization",
    "name": "AND TOOLS"
  }
}
```

---

## テンプレート8：複数スキーマを同時に設定

1ページに複数の構造化データを設定したい場合、`@graph` を使ってまとめて書ける。

```json
{
  "@context": "https://schema.org",
  "@graph": [
    {
      "@type": "WebPage",
      "@id": "https://and-tools.net/tools/tax-calculator#webpage",
      "url": "https://and-tools.net/tools/tax-calculator",
      "name": "フリーランス税金計算ツール | AND TOOLS",
      "isPartOf": {
        "@id": "https://and-tools.net/#website"
      }
    },
    {
      "@type": "FAQPage",
      "@id": "https://and-tools.net/tools/tax-calculator#faq",
      "mainEntity": [
        {
          "@type": "Question",
          "name": "このツールは無料ですか？",
          "acceptedAnswer": {
            "@type": "Answer",
            "text": "はい、すべての機能を無料でご利用いただけます。"
          }
        }
      ]
    },
    {
      "@type": "BreadcrumbList",
      "itemListElement": [
        {
          "@type": "ListItem",
          "position": 1,
          "name": "ホーム",
          "item": "https://and-tools.net/"
        },
        {
          "@type": "ListItem",
          "position": 2,
          "name": "税金計算ツール",
          "item": "https://and-tools.net/tools/tax-calculator"
        }
      ]
    }
  ]
}
```

---

## 確認・テストツール

| ツール | URL | 用途 |
|---|---|---|
| リッチリザルトテスト | https://search.google.com/test/rich-results | 構造化データの検証 |
| スキーママークアップ検証 | https://validator.schema.org/ | schema.orgへの準拠確認 |
| Google Search Console | > 検索結果の強化 | 実際のリッチリザルト表示状況 |

---

## よくあるミス

**1. FAQ の answer に HTML タグを使いすぎる**

Googleは基本的なHTMLタグ（`<a>`, `<br>`, `<ul>`, `<li>` 等）を解釈するが、複雑なHTML構造は避ける。

**2. FAQの質問数が多すぎる**

実際の検索結果に表示されるのは2〜3問のみ。10問以上設定しても表示枠は限られる。重要な質問を上に書く。

**3. スキーマの内容がページ本文と一致しない**

Googleはページ本文とスキーマの整合性をチェックする。スキーマだけで内容を追加しても効果がなく、ペナルティのリスクがある。

---

## まとめ

| スキーマ | 適用ページ | リッチリザルト効果 |
|---|---|---|
| FAQPage | FAQ・よくある質問 | ★★★（高） |
| HowTo | 手順記事 | ★★★（高） |
| Article | ブログ・コラム | ★★（中） |
| BreadcrumbList | 全ページ | ★★（中） |
| SoftwareApplication | ツール・アプリ | ★（低〜中） |

まず `FAQPage` と `BreadcrumbList` から設定するのがコスパが良い。

[ToolShare Lab](https://webatives.com/) や [AND TOOLS](https://and-tools.net/) では、ツールページごとに適切な構造化データを設定している。Search Consoleで「リッチリザルトの強化」を確認しながら継続的に改善している。
