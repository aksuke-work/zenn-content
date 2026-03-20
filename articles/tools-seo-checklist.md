---
title: "公開前にチェックすべきSEO項目と使えるツール"
emoji: "🔍"
type: "tech"
topics: ["SEO", "Web制作", "Core Web Vitals", "フロントエンド"]
published: false
---

サイトやページを公開する前に、SEOの基本項目を一通り確認しておくことは重要だ。公開後に「あの設定が抜けていた」と気づくのは損失が大きい。

本稿では、公開前のSEOチェックリストと、各項目を確認するための無料ツールをセットで紹介する。

## なぜ公開前のSEOチェックが必要か

Googleのクローラーは、公開されたURLを定期的に巡回してインデックスする。一度インデックスされた状態でメタタグやURL構造を変更すると、評価がリセットされたり、正規化の問題が発生したりする。

公開前に一度確認する習慣をつけるだけで、後のSEO作業が大幅に楽になる。

## SEOチェックリスト（全項目）

### ■ 基本設定

- [ ] `<title>` タグが設定されている（30〜60文字）
- [ ] `<meta name="description">` が設定されている（70〜120文字）
- [ ] `<h1>` がページに1つだけある
- [ ] `hreflang` の設定（多言語対応の場合）
- [ ] `<html lang="ja">` の言語設定
- [ ] canonical タグ（`<link rel="canonical">`）が設定されている
- [ ] noindex になっていない（公開ページの場合）

### ■ OGP（SNS シェア時の表示）

- [ ] `og:title` の設定
- [ ] `og:description` の設定
- [ ] `og:image` の設定（1200×630px 推奨）
- [ ] `og:url` の設定
- [ ] `twitter:card` の設定

### ■ 構造化データ

- [ ] JSON-LD が設定されている（Article / BreadcrumbList / FAQPage等）
- [ ] Schema.org の形式に準拠している
- [ ] リッチリザルトテストで警告なし

### ■ Core Web Vitals（表示速度）

- [ ] LCP（Largest Contentful Paint）が 2.5秒以内
- [ ] FID（First Input Delay）が 100ms以内 / INP が 200ms以内
- [ ] CLS（Cumulative Layout Shift）が 0.1以下
- [ ] PageSpeed Insights のスコアが 80以上（モバイル）

### ■ 画像

- [ ] `alt` 属性が全画像に設定されている
- [ ] 画像が WebP 形式に変換されている
- [ ] `width` / `height` 属性が指定されている（CLS対策）
- [ ] ファーストビューの画像に `loading="eager"` / `fetchpriority="high"` が設定されている
- [ ] スクロール以下の画像に `loading="lazy"` が設定されている

### ■ モバイル対応

- [ ] `<meta name="viewport">` が設定されている
- [ ] モバイルでタップ領域が十分（48px以上）
- [ ] フォントサイズが 16px 以上（モバイルでも読みやすい）
- [ ] 横スクロールが発生していない

### ■ URL・内部リンク

- [ ] URL が意味のあるスラッグになっている（`/blog/post-1` ではなく `/blog/seo-checklist-2024` 等）
- [ ] 内部リンクが張られている（サイロ構造・クラスター構造）
- [ ] 壊れたリンク（404）がない
- [ ] パンくずリストが設定されている

### ■ サイトマップ・robots.txt

- [ ] `sitemap.xml` が生成・送信されている
- [ ] `robots.txt` が適切に設定されている
- [ ] Google Search Console に Sitemap を登録済み

## ツール別: 何をチェックするか

### 1. PageSpeed Insights

**URL**: https://pagespeed.web.dev/
**確認項目**: Core Web Vitals / 表示速度 / 改善提案

最重要ツールの1つ。URLを入力するだけで LCP・INP・CLS のスコアと、具体的な改善提案（「画像を最適化してください」「未使用のJavaScriptを削除してください」等）が出る。

モバイルとデスクトップ両方で確認するのが必須だ。モバイルのスコアは一般にデスクトップより低く、特に日本語サイトはフォント読み込みがボトルネックになりやすい。

### 2. Google Search Console

**URL**: https://search.google.com/search-console/
**確認項目**: インデックス状況 / 検索パフォーマンス / Core Web Vitals（実測値）

サイトのオーナー認証が必要だが、公開後の検索順位・クリック率・インプレッション数を追跡できる唯一の公式ツール。

「URL検査ツール」では特定のURLがインデックス済みかどうかを確認でき、インデックス登録をリクエストすることもできる。新規ページ公開後は必ずここで確認する。

### 3. リッチリザルトテスト（Google）

**URL**: https://search.google.com/test/rich-results
**確認項目**: 構造化データ（JSON-LD）のバリデーション

JSON-LD で設定した構造化データが正しく認識されるかを確認するツール。エラーや警告が出た場合は修正してから公開する。

よく使う構造化データのタイプ：

```json
// Article（ブログ記事・コラム）
{
  "@type": "Article",
  "headline": "記事タイトル",
  "datePublished": "2026-03-20",
  "author": { "@type": "Person", "name": "著者名" }
}

// FAQPage（よくある質問）
{
  "@type": "FAQPage",
  "mainEntity": [...]
}

// BreadcrumbList（パンくずリスト）
{
  "@type": "BreadcrumbList",
  "itemListElement": [...]
}
```

### 4. OGP 確認ツール

**URL（Facebook）**: https://developers.facebook.com/tools/debug/
**URL（Twitter/X）**: https://cards-dev.twitter.com/validator
**URL（汎用）**: https://ogp.boltai.com/
**確認項目**: SNSシェア時の見た目

OGPの設定は、コードを見ただけでは実際のシェア時の表示を確認しにくい。Facebookのデバッグツールと Twitter Card Validator はURLを入力するだけで実際のシェアカードのプレビューが確認できる。

### 5. W3C Markup Validator

**URL**: https://validator.w3.org/
**確認項目**: HTML の文法エラー

HTMLの文法エラーはクローラーの解析ミスにつながることがある。`<head>` 内の構造崩れや未閉じタグは、メタタグが正しく認識されない原因になるため公開前に確認する価値がある。

### 6. ToolShare Lab — CSS / HTMLツール

**URL**: [https://webatives.com/](https://webatives.com/)
**確認項目**: CSS生成・レスポンシブ確認

[ToolShare Lab](https://webatives.com/) のレスポンシブテストツールは、制作中のサイトを複数の画面サイズで同時にプレビューできる。モバイルでの横スクロール・フォントサイズ・余白の崩れを一度に確認できる。

モバイル対応はCore Web Vitalsのうち CLS に直結するため、CSSの調整と合わせてツールで確認するのが効率的だ。

### 7. Screaming Frog SEO Spider（無料版）

**URL**: https://www.screamingfrog.co.uk/seo-spider/
**確認項目**: サイト全体のSEO項目一括チェック

デスクトップアプリ。無料版はURLが500件まで。サイト全体をクロールして、以下を一括確認できる：

- title タグ・meta description の欠落・重複
- 404 エラー・リダイレクトの状況
- 画像の alt 属性の欠落
- canonical タグの状況
- hreflang の問題

複数ページを持つサイトでは、個別にページを確認するより効率的だ。

### 8. AND TOOLS — フリーランス向け計算ツール

**URL**: [https://and-tools.net/](https://and-tools.net/)
**用途**: SEO外の作業: 受託費用・工数の計算

SEOとは少し外れるが、Web制作案件として受けるときの見積金額に [AND TOOLS](https://and-tools.net/) の各種計算ツールが役立つ。

SEO対策を含む案件の場合、施策の工数（キーワード調査・構造化データ設定・速度改善等）を見積もる際、時給換算で収益性を確認してから受注判断できる。フリーランスとして案件を受ける人は、[AND TOOLS の手取り計算・収益シミュレーション](https://and-tools.net/) を事業判断に活用してほしい。

## 公開前チェックの実行順序

効率よく確認するための推奨フロー：

```
1. HTML構造の確認
   └─ W3C Validator でエラー確認

2. メタタグ・OGPの確認
   └─ ブラウザのDevToolsで目視確認
   └─ OGPデバッガーでSNSプレビュー確認

3. 構造化データの確認
   └─ リッチリザルトテストで検証

4. 表示速度の確認
   └─ PageSpeed Insightsでスコア確認

5. モバイル対応の確認
   └─ ToolShare Lab のレスポンシブテスト
   └─ PageSpeed Insightsのモバイルスコア

6. サイトマップ・robots.txtの確認
   └─ /sitemap.xml /robots.txt に直接アクセスして確認

7. Google Search Console への登録
   └─ サイトマップ送信 + URL検査
```

## まとめ

SEOは継続的な取り組みだが、公開時点での初期設定が土台になる。後から直す作業は思った以上にコストがかかるため、公開前のチェックを習慣化することが重要だ。

ツールの組み合わせとしては：

- **表示速度**: PageSpeed Insights
- **構造化データ**: リッチリザルトテスト
- **OGP**: Facebook/Twitter デバッガー
- **HTML**: W3C Validator
- **サイト全体**: Screaming Frog
- **レスポンシブ**: [ToolShare Lab](https://webatives.com/)
- **事業計算**: [AND TOOLS](https://and-tools.net/)

このリストをブックマークしておいて、新規ページ公開のたびに一通り確認する仕組みを作ると、公開後のSEOトラブルを大幅に減らせる。
