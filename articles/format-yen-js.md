---
title: "日本円フォーマットの完全実装 — Intl.NumberFormatの落とし穴"
emoji: "¥"
type: "tech"
topics: ["javascript", "i18n", "日本円", "webdev"]
published: false
---

[AND TOOLS](https://and-tools.net/) では34本の計算ツールすべてで日本円のフォーマット処理を行っている。「カンマ区切りにするだけでしょ？」と思うかもしれないが、`Intl.NumberFormat` には意外な落とし穴がある。

## Intl.NumberFormat の基本

```javascript
const formatter = new Intl.NumberFormat('ja-JP', {
  style: 'currency',
  currency: 'JPY',
});

console.log(formatter.format(1234567)); // ￥1,234,567
```

一見簡単だが、これをそのまま使うと問題がいくつか出る。

## 落とし穴1: 「￥」が全角になる

`style: 'currency'` を使うと、ブラウザによって「￥」が**全角**で表示される。Chromeは `￥`（全角）、一部の環境では `¥`（半角）になり、表示が不統一。

```javascript
// ブラウザ依存の出力
const f = new Intl.NumberFormat('ja-JP', { style: 'currency', currency: 'JPY' });
console.log(f.format(1000000));
// Chrome: "￥1,000,000"（全角￥）
// Safari: "¥1,000,000"（半角¥の場合も）
```

解決策: `style: 'currency'` を使わず、数値フォーマットだけ任せる。

```javascript
function formatYen(value) {
  const num = typeof value === 'string' ? parseInt(value, 10) : value;
  if (isNaN(num)) return '0円';

  const formatted = new Intl.NumberFormat('ja-JP').format(Math.abs(num));
  return num < 0 ? `-${formatted}円` : `${formatted}円`;
}

console.log(formatYen(1234567));   // "1,234,567円"
console.log(formatYen(-50000));    // "-50,000円"
console.log(formatYen(0));         // "0円"
```

## 落とし穴2: マイナス値の表示

`Intl.NumberFormat` のマイナス表示はロケールに依存する。

```javascript
const f = new Intl.NumberFormat('ja-JP', { style: 'currency', currency: 'JPY' });
console.log(f.format(-1000));
// "-￥1,000" or "￥-1,000" ← ブラウザによって異なる
```

税金計算では「還付額」をマイナスで表すことがある。表示の一貫性のために手動制御するのが確実。

```javascript
function formatYenSigned(value) {
  const num = parseInt(value, 10);
  if (isNaN(num)) return '0円';

  const abs = Math.abs(num);
  const formatted = new Intl.NumberFormat('ja-JP').format(abs);

  if (num > 0) return `+${formatted}円`;
  if (num < 0) return `-${formatted}円`;
  return '0円';
}

console.log(formatYenSigned(50000));   // "+50,000円"
console.log(formatYenSigned(-50000));  // "-50,000円"
console.log(formatYenSigned(0));       // "0円"
```

## 落とし穴3: 小数点以下の処理

日本円に小数点はないが、計算途中で浮動小数点が出ることがある。

```javascript
const tax = 1000000 * 0.1021;
console.log(tax); // 102100.00000000001

// Intl.NumberFormat はデフォルトで小数を表示してしまう
const f = new Intl.NumberFormat('ja-JP');
console.log(f.format(tax)); // "102,100"（たまたまOK）

// しかし別のケースでは...
console.log(f.format(100 / 3)); // "33.333"
```

対策: フォーマット前に必ず整数化する。

```javascript
function formatYen(value) {
  const num = Math.floor(Number(value));
  if (isNaN(num)) return '0円';

  const formatted = new Intl.NumberFormat('ja-JP', {
    maximumFractionDigits: 0,
  }).format(Math.abs(num));

  return num < 0 ? `-${formatted}円` : `${formatted}円`;
}
```

`maximumFractionDigits: 0` を指定すると、小数点以下が確実に切り捨てられる。`Math.floor` との二重防御。

## 落とし穴4: 大きな数値の表示

億単位の数値は桁が多すぎて読みにくい。

```javascript
console.log(formatYen(123456789)); // "123,456,789円" ← 読みにくい
```

「万」「億」を使った表示を追加する。

```javascript
function formatYenReadable(value) {
  const num = Math.floor(Number(value));
  if (isNaN(num)) return '0円';

  const abs = Math.abs(num);
  const sign = num < 0 ? '-' : '';

  if (abs >= 100000000) {
    // 1億以上
    const oku = Math.floor(abs / 100000000);
    const remainder = abs % 100000000;
    if (remainder === 0) return `${sign}${oku}億円`;
    const man = Math.floor(remainder / 10000);
    return `${sign}${oku}億${man.toLocaleString()}万円`;
  }

  if (abs >= 10000) {
    // 1万以上
    const man = abs / 10000;
    // 整数万なら「○万円」、端数ありなら通常表示
    if (abs % 10000 === 0) {
      return `${sign}${man.toLocaleString()}万円`;
    }
  }

  return `${sign}${abs.toLocaleString()}円`;
}

console.log(formatYenReadable(50000));       // "5万円"
console.log(formatYenReadable(1234567));     // "1,234,567円"
console.log(formatYenReadable(300000000));   // "3億円"
console.log(formatYenReadable(150000000));   // "1億5,000万円"
```

## 手動実装との速度比較

`Intl.NumberFormat` と手動のカンマ挿入、どちらが速いか。

```javascript
// 手動実装
function formatYenManual(value) {
  const num = Math.floor(Number(value));
  if (isNaN(num)) return '0円';

  const abs = Math.abs(num);
  const str = String(abs);
  let result = '';

  for (let i = 0; i < str.length; i++) {
    if (i > 0 && (str.length - i) % 3 === 0) {
      result += ',';
    }
    result += str[i];
  }

  return (num < 0 ? '-' : '') + result + '円';
}

// 正規表現による実装
function formatYenRegex(value) {
  const num = Math.floor(Number(value));
  if (isNaN(num)) return '0円';

  const abs = Math.abs(num);
  const formatted = String(abs).replace(/\B(?=(\d{3})+(?!\d))/g, ',');
  return (num < 0 ? '-' : '') + formatted + '円';
}

// ベンチマーク
function benchmark() {
  const values = Array.from({ length: 10000 }, (_, i) => i * 1000 + 100);

  // Intl.NumberFormat（インスタンス再利用）
  const intlFormatter = new Intl.NumberFormat('ja-JP');
  const intlFn = (v) => intlFormatter.format(v) + '円';

  const methods = [
    { name: 'Intl.NumberFormat', fn: intlFn },
    { name: '手動ループ', fn: formatYenManual },
    { name: '正規表現', fn: formatYenRegex },
  ];

  for (const { name, fn } of methods) {
    const start = performance.now();
    for (const v of values) fn(v);
    const elapsed = performance.now() - start;
    console.log(`${name}: ${elapsed.toFixed(2)}ms (10,000回)`);
  }
}

benchmark();
```

実測結果（Chrome, M1 Mac）:

| 手法 | 10,000回 | 備考 |
|------|---------|------|
| Intl.NumberFormat（再利用） | ~3ms | **最速**。インスタンス再利用が前提 |
| Intl.NumberFormat（毎回生成） | ~80ms | 生成コストが高い |
| 正規表現 | ~5ms | Intlとほぼ同等 |
| 手動ループ | ~4ms | 軽いが可読性が低い |

結論: **`Intl.NumberFormat` のインスタンスを再利用**すれば最速。毎回 `new` するとボトルネックになるので注意。

```javascript
// NG: 毎回インスタンス生成
function formatYenSlow(value) {
  return new Intl.NumberFormat('ja-JP').format(value) + '円';
}

// OK: インスタンスを使い回す
const jaFormatter = new Intl.NumberFormat('ja-JP', { maximumFractionDigits: 0 });

function formatYen(value) {
  const num = Math.floor(Number(value));
  if (isNaN(num)) return '0円';
  const abs = Math.abs(num);
  const formatted = jaFormatter.format(abs);
  return num < 0 ? `-${formatted}円` : `${formatted}円`;
}
```

## AND TOOLSでの最終実装

上記の知見をまとめた、実際に使っているユーティリティ。

```javascript
const jaNumberFormat = new Intl.NumberFormat('ja-JP', {
  maximumFractionDigits: 0,
});

const formatUtils = {
  // 基本: "1,234,567円"
  yen(value) {
    const num = Math.floor(Number(value));
    if (isNaN(num)) return '0円';
    const abs = Math.abs(num);
    return (num < 0 ? '-' : '') + jaNumberFormat.format(abs) + '円';
  },

  // 符号付き: "+1,234,567円" / "-1,234,567円"
  yenSigned(value) {
    const num = Math.floor(Number(value));
    if (isNaN(num) || num === 0) return '0円';
    const abs = Math.abs(num);
    const sign = num > 0 ? '+' : '-';
    return sign + jaNumberFormat.format(abs) + '円';
  },

  // 万円単位: "500万円" / "1,234万5,678円"
  yenMan(value) {
    const num = Math.floor(Number(value));
    if (isNaN(num)) return '0円';
    const abs = Math.abs(num);
    const sign = num < 0 ? '-' : '';
    if (abs >= 10000 && abs % 10000 === 0) {
      return sign + jaNumberFormat.format(abs / 10000) + '万円';
    }
    return sign + jaNumberFormat.format(abs) + '円';
  },

  // パーセント: "10.21%"
  percent(value, digits = 2) {
    const num = Number(value);
    if (isNaN(num)) return '0%';
    return num.toFixed(digits) + '%';
  },
};

// 使用例
console.log(formatUtils.yen(5000000));       // "5,000,000円"
console.log(formatUtils.yenSigned(-30000));  // "-30,000円"
console.log(formatUtils.yenMan(3500000));    // "3,500,000円"
console.log(formatUtils.yenMan(5000000));    // "500万円"
console.log(formatUtils.percent(10.21));     // "10.21%"
```

## まとめ

- `Intl.NumberFormat` は便利だが `style: 'currency'` は**全角￥問題**がある
- マイナス値の表示は**ブラウザ依存**。手動制御が安全
- **`maximumFractionDigits: 0`** を指定して小数を防ぐ
- インスタンスの**再利用**が速度の鍵。毎回 `new` しない
- 万・億の読みやすい表示は別関数で用意する

## 関連ツール・サイト

- [AND TOOLS](https://and-tools.net/) — 全ツールで統一したフォーマットユーティリティを使用
- [ToolShare Lab](https://webatives.com/tools/) — 同様の日本円フォーマットを採用

地味な処理だが、計算ツールでは「数値の見やすさ」がユーザー体験に直結する。
