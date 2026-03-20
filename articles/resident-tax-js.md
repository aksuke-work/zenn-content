---
title: "住民税の所得割+均等割をJavaScriptで計算する — 所得税との違いを実装で理解"
emoji: "🏠"
type: "tech"
topics: ["javascript", "tax", "住民税", "個人開発"]
published: false
---

所得税は累進課税（5%〜45%）だが、住民税は**一律10%**。この単純さの裏に、所得税との微妙な差異が隠れている。[AND TOOLS](https://and-tools.net/) の[住民税計算ツール](https://and-tools.net/tools/resident-tax/)を実装しながら気づいた落とし穴をまとめる。

## 住民税の2つの構成要素

住民税は「所得割」と「均等割」で構成される。

```javascript
function calcResidentTax(taxableIncome) {
  // 非課税判定（単身者の場合）
  if (taxableIncome <= 450000) {
    return { incomePortion: 0, equalPortion: 0, total: 0 };
  }

  // 所得割: 一律10%（市町村民税6% + 道府県民税4%）
  const incomePortion = Math.floor(taxableIncome * 0.10);

  // 調整控除（基礎控除の差額を調整）
  const adjustDeduction = calcAdjustDeduction(taxableIncome);

  // 均等割: 5,000円（市町村3,500円 + 都道府県1,500円）
  // ※2024年度〜 森林環境税1,000円を含む
  const equalPortion = 5000;

  const total = Math.max(incomePortion - adjustDeduction, 0) + equalPortion;

  return { incomePortion, equalPortion, adjustDeduction, total };
}
```

## 所得税と住民税の基礎控除の差

ここが最大のハマりポイント。所得税の基礎控除は48万円（2025年〜58万円）だが、住民税は**43万円**のまま据え置き。この5万円の差を「調整控除」で補正する。

```javascript
function calcAdjustDeduction(taxableIncome) {
  // 合計課税所得200万円以下
  if (taxableIncome <= 2000000) {
    // 人的控除の差の合計額 と 課税所得の小さい方 × 5%
    const diff = 50000; // 基礎控除の差額（48万-43万=5万）
    return Math.floor(Math.min(diff, taxableIncome) * 0.05);
  }

  // 合計課税所得200万円超
  const diff = 50000;
  const adjusted = diff - (taxableIncome - 2000000);
  // 最低2,500円
  return Math.max(Math.floor(adjusted * 0.05), 2500);
}
```

この調整控除は住民税にしか存在しない。実装時に「所得税と同じ控除額でいいだろう」と思い込むとズレる。

## 住民税の課税所得を計算する

住民税用の課税所得は、所得税とは各種控除の金額が異なる。

```javascript
function calcResidentTaxableIncome(income, deductions) {
  // 住民税用の控除額（所得税より低い）
  const residentDeductions = {
    basic: 430000,            // 基礎控除: 43万（所得税48万）
    spouse: 330000,           // 配偶者控除: 33万（所得税38万）
    dependent: 330000,        // 一般扶養: 33万（所得税38万）
    specificDependent: 450000, // 特定扶養: 45万（所得税63万）
    elderlyDependent: 380000,  // 老人扶養: 38万（所得税48万）
  };

  let totalDeduction = residentDeductions.basic;

  // 社会保険料控除は所得税と同額
  totalDeduction += deductions.socialInsurance || 0;

  // 生命保険料控除は上限が異なる（住民税: 最大7万円、所得税: 最大12万円）
  totalDeduction += Math.min(deductions.lifeInsurance || 0, 70000);

  // 扶養控除
  if (deductions.hasSpouse) totalDeduction += residentDeductions.spouse;
  totalDeduction += (deductions.dependents || 0) * residentDeductions.dependent;
  totalDeduction += (deductions.specificDependents || 0) * residentDeductions.specificDependent;

  return Math.max(income - totalDeduction, 0);
}
```

主な差額をテーブルにすると:

| 控除の種類 | 所得税 | 住民税 | 差額 |
|-----------|--------|--------|------|
| 基礎控除 | 48万円 | 43万円 | 5万円 |
| 配偶者控除 | 38万円 | 33万円 | 5万円 |
| 一般扶養控除 | 38万円 | 33万円 | 5万円 |
| 特定扶養控除 | 63万円 | 45万円 | 18万円 |
| 生命保険料控除上限 | 12万円 | 7万円 | 5万円 |

## ふるさと納税の控除上限を概算する

住民税を理解すると、ふるさと納税の控除上限も計算できるようになる。控除上限額は**住民税所得割の20%**がベースになる。

```javascript
function calcFurusatoLimit(income, socialInsurance) {
  // 住民税の課税所得（簡易版）
  const taxableIncome = income - socialInsurance - 430000; // 基礎控除43万
  if (taxableIncome <= 0) return 0;

  // 住民税所得割
  const residentIncomeTax = Math.floor(taxableIncome * 0.10);

  // 所得税の限界税率を取得
  const incomeTaxRate = getIncomeTaxRate(taxableIncome);

  // 控除上限額の計算式
  // 上限 = 住民税所得割 × 20% ÷ (100% - 住民税率10% - 所得税率×1.021) + 2000
  const limit = Math.floor(
    residentIncomeTax * 0.20 / (1 - 0.10 - incomeTaxRate * 1.021)
  ) + 2000;

  return limit;
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
```

この計算の詳細は [住民税 計算ツール](https://and-tools.net/tools/resident-tax/)で確認できる。

## 住民税の非課税ライン

住民税には非課税の基準がある。所得税とは別の非課税ラインが存在する点に注意。

```javascript
function isResidentTaxExempt(income, location, dependents) {
  // 均等割の非課税基準（自治体により異なる）
  // 1級地（東京23区等）: 45万円 × (扶養人数+1) + 21万（扶養あり時）
  // 2級地: 41.5万 × (扶養人数+1) + 18.9万
  // 3級地: 38万 × (扶養人数+1) + 16.8万

  const grades = {
    1: { base: 450000, add: 210000 },
    2: { base: 415000, add: 189000 },
    3: { base: 380000, add: 168000 },
  };

  const grade = grades[location] || grades[1];
  const threshold = grade.base * (dependents + 1)
    + (dependents > 0 ? grade.add : 0);

  return income <= threshold;
}

// 東京23区、単身者
console.log(isResidentTaxExempt(450000, 1, 0)); // true
console.log(isResidentTaxExempt(460000, 1, 0)); // false
```

## 手取り計算への組み込み

所得税と住民税の両方を計算して手取りを出す。

```javascript
function calcTakeHome(annualIncome) {
  const salaryDeduction = calcSalaryDeduction(annualIncome);
  const salaryIncome = annualIncome - salaryDeduction;

  // 社会保険料（概算: 年収の約15%）
  const socialInsurance = Math.floor(annualIncome * 0.15);

  // 所得税の課税所得
  const incomeTaxBase = Math.max(salaryIncome - socialInsurance - 480000, 0);
  const incomeTax = calcIncomeTax(incomeTaxBase);

  // 住民税の課税所得（基礎控除が5万円少ない）
  const residentTaxBase = Math.max(salaryIncome - socialInsurance - 430000, 0);
  const residentTax = calcResidentTax(residentTaxBase).total;

  const takeHome = annualIncome - incomeTax - residentTax - socialInsurance;

  return { annualIncome, takeHome, incomeTax, residentTax, socialInsurance };
}
```

[手取り計算シミュレーター](https://and-tools.net/tools/take-home-pay/)では、扶養家族や各種控除を加味した精密な計算ができる。

## まとめ

住民税の実装で注意すべきポイント:

1. **税率は一律10%**だが、控除額が所得税と異なる
2. **基礎控除の差額5万円**は調整控除で補正される
3. **均等割5,000円**（森林環境税含む）は所得割とは別計算
4. **非課税ラインは自治体の級地**で異なる
5. ふるさと納税の上限額は**住民税所得割の20%**が基準

住民税は所得税の「ほぼコピー」に見えて、控除額の差異がいたるところに潜んでいる。[178万円の壁 シミュレーション](https://and-tools.net/tools/178-wall/)でも住民税と所得税の両面から壁の影響を可視化できる。

---

[AND TOOLS](https://and-tools.net/) では、フリーランス・個人事業主向けの税金計算ツールを34個以上無料で公開しています。
