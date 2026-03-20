---
title: "源泉徴収税の10.21%計算と逆算ロジック — 手取りから請求額を求める"
emoji: "✂️"
type: "tech"
topics: ["javascript", "源泉徴収", "フリーランス", "個人開発"]
published: false
---

フリーランスとして働いていると、源泉徴収税の計算は避けて通れない。[AND TOOLS](https://and-tools.net/) で源泉徴収の計算・逆算ツールを作った経験から、このロジックを詳しく解説する。

## 源泉徴収税の基本 — なぜ10.21%なのか

フリーランスへの報酬から天引きされる源泉徴収税は、所得税10% + 復興特別所得税0.21% = **10.21%**。

ただし100万円を超える部分は税率が倍になる。

```javascript
function calcWithholding(grossAmount) {
  if (grossAmount <= 0) return 0;

  if (grossAmount <= 1000000) {
    // 100万以下: 10.21%
    return Math.floor(grossAmount * 0.1021);
  }

  // 100万超: 100万までは10.21%、超過分は20.42%
  const baseTax = Math.floor(1000000 * 0.1021);    // 102,100円
  const overTax = Math.floor((grossAmount - 1000000) * 0.2042);
  return baseTax + overTax;
}
```

ポイント:
- `Math.floor` で**1円未満切捨て**。国税の端数処理ルール
- 100万円超の部分が20.42%になるのは、所得税20% + 復興特別所得税0.42%
- 復興特別所得税は2037年末で終了予定

実際に計算してみる:

```javascript
console.log(calcWithholding(500000));   // 51,050円
console.log(calcWithholding(1000000));  // 102,100円
console.log(calcWithholding(1500000));  // 204,200円（102,100 + 102,100）
console.log(calcWithholding(3000000));  // 510,500円（102,100 + 408,400）
```

## 逆算 — 手取りから請求額を求める

フリーランスが「手取り50万円欲しい」と思ったとき、クライアントにいくら請求すればいいか。これが意外と難しい。

### 数式による逆算

```javascript
function reverseWithholding(netAmount) {
  if (netAmount <= 0) return 0;

  // 100万以下の手取り上限: 1,000,000 × (1 - 0.1021) = 897,900円
  if (netAmount <= 897900) {
    // 手取り = 報酬 × (1 - 0.1021)
    // 報酬 = 手取り / 0.8979
    return Math.ceil(netAmount / 0.8979);
  }

  // 100万超の場合
  // 手取り = 897,900 + (報酬 - 1,000,000) × (1 - 0.2042)
  // 手取り = 897,900 + (報酬 - 1,000,000) × 0.7958
  // 報酬 = (手取り - 897,900) / 0.7958 + 1,000,000
  return Math.ceil((netAmount - 897900) / 0.7958 + 1000000);
}
```

ここで `Math.ceil`（切り上げ）を使うのがポイント。切り捨てだと手取りが目標を下回る可能性がある。

### 数式による逆算の問題点

実は上の数式には落とし穴がある。100万円のボーダー付近で**1円ズレる**ことがある。

```javascript
// 報酬100万 → 源泉 102,100 → 手取り 897,900
console.log(reverseWithholding(897900));  // 1,000,000 ✓

// 報酬100万1円 → 源泉 102,100 → 手取り 897,901
// でも数式で逆算すると...
console.log(reverseWithholding(897901));  // 1,000,002?
```

原因は `Math.floor` と `Math.ceil` の組み合わせによる端数の不一致。

### 二分探索による確実な逆算

端数問題を完全に解消するには、二分探索が最も安全。

```javascript
function reverseWithholdingSafe(targetNet) {
  if (targetNet <= 0) return 0;

  let low = targetNet;
  let high = Math.ceil(targetNet * 1.3); // 余裕を持った上限

  while (high - low > 1) {
    const mid = Math.floor((low + high) / 2);
    const tax = calcWithholding(mid);
    const net = mid - tax;

    if (net < targetNet) {
      low = mid;
    } else {
      high = mid;
    }
  }

  // 検証: lowとhighの両方をチェック
  const taxLow = calcWithholding(low);
  if (low - taxLow === targetNet) return low;
  return high;
}
```

二分探索なら:
- 端数処理の不一致を気にしなくていい
- `calcWithholding` を正として、それと整合する逆算ができる
- 収束は**約20回のループ**で済む（log2(報酬額)回）

## インボイス制度との関係

2023年10月のインボイス制度導入で、源泉徴収の計算に**消費税の扱い**が影響するようになった。

```javascript
function calcWithholdingWithTax(amount, isTaxIncluded, taxRate = 0.10) {
  if (isTaxIncluded) {
    // 税込金額から源泉徴収を計算
    // 請求書で消費税が区分されている場合は税抜で計算可能
    const taxExcluded = Math.floor(amount / (1 + taxRate));
    return calcWithholding(taxExcluded);
  }

  // 税抜金額からそのまま計算
  return calcWithholding(amount);
}
```

重要なルール:
- 請求書で消費税額が**明確に区分**されていれば、**税抜金額**に対して源泉徴収を計算できる
- 区分されていなければ**税込金額の全額**が源泉徴収の対象

フリーランスにとっては消費税を区分して記載するほうが源泉徴収額が減るため、手取りが増える。

## 実務でのよくある間違い

### 1. 復興特別所得税の漏れ

```javascript
// NG: 10%で計算してしまう
const tax = amount * 0.10;

// OK: 10.21%で計算する
const tax = Math.floor(amount * 0.1021);
```

### 2. 100万円超の二段階計算を忘れる

```javascript
// NG: 全額に20.42%をかけてしまう
const tax = Math.floor(1500000 * 0.2042); // 306,300（間違い）

// OK: 100万までは10.21%、超過分は20.42%
const tax = Math.floor(1000000 * 0.1021)
          + Math.floor(500000 * 0.2042);  // 204,200（正しい）
```

### 3. 源泉徴収が不要な報酬

すべての報酬に源泉徴収がかかるわけではない:
- **必要**: 原稿料、デザイン料、講演料、コンサルティング料
- **不要**: 物品の売買代金、Webサイト制作費（※判断が難しいケースあり）

Webサイト制作が「デザイン」と「プログラミング」のどちらに分類されるかで源泉徴収の要否が変わる。実務上は税理士に確認するのが安全だ。

## まとめ

源泉徴収の計算ロジックをまとめると:

- **順方向**: 10.21%（100万以下）/ 20.42%（100万超）の二段階
- **逆方向**: 数式でも解けるが、**二分探索が端数に強い**
- **インボイス制度**: 消費税を区分記載すれば税抜金額で計算可能
- **復興特別所得税**: 2037年末まで加算。忘れがち

## 関連ツール・サイト

- [源泉徴収税 計算ツール](https://and-tools.net/tools/tax-withholding/) — 順方向・逆方向の両方に対応
- [インボイス計算ツール](https://and-tools.net/tools/invoice-calc/) — インボイス制度対応の請求書計算

全34ツールは [AND TOOLS](https://and-tools.net/) で無料公開中。
