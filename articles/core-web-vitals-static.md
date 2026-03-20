---
title: "静的サイトのCore Web Vitalsを最適化する — LCP/FID/CLS対策"
emoji: "⚡"
type: "tech"
topics: ["CoreWebVitals", "パフォーマンス", "SEO", "静的サイト"]
published: false
---

Core Web Vitals（CWV）はGoogleのランキング要素に含まれている。[AND TOOLS](https://and-tools.net/) と [ToolShare Lab](https://and-and.net/) という2つの静的サイトを運営していて、CWVのスコア改善に取り組んできた経験をまとめる。

静的サイトはNext.jsやWordPressと比べてCWVが良くなりやすいが、AdSenseの広告やJavaScriptのツール処理でスコアが下がりやすいポイントがある。そこを中心に解説する。

## Core Web Vitalsの3指標を理解する

### LCP（Largest Contentful Paint）

ページの中で最も大きな要素が描画されるまでの時間。

**判定基準:**
- 良好: 2.5秒以下
- 改善が必要: 2.5〜4.0秒
- 低速: 4.0秒超

**静的サイトでLCPが遅くなる主な原因:**

1. ファーストビューに大きな画像がある
2. 画像の最適化ができていない（PNGのまま、サイズが大きい）
3. レンダーブロッキングなCSSやJSがある

### CLS（Cumulative Layout Shift）

ページ読み込み中にレイアウトがどれだけズレたか。

**判定基準:**
- 良好: 0.1以下
- 改善が必要: 0.1〜0.25
- 低速: 0.25超

**静的サイトでCLSが発生する主な原因:**

1. AdSenseなどの広告の遅延読み込みによるレイアウトシフト
2. `width`/`height`属性のない画像
3. 非同期で読み込まれるフォント

### INP（Interaction to Next Paint）

2024年3月にFID（First Input Delay）の後継として正式指標になった。ユーザーの操作（クリック、キー入力）に対してページがどれだけ素早く応答するかを測定する。

**判定基準:**
- 良好: 200ms以下
- 改善が必要: 200〜500ms
- 低速: 500ms超

**静的サイトでINPが悪化する主な原因:**

1. ツールの計算処理がメインスレッドをブロックする
2. イベントハンドラが重い処理を同期的に実行する

## LCP改善の実践

### 画像の最適化

AND TOOLSのトップページでヒーロー画像を使っていた時期にLCPが3.2秒だった。以下の対応でLCPが1.8秒に改善した。

**WebP変換:**

```bash
# cwebpでPNG/JPGをWebPに変換
cwebp -q 80 hero.png -o hero.webp

# ImageMagickで一括変換
for file in *.jpg; do
  convert "$file" -quality 80 "${file%.jpg}.webp"
done
```

**`<picture>` タグでfallbackを用意:**

```html
<picture>
  <source srcset="hero.webp" type="image/webp">
  <img src="hero.jpg" alt="AND TOOLS" width="1200" height="600">
</picture>
```

**`loading="eager"` と `fetchpriority="high"` でLCP要素を優先:**

```html
<!-- LCP要素には必ずfetchpriority="high"を付ける -->
<img
  src="hero.webp"
  alt="AND TOOLS"
  width="1200"
  height="630"
  loading="eager"
  fetchpriority="high"
>
```

LCP要素（最大の画像やテキストブロック）には `loading="lazy"` を付けてはいけない。逆効果になる。

### Critical CSSのインライン化

外部CSSファイルはレンダーブロッキングリソースになる。ファーストビューに必要なCSSだけを `<head>` にインラインで書き、残りは非同期で読み込む。

```html
<head>
  <!-- Critical CSSをインライン化 -->
  <style>
    /* ファーストビューに必要なスタイルのみ */
    body { margin: 0; font-family: sans-serif; }
    .hero { width: 100%; background: #f5f5f5; }
    /* ... */
  </style>

  <!-- 残りのCSSを非同期で読み込む -->
  <link
    rel="preload"
    href="/css/main.css"
    as="style"
    onload="this.onload=null;this.rel='stylesheet'"
  >
  <noscript>
    <link rel="stylesheet" href="/css/main.css">
  </noscript>
</head>
```

GulpやWebpackで自動化する場合は `critical` パッケージが便利だ。

```bash
npm install critical --save-dev
```

### フォントのプリロード

Googleフォントなど外部フォントはLCPを遅らせる原因になる。

**セルフホスティングに切り替える:**

```html
<!-- NG: 外部フォントの直接読み込み -->
<link href="https://fonts.googleapis.com/css2?family=Noto+Sans+JP" rel="stylesheet">

<!-- OK: フォントをダウンロードしてセルフホスティング -->
<link rel="preload" href="/fonts/NotoSansJP-Regular.woff2" as="font" type="font/woff2" crossorigin>
```

```css
@font-face {
  font-family: 'Noto Sans JP';
  src: url('/fonts/NotoSansJP-Regular.woff2') format('woff2');
  font-weight: 400;
  font-display: swap;
}
```

`font-display: swap` は必ず指定する。フォントが読み込まれるまでシステムフォントを表示してFOUT（Flash of Unstyled Text）を許容することで、LCPを改善できる。

## CLS改善の実践

### AdSense広告枠の高さを事前確保

これがツールサイトでCLSを悪化させる最大の原因だ。

```html
<!-- NG: 高さを指定しない広告枠 -->
<div>
  <ins class="adsbygoogle" data-ad-slot="1234567890"></ins>
</div>

<!-- OK: 最小高さを確保する -->
<div style="min-height: 250px;">
  <ins class="adsbygoogle" data-ad-slot="1234567890"></ins>
</div>
```

AND TOOLSでCLSが0.35だった時期があったが、広告枠に `min-height` を設定して0.05に改善した。

**レスポンシブ広告枠の場合:**

```css
.ad-container {
  min-height: 90px; /* モバイル横長 */
}

@media (min-width: 768px) {
  .ad-container {
    min-height: 250px; /* デスクトップ レクタングル */
  }
}
```

### 画像のサイズ属性を必ず指定する

```html
<!-- NG: サイズ指定なし（CLS発生の原因） -->
<img src="tool-screenshot.png" alt="ツールのスクリーンショット">

<!-- OK: width/heightを必ず指定 -->
<img src="tool-screenshot.png" alt="ツールのスクリーンショット" width="800" height="450">
```

`width` と `height` を指定するとブラウザはアスペクト比を事前計算できる。これだけでCLSを大幅に減らせる。

## INP改善の実践

### 計算処理をWeb Workersに移す

AND TOOLSでは複雑な税額計算をメインスレッドで実行していたため、計算中にスクロールやボタンクリックへの応答が遅れることがあった。Web Workersで別スレッドに移すことで解決した。

```javascript
// main.js
const worker = new Worker('/js/calc-worker.js');

document.getElementById('calculate').addEventListener('click', () => {
  const income = parseInt(document.getElementById('income').value);

  // Workerに計算を依頼（メインスレッドはブロックされない）
  worker.postMessage({ type: 'calc', income });
});

worker.onmessage = (e) => {
  // 計算結果を受け取ってUIを更新
  document.getElementById('result').textContent = e.data.result;
};
```

```javascript
// calc-worker.js
self.onmessage = (e) => {
  if (e.data.type === 'calc') {
    const { income } = e.data;
    // 重い計算処理
    const result = calculateTax(income);
    self.postMessage({ result });
  }
};

function calculateTax(income) {
  // 複雑な計算...
  return result;
}
```

### イベントハンドラのdebounce

入力フォームのリアルタイム計算でINPが悪化することがある。

```javascript
// NG: 入力のたびに重い計算を実行
input.addEventListener('input', () => {
  calculateAndUpdate(); // 重い処理
});

// OK: debounceで間引く
function debounce(fn, delay) {
  let timeout;
  return function(...args) {
    clearTimeout(timeout);
    timeout = setTimeout(() => fn.apply(this, args), delay);
  };
}

input.addEventListener('input', debounce(() => {
  calculateAndUpdate();
}, 300)); // 300ms後に実行
```

## 計測と継続的なモニタリング

### PageSpeed Insightsで定期チェック

```bash
# APIで計測（要APIキー）
curl "https://www.googleapis.com/pagespeedonline/v5/runPagespeed?url=https://and-tools.net/tool/consumption-tax/&strategy=mobile&key=YOUR_API_KEY"
```

手動でチェックする場合はPageSpeed Insightsに直接URLを入れる。モバイルとデスクトップの両方を確認する。

### Search Consoleのコアウェブバイタルレポート

Search Console → コアウェブバイタル で、実際のユーザーデータ（フィールドデータ）を確認できる。PageSpeed Insightsは合成テストだが、こちらは実際のユーザーの体験データだ。

**確認のポイント:**
- 「低速」に分類されているURLを優先的に改善
- URLをクリックして類似ページをグループで確認
- モバイルとデスクトップを別々に確認

[ToolShare Lab](https://and-and.net/) はGulp 5 + 純粋な静的HTMLで構築しているため、JavaScriptフレームワークのオーバーヘッドがない。この構成はCWVの観点では有利で、ほとんどのページでGood判定を維持できている。

## まとめ：静的サイトのCWV改善チェックリスト

**LCP対策:**
- [ ] LCP要素の画像をWebP化
- [ ] LCP要素に `fetchpriority="high"` を付ける
- [ ] Critical CSSをインライン化
- [ ] フォントをセルフホスティング + `font-display: swap`

**CLS対策:**
- [ ] 全画像に `width` / `height` 属性を指定
- [ ] 広告枠に `min-height` を設定
- [ ] 動的コンテンツの高さを事前確保

**INP対策:**
- [ ] 重い計算処理をWeb Workersに移す
- [ ] イベントハンドラをdebounce/throttleで最適化
- [ ] 不要なJavaScriptを削除する

CWVの改善は一度やって終わりではない。コンテンツを追加するたびに再チェックする習慣をつけることが大切だ。
