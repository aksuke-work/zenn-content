---
title: "税金計算における二分探索の活用 — 非線形関数の逆問題を解く"
emoji: "🔎"
type: "tech"
topics: ["javascript", "algorithm", "二分探索", "個人開発"]
published: false
---

税金計算ツールサイト [AND TOOLS](https://and-tools.net/) では「手取り○○万円にするには年収いくら？」「時給○○円で年収○○万円にするには月何時間？」といった**逆算**を多用している。この逆算に**二分探索**が驚くほど相性がいい。

## なぜ税金の逆算は難しいのか

税金の順方向計算（年収 → 手取り）は単純な関数の組み合わせだ:

```
手取り = 年収 - 所得税 - 住民税 - 社会保険料
```

ところが逆方向（手取り → 年収）は一筋縄ではいかない。理由は3つ。

### 1. 累進課税の非線形性

所得税は7段階の累進課税。年収が増えると税率がジャンプし、関数が区分線形になる。

```javascript
// 所得税の順方向計算
function calcIncomeTax(taxableIncome) {
  const brackets = [
    { limit: 1950000,  rate: 0.05, deduction: 0 },
    { limit: 3300000,  rate: 0.10, deduction: 97500 },
    { limit: 6950000,  rate: 0.20, deduction: 427500 },
    { limit: 9000000,  rate: 0.23, deduction: 636000 },
    { limit: 18000000, rate: 0.33, deduction: 1536000 },
    { limit: 40000000, rate: 0.40, deduction: 2796000 },
    { limit: Infinity,  rate: 0.45, deduction: 4796000 },
  ];

  for (const b of brackets) {
    if (taxableIncome <= b.limit) {
      return Math.floor(taxableIncome * b.rate - b.deduction);
    }
  }
  return 0;
}
```

逆関数を数式で求めることは可能だが、ブラケットの境界をまたぐケースの処理が面倒になる。

### 2. 控除の段階的変化

基礎控除（2026年〜の9段階テーブル）、配偶者控除、給与所得控除...これらはすべて所得に応じて段階的に変化する。**入力が変わると控除額も変わり、控除額が変わると課税所得が変わり、課税所得が変わると税額が変わる**。循環する依存関係がある。

### 3. Math.floorによる情報損失

国税の端数処理は1円未満切捨て。`Math.floor` を通すと情報が失われ、厳密な逆関数が存在しなくなる。

```javascript
Math.floor(123456 * 0.1021); // 12,608
// 12,608 から元の 123,456 を復元できるか？
// 12608 / 0.1021 = 123,487.76... → 123,456 に戻らない
```

## 二分探索の基本実装

これらの問題をすべて解決するのが二分探索だ。順方向の関数さえ正確なら、逆算は機械的にできる。

```javascript
function binarySearchReverse(targetNet, calcNetFromGross) {
  let low = 0;
  let high = targetNet * 3; // 十分大きな上限

  let iterations = 0;
  while (high - low > 1) {
    const mid = Math.floor((low + high) / 2);
    const net = calcNetFromGross(mid);

    if (net < targetNet) {
      low = mid;
    } else {
      high = mid;
    }
    iterations++;
  }

  console.log(`収束まで ${iterations} 回`);
  return high;
}
```

前提条件:
- `calcNetFromGross` が**単調増加**であること（年収が増えれば手取りも増える）
- 税金計算ではこの前提がほぼ成り立つ（限界税率100%超はありえない）

## 実践: 手取りから年収を逆算

```javascript
function calcTakeHomePay(grossSalary) {
  // 給与所得控除
  const salaryDeduction = calcSalaryDeduction(grossSalary);
  const salaryIncome = grossSalary - salaryDeduction;

  // 社会保険料（概算）
  const socialInsurance = Math.floor(grossSalary * 0.15);

  // 基礎控除
  const basicDeduction = 580000;

  // 課税所得
  const taxableIncome = Math.max(
    Math.floor((salaryIncome - socialInsurance - basicDeduction) / 1000) * 1000,
    0
  );

  // 所得税 + 復興特別所得税
  const incomeTax = Math.floor(calcIncomeTax(taxableIncome) * 1.021);

  // 住民税（簡略）
  const residentTax = Math.floor(taxableIncome * 0.10) + 5000;

  return grossSalary - incomeTax - residentTax - socialInsurance;
}

function calcSalaryDeduction(income) {
  if (income <= 1625000) return 550000;
  if (income <= 1800000) return Math.floor(income * 0.4 - 100000);
  if (income <= 3600000) return Math.floor(income * 0.3 + 80000);
  if (income <= 6600000) return Math.floor(income * 0.2 + 440000);
  if (income <= 8500000) return Math.floor(income * 0.1 + 1100000);
  return 1950000;
}

// 「手取り400万」に必要な年収は？
const requiredGross = binarySearchReverse(4000000, calcTakeHomePay);
console.log(`必要年収: ${requiredGross.toLocaleString()}円`);
console.log(`検証 手取り: ${calcTakeHomePay(requiredGross).toLocaleString()}円`);
```

## 収束条件と精度

二分探索の収束回数は `O(log2(探索範囲))` で決まる。

```javascript
function analyzeConvergence(targetNet) {
  const searchRange = targetNet * 3;
  const theoreticalIterations = Math.ceil(Math.log2(searchRange));

  console.log(`探索範囲: 0 〜 ${searchRange.toLocaleString()}`);
  console.log(`理論上の最大反復回数: ${theoreticalIterations}`);

  // 実測
  let low = 0;
  let high = searchRange;
  let count = 0;

  while (high - low > 1) {
    const mid = Math.floor((low + high) / 2);
    const net = calcTakeHomePay(mid);
    if (net < targetNet) low = mid;
    else high = mid;
    count++;
  }

  console.log(`実際の反復回数: ${count}`);
  return { theoretical: theoreticalIterations, actual: count };
}

// 手取り500万 → 探索範囲1500万
analyzeConvergence(5000000);
// 理論: 24回、実測: 24回程度
```

年収1,500万円の範囲を探索しても**24回のループ**で1円単位の精度が出る。パフォーマンスの心配は不要だ。

## 精度1円 vs 100円 — トレードオフ

用途によっては1円の精度は不要。ツールのUIでは「万円単位」で表示することも多い。

```javascript
function binarySearchWithPrecision(targetNet, calcNet, precision = 1) {
  let low = 0;
  let high = targetNet * 3;

  while (high - low > precision) {
    const mid = Math.floor((low + high) / 2);
    const net = calcNet(mid);
    if (net < targetNet) low = mid;
    else high = mid;
  }

  // precisionに応じた丸め
  return Math.ceil(high / precision) * precision;
}

// 1円単位（デフォルト）
console.log(binarySearchWithPrecision(4000000, calcTakeHomePay, 1));

// 1万円単位（UIで十分な場合）
console.log(binarySearchWithPrecision(4000000, calcTakeHomePay, 10000));
```

`precision = 10000` にすると反復回数は**10回程度**に減る。ユーザーに「概算」で十分な場合はこれで良い。

## 応用: フリーランスの時給逆算

「年収600万にするには時給いくら？」を二分探索で解く。

```javascript
function calcAnnualFromHourlyRate(hourlyRate, hoursPerMonth = 160) {
  const monthlyRevenue = hourlyRate * hoursPerMonth;
  const annualRevenue = monthlyRevenue * 12;

  // 個人事業主の手取り計算（簡略版）
  const expenses = Math.floor(annualRevenue * 0.1); // 経費10%想定
  const income = annualRevenue - expenses;
  const basicDeduction = 580000;
  const blueDeduction = 650000;
  const taxable = Math.max(income - basicDeduction - blueDeduction, 0);
  const incomeTax = Math.floor(calcIncomeTax(taxable) * 1.021);
  const residentTax = Math.floor(taxable * 0.10) + 5000;
  const socialInsurance = Math.floor(income * 0.15);

  return annualRevenue - incomeTax - residentTax - socialInsurance;
}

function findHourlyRate(targetAnnualNet, hoursPerMonth = 160) {
  let low = 0;
  let high = 50000; // 時給5万円を上限

  while (high - low > 1) {
    const mid = Math.floor((low + high) / 2);
    const net = calcAnnualFromHourlyRate(mid, hoursPerMonth);
    if (net < targetAnnualNet) low = mid;
    else high = mid;
  }

  return high;
}

const rate = findHourlyRate(6000000);
console.log(`手取り600万に必要な時給: ${rate.toLocaleString()}円`);
```

順方向関数（時給 → 年間手取り）が正確に書ければ、逆算は二分探索に丸投げできる。**新しい逆算の数式を導出する必要がない**のが最大のメリット。

## まとめ

- 税金計算の逆算は**累進課税・段階的控除・Math.floor** のせいで数式では解きにくい
- **二分探索なら順方向関数だけあれば逆算できる**
- 年収1,500万の範囲でも**24回**で1円精度に収束する
- 精度を落とせば反復回数をさらに減らせる
- 新しい逆算を追加するコストが**ほぼゼロ**

## 関連ツール・サイト

- [フリーランス単価 計算ツール](https://and-tools.net/tools/freelance-rate-calc/) — 時給・月額・年収の逆算
- [手取りから逆算ツール](https://and-tools.net/tools/reverse-salary/) — 手取り → 必要年収

全34ツールは [AND TOOLS](https://and-tools.net/) で無料公開中。
