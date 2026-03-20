---
title: "医療費控除とセルフメディケーション税制をJSで実装する"
emoji: "💊"
type: "tech"
topics: ["javascript", "医療費控除", "tax", "個人開発"]
published: false
---

医療費控除は「10万円を超えた分が控除される」とざっくり理解されているが、実は所得200万円未満の場合は「所得の5%」が基準になる。さらにセルフメディケーション税制との選択適用があり、どちらが得かの判定が必要だ。[AND TOOLS](https://and-tools.net/) の[医療費控除 計算ツール](https://and-tools.net/tools/medical-deduction/)で実装したロジックを解説する。

## 医療費控除の基本計算

```javascript
function calcMedicalDeduction(totalMedicalExpenses, insuranceReimbursement, totalIncome) {
  // 保険金等で補填される金額を差し引く
  const netExpenses = Math.max(totalMedicalExpenses - insuranceReimbursement, 0);

  // 控除の基準額: 10万円 or 総所得の5% の小さい方
  const threshold = Math.min(100000, Math.floor(totalIncome * 0.05));

  // 控除額 = 支払った医療費 - 補填金額 - 基準額（上限200万円）
  const deduction = Math.min(
    Math.max(netExpenses - threshold, 0),
    2000000 // 上限200万円
  );

  return {
    totalExpenses: totalMedicalExpenses,
    insuranceReimbursement,
    netExpenses,
    threshold,
    deduction,
    meetsMinimum: netExpenses > threshold,
  };
}

// テスト
console.log(calcMedicalDeduction(150000, 0, 5000000));
// → deduction: 50,000 (15万-10万)

console.log(calcMedicalDeduction(150000, 0, 1500000));
// → deduction: 75,000 (15万-7.5万) ※所得200万未満で基準が下がる

console.log(calcMedicalDeduction(80000, 0, 1200000));
// → deduction: 20,000 (8万-6万) ※所得の5%=6万が基準
```

所得200万円未満の人は基準額が下がるため、10万円に届かなくても控除が受けられる。これを知らない人が多い。

## セルフメディケーション税制

2017年から始まった制度で、スイッチOTC医薬品（市販薬）の購入額が年間**1.2万円を超えた部分**が控除される。

```javascript
function calcSelfMedicationDeduction(otcExpenses) {
  const threshold = 12000; // 1.2万円
  const cap = 88000;       // 上限8.8万円

  const deduction = Math.min(
    Math.max(otcExpenses - threshold, 0),
    cap
  );

  return {
    otcExpenses,
    threshold,
    deduction,
    cap,
    meetsMinimum: otcExpenses > threshold,
  };
}

// テスト
console.log(calcSelfMedicationDeduction(50000));
// → deduction: 38,000 (5万-1.2万)

console.log(calcSelfMedicationDeduction(120000));
// → deduction: 88,000 (上限到達)
```

## 医療費控除 vs セルフメディケーション: どちらが得か

この2つは**選択適用**で、両方同時には使えない。どちらが有利かを判定するロジックを実装する。

```javascript
function compareMedicalDeductions(medicalExpenses, insuranceReimbursement, otcExpenses, totalIncome) {
  // 医療費控除の計算
  const medicalResult = calcMedicalDeduction(
    medicalExpenses,
    insuranceReimbursement,
    totalIncome
  );

  // セルフメディケーション税制の計算
  const selfMedResult = calcSelfMedicationDeduction(otcExpenses);

  // 所得税の限界税率を取得
  const taxRate = getIncomeTaxRate(totalIncome - 480000); // 基礎控除後

  // 各控除による節税額
  const medicalSavings = Math.floor(medicalResult.deduction * (taxRate * 1.021 + 0.10));
  const selfMedSavings = Math.floor(selfMedResult.deduction * (taxRate * 1.021 + 0.10));

  const recommendation = medicalSavings >= selfMedSavings
    ? '医療費控除が有利'
    : 'セルフメディケーション税制が有利';

  return {
    medical: {
      deduction: medicalResult.deduction,
      taxSavings: medicalSavings,
    },
    selfMedication: {
      deduction: selfMedResult.deduction,
      taxSavings: selfMedSavings,
    },
    recommendation,
    difference: Math.abs(medicalSavings - selfMedSavings),
  };
}

function getIncomeTaxRate(taxableIncome) {
  if (taxableIncome <= 1950000) return 0.05;
  if (taxableIncome <= 3300000) return 0.10;
  if (taxableIncome <= 6950000) return 0.20;
  if (taxableIncome <= 9000000) return 0.23;
  if (taxableIncome <= 18000000) return 0.33;
  if (taxableIncome <= 40000000) return 0.40;
  return 0.45;
}

// シナリオ1: 医療費が多い場合
console.log('=== シナリオ1: 医療費15万、OTC3万 ===');
console.log(compareMedicalDeductions(150000, 0, 30000, 5000000));
// 医療費控除: 5万円の控除 → 節税約1.5万円
// セルフメディケーション: 1.8万円の控除 → 節税約0.5万円
// → 医療費控除が有利

// シナリオ2: OTCが多い場合
console.log('=== シナリオ2: 医療費6万、OTC8万 ===');
console.log(compareMedicalDeductions(60000, 0, 80000, 5000000));
// 医療費控除: 0円（10万円未満で控除なし）
// セルフメディケーション: 6.8万円の控除 → 節税約2万円
// → セルフメディケーション税制が有利
```

## 保険金等の補填をどう扱うか

医療費控除で最も間違えやすいのが、保険金の取り扱い。保険金は**その医療費に対してのみ**差し引く。他の医療費から差し引いてはいけない。

```javascript
function calcMedicalDeductionDetailed(medicalItems, totalIncome) {
  // 各医療費項目ごとに保険金を差し引く
  let totalNetExpenses = 0;

  const details = medicalItems.map(item => {
    // 各項目で保険金を差し引く（マイナスにはならない）
    const netExpense = Math.max(item.expense - (item.insurance || 0), 0);
    totalNetExpenses += netExpense;

    return {
      description: item.description,
      expense: item.expense,
      insurance: item.insurance || 0,
      netExpense,
    };
  });

  const threshold = Math.min(100000, Math.floor(totalIncome * 0.05));
  const deduction = Math.min(Math.max(totalNetExpenses - threshold, 0), 2000000);

  return { details, totalNetExpenses, threshold, deduction };
}

// 例: 入院（高額）と通院（少額）がある場合
const items = [
  { description: '入院手術', expense: 500000, insurance: 400000 },
  { description: '通院（歯科）', expense: 80000, insurance: 0 },
  { description: '通院（皮膚科）', expense: 30000, insurance: 0 },
  { description: '処方薬', expense: 20000, insurance: 0 },
];

const result = calcMedicalDeductionDetailed(items, 5000000);
console.log(result);
// 入院: 50万-40万=10万（保険金はこの項目だけに充当）
// 通院歯科: 8万
// 通院皮膚科: 3万
// 処方薬: 2万
// 合計: 23万 - 基準額10万 = 13万円の控除
```

## 医療費控除の対象になるもの・ならないもの

プログラムで判定するのは難しいが、よく問い合わせがあるものをデータ化しておくと便利。

```javascript
const MEDICAL_CATEGORIES = {
  deductible: [
    { name: '診療費・治療費', note: '保険適用・自由診療とも' },
    { name: '処方薬', note: '医師の処方によるもの' },
    { name: '入院費', note: '食事代含む（差額ベッド代は原則対象外）' },
    { name: '通院交通費', note: '公共交通機関。タクシーは緊急時のみ' },
    { name: '歯科矯正', note: '子供の場合。大人は審美目的だと対象外' },
    { name: 'レーシック', note: '視力回復目的' },
    { name: '介護サービス', note: '医療系サービスのみ' },
    { name: '妊婦健診・出産費用', note: '出産育児一時金は補填として差し引く' },
  ],
  notDeductible: [
    { name: '美容整形', note: '治療目的でないもの' },
    { name: '健康診断・人間ドック', note: '異常が見つかり治療に移行した場合は対象' },
    { name: 'サプリメント', note: '治療目的でないもの' },
    { name: '差額ベッド代', note: '自己都合の場合' },
    { name: 'メガネ・コンタクト', note: '通常の視力補正用' },
    { name: '予防接種', note: 'インフルエンザ等' },
  ],
};

function checkDeductibility(itemName) {
  const found = MEDICAL_CATEGORIES.deductible.find(
    item => item.name.includes(itemName)
  );
  if (found) {
    return { deductible: true, note: found.note };
  }

  const notFound = MEDICAL_CATEGORIES.notDeductible.find(
    item => item.name.includes(itemName)
  );
  if (notFound) {
    return { deductible: false, note: notFound.note };
  }

  return { deductible: null, note: '個別判断が必要。税務署に確認を推奨' };
}
```

## 家族の医療費を合算する

医療費控除は生計を一にする家族の分もまとめて申告できる。所得税率が高い人が申告するのが有利。

```javascript
function calcFamilyMedicalDeduction(familyMembers) {
  // 全員の医療費を合算
  const totalExpenses = familyMembers.reduce(
    (sum, member) => sum + member.medicalExpenses, 0
  );
  const totalInsurance = familyMembers.reduce(
    (sum, member) => sum + (member.insurance || 0), 0
  );

  // 最も所得税率が高い人が申告するのが有利
  const bestApplicant = familyMembers
    .filter(m => m.income > 0)
    .sort((a, b) => {
      const rateA = getIncomeTaxRate(a.income - 480000);
      const rateB = getIncomeTaxRate(b.income - 480000);
      return rateB - rateA;
    })[0];

  const deduction = calcMedicalDeduction(totalExpenses, totalInsurance, bestApplicant.income);
  const taxRate = getIncomeTaxRate(bestApplicant.income - 480000);
  const savings = Math.floor(deduction.deduction * (taxRate * 1.021 + 0.10));

  return {
    totalFamilyExpenses: totalExpenses,
    bestApplicant: bestApplicant.name,
    bestApplicantRate: `${taxRate * 100}%`,
    deduction: deduction.deduction,
    estimatedSavings: savings,
  };
}

// 例: 夫婦+子供の医療費
const family = [
  { name: '夫', income: 7000000, medicalExpenses: 50000, insurance: 0 },
  { name: '妻', income: 1500000, medicalExpenses: 80000, insurance: 0 },
  { name: '子供', income: 0, medicalExpenses: 30000, insurance: 0 },
];

console.log(calcFamilyMedicalDeduction(family));
// 合計16万円、夫（税率20%）が申告 → 控除6万円 → 節税約1.8万円
```

[手取り計算シミュレーター](https://and-tools.net/tools/take-home-pay/)では、医療費控除を含めた年間の税額を計算できる。[所得税 計算ツール](https://and-tools.net/tools/income-tax/)と合わせて使うと、控除の効果を具体的に確認できる。

## まとめ

医療費控除の実装ポイント:

1. **基準額は10万円 or 所得の5%** の小さい方（所得200万未満は5%）
2. **上限200万円**、セルフメディケーション税制は**上限8.8万円**
3. 2つの制度は**選択適用** — 節税額で比較して有利な方を選ぶ
4. 保険金は**該当する医療費にのみ充当** — 他の医療費から差し引かない
5. 家族の医療費は合算可能。**所得税率が高い人**が申告するのが有利

「10万円を超えないから関係ない」と思い込んでいる人が多いが、所得200万円未満なら基準額が下がる。また、セルフメディケーション税制は1.2万円超から使える。[医療費控除 計算ツール](https://and-tools.net/tools/medical-deduction/)で自分のケースを試算してみてほしい。

---

[AND TOOLS](https://and-tools.net/) では、フリーランス・個人事業主向けの税金計算ツールを34個以上無料で公開しています。
