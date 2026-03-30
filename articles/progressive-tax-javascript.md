---
title: "累進課税をJavaScriptで正確に実装する — 所得税の速算表と復興特別所得税"
emoji: "📊"
type: "tech"
topics: ["javascript", "tax", "algorithm", "個人開発"]
published: false
---

フリーランス向け税金計算ツールサイト [AND TOOLS](https://and-tools.net/) を開発・運営している。この記事では、日本の所得税の累進課税を正確にJavaScriptで実装する方法を解説する。

## 累進課税とは何か — エンジニア目線で

「年収600万だから税率20%で120万円」は**よくある誤解**。日本の所得税は**超過累進税率**を採用しており、所得を7つのブラケットに分割して、各ブラケットごとに異なる税率を適用する。

| 課税所得 | 税率 | 控除額 |
|---|---|---|
| 〜195万円 | 5% | 0円 |
| 〜330万円 | 10% | 97,500円 |
| 〜695万円 | 20% | 427,500円 |
| 〜900万円 | 23% | 636,000円 |
| 〜1,800万円 | 33% | 1,536,000円 |
| 〜4,000万円 | 40% | 2,796,000円 |
| 4,000万円超 | 45% | 4,796,000円 |

つまり、課税所得400万円の場合:
- 195万円 × 5% = 97,500円
- 135万円（330万-195万）× 10% = 135,000円
- 70万円（400万-330万）× 20% = 140,000円
- **合計: 372,500円**（実効税率は約9.3%）

## 素朴な実装 vs 速算表

ブラケットごとに分割して計算する素朴な実装を書くと、こうなる。

```javascript
function calcIncomeTaxNaive(taxableIncome) {
  const brackets = [
    { upper: 1950000, rate: 0.05 },
    { upper: 3300000, rate: 0.10 },
    { upper: 6950000, rate: 0.20 },
    { upper: 9000000, rate: 0.23 },
    { upper: 18000000, rate: 0.33 },
    { upper: 40000000, rate: 0.40 },
    { upper: Infinity, rate: 0.45 },
  ];

  let tax = 0;
  let prev = 0;

  for (const b of brackets) {
    if (taxableIncome <= prev) break;
    const taxable = Math.min(taxableIncome, b.upper) - prev;
    tax += taxable * b.rate;
    prev = b.upper;
  }

  return Math.floor(tax);
}
```

これは正しいが、国税庁の公式計算方法は**速算表**を使う。速算表は「所得 × 税率 - 控除額」の1行で同じ結果を出せる。

```javascript
function calcIncomeTax(taxableIncome) {
  const brackets = [
    { limit: 1950000, rate: 0.05, deduction: 0 },
    { limit: 3300000, rate: 0.10, deduction: 97500 },
    { limit: 6950000, rate: 0.20, deduction: 427500 },
    { limit: 9000000, rate: 0.23, deduction: 636000 },
    { limit: 18000000, rate: 0.33, deduction: 1536000 },
    { limit: 40000000, rate: 0.40, deduction: 2796000 },
    { limit: Infinity, rate: 0.45, deduction: 4796000 },
  ];

  for (const b of brackets) {
    if (taxableIncome <= b.limit) {
      return Math.floor(taxableIncome * b.rate - b.deduction);
    }
  }
  return 0;
}
```

速算表の `deduction` は、下位ブラケットとの税率差を吸収するための定数。数学的に素朴な方法と完全に同値になる。ループが1回で終わるので効率的だし、国税庁の計算方法と一致するので検算もしやすい。

実装: [所得税 計算ツール](https://and-tools.net/tools/income-tax/)

## 復興特別所得税 — 忘れがちな2.1%

2013年から2037年まで、所得税額に対して **2.1%** の復興特別所得税が加算される。

```javascript
function calcIncomeTaxWithReconstruction(taxableIncome) {
  const baseTax = calcIncomeTax(taxableIncome);
  const reconstructionTax = Math.floor(baseTax * 0.021);
  return baseTax + reconstructionTax;
}
```

注意点:
- 復興税は**所得に対して**ではなく、**所得税額に対して** 2.1%
- `Math.floor(baseTax * 0.021)` で1円未満切り捨て（国税の原則）
- 簡易的に `Math.floor(baseTax * 1.021)` と書くこともあるが、厳密には基本税額と復興税を分けて計算するのが正しい

## 2026年税制改正 — 基礎控除の段階的引き上げ

2025年度税制改正で、基礎控除が所得に応じて段階的に引き上げられた。これが実装上かなり厄介。

```javascript
function getBasicDeduction(totalIncome, year = 2026) {
  if (year < 2025) return 480000;

  // 2025年度改正: 基礎控除の引き上げ
  if (totalIncome <= 1320000) return 950000;   // 95万円
  if (totalIncome <= 1500000) return 880000;   // 88万円
  if (totalIncome <= 2100000) return 680000;   // 68万円
  if (totalIncome <= 2450000) return 580000;   // 58万円
  if (totalIncome <= 2500000) return 480000;   // 48万円（従来どおり）
  return 0; // 2,500万超は基礎控除なし
}
```

ポイント:
- 低所得層ほど控除額が大きい（最大95万円）
- 所得2,500万円超は基礎控除ゼロ（従来どおり）
- 年金受給者と給与所得者で異なる経過措置がある

税制改正のたびにテーブルが変わるので、**年度をパラメータとして受け取る設計**にしておくのが重要。実際のプロダクションコードでは、年度ごとのテーブルをオブジェクトで管理している。

実装: [178万の壁 計算ツール](https://and-tools.net/tools/178-wall/)

## 課税所得の算出 — 控除の積み上げ

所得税計算の全体像を組み立てると、こうなる。

```javascript
function calcFullIncomeTax(annualIncome, options = {}) {
  const { age = 30, iDeCo = 0, dependents = 0 } = options;

  // 1. 給与所得控除
  const salaryDeduction = calcSalaryDeduction(annualIncome);
  const salaryIncome = annualIncome - salaryDeduction;

  // 2. 所得控除の積み上げ
  const basicDeduction = getBasicDeduction(salaryIncome);
  const socialInsurance = estimateSocialInsurance(annualIncome, age);
  const dependentDeduction = dependents * 380000;

  const totalDeductions = basicDeduction + socialInsurance + iDeCo + dependentDeduction;

  // 3. 課税所得（0未満にはならない）
  const taxableIncome = Math.max(0, salaryIncome - totalDeductions);

  // 4. 所得税 + 復興税
  return calcIncomeTaxWithReconstruction(taxableIncome);
}
```

控除の種類が多いのが日本の税制の特徴で、ここに配偶者控除、医療費控除、生命保険料控除...と積み上がっていく。[AND TOOLSの手取り計算ツール](https://and-tools.net/tools/take-home-pay/)では、主要な控除を網羅して実装している。

## テストの書き方 — 国税庁の計算例で検証

税金計算のテストは、国税庁の公式計算例を使うのが確実。

```javascript
// 国税庁の速算表に基づく検算
console.assert(calcIncomeTax(1950000) === 97500);   // 195万×5%
console.assert(calcIncomeTax(3300000) === 232500);   // 330万×10%-97500
console.assert(calcIncomeTax(5000000) === 572500);   // 500万×20%-427500
console.assert(calcIncomeTax(10000000) === 1764000); // 1000万×33%-1536000
```

実際のプロダクトでは、年収100万〜2,000万まで100万刻みのテストケースを用意して、素朴な方法と速算表の結果が一致することも確認している。

## まとめ

累進課税の実装ポイント:

1. **速算表方式**を使えば1行で計算できる（ループ不要）
2. **復興特別所得税**は所得税額の2.1%（所得の2.1%ではない）
3. **基礎控除の段階的引き上げ**に対応するなら、年度パラメータが必要
4. **端数処理は切り捨て**が国税の原則（`Math.floor`）
5. **テストは国税庁の計算例**で検証する

## AND TOOLSについて

[AND TOOLS](https://and-tools.net/) では、フリーランス・個人事業主向けの税金計算ツールを34個以上無料で公開しています。この記事で紹介した計算ロジックを実際に試せます。
