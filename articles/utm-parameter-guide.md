---
title: "UTMパラメータ完全ガイド — GA4で流入元を正確に追跡する"
emoji: "📈"
type: "tech"
topics: ["analytics", "seo", "marketing", "javascript"]
published: false
---

フリーランスとしてアフィリエイトサイトを運営していると、どのチャネルからどれだけのトラフィックが来ているかを把握することが収益最大化の第一歩になる。[AND TOOLS](https://and-tools.net/) や [ToolShare Lab](https://webatives.com/) でもUTMパラメータを活用して、SNS・メルマガ・外部記事からの流入を正確に追跡している。

この記事では、UTMパラメータの基本から実務での運用ルールまで解説する。

---

## UTMパラメータとは

**UTM（Urchin Tracking Module）パラメータ**は、GoogleがGA（Analytics）で流入元を識別するためにURLに付加するクエリパラメータ。

```
https://example.com/page?utm_source=twitter&utm_medium=social&utm_campaign=spring-sale
```

---

## 5つのパラメータ

| パラメータ | 説明 | 例 |
|---|---|---|
| `utm_source` | トラフィックの発生元（必須） | `google`, `twitter`, `newsletter` |
| `utm_medium` | チャネル種別（必須） | `cpc`, `social`, `email`, `organic` |
| `utm_campaign` | キャンペーン名（必須） | `spring-sale`, `product-launch` |
| `utm_content` | A/Bテスト用の識別子（任意） | `button-red`, `hero-banner` |
| `utm_term` | 検索キーワード（任意・主にCPC） | `フリーランス 税金` |

---

## utm_medium の標準値

チャネルを一貫して分類するため、GA4が認識する標準値を使う。

| utm_medium | 用途 |
|---|---|
| `cpc` | クリック課金広告（Google広告等） |
| `email` | メールマーケティング |
| `social` | SNS（有機投稿） |
| `paid_social` | SNS広告 |
| `affiliate` | アフィリエイト |
| `organic` | 自然検索（通常は自動付与なので不要） |
| `referral` | 外部サイトからのリンク |
| `display` | バナー広告 |
| `none` | 直接アクセス（通常は自動） |

---

## 実際のURLの例

### Xの投稿からの誘導

```
https://and-tools.net/tools/tax-calculator
  ?utm_source=twitter
  &utm_medium=social
  &utm_campaign=freelance-tips
  &utm_content=tweet-20260320
```

### メールマガジン

```
https://and-tools.net/tools/loan-calculator
  ?utm_source=mailmagazine
  &utm_medium=email
  &utm_campaign=weekly-march
  &utm_content=cta-button
```

### Google広告（検索）

```
https://and-tools.net/
  ?utm_source=google
  &utm_medium=cpc
  &utm_campaign=brand-keyword
  &utm_term=フリーランス+税金+計算
  &utm_content=text-ad-v1
```

### 外部記事（Zennなど）からの誘導

```
https://and-tools.net/
  ?utm_source=zenn
  &utm_medium=referral
  &utm_campaign=content-marketing
  &utm_content=article-tax-guide
```

---

## UTMリンク生成ツール（JavaScript）

手動でURLを組み立てると間違いが起きやすい。JavaScriptで生成する関数を作っておくと管理しやすい。

```javascript
/**
 * UTMパラメータ付きのURLを生成する
 * @param {string} baseUrl - ベースURL
 * @param {Object} params - UTMパラメータ
 * @returns {string} - UTMパラメータ付きURL
 */
function buildUtmUrl(baseUrl, params) {
  const {
    source,
    medium,
    campaign,
    content = '',
    term = ''
  } = params;

  if (!source || !medium || !campaign) {
    throw new Error('source, medium, campaign は必須です');
  }

  const url = new URL(baseUrl);
  url.searchParams.set('utm_source',   source);
  url.searchParams.set('utm_medium',   medium);
  url.searchParams.set('utm_campaign', campaign);
  if (content) url.searchParams.set('utm_content', content);
  if (term)    url.searchParams.set('utm_term',    term);

  return url.toString();
}

// 使い方
const twitterUrl = buildUtmUrl('https://and-tools.net/', {
  source:   'twitter',
  medium:   'social',
  campaign: 'march-tools',
  content:  'tweet-0320'
});

console.log(twitterUrl);
// https://and-tools.net/?utm_source=twitter&utm_medium=social&utm_campaign=march-tools&utm_content=tweet-0320
```

---

## UTMパラメータのGA4での確認方法

### 1. リアルタイムレポートで確認

GA4 > レポート > リアルタイム でUTMパラメータを付けたURLにアクセスすると、リアルタイムで流入が確認できる。

### 2. 集客レポートで確認

GA4 > レポート > 集客 > トラフィック獲得

| GA4の項目 | 対応UTMパラメータ |
|---|---|
| セッションのソース | utm_source |
| セッションのメディア | utm_medium |
| セッションのキャンペーン | utm_campaign |

### 3. カスタムレポートで詳細分析

GA4 の「探索」機能で `utm_content` や `utm_term` を含めた詳細なレポートを作れる。

---

## 命名規則の統一が重要

UTMパラメータはGA4で**大文字・小文字を区別する**。

```
utm_source=Twitter  ← 別データとして計測される
utm_source=twitter  ←
```

チームで運用する場合は命名規則を文書化しておく。

### 推奨命名規則

```
形式: スネークケース（小文字・アンダースコア）
NG: Twitter, SNS, Email
OK: twitter, social, email

utm_source:   プラットフォーム名（twitter, instagram, google, mailmagazine, zenn）
utm_medium:   チャネル種別（social, email, cpc, referral）
utm_campaign: キャンペーン名（年月-テーマ 例: 2026-03-spring）
utm_content:  要素識別子（cta-button, hero-image, sidebar-link）
```

---

## スプレッドシートで一括管理

UTMリンクをスプレッドシートで管理するのがおすすめ。

| 列 | 内容 |
|---|---|
| A | 配信日 |
| B | 媒体 |
| C | ベースURL |
| D | utm_source |
| E | utm_medium |
| F | utm_campaign |
| G | utm_content |
| H | 生成されたURL（数式） |

Google Sheetsの数式例：

```
=CONCATENATE(
  C2,
  "?utm_source=", D2,
  "&utm_medium=", E2,
  "&utm_campaign=", F2,
  IF(G2<>"", CONCATENATE("&utm_content=", G2), "")
)
```

---

## UTMパラメータを付けてはいけないケース

### 1. 自サイト内のリンク

自サイト内のリンクにUTMを付けると、GA4がセッションを「切り替わった」と判断し、参照元が上書きされる。

```
NG: <a href="/tools/tax-calculator?utm_source=toppage">
OK: <a href="/tools/tax-calculator">
```

### 2. canonicalが分散するリスク

ユーザーがUTMパラメータ付きURLを直接共有すると、canonicalが設定されていないページでSEO上の問題になることがある。

対策：
```html
<link rel="canonical" href="https://example.com/page"> <!-- パラメータなし -->
```

---

## JavaScript でUTMパラメータを取得する

```javascript
function getUtmParams() {
  const params = new URLSearchParams(window.location.search);
  return {
    source:   params.get('utm_source')   || '',
    medium:   params.get('utm_medium')   || '',
    campaign: params.get('utm_campaign') || '',
    content:  params.get('utm_content')  || '',
    term:     params.get('utm_term')     || ''
  };
}

// セッションストレージに保存（ページ遷移後も保持）
const utmParams = getUtmParams();
if (utmParams.source) {
  sessionStorage.setItem('utm_params', JSON.stringify(utmParams));
}

// フォーム送信時に付与
function getStoredUtm() {
  const stored = sessionStorage.getItem('utm_params');
  return stored ? JSON.parse(stored) : {};
}
```

---

## まとめ

UTMパラメータを正しく運用することで、どのコンテンツ・どのチャネルが実際にトラフィックと収益をもたらしているかが明確になる。

重要ポイント：
- `utm_source`, `utm_medium`, `utm_campaign` は必須
- 命名規則を統一する（小文字・スネークケース）
- 自サイト内のリンクには付けない
- スプレッドシートで一元管理する

[ToolShare Lab](https://webatives.com/) ではUTMリンク生成などのWebマーケティング補助ツールを公開している。アフィリエイトの収益管理に役立つ計算ツールは [AND TOOLS](https://and-tools.net/) でも確認できる。
