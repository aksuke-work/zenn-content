---
title: "フリーランスの老後資金計画 — 国民年金だけでは月6.5万円"
emoji: "🔮"
type: "tech"
topics: ["フリーランス", "老後資金", "iDeCo", "資産形成"]
published: false
---

「老後は考えていない」というフリーランスは多いが、これは最もコストが高い判断だ。国民年金の満額は月約6.5万円。これが唯一の収入源では生活が成り立たない。

[AND TOOLS](https://and-tools.net/) で老後資金計算ツールを設計した際のデータをもとに、フリーランスの現実的な老後資金計画を解説する。

---

## 現実の数字：フリーランスの老後はいくら必要か

### 生活費の試算

総務省の家計調査によると、老後の生活費の目安は夫婦で月約23万円、単身で約14万円とされている。

```javascript
// 老後資金の必要額計算
function calcRetirementNeed(config) {
  const {
    currentAge,        // 現在の年齢
    retirementAge,     // 退職年齢
    lifeExpectancy,    // 想定寿命
    monthlyExpense,    // 老後の月間生活費
    nationalPensionMonthly, // 国民年金の月額受給額
    otherIncomeMonthly,    // その他収入（アルバイト等）
  } = config;

  const retirementYears = lifeExpectancy - retirementAge;
  const monthlyShortfall = monthlyExpense - nationalPensionMonthly - otherIncomeMonthly;
  const totalNeed = Math.max(monthlyShortfall * 12 * retirementYears, 0);

  const workingYears = retirementAge - currentAge;
  const monthlyTarget = Math.ceil(totalNeed / (workingYears * 12));

  return {
    retirementYears,
    monthlyExpense,
    nationalPensionMonthly,
    monthlyShortfall,
    totalNeed,
    workingYears,
    monthlyTarget,
  };
}

// シミュレーション例
const plan = calcRetirementNeed({
  currentAge: 35,
  retirementAge: 65,
  lifeExpectancy: 90,
  monthlyExpense: 200000,      // 月20万円の生活費
  nationalPensionMonthly: 65000, // 国民年金満額（月6.5万円）
  otherIncomeMonthly: 0,
});

console.log("=== 老後資金シミュレーション ===");
console.log(`老後期間: ${plan.retirementYears}年間`);
console.log(`月間生活費: ¥${plan.monthlyExpense.toLocaleString()}`);
console.log(`国民年金: ¥${plan.nationalPensionMonthly.toLocaleString()}/月`);
console.log(`月間不足額: ¥${plan.monthlyShortfall.toLocaleString()}`);
console.log(`必要総額: ¥${plan.totalNeed.toLocaleString()}`);
console.log(`今から毎月積み立て目標: ¥${plan.monthlyTarget.toLocaleString()}`);
```

---

## フリーランスが活用できる老後資金の手段

### 老後資金の4層構造

```
Layer 4: 投資（株式・ETF・不動産）
Layer 3: 小規模企業共済（退職金代わり）
Layer 2: iDeCo（確定拠出年金）
Layer 1: 国民年金 + 付加年金（必須）
```

各層を積み上げることで、老後の収入源を多様化できる。

---

## Layer 1：国民年金 + 付加年金

```javascript
// 国民年金の受給額と付加年金の効果
function calcBaseLayer(paymentYears) {
  // 国民年金
  const fullPension = 65000; // 満額（月額・2024年度）
  const nationalPension = Math.floor(fullPension * (paymentYears / 40));

  // 付加年金（月400円を paymentYears 年間納付）
  const paidMonths = paymentYears * 12;
  const additionalPension = 200 * paidMonths; // 月額追加

  return {
    nationalPension,
    additionalPension,
    totalPension: nationalPension + additionalPension,
    yearlyTotal: (nationalPension + additionalPension) * 12,
  };
}

const base30 = calcBaseLayer(30); // 30年間納付
console.log(`30年納付: 月¥${base30.totalPension.toLocaleString()} / 年¥${base30.yearlyTotal.toLocaleString()}`);

const base40 = calcBaseLayer(40); // 40年間納付（満額）
console.log(`40年納付: 月¥${base40.totalPension.toLocaleString()} / 年¥${base40.yearlyTotal.toLocaleString()}`);
```

---

## Layer 2：iDeCo（個人型確定拠出年金）

```javascript
// iDeCoの複利シミュレーション
function calcIdeco(config) {
  const {
    monthlyContribution, // 月額拠出
    years,               // 運用年数
    annualReturn,        // 年間利回り
    taxRate,             // 実効税率
  } = config;

  const yearlyContribution = monthlyContribution * 12;

  // 積立額の複利計算
  let totalAmount = 0;
  for (let i = 0; i < years; i++) {
    totalAmount = (totalAmount + yearlyContribution) * (1 + annualReturn);
  }

  const totalContribution = yearlyContribution * years;
  const investmentGain = totalAmount - totalContribution;
  const totalTaxSaving = yearlyContribution * taxRate * years;

  // 受取時の税金（退職所得控除を活用）
  // 勤続年数20年超: 800万 + 70万 × (年数 - 20年)
  const retirementDeduction = years > 20
    ? 8000000 + 700000 * (years - 20)
    : 400000 * years;
  const taxableRetirement = Math.max(totalAmount - retirementDeduction, 0);
  const receiveTax = taxableRetirement * 0.5 * taxRate; // 1/2課税

  return {
    totalContribution,
    totalAmount: Math.floor(totalAmount),
    investmentGain: Math.floor(investmentGain),
    totalTaxSaving,
    retirementDeduction,
    receiveTax: Math.floor(receiveTax),
    netReceiveAmount: Math.floor(totalAmount - receiveTax),
  };
}

// 月3万円・年利4%・30年間・税率20%
const idecoResult = calcIdeco({
  monthlyContribution: 30000,
  years: 30,
  annualReturn: 0.04,
  taxRate: 0.20,
});

console.log("=== iDeCo 30年シミュレーション ===");
console.log(`総拠出額: ¥${idecoResult.totalContribution.toLocaleString()}`);
console.log(`受取総額（概算）: ¥${idecoResult.totalAmount.toLocaleString()}`);
console.log(`運用益: ¥${idecoResult.investmentGain.toLocaleString()}`);
console.log(`積立期間中の節税合計: ¥${idecoResult.totalTaxSaving.toLocaleString()}`);
console.log(`退職所得控除: ¥${idecoResult.retirementDeduction.toLocaleString()}`);
console.log(`受取後の実質手取り: ¥${idecoResult.netReceiveAmount.toLocaleString()}`);
```

---

## Layer 3：小規模企業共済

```javascript
// 小規模企業共済の積み立てシミュレーション
function calcShokiboSavings(monthlyContribution, years) {
  const yearlyContribution = monthlyContribution * 12;
  const totalContribution = yearlyContribution * years;

  // 掛金月数と想定共済金（概算）
  const paidMonths = years * 12;

  // 共済金の計算（実際は複雑な計算式があるが概算）
  // 長期（20年以上）では元本の約110〜120%が目安
  const returnRate = years >= 20 ? 1.15 : years >= 10 ? 1.05 : 1.0;
  const estimatedBenefit = Math.floor(totalContribution * returnRate);

  const taxSaving = yearlyContribution * 0.20 * years; // 税率20%での概算

  return {
    monthlyContribution,
    years,
    totalContribution,
    estimatedBenefit,
    taxSaving,
    totalReturn: estimatedBenefit + taxSaving,
  };
}

// 月5万円・20年間
const shokibo = calcShokiboSavings(50000, 20);
console.log("=== 小規模企業共済 20年シミュレーション ===");
console.log(`月額掛金: ¥${shokibo.monthlyContribution.toLocaleString()}`);
console.log(`総拠出額: ¥${shokibo.totalContribution.toLocaleString()}`);
console.log(`受取概算: ¥${shokibo.estimatedBenefit.toLocaleString()}`);
console.log(`積立期間中の節税合計: ¥${shokibo.taxSaving.toLocaleString()}`);
console.log(`実質総リターン: ¥${shokibo.totalReturn.toLocaleString()}`);
```

[AND TOOLSの小規模企業共済シミュレーター](https://and-tools.net/tools/mutual-aid/) でより詳細な試算ができる。

---

## Layer 4：投資（NISA・株式・不動産）

iDeCoと小規模企業共済だけでは不足する場合、NISA（新NISA）での投資も老後資金の一部になる。

```javascript
// 新NISAでの積み立てシミュレーション
function calcNISA(monthlyContribution, years, annualReturn = 0.05) {
  // 新NISAの年間上限: 360万円（成長投資枠240万 + つみたて投資枠120万）
  const maxMonthly = 300000;
  const actualMonthly = Math.min(monthlyContribution, maxMonthly);
  const yearlyContribution = actualMonthly * 12;

  let totalAmount = 0;
  for (let i = 0; i < years; i++) {
    totalAmount = (totalAmount + yearlyContribution) * (1 + annualReturn);
  }

  return {
    monthlyContribution: actualMonthly,
    totalContribution: yearlyContribution * years,
    totalAmount: Math.floor(totalAmount),
    investmentGain: Math.floor(totalAmount - yearlyContribution * years),
    note: "新NISAは運用益・売却益が非課税",
  };
}

// 月5万円・年利5%・30年間
const nisa = calcNISA(50000, 30);
console.log("=== 新NISA 30年シミュレーション ===");
console.log(`総拠出額: ¥${nisa.totalContribution.toLocaleString()}`);
console.log(`受取概算: ¥${nisa.totalAmount.toLocaleString()}`);
console.log(`運用益（非課税）: ¥${nisa.investmentGain.toLocaleString()}`);
```

---

## 統合シミュレーション：65歳時点での資産

```javascript
function integratedPlan(currentAge = 35, retirementAge = 65) {
  const years = retirementAge - currentAge;

  // 各制度の積み立て
  const ideco = calcIdeco({ monthlyContribution: 30000, years, annualReturn: 0.04, taxRate: 0.20 });
  const shokibo = calcShokiboSavings(50000, years);
  const nisa = calcNISA(50000, years, 0.05);

  const totalAssets = ideco.netReceiveAmount + shokibo.estimatedBenefit + nisa.totalAmount;
  const monthlyPension = calcBaseLayer(years).totalPension;

  // 老後の月間利用可能額（25年間取り崩す場合）
  const monthlyFromAssets = Math.floor(totalAssets / (25 * 12));
  const totalMonthlyIncome = monthlyPension + monthlyFromAssets;

  console.log("=== 65歳時点の統合シミュレーション ===");
  console.log(`iDeCo受取（税引後）: ¥${ideco.netReceiveAmount.toLocaleString()}`);
  console.log(`小規模企業共済: ¥${shokibo.estimatedBenefit.toLocaleString()}`);
  console.log(`NISA: ¥${nisa.totalAmount.toLocaleString()}`);
  console.log(`資産合計: ¥${totalAssets.toLocaleString()}`);
  console.log("---");
  console.log(`年金受給（月）: ¥${monthlyPension.toLocaleString()}`);
  console.log(`資産取り崩し（月）: ¥${monthlyFromAssets.toLocaleString()}`);
  console.log(`月間合計: ¥${totalMonthlyIncome.toLocaleString()}`);
}

integratedPlan(35, 65);
```

---

## 何から始めるか：優先順位

| 優先度 | アクション | 月額目安 |
|--------|----------|---------|
| 1位 | 国民年金の未納解消 | 16,980円（必須） |
| 2位 | 付加年金の申請 | +400円（即効性高い） |
| 3位 | 小規模企業共済の加入 | 1万〜7万円 |
| 4位 | iDeCoの開設・拠出 | 1万〜6.8万円 |
| 5位 | 新NISAでの投資 | 残り余力で |

---

## まとめ

フリーランスの老後資金は「国民年金だけ」では月6.5万円にしかならない。会社員の厚生年金との差を埋めるために、4層構造で資産を積み上げる必要がある。

- **Layer 1**: 国民年金 + 付加年金（月約7〜10万円）
- **Layer 2**: iDeCo（60歳以降に受取）
- **Layer 3**: 小規模企業共済（廃業・退職時に受取）
- **Layer 4**: 新NISA（いつでも引き出せる流動性）

[AND TOOLS](https://and-tools.net/) では [所得税計算](https://and-tools.net/tools/income-tax/)・[手取り計算](https://and-tools.net/tools/take-home/)・[小規模企業共済シミュレーター](https://and-tools.net/tools/mutual-aid/) など、老後資金計画に役立つ無料ツールを公開している。今の手取りを把握した上で、老後積立の余力を計算するのに活用してほしい。
