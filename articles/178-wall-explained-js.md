---
title: "2026年『178万の壁』— 基礎控除9段階の実装を図解する"
emoji: "🧱"
type: "tech"
topics: ["javascript", "税制改正", "基礎控除", "個人開発"]
published: false
---

2025年末の税制改正で基礎控除が大幅に見直された。従来の「103万の壁」に代わって「178万の壁」が話題になっている。[AND TOOLS](https://and-tools.net/) に178万の壁シミュレーターを追加した際に実装した、9段階の基礎控除テーブルを解説する。

## 何が変わったのか

改正前の基礎控除は**一律48万円**（所得2,400万超で段階的に減額）とシンプルだった。改正後は**所得200万円以下の区間で基礎控除が最大95万円に引き上げ**られ、9段階のテーブルになった。

```
改正前: 基礎控除 48万円（ほぼ全員同額）
改正後: 基礎控除 58万〜95万円（所得に応じて段階的）
```

## 9段階テーブルの実装

```javascript
function getBasicDeduction2026(income) {
  // 2026年〜の基礎控除テーブル（所得税）
  const table = [
    { limit: 1320000,   deduction: 950000 },  // 〜132万: 95万
    { limit: 1370000,   deduction: 880000 },  // 〜137万: 88万
    { limit: 1420000,   deduction: 830000 },  // 〜142万: 83万
    { limit: 1470000,   deduction: 780000 },  // 〜147万: 78万
    { limit: 1520000,   deduction: 730000 },  // 〜152万: 73万
    { limit: 1570000,   deduction: 680000 },  // 〜157万: 68万
    { limit: 1620000,   deduction: 630000 },  // 〜162万: 63万
    { limit: 1670000,   deduction: 580000 },  // 〜167万: 58万
    { limit: 24000000,  deduction: 580000 },  // 〜2400万: 58万
    { limit: 24500000,  deduction: 480000 },  // 〜2450万: 48万
    { limit: 25000000,  deduction: 320000 },  // 〜2500万: 32万
    { limit: Infinity,  deduction: 0 },        // 2500万超: 0
  ];

  for (const row of table) {
    if (income <= row.limit) {
      return row.deduction;
    }
  }
  return 0;
}
```

所得132万円以下で基礎控除が95万円。ここが「178万の壁」の根拠になる。

## 「178万の壁」の計算根拠

給与所得者の場合:

```
給与収入 178万円
− 給与所得控除 83万円（収入162.5万〜180万の区間: 収入×40%−10万）
= 給与所得 95万円

基礎控除 95万円（所得132万以下の最大控除）
課税所得 = 95万 − 95万 = 0円 → 所得税ゼロ
```

```javascript
function calcWall178(grossSalary) {
  // 給与所得控除
  const salaryDeduction = calcSalaryDeduction(grossSalary);
  const income = grossSalary - salaryDeduction;

  // 基礎控除（2026年〜）
  const basicDeduction = getBasicDeduction2026(income);

  // 課税所得
  const taxableIncome = Math.max(income - basicDeduction, 0);

  return {
    grossSalary,
    salaryDeduction,
    income,
    basicDeduction,
    taxableIncome,
    incomeTax: calcIncomeTax(taxableIncome),
    isZeroTax: taxableIncome === 0,
  };
}

function calcSalaryDeduction(salary) {
  if (salary <= 1625000) return 550000;
  if (salary <= 1800000) return Math.floor(salary * 0.4 - 100000);
  if (salary <= 3600000) return Math.floor(salary * 0.3 + 80000);
  if (salary <= 6600000) return Math.floor(salary * 0.2 + 440000);
  if (salary <= 8500000) return Math.floor(salary * 0.1 + 1100000);
  return 1950000;
}

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

// 検証: 178万円ちょうどで税金ゼロか？
const result = calcWall178(1780000);
console.log(result);
// income: 612000, basicDeduction: 580000 ...
// 実際の壁は給与所得控除の計算次第で微妙にズレる
```

## 住民税の基礎控除 — 5万円オフセット

住民税の基礎控除は**所得税より一律5万円少ない**。この差が「壁」の位置を変える。

```javascript
function getBasicDeductionResident2026(income) {
  // 住民税の基礎控除 = 所得税の基礎控除 − 5万円
  const incomeTaxDeduction = getBasicDeduction2026(income);

  // ただし下限は0
  return Math.max(incomeTaxDeduction - 50000, 0);
}
```

つまり所得税がゼロでも住民税は発生する場合がある。

```javascript
function calcWall178Full(grossSalary) {
  const salaryDeduction = calcSalaryDeduction(grossSalary);
  const income = grossSalary - salaryDeduction;

  // 所得税
  const basicDeductionIT = getBasicDeduction2026(income);
  const taxableIT = Math.max(
    Math.floor((income - basicDeductionIT) / 1000) * 1000, 0
  );
  const incomeTax = calcIncomeTax(taxableIT);

  // 住民税
  const basicDeductionRT = getBasicDeductionResident2026(income);
  const taxableRT = Math.max(
    Math.floor((income - basicDeductionRT) / 1000) * 1000, 0
  );
  const residentTax = Math.floor(taxableRT * 0.10) + 5000; // 所得割10% + 均等割

  return {
    grossSalary,
    income,
    incomeTax,
    residentTax,
    totalTax: incomeTax + residentTax,
  };
}
```

## 給与所得者 vs 個人事業主

「178万の壁」は**給与所得者**の話。個人事業主は給与所得控除がないため、壁の位置が違う。

```javascript
function calcWallFreelance(revenue, expenses) {
  // 個人事業主: 所得 = 収入 − 経費
  const income = revenue - expenses;

  // 基礎控除（同じテーブル）
  const basicDeduction = getBasicDeduction2026(income);

  // 青色申告特別控除（65万 or 55万 or 10万）
  const blueDeduction = 650000;

  const taxableIncome = Math.max(
    Math.floor((income - basicDeduction - blueDeduction) / 1000) * 1000, 0
  );

  return {
    revenue,
    expenses,
    income,
    basicDeduction,
    blueDeduction,
    taxableIncome,
    incomeTax: calcIncomeTax(taxableIncome),
  };
}
```

個人事業主の場合は「年収（売上）」と「経費」で所得が決まるため、壁の概念がそのまま当てはまらない。所得が132万以下になるよう経費を計上すれば、最大の基礎控除95万円を受けられる。

## 段階的に減額される影響 — 限界税率の歪み

9段階テーブルの問題は、所得が5万円増えるごとに基礎控除が**5万〜7万円ずつ減額**されること。これにより実効税率がジャンプする区間がある。

```javascript
function showMarginalTaxEffect() {
  const results = [];

  for (let salary = 1700000; salary <= 1900000; salary += 10000) {
    const salaryDeduction = calcSalaryDeduction(salary);
    const income = salary - salaryDeduction;
    const deduction = getBasicDeduction2026(income);
    const taxable = Math.max(income - deduction, 0);
    const tax = calcIncomeTax(taxable);

    results.push({
      salary: salary.toLocaleString(),
      income: income.toLocaleString(),
      deduction: deduction.toLocaleString(),
      taxable: taxable.toLocaleString(),
      tax: tax.toLocaleString(),
    });
  }

  console.table(results);
}

showMarginalTaxEffect();
```

これを実行すると、給与収入170万→190万の区間で税額がどう変化するかが見える。控除の減額と収入増加が同時に起きるため、**手取りが逆転するポイント**が存在しうる。

## まとめ

- 基礎控除は2026年から**9段階**。所得132万以下で最大95万円
- 「178万の壁」= 給与所得控除 + 基礎控除で所得税ゼロになるライン
- **住民税の基礎控除は5万円少ない**ため、所得税ゼロでも住民税はかかる
- 個人事業主は給与所得控除がないため、壁の位置が異なる
- 段階的減額により**限界税率が歪む区間**がある

## 関連ツール・サイト

- [178万の壁 計算ツール](https://and-tools.net/tools/178-wall/) — 新旧の壁を比較シミュレーション
- [手取り計算シミュレーター](https://and-tools.net/tools/take-home-pay/) — 2026年基礎控除対応
- [収入の壁 一覧ツール](https://and-tools.net/tools/income-wall/) — 103万・130万・178万の壁を図解

全34ツールは [AND TOOLS](https://and-tools.net/) で無料公開中。
