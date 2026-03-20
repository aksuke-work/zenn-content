---
title: "国民健康保険料の計算ロジック — 自治体ごとの差をどう扱うか"
emoji: "🏛️"
type: "tech"
topics: ["javascript", "国保", "フリーランス", "個人開発"]
published: false
---

フリーランスが加入する国民健康保険（国保）は、自治体によって保険料が大きく異なる。同じ所得でも東京23区と大阪市で年間10万円以上変わることもある。[AND TOOLS](https://and-tools.net/) の[手取り計算ツール](https://and-tools.net/tools/take-home-pay/)で国保の概算計算を実装した際のロジックをまとめる。

## 国保の3つの構成要素

国保の保険料は「医療分」「後期高齢者支援金分」「介護分（40歳以上）」の3区分に分かれ、それぞれに「所得割」「均等割」「平等割」がある。

```javascript
function calcNationalHealthInsurance(income, age, city) {
  // 旧ただし書き所得 = 総所得 - 基礎控除43万円
  const taxableIncome = Math.max(income - 430000, 0);

  // 各区分の計算
  const medical = calcPortion(taxableIncome, city.medical);
  const support = calcPortion(taxableIncome, city.support);
  const nursing = age >= 40 && age < 65
    ? calcPortion(taxableIncome, city.nursing)
    : { incomeRate: 0, equalRate: 0, flatRate: 0, total: 0 };

  // 各区分に上限額がある
  const medicalCapped = Math.min(medical.total, city.medical.cap);
  const supportCapped = Math.min(support.total, city.support.cap);
  const nursingCapped = Math.min(nursing.total, city.nursing?.cap || 0);

  return {
    medical: medicalCapped,
    support: supportCapped,
    nursing: nursingCapped,
    total: medicalCapped + supportCapped + nursingCapped,
  };
}

function calcPortion(taxableIncome, rates) {
  const incomeRate = Math.floor(taxableIncome * rates.incomeRate);
  const equalRate = rates.equalRate;   // 均等割（1人あたり）
  const flatRate = rates.flatRate || 0; // 平等割（1世帯あたり）

  return {
    incomeRate,
    equalRate,
    flatRate,
    total: incomeRate + equalRate + flatRate,
  };
}
```

## 自治体別の料率データ

ここが一番やっかいなところ。全国約1,700の自治体それぞれに料率が異なる。現実的にはすべてを網羅するのは不可能なので、主要都市のデータをプリセットとして用意し、カスタム入力にも対応する。

```javascript
// 主要都市の国保料率（2026年度概算）
const NHI_RATES = {
  '東京23区': {
    medical: {
      incomeRate: 0.0769,  // 所得割 7.69%
      equalRate: 47300,     // 均等割 47,300円/人
      flatRate: 0,          // 平等割なし
      cap: 680000,          // 上限68万円
    },
    support: {
      incomeRate: 0.0280,
      equalRate: 16800,
      flatRate: 0,
      cap: 240000,
    },
    nursing: {
      incomeRate: 0.0222,
      equalRate: 17700,
      flatRate: 0,
      cap: 170000,
    },
  },
  '大阪市': {
    medical: {
      incomeRate: 0.0889,
      equalRate: 34976,
      flatRate: 34776,
      cap: 680000,
    },
    support: {
      incomeRate: 0.0316,
      equalRate: 12430,
      flatRate: 12359,
      cap: 240000,
    },
    nursing: {
      incomeRate: 0.0282,
      equalRate: 15616,
      flatRate: 7061,
      cap: 170000,
    },
  },
  '名古屋市': {
    medical: {
      incomeRate: 0.0805,
      equalRate: 41690,
      flatRate: 0,
      cap: 680000,
    },
    support: {
      incomeRate: 0.0280,
      equalRate: 14515,
      flatRate: 0,
      cap: 240000,
    },
    nursing: {
      incomeRate: 0.0244,
      equalRate: 15515,
      flatRate: 0,
      cap: 170000,
    },
  },
};
```

東京23区は「平等割なし」だが、大阪市は「平等割あり」。この構造の違いが同じコードで処理できるよう、`flatRate: 0` をデフォルトにしておく。

## 上限額の推移

国保の上限額は毎年引き上げ傾向にある。2026年度の上限額合計は**106万円**（医療68万+支援24万+介護17万=109万円）。

```javascript
function checkCap(result, rates) {
  const warnings = [];

  if (result.medical >= rates.medical.cap) {
    warnings.push(`医療分が上限${rates.medical.cap / 10000}万円に到達`);
  }
  if (result.support >= rates.support.cap) {
    warnings.push(`支援金分が上限${rates.support.cap / 10000}万円に到達`);
  }
  if (result.nursing >= (rates.nursing?.cap || 0)) {
    warnings.push(`介護分が上限${rates.nursing.cap / 10000}万円に到達`);
  }

  return warnings;
}

// 所得800万で上限チェック
const result = calcNationalHealthInsurance(8000000, 45, NHI_RATES['東京23区']);
const warnings = checkCap(result, NHI_RATES['東京23区']);
console.log(`国保年額: ${result.total.toLocaleString()}円`);
console.log(`警告: ${warnings.join(', ') || 'なし'}`);
```

## 軽減判定ロジック

所得が一定以下の場合、均等割と平等割が7割・5割・2割軽減される。

```javascript
function calcReduction(totalIncome, householdSize, rates) {
  // 軽減判定所得 = 世帯主の総所得 + 被保険者の総所得
  const baseAmount = 430000; // 基礎控除相当
  const perPerson = 295000;  // 5割軽減の加算額（1人あたり）
  const perPerson2 = 545000; // 2割軽減の加算額（1人あたり）

  let reductionRate = 0;

  // 7割軽減: 所得 ≤ 43万 + 10万 × (給与所得者数-1)
  if (totalIncome <= baseAmount) {
    reductionRate = 0.7;
  }
  // 5割軽減: 所得 ≤ 43万 + 29.5万 × 被保険者数
  else if (totalIncome <= baseAmount + perPerson * householdSize) {
    reductionRate = 0.5;
  }
  // 2割軽減: 所得 ≤ 43万 + 54.5万 × 被保険者数
  else if (totalIncome <= baseAmount + perPerson2 * householdSize) {
    reductionRate = 0.2;
  }

  // 均等割と平等割のみ軽減（所得割は軽減なし）
  const equalReduction = Math.floor(
    (rates.medical.equalRate + rates.support.equalRate + (rates.nursing?.equalRate || 0))
    * reductionRate
  );
  const flatReduction = Math.floor(
    ((rates.medical.flatRate || 0) + (rates.support.flatRate || 0) + (rates.nursing?.flatRate || 0))
    * reductionRate
  );

  return {
    reductionRate: `${reductionRate * 100}%`,
    equalReduction,
    flatReduction,
    totalReduction: equalReduction + flatReduction,
  };
}
```

## 概算モードの実装

ツールのUIでは、自治体を選択せずに「全国平均」で概算する機能も用意した。

```javascript
function calcNHIEstimate(income, age) {
  // 全国平均の概算料率
  const estimateRate = age >= 40 ? 0.145 : 0.12;
  const taxableIncome = Math.max(income - 430000, 0);

  const estimated = Math.floor(taxableIncome * estimateRate);
  const cap = age >= 40 ? 1060000 : 890000; // 上限

  return Math.min(estimated, cap);
}

// テスト: 各所得帯での国保概算
[3000000, 5000000, 7000000, 10000000].forEach(income => {
  const nhi = calcNHIEstimate(income, 35);
  console.log(`所得${income / 10000}万 → 国保概算: ${(nhi / 10000).toFixed(1)}万円/年`);
});
// 所得300万 → 国保概算: 30.8万円/年
// 所得500万 → 国保概算: 54.8万円/年
// 所得700万 → 国保概算: 78.8万円/年
// 所得1000万 → 国保概算: 89.0万円/年（上限到達）
```

## 社会保険との比較

法人化して社会保険に加入するのと、国保のままでいるのはどちらが有利か。この比較は[フリーランス単価計算ツール](https://and-tools.net/tools/freelance-rate-calc/)で試算できる。

```javascript
function compareInsurance(income, age) {
  // 国保 + 国民年金
  const nhi = calcNHIEstimate(income, age);
  const nationalPension = 16980 * 12; // 2026年度
  const freelanceTotal = nhi + nationalPension;

  // 社会保険（月給に換算）
  const monthly = Math.floor(income / 12);
  const si = calcSocialInsurance(monthly, '東京', age);
  const corpTotal = si.employeeTotal * 12;
  const corpWithCompany = (si.employeeTotal + si.companyTotal) * 12;

  return {
    freelance: freelanceTotal,
    corpEmployee: corpTotal,
    corpTotal: corpWithCompany,
    diff: freelanceTotal - corpTotal,
  };
}
```

[経費率 判定ツール](https://and-tools.net/tools/expense-rate/)も合わせて使うと、フリーランスの実質的な負担を把握しやすい。

## まとめ

国保の計算で押さえるべきポイント:

1. **3区分 × 3要素** — 医療・支援・介護それぞれに所得割・均等割・平等割がある
2. **自治体差が大きい** — 料率だけでなく、平等割の有無も異なる
3. **上限額は毎年変動** — 2026年度合計106万円
4. **軽減判定**で均等割・平等割が7割/5割/2割減額される
5. 正確な計算には自治体別データが必要だが、**概算では全国平均12〜14.5%**で十分

国保は「高い」と言われるが、実際にいくらかは所得と住んでいる自治体で決まる。引っ越しで保険料が変わることもあるので、複数自治体で試算することを勧める。

---

[AND TOOLS](https://and-tools.net/) では、フリーランス・個人事業主向けの税金計算ツールを34個以上無料で公開しています。
