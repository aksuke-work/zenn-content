---
title: "会社員の副業税金を正確に計算する — 給与所得・事業所得・雑所得の3パターン実装"
emoji: "💼"
type: "tech"
topics: ["javascript", "tax", "副業", "個人開発"]
published: false
---

副業エンジニアが増えているが、「副業で年50万稼いだら税金いくら？」に正確に答えられる人は少ない。[AND TOOLS](https://and-tools.net/) の副業税金計算ツールを作った経験から、3つの所得分類ごとの計算ロジックを解説する。

## 副業所得の3分類

副業の収入は、税法上3つのどれかに分類される。分類によって控除額が大きく変わる。

```javascript
const INCOME_TYPES = {
  SALARY: 'salary',       // 給与所得（アルバイト等）
  BUSINESS: 'business',   // 事業所得（開業届あり + 反復継続）
  MISC: 'misc',           // 雑所得（上記以外）
};
```

| 分類 | 例 | 特徴 |
|---|---|---|
| 給与所得 | 副業アルバイト | 給与所得控除の合算計算 |
| 事業所得 | フリーランス案件（開業届あり） | 青色申告特別控除（最大65万） |
| 雑所得 | 単発案件、副業プログラミング | 控除なし、赤字の損益通算不可 |

## 給与所得控除の合算計算

副業がアルバイト（給与所得）の場合、本業と副業の給与を**合算してから**給与所得控除を適用する。

```javascript
function calcSalaryDeduction(totalSalary) {
  if (totalSalary <= 1625000) return 550000;
  if (totalSalary <= 1800000) return Math.floor(totalSalary * 0.4 - 100000);
  if (totalSalary <= 3600000) return Math.floor(totalSalary * 0.3 + 80000);
  if (totalSalary <= 6600000) return Math.floor(totalSalary * 0.2 + 440000);
  if (totalSalary <= 8500000) return Math.floor(totalSalary * 0.1 + 1100000);
  return 1950000;
}

function calcSideJobTax_Salary(mainSalary, sideSalary) {
  // 本業だけの場合の税金
  const mainDeduction = calcSalaryDeduction(mainSalary);
  const mainIncome = mainSalary - mainDeduction;
  const mainTax = calcIncomeTax(Math.max(0, mainIncome - 580000));

  // 本業+副業を合算した場合の税金
  const totalSalary = mainSalary + sideSalary;
  const totalDeduction = calcSalaryDeduction(totalSalary);
  const totalIncome = totalSalary - totalDeduction;
  const totalTax = calcIncomeTax(Math.max(0, totalIncome - 580000));

  // 差額が副業による増税分
  return {
    additionalTax: totalTax - mainTax,
    effectiveRate: ((totalTax - mainTax) / sideSalary * 100).toFixed(1),
  };
}
```

ポイント: 給与所得控除は**合算額に対して1回だけ**適用される。副業分だけ別に控除されるわけではない。副業が給与所得だと、副業収入の大部分がそのまま課税対象になりやすい。

## 事業所得 — 青色申告で最大65万円控除

開業届を出して確定申告を青色申告で行えば、最大65万円の特別控除が受けられる。

```javascript
function calcSideJobTax_Business(mainSalary, sideRevenue, sideExpenses, options = {}) {
  const {
    isBlueReturn = true,
    blueDeduction = 650000, // 65万円（e-Tax + 複式簿記）
  } = options;

  // 本業の給与所得
  const salaryDeduction = calcSalaryDeduction(mainSalary);
  const salaryIncome = mainSalary - salaryDeduction;

  // 副業の事業所得
  let businessIncome = sideRevenue - sideExpenses;
  if (isBlueReturn && businessIncome > 0) {
    businessIncome = Math.max(0, businessIncome - blueDeduction);
  }

  // 合算して課税所得を計算
  const totalIncome = salaryIncome + businessIncome;
  const taxableIncome = Math.max(0, totalIncome - 580000); // 基礎控除
  const totalTax = calcIncomeTax(taxableIncome);

  // 本業だけの税金との差額
  const mainOnlyTax = calcIncomeTax(Math.max(0, salaryIncome - 580000));

  return {
    businessIncome,
    additionalTax: totalTax - mainOnlyTax,
    blueDeductionBenefit: isBlueReturn ? blueDeduction : 0,
  };
}
```

青色申告特別控除の3段階:
- **65万円**: e-Tax + 複式簿記 + 貸借対照表
- **55万円**: 複式簿記 + 貸借対照表（紙提出）
- **10万円**: 簡易簿記

```javascript
function getBlueDeduction(options) {
  const { isETax = false, isDoubleEntry = false, hasBalanceSheet = false } = options;

  if (isETax && isDoubleEntry && hasBalanceSheet) return 650000;
  if (isDoubleEntry && hasBalanceSheet) return 550000;
  return 100000;
}
```

## 雑所得 — 控除なしの厳しい世界

副業が「雑所得」に分類されると、特別控除が一切ない。

```javascript
function calcSideJobTax_Misc(mainSalary, sideRevenue, sideExpenses) {
  const salaryDeduction = calcSalaryDeduction(mainSalary);
  const salaryIncome = mainSalary - salaryDeduction;

  // 雑所得 = 収入 - 経費（特別控除なし）
  const miscIncome = Math.max(0, sideRevenue - sideExpenses);

  const totalIncome = salaryIncome + miscIncome;
  const taxableIncome = Math.max(0, totalIncome - 580000);
  const totalTax = calcIncomeTax(taxableIncome);

  const mainOnlyTax = calcIncomeTax(Math.max(0, salaryIncome - 580000));

  return {
    miscIncome,
    additionalTax: totalTax - mainOnlyTax,
    // 雑所得は赤字でも他の所得と損益通算できない
    canOffsetLoss: false,
  };
}
```

事業所得 vs 雑所得の最大の違い:
- **事業所得**: 赤字を給与所得と**損益通算**できる
- **雑所得**: 赤字でも損益通算**不可**

実装: [副業税金 計算ツール](https://and-tools.net/tools/side-job-tax/)

## 「20万円ルール」の正確な判定

「副業が20万以下なら確定申告不要」は**半分正しくて半分間違い**。

```javascript
function check20manRule(sideIncome, incomeType) {
  const result = {
    incomeTaxExempt: false,   // 所得税の確定申告不要
    residentTaxExempt: false, // 住民税の申告不要
  };

  // 所得税: 20万円以下なら確定申告不要（給与所得者で年末調整済みの場合）
  if (sideIncome <= 200000) {
    result.incomeTaxExempt = true;
  }

  // 住民税: 20万円ルールは適用されない！
  // 1円でも副業所得があれば住民税の申告が必要
  result.residentTaxExempt = sideIncome <= 0;

  return result;
}
```

これは多くの人が見落とすポイント:
- **所得税**: 副業所得20万円以下 → 確定申告不要
- **住民税**: 20万円ルール**なし** → 1円でも申告が必要

```javascript
// 住民税の申告が必要なケース
const sideIncome = 150000; // 15万円の副業所得
const rule = check20manRule(sideIncome, 'misc');

console.log(rule.incomeTaxExempt);   // true（確定申告不要）
console.log(rule.residentTaxExempt); // false（住民税申告は必要！）
```

## 副業の税金増分を差額計算で求める

3つの所得タイプを統合した計算関数。

```javascript
function calcSideJobTaxIncrease(params) {
  const {
    mainSalary,
    sideRevenue,
    sideExpenses = 0,
    incomeType = 'misc',
    isBlueReturn = false,
  } = params;

  // 本業のみの税金
  const mainDeduction = calcSalaryDeduction(mainSalary);
  const mainIncome = mainSalary - mainDeduction;
  const mainTaxableIncome = Math.max(0, mainIncome - 580000);
  const mainIncomeTax = calcIncomeTax(mainTaxableIncome);
  const mainResidentTax = Math.floor(mainTaxableIncome * 0.1) + 5000;

  // 副業込みの計算
  let sideIncome;
  if (incomeType === 'salary') {
    const totalDeduction = calcSalaryDeduction(mainSalary + sideRevenue);
    sideIncome = (mainSalary + sideRevenue) - totalDeduction - mainIncome;
  } else if (incomeType === 'business') {
    sideIncome = sideRevenue - sideExpenses;
    if (isBlueReturn) sideIncome = Math.max(0, sideIncome - 650000);
  } else {
    sideIncome = Math.max(0, sideRevenue - sideExpenses);
  }

  const totalTaxableIncome = Math.max(0, mainIncome + Math.max(0, sideIncome) - 580000);
  const totalIncomeTax = calcIncomeTax(totalTaxableIncome);
  const totalResidentTax = Math.floor(totalTaxableIncome * 0.1) + 5000;

  return {
    sideIncome: Math.max(0, sideIncome),
    additionalIncomeTax: totalIncomeTax - mainIncomeTax,
    additionalResidentTax: totalResidentTax - mainResidentTax,
    totalAdditionalTax: (totalIncomeTax - mainIncomeTax) + (totalResidentTax - mainResidentTax),
    effectiveRate: (
      ((totalIncomeTax - mainIncomeTax) + (totalResidentTax - mainResidentTax)) /
      sideRevenue * 100
    ).toFixed(1),
  };
}
```

実装: [手取り計算シミュレーター](https://and-tools.net/tools/take-home-pay/)

## 住民税の普通徴収 — 副業バレを防ぐ

副業の住民税を「普通徴収（自分で納付）」にすれば、会社の給与天引きに副業分が上乗せされない。

```javascript
function splitResidentTax(mainTaxable, sideTaxable) {
  const mainResidentTax = Math.floor(mainTaxable * 0.1);
  const sideResidentTax = Math.floor(sideTaxable * 0.1);

  return {
    // 特別徴収（給与天引き）: 本業分のみ
    specialCollection: mainResidentTax + 5000,
    // 普通徴収（自分で納付）: 副業分
    ordinaryCollection: sideResidentTax,
    note: '確定申告書の「住民税に関する事項」で「自分で納付」を選択',
  };
}
```

注意: 副業が**給与所得**の場合、普通徴収を選択できない自治体もある。事業所得・雑所得なら基本的に普通徴収が可能。

実装: [源泉徴収税 計算ツール](https://and-tools.net/tools/tax-withholding/)

## まとめ

1. **所得分類で税額が大きく変わる**: 事業所得なら青色申告65万控除が使える
2. **給与所得控除は合算計算**: 副業分だけ別に控除されるわけではない
3. **20万円ルールは所得税だけ**: 住民税には適用されない（1円でも申告必要）
4. **差額計算が正確**: 本業のみの税金と副業込みの税金の差額を求める
5. **普通徴収で副業分を分離**: 確定申告時に「自分で納付」を選択する

## AND TOOLSについて

[AND TOOLS](https://and-tools.net/) では、フリーランス・個人事業主向けの税金計算ツールを34個以上無料で公開しています。この記事で紹介した計算ロジックを実際に試せます。
