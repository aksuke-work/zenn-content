---
title: "社会保険料の標準報酬月額テーブルをJSで実装する"
emoji: "🏥"
type: "tech"
topics: ["javascript", "社会保険", "フリーランス", "個人開発"]
published: false
---

社会保険料の計算は税金より厄介だ。月額報酬を「標準報酬月額」に変換し、そこに保険料率を掛ける。このテーブル参照のロジックを [AND TOOLS](https://and-tools.net/) の[社会保険料 計算ツール](https://and-tools.net/tools/social-insurance/)で実装した知見をまとめる。

## 標準報酬月額とは

実際の月額報酬をそのまま使わず、1〜50等級のテーブルに当てはめた金額で保険料を計算する。なぜこんな仕組みかというと、毎月変動する報酬に対して保険料を一定にするためだ。

```javascript
// 標準報酬月額テーブル（健康保険: 1〜50等級）
const HEALTH_INSURANCE_TABLE = [
  { grade:  1, lower:      0, upper:  63000, standard:  58000 },
  { grade:  2, lower:  63000, upper:  73000, standard:  68000 },
  { grade:  3, lower:  73000, upper:  83000, standard:  78000 },
  { grade:  4, lower:  83000, upper:  93000, standard:  88000 },
  { grade:  5, lower:  93000, upper: 101000, standard:  98000 },
  { grade:  6, lower: 101000, upper: 107000, standard: 104000 },
  { grade:  7, lower: 107000, upper: 114000, standard: 110000 },
  { grade:  8, lower: 114000, upper: 122000, standard: 118000 },
  { grade:  9, lower: 122000, upper: 130000, standard: 126000 },
  { grade: 10, lower: 130000, upper: 138000, standard: 134000 },
  { grade: 11, lower: 138000, upper: 146000, standard: 142000 },
  { grade: 12, lower: 146000, upper: 155000, standard: 150000 },
  { grade: 13, lower: 155000, upper: 165000, standard: 160000 },
  { grade: 14, lower: 165000, upper: 175000, standard: 170000 },
  { grade: 15, lower: 175000, upper: 185000, standard: 180000 },
  { grade: 16, lower: 185000, upper: 195000, standard: 190000 },
  { grade: 17, lower: 195000, upper: 210000, standard: 200000 },
  { grade: 18, lower: 210000, upper: 230000, standard: 220000 },
  { grade: 19, lower: 230000, upper: 250000, standard: 240000 },
  { grade: 20, lower: 250000, upper: 270000, standard: 260000 },
  { grade: 21, lower: 270000, upper: 290000, standard: 280000 },
  { grade: 22, lower: 290000, upper: 310000, standard: 300000 },
  { grade: 23, lower: 310000, upper: 330000, standard: 320000 },
  { grade: 24, lower: 330000, upper: 350000, standard: 340000 },
  { grade: 25, lower: 350000, upper: 370000, standard: 360000 },
  { grade: 26, lower: 370000, upper: 395000, standard: 380000 },
  { grade: 27, lower: 395000, upper: 425000, standard: 410000 },
  { grade: 28, lower: 425000, upper: 455000, standard: 440000 },
  { grade: 29, lower: 455000, upper: 485000, standard: 470000 },
  { grade: 30, lower: 485000, upper: 515000, standard: 500000 },
  { grade: 31, lower: 515000, upper: 545000, standard: 530000 },
  { grade: 32, lower: 545000, upper: 575000, standard: 560000 },
  { grade: 33, lower: 575000, upper: 605000, standard: 590000 },
  { grade: 34, lower: 605000, upper: 635000, standard: 620000 },
  { grade: 35, lower: 635000, upper: 665000, standard: 650000 },
  { grade: 36, lower: 665000, upper: 695000, standard: 680000 },
  { grade: 37, lower: 695000, upper: 730000, standard: 710000 },
  { grade: 38, lower: 730000, upper: 770000, standard: 750000 },
  { grade: 39, lower: 770000, upper: 810000, standard: 790000 },
  { grade: 40, lower: 810000, upper: 855000, standard: 830000 },
  { grade: 41, lower: 855000, upper: 905000, standard: 880000 },
  { grade: 42, lower: 905000, upper: 955000, standard: 930000 },
  { grade: 43, lower: 955000, upper: 1005000, standard: 980000 },
  { grade: 44, lower: 1005000, upper: 1055000, standard: 1030000 },
  { grade: 45, lower: 1055000, upper: 1115000, standard: 1090000 },
  { grade: 46, lower: 1115000, upper: 1175000, standard: 1150000 },
  { grade: 47, lower: 1175000, upper: 1235000, standard: 1210000 },
  { grade: 48, lower: 1235000, upper: 1295000, standard: 1270000 },
  { grade: 49, lower: 1295000, upper: 1355000, standard: 1330000 },
  { grade: 50, lower: 1355000, upper: Infinity, standard: 1390000 },
];

function getStandardMonthly(monthlyIncome) {
  for (const row of HEALTH_INSURANCE_TABLE) {
    if (monthlyIncome >= row.lower && monthlyIncome < row.upper) {
      return row;
    }
  }
  // フォールバック（最高等級）
  return HEALTH_INSURANCE_TABLE[HEALTH_INSURANCE_TABLE.length - 1];
}
```

50等級すべてを定義するのが面倒だが、正確な計算のためには省略できない。

## 保険料率と折半計算

健康保険料率は都道府県ごとに異なる。厚生年金保険料率は全国一律18.3%で固定されている。

```javascript
// 都道府県別の健康保険料率（協会けんぽ 2026年度・一部抜粋）
const HEALTH_RATES = {
  '北海道': 0.1029,
  '東京':   0.0998,
  '大阪':   0.1033,
  '福岡':   0.1038,
  '佐賀':   0.1089, // 最も高い
  '新潟':   0.0935, // 最も低い
};

function calcSocialInsurance(monthlyIncome, prefecture, age) {
  const { standard, grade } = getStandardMonthly(monthlyIncome);

  // 健康保険料率（都道府県別）
  const healthRate = HEALTH_RATES[prefecture] || 0.10;

  // 介護保険料率（40歳以上65歳未満）
  const nursingRate = (age >= 40 && age < 65) ? 0.016 : 0;

  // 厚生年金保険料率（固定）
  const pensionRate = 0.183;

  // 厚生年金の標準報酬月額上限は65万円（32等級）
  const pensionStandard = Math.min(standard, 650000);

  // 会社負担と個人負担で折半
  const healthEmployee = Math.floor(standard * (healthRate + nursingRate) / 2);
  const pensionEmployee = Math.floor(pensionStandard * pensionRate / 2);

  const healthCompany = Math.floor(standard * (healthRate + nursingRate) / 2);
  const pensionCompany = Math.floor(pensionStandard * pensionRate / 2);

  return {
    grade,
    standardMonthly: standard,
    health: {
      employee: healthEmployee,
      company: healthCompany,
      total: healthEmployee + healthCompany,
    },
    pension: {
      employee: pensionEmployee,
      company: pensionCompany,
      total: pensionEmployee + pensionCompany,
    },
    employeeTotal: healthEmployee + pensionEmployee,
    companyTotal: healthCompany + pensionCompany,
    grandTotal: healthEmployee + pensionEmployee + healthCompany + pensionCompany,
  };
}
```

## 端数処理の落とし穴

折半計算での端数処理が実は曲者。法律上は「事業主が納付義務を負う」ため、端数は原則として**50銭以下切捨て、50銭超切上げ**。ただし労使で取り決めがあればそちらが優先される。

```javascript
function halfRound(amount) {
  // 折半して端数処理
  const half = amount / 2;
  const decimal = half - Math.floor(half);

  // 50銭以下（0.5以下）は切捨て、50銭超は切上げ
  if (decimal <= 0.5) {
    return Math.floor(half);
  }
  return Math.ceil(half);
}

// 例: 奇数の保険料が発生した場合
const totalPremium = 29999;
console.log(halfRound(totalPremium)); // 14999（端数は事業主側に）
```

## 年収から年間社会保険料を概算する

手取り計算では年額で社会保険料を出す必要がある。月額 × 12 が基本だが、賞与は別の標準賞与額で計算する。

```javascript
function calcAnnualSocialInsurance(annualIncome, bonusMonths, prefecture, age) {
  // 月額報酬を算出（賞与月数を考慮）
  const monthlyBase = Math.floor(annualIncome / (12 + bonusMonths));
  const annualBonus = annualIncome - monthlyBase * 12;

  // 月額保険料
  const monthly = calcSocialInsurance(monthlyBase, prefecture, age);

  // 賞与の保険料（標準賞与額: 1,000円未満切捨て）
  const standardBonus = Math.floor(annualBonus / 1000) * 1000;
  // 健康保険の賞与上限: 年度累計573万円
  const healthBonusCap = 5730000;
  // 厚生年金の賞与上限: 1回あたり150万円
  const pensionBonusCap = 1500000;

  const healthRate = (HEALTH_RATES[prefecture] || 0.10) + ((age >= 40 && age < 65) ? 0.016 : 0);
  const bonusHealth = Math.floor(Math.min(standardBonus, healthBonusCap) * healthRate / 2);
  const bonusPension = Math.floor(Math.min(standardBonus, pensionBonusCap) * 0.183 / 2);

  return {
    monthlyEmployee: monthly.employeeTotal,
    annualFromMonthly: monthly.employeeTotal * 12,
    bonusEmployee: bonusHealth + bonusPension,
    annualTotal: monthly.employeeTotal * 12 + bonusHealth + bonusPension,
  };
}
```

## 法人成りしたときの社会保険料

フリーランスが法人成りすると、国保+国民年金から社会保険に切り替わる。役員報酬の設定で保険料が決まるため、シミュレーションが重要になる。

```javascript
function compareSoloVsCorp(revenue, expenses) {
  const profit = revenue - expenses;

  // フリーランス: 国保 + 国民年金
  const freelanceNationalPension = 16980 * 12; // 月額16,980円（2026年度）
  const freelanceNHI = calcNationalHealthInsurance(profit); // 別途実装

  // 法人: 社会保険（役員報酬を利益の半分と仮定）
  const directorSalary = Math.floor(profit * 0.5);
  const corpSI = calcAnnualSocialInsurance(directorSalary, 0, '東京', 35);

  console.log(`フリーランス: 国保+年金 = ${freelanceNationalPension + freelanceNHI}円/年`);
  console.log(`法人: 社保(個人) = ${corpSI.annualTotal}円/年`);
  console.log(`法人: 社保(会社負担含む) = ${corpSI.annualTotal * 2}円/年`);
}
```

法人成りのシミュレーションは[法人成り計算ツール](https://and-tools.net/tools/solo-corp-sim/)で詳しく試算できる。

## まとめ

社会保険料の実装ポイント:

1. **50等級のテーブル**を省略せず定義する（二分探索で高速化は可能）
2. **健康保険料率は都道府県別** — 佐賀と新潟で約1.5%の差がある
3. **厚生年金の上限は65万円** — 月収65万超でも保険料は変わらない
4. **折半の端数処理**は50銭ルールに従う
5. **賞与は標準賞与額**で別計算。上限額に注意

全34ツールは [AND TOOLS](https://and-tools.net/) で動いている。[手取り計算シミュレーター](https://and-tools.net/tools/take-home-pay/)では社会保険料を含めた総合的な手取り額を計算できる。

---

[AND TOOLS](https://and-tools.net/) では、フリーランス・個人事業主向けの税金計算ツールを34個以上無料で公開しています。
