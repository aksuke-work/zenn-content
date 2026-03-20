---
title: "日本語Webフォントの選び方 — Google Fonts+Noto Sansの最適設定"
emoji: "🔤"
type: "tech"
topics: ["css", "webdesign", "frontend", "typography"]
published: false
---

日本語Webフォントはファイルサイズが大きく、使い方を誤るとページの読み込みが遅くなる。[ToolShare Lab](https://and-and.net/) を作るときに、日本語フォントの最適化で試行錯誤した経験をまとめた。

この記事では、Google Fontsを中心に日本語Webフォントの選び方と、Core Web Vitalsを悪化させない実装方法を解説する。

---

## 日本語Webフォントの課題

日本語フォントは文字数が多い（常用漢字だけでも2136字）ため、1ウェイトあたり数MB〜数十MBになる。そのまま読み込むとFOIT（Flash of Invisible Text）やFOUT（Flash of Unstyled Text）が発生し、ユーザー体験と Core Web Vitals（CLS/LCP）に悪影響が出る。

解決策：
1. Google Fontsのサブセット機能を活用する
2. `font-display` を適切に設定する
3. 必要なウェイトのみ読み込む

---

## Google Fontsで日本語フォントを使う

### 主要な日本語フォント一覧

| フォント名 | 印象 | 用途 |
|---|---|---|
| Noto Sans JP | モダン・クリーン | 汎用・UI |
| Noto Serif JP | 伝統的・上品 | 読み物・和風 |
| M PLUS 1p | 丸みがある・親しみやすい | LP・カジュアル |
| M PLUS Rounded 1c | やわらかい | 子ども向け・ポップ |
| Zen Kaku Gothic New | すっきり・現代的 | UI・テキスト |
| Shippori Mincho | 格調高い明朝 | 和風・高級感 |
| Kaisei Decol | デザイン性高い明朝 | ヘッダー・アクセント |

### Google Fonts URL の組み立て

```html
<!-- Noto Sans JP：400・500・700のみ -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link rel="stylesheet"
  href="https://fonts.googleapis.com/css2?family=Noto+Sans+JP:wght@400;500;700&display=swap">
```

### `display=swap` の重要性

`&display=swap` を付けると `font-display: swap` が適用される。フォントの読み込み中はシステムフォントで表示し、読み込み完了後に切り替わる。これによりFOITを防げる。

---

## CSS でのフォント設定

### 基本設定

```css
body {
  font-family:
    'Noto Sans JP',
    'Hiragino Sans',      /* macOS */
    'Hiragino Kaku Gothic ProN',
    'Yu Gothic Medium',   /* Windows */
    'Yu Gothic',
    Meiryo,
    sans-serif;
  font-weight: 400;
  font-size: 16px;
  line-height: 1.8;       /* 日本語は1.7〜1.9が読みやすい */
  letter-spacing: 0.02em; /* 少し広めに */
}
```

### タイポグラフィのルール（日本語）

```css
:root {
  /* フォントスタック */
  --font-sans:   'Noto Sans JP', 'Hiragino Sans', 'Yu Gothic Medium', sans-serif;
  --font-serif:  'Noto Serif JP', 'Hiragino Mincho ProN', 'Yu Mincho', serif;
  --font-mono:   'JetBrains Mono', 'Fira Code', Consolas, monospace;

  /* フォントサイズ */
  --text-sm:   clamp(12px, 0.8vw + 9px, 13px);
  --text-base: clamp(15px, 1vw + 11px, 16px);
  --text-lg:   clamp(17px, 1.2vw + 12.5px, 18px);
  --text-xl:   clamp(20px, 1.8vw + 13px, 24px);
  --text-2xl:  clamp(24px, 2.8vw + 13px, 32px);
  --text-3xl:  clamp(28px, 4vw + 13px, 40px);

  /* 行間 */
  --leading-tight:  1.4;
  --leading-normal: 1.7;
  --leading-relaxed:1.9;

  /* 字間 */
  --tracking-tight:  -0.01em;
  --tracking-normal: 0.02em;
  --tracking-wide:   0.08em;
}
```

---

## Noto Sans JPの最適設定

### フォントウェイトと用途のマッピング

```css
/* 本文 */
p {
  font-weight: 400;
  line-height: var(--leading-normal);
  letter-spacing: var(--tracking-normal);
}

/* 見出し（h2, h3） */
h2, h3 {
  font-weight: 700;
  line-height: var(--leading-tight);
  letter-spacing: var(--tracking-tight);
}

/* キャプション・注釈 */
small, .caption {
  font-weight: 400;
  font-size: var(--text-sm);
  color: var(--color-text-muted);
}

/* ナビゲーション・ラベル */
nav, label {
  font-weight: 500;
  letter-spacing: var(--tracking-normal);
}
```

---

## `font-feature-settings` で組版品質を上げる

### palt（プロポーショナル詰め）

日本語フォントは等幅設計のものが多く、そのままだと文字間が広く感じる。`palt` を有効にすると文字ごとに適切な幅に調整される。

```css
body {
  font-feature-settings: 'palt'; /* プロポーショナル詰め */
}

/* または個別に */
.heading {
  font-feature-settings: 'palt', 'pkna';
}
```

### よく使う font-feature-settings

| 値 | 効果 |
|---|---|
| `'palt'` | プロポーショナル詰め（一番効果が大きい） |
| `'pkna'` | プロポーショナル仮名 |
| `'liga'` | 合字（fi, fl 等） |
| `'kern'` | カーニング |
| `'tnum'` | 等幅数字（テーブルの数値揃え） |

### 数字を等幅にする

テーブルや金額表示では `tnum` を使うと数字が揃って見やすくなる。

```css
.price, .table td:has(> .number) {
  font-feature-settings: 'tnum';
  font-variant-numeric: tabular-nums;
}
```

---

## フォント読み込みの最適化

### 1. preconnect で事前接続

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
```

### 2. preload で最重要フォントを先読み

ファーストビューに必要なフォントファイルを `preload` で先読みする。

```html
<!-- フォントファイルのURLはGoogle FontsのCSSを確認して取得 -->
<link rel="preload"
  href="https://fonts.gstatic.com/s/notosansjp/v53/xxx.woff2"
  as="font"
  type="font/woff2"
  crossorigin>
```

### 3. font-display の選択

```css
@font-face {
  font-family: 'MyFont';
  src: url('/fonts/myfont.woff2') format('woff2');
  font-display: swap;     /* 推奨：即座にフォールバック、切り替わったらスワップ */
  /* font-display: optional; を使うとCLSがより安定する */
}
```

| font-display | 挙動 | 推奨 |
|---|---|---|
| `block` | フォント読み込みまで非表示（FOIT） | 非推奨 |
| `swap` | フォールバック表示 → スワップ | 本文テキスト |
| `fallback` | 短い非表示 → スワップ | — |
| `optional` | 接続速度が遅い場合フォールバックのまま | Core Web Vitals重視 |

---

## セルフホスティング（推奨設定）

Google Fontsに依存しない場合、フォントをサーバーに置いてセルフホスティングする。プライバシー要件がある場合（GDPRなど）や、外部CDN依存を避けたい場合に有効。

```css
@font-face {
  font-family: 'Noto Sans JP';
  font-style: normal;
  font-weight: 400;
  font-display: swap;
  src: url('/fonts/NotoSansJP-Regular.woff2') format('woff2');
  unicode-range: U+3000-30FF, U+4E00-9FFF, U+FF00-FFEF; /* 日本語範囲 */
}

@font-face {
  font-family: 'Noto Sans JP';
  font-style: normal;
  font-weight: 700;
  font-display: swap;
  src: url('/fonts/NotoSansJP-Bold.woff2') format('woff2');
  unicode-range: U+3000-30FF, U+4E00-9FFF, U+FF00-FFEF;
}
```

`unicode-range` で日本語に必要な範囲だけを指定すると、英数字は別のフォント（またはシステムフォント）に任せられる。

---

## パフォーマンスチェックリスト

- [ ] `preconnect` をGoogle Fonts用に設定している
- [ ] `&display=swap` を付けている
- [ ] 使うウェイトを3種類以内に絞っている（400・500・700が標準）
- [ ] `font-feature-settings: 'palt'` を設定している
- [ ] 数値表示箇所に `font-variant-numeric: tabular-nums` を設定している
- [ ] PageSpeed Insightsで「フォントの読み込みを最適化してください」が出ていないか確認した

---

## まとめ

日本語Webフォントは選択肢が増えたが、Noto Sans JPを中心に使うのがコスパがよい。

重要ポイント：
- Google Fonts は必ず `display=swap` を付ける
- `font-feature-settings: 'palt'` で組版品質が上がる
- ウェイトは3種類以内に絞る
- `preconnect` で読み込みを早める

[ToolShare Lab](https://and-and.net/) ではフォント選定や組版に関するTipsを発信している。フォントにまつわるWeb制作の実務ノウハウは、[AND TOOLS](https://and-tools.net/) でも関連するツールを無料公開している。
