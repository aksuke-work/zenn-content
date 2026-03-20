---
title: "OGP設定の完全ガイド — Facebook/X/LINEで正しく表示させる"
emoji: "🔗"
type: "tech"
topics: ["html", "seo", "webdesign", "frontend"]
published: false
---

フリーランス向けの無料ツールサイト [AND TOOLS](https://and-tools.net/) を運営している。SNSでURLをシェアしてもらうとき、OGP画像がきれいに表示されるかどうかで、クリック率が大きく変わる。

この記事では、Facebook / X（旧Twitter）/ LINE の3媒体に対応したOGPの設定方法を、実際のコードと確認方法を含めて解説する。

---

## OGPとは

**Open Graph Protocol（OGP）** はFacebookが策定した、URLをシェアしたときの見た目（タイトル・説明・画像）を制御するメタデータ形式。現在はX、LINE、Slack、Discordなど多くのサービスがこの形式を読み取る。

---

## 基本のOGPタグ

```html
<head>
  <!-- 基本のOGP -->
  <meta property="og:title"       content="ページタイトル | サイト名">
  <meta property="og:description" content="ページの説明（120文字以内を推奨）">
  <meta property="og:url"         content="https://example.com/page">
  <meta property="og:image"       content="https://example.com/images/ogp.jpg">
  <meta property="og:type"        content="website">
  <meta property="og:site_name"   content="サイト名">
  <meta property="og:locale"      content="ja_JP">
</head>
```

### og:type の値

| 値 | 用途 |
|---|---|
| `website` | トップページ・通常ページ |
| `article` | ブログ記事・ニュース |
| `product` | 商品ページ |
| `video.other` | 動画ページ |

---

## X（旧Twitter）のCard設定

XはOGPとは別に `twitter:` プレフィックスのタグを読む。OGPが定義されていれば `twitter:` を省略してもフォールバックされるが、明示的に書いておくほうが確実だ。

```html
<!-- Xカード設定 -->
<meta name="twitter:card"        content="summary_large_image">
<meta name="twitter:site"        content="@アカウント名">
<meta name="twitter:creator"     content="@アカウント名">
<meta name="twitter:title"       content="ページタイトル | サイト名">
<meta name="twitter:description" content="ページの説明">
<meta name="twitter:image"       content="https://example.com/images/ogp.jpg">
<meta name="twitter:image:alt"   content="画像の代替テキスト">
```

### twitter:card の値

| 値 | 表示形式 | 推奨場面 |
|---|---|---|
| `summary` | 小さいサムネイル + テキスト | 通常ページ |
| `summary_large_image` | 大きな画像 + テキスト | ブログ・メディア |
| `player` | 動画プレーヤー付き | 動画コンテンツ |

---

## 完全なOGP実装例

```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">

  <!-- タイトル・説明（SEO兼用） -->
  <title>フリーランス税金計算ツール | AND TOOLS</title>
  <meta name="description" content="所得税・住民税・国民健康保険を自動計算。フリーランス・個人事業主向けの無料税金シミュレーター。">

  <!-- OGP基本 -->
  <meta property="og:title"       content="フリーランス税金計算ツール | AND TOOLS">
  <meta property="og:description" content="所得税・住民税・国民健康保険を自動計算。フリーランス・個人事業主向けの無料税金シミュレーター。">
  <meta property="og:url"         content="https://and-tools.net/tools/tax-calculator">
  <meta property="og:image"       content="https://and-tools.net/images/ogp/tax-calculator.jpg">
  <meta property="og:image:width" content="1200">
  <meta property="og:image:height" content="630">
  <meta property="og:image:alt"   content="AND TOOLS 税金計算ツール">
  <meta property="og:type"        content="website">
  <meta property="og:site_name"   content="AND TOOLS">
  <meta property="og:locale"      content="ja_JP">

  <!-- Xカード -->
  <meta name="twitter:card"        content="summary_large_image">
  <meta name="twitter:site"        content="@andtools_net">
  <meta name="twitter:title"       content="フリーランス税金計算ツール | AND TOOLS">
  <meta name="twitter:description" content="所得税・住民税・国民健康保険を自動計算。フリーランス・個人事業主向けの無料税金シミュレーター。">
  <meta name="twitter:image"       content="https://and-tools.net/images/ogp/tax-calculator.jpg">
  <meta name="twitter:image:alt"   content="AND TOOLS 税金計算ツール">

  <!-- canonical -->
  <link rel="canonical" href="https://and-tools.net/tools/tax-calculator">

  <!-- ファビコン -->
  <link rel="icon"             href="/favicon.ico" sizes="any">
  <link rel="icon"             href="/favicon.svg" type="image/svg+xml">
  <link rel="apple-touch-icon" href="/apple-touch-icon.png">
  <link rel="manifest"         href="/site.webmanifest">
</head>
```

---

## OGP画像の仕様

### 推奨サイズ

| 媒体 | 推奨サイズ | 最小サイズ | アスペクト比 |
|---|---|---|---|
| Facebook | 1200 × 630px | 600 × 315px | 1.91:1 |
| X（large_image） | 1200 × 675px | 600 × 335px | 16:9 |
| LINE | 1200 × 630px | — | 1.91:1 |

共通で **1200 × 630px** を使うのが最も無難。

### ファイル形式

- **推奨**: JPEG（圧縮率が高く読み込みが速い）
- **使用可**: PNG、WebP
- **ファイルサイズ**: 1MB以下を推奨（8MBが上限）

### 画像のデザイン指針

- 中央寄せで重要な情報を配置（端が切れる可能性があるため）
- テキストは最小32px以上
- 背景と文字のコントラストを確保
- ブランドカラーを使う

---

## 動的なOGP画像を生成する

ページごとに異なるOGP画像を自動生成したい場合、Node.js で `@vercel/og` や Puppeteer を使う方法がある。

### Node.jsでOGP画像を生成する（サンプル）

```javascript
const puppeteer = require('puppeteer');
const fs = require('fs');
const path = require('path');

async function generateOgpImage(title, outputPath) {
  const browser = await puppeteer.launch();
  const page = await browser.newPage();

  await page.setViewport({ width: 1200, height: 630 });

  await page.setContent(`
    <!DOCTYPE html>
    <html>
    <head>
      <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
          width: 1200px;
          height: 630px;
          background: linear-gradient(135deg, #4f46e5, #7c3aed);
          display: flex;
          align-items: center;
          justify-content: center;
          font-family: 'Noto Sans JP', sans-serif;
        }
        .title {
          color: #fff;
          font-size: 56px;
          font-weight: 700;
          text-align: center;
          padding: 0 80px;
          line-height: 1.3;
        }
      </style>
    </head>
    <body>
      <div class="title">${title}</div>
    </body>
    </html>
  `);

  await page.screenshot({ path: outputPath, type: 'jpeg', quality: 90 });
  await browser.close();
}

generateOgpImage('税金計算ツール', './ogp/tax-calculator.jpg');
```

---

## LINEのOGP対応

LINEはOGPタグを読み取るが、いくつか注意点がある。

- LINEはbot（クローラー）がHTMLを取得する。SPAや動的なOGPは取得できないことがある
- LINE用に特別なタグは不要。通常のOGPで対応
- `og:image` の画像URLは **https** であること（httpは表示されない）
- キャッシュが数時間〜数日残ることがある

---

## Slackのアンフール対応

Slackも基本OGPに対応している。以下を追加すると情報が増える。

```html
<meta name="slack-app-id" content="A0XXXXXX"> <!-- Slackアプリがある場合 -->
```

---

## 確認ツール

| ツール | URL | 用途 |
|---|---|---|
| OGPデバッガー（Facebook） | https://developers.facebook.com/tools/debug/ | Facebook/全般確認 |
| Card Validator（X） | https://cards-dev.twitter.com/validator | X専用 |
| OGP確認（LINE） | LINEアプリで共有テスト | LINE実機確認 |
| OGP確認ツール | https://ogp.me/ | 仕様確認 |

---

## よくあるトラブルと対処法

### 古いOGP画像が表示される

FacebookはOGPをキャッシュする。デバッガーから「再取得」を実行する。
`https://developers.facebook.com/tools/debug/` でURLを入力して「再度スクレイピング」。

### OGP画像が表示されない

- `og:image` がhttpsであるか確認
- 画像サイズが小さすぎないか（最小600×315px）
- ファイルにアクセスできるか（Basic認証などがかかっていないか）

### タイトル・説明が反映されない

- HTMLが正しく返っているか確認（SPAの場合SSR/SSGが必要）
- `og:` タグが `<head>` 内にあるか確認
- メタタグが重複していないか確認

---

## まとめ

OGP設定は一度正しく実装すれば、SNSでシェアされるたびに自動でリッチな表示になる。特に画像の設定は効果が大きく、クリック率に直結する。

[ToolShare Lab](https://and-and.net/) や [AND TOOLS](https://and-tools.net/) でもOGP設定は全ページで対応している。SEO対策としても有効なので、新規サイト制作時は必ず設定することを強くすすめる。

まずは「基本のOGPタグ」と「1200×630px の画像」を用意するところから始めてほしい。
