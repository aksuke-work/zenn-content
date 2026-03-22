---
title: "年末調整のロジックをプログラマ視点で理解する — 概算源泉徴収から精算まで"
emoji: "📅"
type: "tech"
topics: ["javascript", "年末調整", "tax", "個人開発"]
published: true
---

フリーランス向け税金計算ツールサイト [AND TOOLS](https://and-tools.net/) を運営している。年末調整の計算ロジックを実装したときに「毎月の源泉徴収は"仮払い"だった」と改めて実感した。プログラマ視点で年末調整の全体像を解説する。

## 年末調整とは — システム設計で考える

年末調整を一言で表すと**「毎月の概算値と年間の確定値の差分を精算するバッチ処理」**だ。

毎月の給与からは概算の所得税（源泉徴収税）が天引きされるが、以下の理由で年間の正確な税額とズレる:

- 毎月の計算は**月額テーブル**による概算
- **配偶者控除・扶養控除**は年末に確定する
- **生命保険料控除**や**iDeCo控除**は年間額が必要
- 途中で**昇給・賞与**があると税率区分が変わる

12月の給与計算時にこのズレを精算するのが年末調整だ。

## 月次の概算源泉徴収

毎月の源泉徴収は「給与所得の源泉徴収税額表（月額表）」で求める。甲欄（扶養控除等申告書を提出済み）と乙欄（未提出）で大きく異なる。

```javascript
// 月額表の甲欄（簡略化版）
function calcMonthlyWithholding(monthlySalary, dependents) {
  // 社会保険料控除後の給与等の金額
  const socialInsurance = Math.floor(monthlySalary * 0.15);
  const taxable = monthlySalary - socialInsurance;

  // 扶養人数による控除（1人あたり約31,667円/月）
  const dependentDeduction = dependents * 31667;
  const adjusted = Math.max(taxable - dependentDeduction, 0);

  // 簡略化した月額テーブル（実際は細かく区分されている）
  const brackets = [
    { limit: 88000,  rate: 0,     deduction: 0 },
    { limit: 162500, rate: 0.05,  deduction: 0 },
    { limit: 275000, rate: 0.10,  deduction: 4170 },
    { limit: 579167, rate: 0.20,  deduction: 31600 },
    { limit: 750000, rate: 0.23,  deduction: 49000 },
    { limit: Infinity, rate: 0.33, deduction: 124000 },
  ];

  for (const b of brackets) {
    if (adjusted <= b.limit) {
      return Math.floor(adjusted * b.rate - b.deduction);
    }
  }
  return 0;
}
```

ポイント:
- 月額88,000円以下は源泉徴収ゼロ（甲欄の場合）
- 扶養親族1人につき月額で約31,667円の控除
- これはあくまで**概算**。年間の正確な税額ではない

## 年間税額の確定計算

年末調整では、1年間の給与総額から正確な年間税額を計算する。

```javascript
function calcAnnualTax(annualSalary, options = {}) {
  const {
    dependents = 0,        // 扶養親族の数
    spouseIncome = 0,      // 配偶者の所得
    lifeInsurance = 0,     // 生命保険料（支払額）
    ideco = 0,             // iDeCo拠出額
    socialInsurance = 0,   // 年間社会保険料
  } = options;

  // 1. 給与所得控除
  const salaryDeduction = calcSalaryDeduction(annualSalary);
  const salaryIncome = annualSalary - salaryDeduction;

  // 2. 所得控除の積み上げ
  let deductions = 0;
  deductions += 580000;                                // 基礎控除（2025年〜58万円）
  deductions += socialInsurance;                       // 社会保険料控除
  deductions += calcLifeInsuranceDeduction(lifeInsurance); // 生命保険料控除
  deductions += ideco;                                 // 小規模企業共済等掛金控除
  deductions += calcSpouseDeduction(salaryIncome, spouseIncome); // 配偶者控除
  deductions += dependents * 380000;                   // 扶養控除（一般）

  // 3. 課税所得
  const taxableIncome = Math.max(
    Math.floor((salaryIncome - deductions) / 1000) * 1000,
    0
  );

  // 4. 所得税（累進課税 + 復興特別所得税）
  return calcIncomeTax(taxableIncome);
}

function calcSalaryDeduction(income) {
  if (income <= 1625000) return 550000;
  if (income <= 1800000) return Math.floor(income * 0.4 - 100000);
  if (income <= 3600000) return Math.floor(income * 0.3 + 80000);
  if (income <= 6600000) return Math.floor(income * 0.2 + 440000);
  if (income <= 8500000) return Math.floor(income * 0.1 + 1100000);
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
```

課税所得の計算で `Math.floor(... / 1000) * 1000` としている。課税所得は**1,000円未満切捨て**が国税のルール。

## 過不足の精算

年末調整の核心は「毎月天引きした合計」と「年間の正確な税額」の差分だ。

```javascript
function calcYearEndAdjustment(monthlyRecords, annualTaxOptions) {
  // 毎月の源泉徴収税額の合計
  const totalWithheld = monthlyRecords.reduce(
    (sum, record) => sum + record.withheld, 0
  );

  // 年間の正確な税額
  const annualSalary = monthlyRecords.reduce(
    (sum, record) => sum + record.salary, 0
  );
  const correctTax = calcAnnualTax(annualSalary, annualTaxOptions);

  // 過不足
  const difference = totalWithheld - correctTax;

  return {
    totalWithheld,   // 年間源泉徴収済み額
    correctTax,      // 正しい年税額
    refund: Math.max(difference, 0),      // 還付額（払いすぎ）
    shortage: Math.max(-difference, 0),   // 不足額（追加徴収）
  };
}
```

多くの場合は**還付**（払いすぎの返金）になる。理由は:
- 生命保険料控除やiDeCo控除が年末にまとめて反映される
- 配偶者の所得が確定し、配偶者控除が適用される
- ボーナスに対する源泉徴収が多めに計算されている

逆に**不足**（追加徴収）になるケース:
- 扶養親族が減った（子供が独立等）
- 年途中で大幅に昇給した
- 2箇所以上から給与を受けている

## 配偶者控除の段階的計算

配偶者控除は2018年の改正で複雑になった。本人の所得と配偶者の所得の**2軸**で控除額が変わる。

```javascript
function calcSpouseDeduction(myIncome, spouseIncome) {
  // 本人の所得が1,000万超は適用外
  if (myIncome > 10000000) return 0;

  // 配偶者の所得が48万以下 → 配偶者控除
  if (spouseIncome <= 480000) {
    if (myIncome <= 9000000) return 380000;
    if (myIncome <= 9500000) return 260000;
    return 130000; // 〜1,000万
  }

  // 配偶者の所得が48万超133万以下 → 配偶者特別控除
  if (spouseIncome > 1330000) return 0;

  // 段階的に減額（簡略化）
  const steps = [
    { limit: 500000,  amounts: [380000, 260000, 130000] },
    { limit: 550000,  amounts: [360000, 240000, 120000] },
    { limit: 600000,  amounts: [310000, 210000, 110000] },
    { limit: 670000,  amounts: [260000, 180000,  90000] },
    { limit: 750000,  amounts: [210000, 140000,  70000] },
    { limit: 830000,  amounts: [160000, 110000,  60000] },
    { limit: 900000,  amounts: [110000,  80000,  40000] },
    { limit: 950000,  amounts: [ 60000,  40000,  20000] },
    { limit: 1330000, amounts: [ 30000,  20000,  10000] },
  ];

  const incomeIdx = myIncome <= 9000000 ? 0 :
                    myIncome <= 9500000 ? 1 : 2;

  for (const step of steps) {
    if (spouseIncome <= step.limit) {
      return step.amounts[incomeIdx];
    }
  }
  return 0;
}
```

これを見ると「配偶者の年収103万の壁」が単なるキリの良い数字ではなく、**給与所得控除55万 + 基礎控除48万 = 103万**という計算から来ていることがわかる。

## 生命保険料控除

生命保険料控除は「旧制度」と「新制度」で計算式が異なる。

```javascript
function calcLifeInsuranceDeduction(premium) {
  // 新制度（2012年1月以降の契約）の場合
  if (premium <= 20000) return premium;
  if (premium <= 40000) return Math.floor(premium / 2 + 10000);
  if (premium <= 80000) return Math.floor(premium / 4 + 20000);
  return 40000; // 上限4万円

  // 一般・介護医療・個人年金の3種で各4万円
  // 合計上限12万円
}
```

## まとめ

年末調整をコードで書くと、その本質が見える:

- **毎月の源泉徴収はあくまで概算**。月額テーブルで近似値を出しているだけ
- **12月に年間の正確な税額を計算**し、概算との差分を精算する
- **控除が増えるほど還付額が大きくなる**。iDeCoや生命保険は節税効果が直接見える
- **配偶者控除は2軸のテーブル**。103万・150万・201万の壁はこのテーブルから来ている

## 関連ツール・サイト

これらのロジックを使った計算ツールを公開している:

- [手取り計算シミュレーター](https://and-tools.net/tools/take-home-pay/) — 年収から手取りを一括計算
- [所得税 計算ツール](https://and-tools.net/tools/income-tax/) — 累進課税の計算
- [178万の壁 計算ツール](https://and-tools.net/tools/178-wall/) — 2026年の基礎控除改正に対応

全34ツールは [AND TOOLS](https://and-tools.net/) で無料公開中。
