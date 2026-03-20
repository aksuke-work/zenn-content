---
title: "インボイス経過措置の消費税計算をプログラムで理解する — 80%→50%→0%の3段階シミュレーション"
emoji: "🧾"
type: "tech"
topics: ["javascript", "tax", "インボイス", "個人開発"]
published: false
---

インボイス制度が始まって2年以上経つが、経過措置の計算が複雑すぎて正確に理解しているエンジニアは少ない。[AND TOOLS](https://and-tools.net/) でインボイス関連ツールを作った経験をもとに、プログラムで整理する。

## インボイス経過措置の全体像

免税事業者からの仕入れに対する仕入税額控除が、3段階で縮小される。

| 期間 | 控除率 |
|---|---|
| 2023年10月〜2026年9月 | **80%** |
| 2026年10月〜2029年9月 | **50%** |
| 2029年10月〜 | **0%**（控除不可） |

つまり、課税事業者が免税事業者に110万円（税込）を支払った場合:

```javascript
function calcDeductibleTax(totalAmount, transactionDate) {
  const taxAmount = Math.floor(totalAmount * 10 / 110); // 消費税額

  const rate = getTransitionRate(transactionDate);
  return Math.floor(taxAmount * rate);
}

function getTransitionRate(date) {
  const d = new Date(date);
  if (d < new Date('2023-10-01')) return 1.0;   // 制度開始前: 全額控除
  if (d < new Date('2026-10-01')) return 0.8;    // 80%控除
  if (d < new Date('2029-10-01')) return 0.5;    // 50%控除
  return 0.0;                                     // 控除不可
}
```

実装: [インボイス経過措置 計算ツール](https://and-tools.net/tools/invoice-transition/)

## 消費税の按分計算 — 10/110 の意味

税込金額から消費税を抜き出すとき、`× 10 / 110` という計算をする。

```javascript
// 税込 1,100,000円 の場合
const taxIncluded = 1100000;
const taxAmount = Math.floor(taxIncluded * 10 / 110); // 100,000円
const preTax = taxIncluded - taxAmount;                // 1,000,000円
```

なぜ `/ 1.1` ではなく `* 10 / 110` か:
- 浮動小数点数の誤差を避けるため
- `1100000 / 1.1` は `999999.9999999999` になる場合がある
- 整数演算 `1100000 * 10 / 110` なら正確に `100000` が出る

```javascript
// 浮動小数点の罠
console.log(1100000 / 1.1);       // 999999.9999999999（環境依存）
console.log(1100000 * 10 / 110);  // 1000000（正確）
```

## 課税事業者 vs 免税事業者の2つの計算パス

インボイス制度では、自分が課税事業者か免税事業者かで計算が全く変わる。

```javascript
function calcConsumptionTax(revenue, expenses, options = {}) {
  const {
    isTaxable = true,        // 課税事業者か
    useSimplified = false,   // 簡易課税か
    use20PercentRule = false, // 2割特例か
    businessType = 5,        // 簡易課税の事業区分（1-6）
  } = options;

  // 免税事業者: 消費税の申告義務なし
  if (!isTaxable) {
    return { taxPayable: 0, note: '免税事業者のため申告不要' };
  }

  const receivedTax = Math.floor(revenue * 10 / 110);

  // 2割特例（2026年分まで適用可能）
  if (use20PercentRule) {
    return {
      taxPayable: Math.floor(receivedTax * 0.2),
      note: '2割特例適用',
    };
  }

  // 簡易課税
  if (useSimplified) {
    const deemRate = getDeemedPurchaseRate(businessType);
    const deductible = Math.floor(receivedTax * deemRate);
    return {
      taxPayable: receivedTax - deductible,
      note: `簡易課税（みなし仕入率${deemRate * 100}%）`,
    };
  }

  // 本則課税
  const paidTax = Math.floor(expenses * 10 / 110);
  return {
    taxPayable: receivedTax - paidTax,
    note: '本則課税',
  };
}
```

実装: [消費税 計算ツール](https://and-tools.net/tools/consumption-tax/)

## 2割特例 — 最もシンプルな計算

インボイス制度で新たに課税事業者になった人向けの特例。預かった消費税の20%だけ納めればいい。

```javascript
function calc20PercentRule(annualRevenue) {
  const receivedTax = Math.floor(annualRevenue * 10 / 110);
  const taxPayable = Math.floor(receivedTax * 0.2);

  return {
    receivedTax,
    taxPayable,
    effectiveRate: (taxPayable / annualRevenue * 100).toFixed(2),
  };
}

// 年商550万円（税込）の場合
const result = calc20PercentRule(5500000);
// receivedTax: 500,000円
// taxPayable: 100,000円（実質負担率 1.82%）
```

2割特例は2026年分の申告まで使える。フリーランスエンジニアの多くがこの特例を使っているはず。

## 簡易課税のみなし仕入率テーブル

簡易課税は業種ごとの「みなし仕入率」で控除額を計算する。

```javascript
function getDeemedPurchaseRate(businessType) {
  const rates = {
    1: 0.90, // 卸売業
    2: 0.80, // 小売業
    3: 0.70, // 製造業等
    4: 0.60, // その他（飲食店等）
    5: 0.50, // サービス業等（エンジニアはここ）
    6: 0.40, // 不動産業
  };

  return rates[businessType] || 0.50;
}
```

エンジニア・デザイナーなどのフリーランスは**第5種（サービス業）**に該当するので、みなし仕入率は50%。つまり預かり消費税の半分を納める。

```javascript
// フリーランスエンジニアの例: 年商880万円（税込）
const revenue = 8800000;
const receivedTax = Math.floor(revenue * 10 / 110); // 800,000円
const deductible = Math.floor(receivedTax * 0.5);    // 400,000円
const taxPayable = receivedTax - deductible;          // 400,000円

// 2割特例なら: 800,000 × 0.2 = 160,000円
// → 2割特例のほうが24万円お得
```

## 3段階経過措置のシミュレーション

免税事業者と取引する場合のコスト増を、年度ごとにシミュレーションする。

```javascript
function simulateTransitionCost(annualPurchase, years) {
  const results = [];

  for (const year of years) {
    const date = `${year}-04-01`; // 年度開始日
    const taxAmount = Math.floor(annualPurchase * 10 / 110);
    const rate = getTransitionRate(date);
    const deductible = Math.floor(taxAmount * rate);
    const nonDeductible = taxAmount - deductible;

    results.push({
      year,
      rate: `${rate * 100}%`,
      deductible,
      nonDeductible,
      additionalCost: nonDeductible,
    });
  }

  return results;
}

// 免税事業者への年間支払い 330万円（税込）の場合
const sim = simulateTransitionCost(3300000, [2024, 2025, 2027, 2030]);

// 2024: 控除率80% → 追加コスト 60,000円
// 2025: 控除率80% → 追加コスト 60,000円
// 2027: 控除率50% → 追加コスト 150,000円
// 2030: 控除率0%  → 追加コスト 300,000円
```

実装: [インボイス 計算ツール](https://and-tools.net/tools/invoice-calc/)

## 判定フローチャートをコードで書く

「自分はどの制度を使うべきか」を判定するロジック。

```javascript
function recommendTaxMethod(options) {
  const { revenue, expenses, isNewTaxable, baseYearRevenue } = options;

  // 免税判定: 基準期間の売上が1,000万以下
  if (baseYearRevenue <= 10000000 && !isNewTaxable) {
    return { method: '免税', reason: '基準期間の売上1,000万以下' };
  }

  const receivedTax = Math.floor(revenue * 10 / 110);
  const paidTax = Math.floor(expenses * 10 / 110);

  // 3つの方法で税額を計算して比較
  const honzei = receivedTax - paidTax;
  const simplified = Math.floor(receivedTax * 0.5); // 第5種
  const twentyPercent = Math.floor(receivedTax * 0.2);

  const methods = [
    { method: '本則課税', tax: honzei },
    { method: '簡易課税', tax: simplified },
  ];

  // 2割特例は条件付き
  if (isNewTaxable) {
    methods.push({ method: '2割特例', tax: twentyPercent });
  }

  // 最も税額が少ない方法を推奨
  methods.sort((a, b) => a.tax - b.tax);
  return methods[0];
}
```

## まとめ

1. **経過措置は3段階**で控除率が下がる（80%→50%→0%）
2. **按分計算は `* 10 / 110`** で浮動小数点誤差を回避
3. **2割特例**はフリーランスにとって最もお得な選択肢（2026年分まで）
4. **簡易課税のみなし仕入率**はエンジニアなら50%（第5種）
5. 本則・簡易・2割特例を**全部計算して比較**するのが正しい判断方法

## AND TOOLSについて

[AND TOOLS](https://and-tools.net/) では、フリーランス・個人事業主向けの税金計算ツールを34個以上無料で公開しています。この記事で紹介した計算ロジックを実際に試せます。
