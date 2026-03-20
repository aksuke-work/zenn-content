---
title: "簡易課税のみなし仕入率テーブル実装 — 6業種の判定ロジック"
emoji: "🧮"
type: "tech"
topics: ["javascript", "消費税", "簡易課税", "個人開発"]
published: false
---

消費税の簡易課税制度をJavaScriptで実装した。[AND TOOLS](https://and-tools.net/) のインボイス関連ツールを作るなかで、みなし仕入率の6業種テーブルと、2種以上の事業を営む場合の按分計算に取り組んだ記録。

## 簡易課税とは — エンジニア向けの説明

通常の消費税計算（本則課税）は「売上にかかる消費税 − 仕入にかかる消費税」で納税額を求める。一方、簡易課税は**仕入税額を実際に計算せず、売上の一定割合を「みなし仕入率」として控除**する。

```
本則課税: 納税額 = 売上消費税 - 仕入消費税（実額）
簡易課税: 納税額 = 売上消費税 - 売上消費税 × みなし仕入率
        = 売上消費税 × (1 - みなし仕入率)
```

適用条件:
- 基準期間の課税売上高が**5,000万円以下**
- 事前に**「簡易課税制度選択届出書」**を提出済み

## みなし仕入率テーブル

6業種で仕入率が異なる。

```javascript
const DEEMED_PURCHASE_RATES = [
  { type: 1, name: '卸売業',         rate: 0.90 },
  { type: 2, name: '小売業',         rate: 0.80 },
  { type: 3, name: '製造業等',       rate: 0.70 },
  { type: 4, name: 'その他の事業',   rate: 0.60 },
  { type: 5, name: 'サービス業等',   rate: 0.50 },
  { type: 6, name: '不動産業',       rate: 0.40 },
];

function getDeemedPurchaseRate(businessType) {
  const entry = DEEMED_PURCHASE_RATES.find(r => r.type === businessType);
  if (!entry) throw new Error(`Unknown business type: ${businessType}`);
  return entry.rate;
}
```

フリーランスのWeb制作やコンサルティングは**第5種（サービス業等）**に該当する。みなし仕入率は50%なので、売上消費税の半分を納税するイメージだ。

## 単一事業の計算

事業が1種類だけなら計算はシンプル。

```javascript
function calcSimplifiedTax(salesAmount, businessType) {
  const taxRate = 0.10;
  const salesTax = Math.floor(salesAmount * taxRate / (1 + taxRate));
  const deemedRate = getDeemedPurchaseRate(businessType);
  const deemedPurchaseTax = Math.floor(salesTax * deemedRate);
  const taxPayable = salesTax - deemedPurchaseTax;

  return {
    salesAmount,
    salesTax,
    deemedRate,
    deemedPurchaseTax,
    taxPayable,
  };
}

// 例: サービス業で年商800万円（税込）
const result = calcSimplifiedTax(8000000, 5);
console.log(result);
// {
//   salesAmount: 8000000,
//   salesTax: 727272,
//   deemedRate: 0.5,
//   deemedPurchaseTax: 363636,
//   taxPayable: 363636
// }
```

ポイント:
- 税込金額から消費税を抽出するには `金額 × 10 / 110`
- `Math.floor` で1円未満切捨て

## 2種以上の事業を営む場合 — 按分計算

ここからが複雑になる。2種類以上の事業を行っている場合、原則として**事業区分ごとに按分計算**する。

```javascript
function calcSimplifiedTaxMultiple(businessEntries) {
  // businessEntries: [{ type: 3, sales: 5000000 }, { type: 5, sales: 3000000 }]

  const taxRate = 0.10;

  // 全体の売上消費税
  const totalSales = businessEntries.reduce((sum, e) => sum + e.sales, 0);
  const totalSalesTax = Math.floor(totalSales * taxRate / (1 + taxRate));

  // 事業区分ごとのみなし仕入税額を計算
  let totalDeemedTax = 0;
  const breakdown = businessEntries.map(entry => {
    const salesTax = Math.floor(entry.sales * taxRate / (1 + taxRate));
    const rate = getDeemedPurchaseRate(entry.type);
    const deemedTax = Math.floor(salesTax * rate);
    totalDeemedTax += deemedTax;

    return {
      type: entry.type,
      sales: entry.sales,
      salesTax,
      rate,
      deemedTax,
    };
  });

  return {
    totalSales,
    totalSalesTax,
    totalDeemedTax,
    taxPayable: totalSalesTax - totalDeemedTax,
    breakdown,
  };
}

// 例: 製造業500万 + サービス業300万（ともに税込）
const result = calcSimplifiedTaxMultiple([
  { type: 3, sales: 5000000 },
  { type: 5, sales: 3000000 },
]);
console.log(result.taxPayable);
```

## 75%ルール — 特例計算

2種以上の事業を行っていても、**1つの事業の売上が全体の75%以上**を占める場合、その事業のみなし仕入率を全体に適用できる特例がある。

```javascript
function calcWithSpecialRule(businessEntries) {
  const totalSales = businessEntries.reduce((sum, e) => sum + e.sales, 0);

  // 75%ルールのチェック
  for (const entry of businessEntries) {
    const ratio = entry.sales / totalSales;
    if (ratio >= 0.75) {
      // この事業のみなし仕入率を全体に適用
      const rate = getDeemedPurchaseRate(entry.type);
      const totalSalesTax = Math.floor(totalSales * 0.10 / 1.10);
      const deemedTax = Math.floor(totalSalesTax * rate);

      return {
        specialRule: true,
        dominantType: entry.type,
        dominantRate: rate,
        totalSalesTax,
        deemedTax,
        taxPayable: totalSalesTax - deemedTax,
      };
    }
  }

  // 75%を超える事業がない → 原則計算
  return {
    specialRule: false,
    ...calcSimplifiedTaxMultiple(businessEntries),
  };
}
```

この特例は**みなし仕入率が高い事業（卸売業90%など）が75%以上**を占める場合に有利になる。逆にサービス業（50%）が支配的なら、製造業（70%）の売上にも50%が適用されるので不利。

実装時は原則計算と特例計算の**両方を計算して有利なほうを採用**するのが正しい。

```javascript
function calcOptimalSimplifiedTax(businessEntries) {
  const normalResult = calcSimplifiedTaxMultiple(businessEntries);
  const specialResult = calcWithSpecialRule(businessEntries);

  // 納税額が少ないほうを採用
  if (specialResult.specialRule &&
      specialResult.taxPayable < normalResult.taxPayable) {
    return specialResult;
  }
  return normalResult;
}
```

## 本則課税との比較判断

簡易課税を選ぶべきかどうかは、実際の仕入率との比較で決まる。

```javascript
function shouldUseSimplified(salesAmount, actualPurchases, businessType) {
  const taxRate = 0.10;
  const salesTax = Math.floor(salesAmount * taxRate / (1 + taxRate));

  // 本則課税の納税額
  const actualPurchaseTax = Math.floor(actualPurchases * taxRate / (1 + taxRate));
  const normalTax = salesTax - actualPurchaseTax;

  // 簡易課税の納税額
  const deemedRate = getDeemedPurchaseRate(businessType);
  const simplifiedTax = salesTax - Math.floor(salesTax * deemedRate);

  return {
    normalTax,
    simplifiedTax,
    recommendation: simplifiedTax < normalTax ? '簡易課税が有利' : '本則課税が有利',
    difference: Math.abs(normalTax - simplifiedTax),
  };
}

// 例: サービス業で売上800万、仕入200万（ともに税込）
console.log(shouldUseSimplified(8000000, 2000000, 5));
// → 簡易課税が有利（仕入が少ないサービス業の典型）
```

フリーランスのように**仕入が少ない業種**は、簡易課税のほうが有利になることが多い。

## まとめ

- みなし仕入率は6業種で**40%〜90%**。フリーランスは大半が第5種（50%）
- 2種以上は**按分計算**。75%ルールの特例もある
- 本則課税と簡易課税の**両方を試算して有利なほうを選ぶ**
- インボイス制度導入で消費税の計算は避けて通れなくなった

## 関連ツール・サイト

- [インボイス経過措置 計算ツール](https://and-tools.net/tools/invoice-transition/) — 2割特例・簡易課税の比較
- [消費税 計算ツール](https://and-tools.net/tools/consumption-tax/) — 税込⇔税抜の変換

全34ツールは [AND TOOLS](https://and-tools.net/) で無料公開中。
