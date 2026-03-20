---
title: "退職所得控除の計算 — 一括受取と分割受取の税金比較をJSで実装"
emoji: "🎓"
type: "tech"
topics: ["javascript", "退職金", "節税", "個人開発"]
published: false
---

退職金は通常の所得とは異なる税計算が適用される。勤続年数に応じた「退職所得控除」と「1/2課税」により、大幅に税負担が軽減される仕組みだ。[AND TOOLS](https://and-tools.net/) の[退職金手取り計算ツール](https://and-tools.net/tools/retirement-pay/)で実装したロジックを解説する。

## 退職所得控除の計算

退職所得控除は勤続年数で2段階に分かれる。20年を境に計算方法が変わる。

```javascript
function calcRetirementDeduction(yearsOfService) {
  if (yearsOfService <= 0) return 0;

  if (yearsOfService <= 20) {
    // 20年以下: 40万円 × 勤続年数（最低80万円）
    return Math.max(400000 * yearsOfService, 800000);
  }

  // 20年超: 800万円 + 70万円 × (勤続年数 - 20年)
  return 8000000 + 700000 * (yearsOfService - 20);
}

// テスト
const testYears = [5, 10, 15, 20, 25, 30, 35, 38];
testYears.forEach(years => {
  const deduction = calcRetirementDeduction(years);
  console.log(
    `勤続${years}年 → 控除額: ${(deduction / 10000).toFixed(0)}万円`
  );
});
// 勤続5年 → 控除額: 200万円
// 勤続10年 → 控除額: 400万円
// 勤続15年 → 控除額: 600万円
// 勤続20年 → 控除額: 800万円
// 勤続25年 → 控除額: 1150万円
// 勤続30年 → 控除額: 1500万円
// 勤続35年 → 控除額: 1850万円
// 勤続38年 → 控除額: 2060万円
```

20年超で1年あたりの控除額が40万→70万に跳ね上がる。長期勤続者ほど有利な設計だ。

## 退職所得の計算（1/2課税）

退職所得は控除後の金額をさらに**1/2にしてから**課税する。この1/2課税が退職金の税負担を大幅に下げるポイント。

```javascript
function calcRetirementIncome(severancePay, yearsOfService, isOfficer, officerYears) {
  const deduction = calcRetirementDeduction(yearsOfService);
  const afterDeduction = Math.max(severancePay - deduction, 0);

  // 特定役員退職手当等（勤続5年以下の役員）は1/2課税なし
  if (isOfficer && officerYears <= 5) {
    return afterDeduction;
  }

  // 短期退職手当等（勤続5年以下の一般従業員）
  // 2022年〜: 300万円超の部分は1/2課税なし
  if (yearsOfService <= 5 && !isOfficer) {
    if (afterDeduction <= 3000000) {
      return Math.floor(afterDeduction / 2);
    }
    // 300万超の部分は全額課税
    return Math.floor(3000000 / 2) + (afterDeduction - 3000000);
  }

  // 通常: 1/2課税
  return Math.floor(afterDeduction / 2);
}
```

2022年の改正で、勤続5年以下の一般従業員も300万円超部分は1/2課税が適用されなくなった。短期転職を繰り返して退職金を受け取る「退職金課税の抜け穴」を塞ぐ趣旨だ。

## 退職金にかかる所得税と住民税

退職所得は**分離課税**。他の所得とは合算せず、退職所得だけで税額を計算する。

```javascript
function calcRetirementTax(severancePay, yearsOfService, isOfficer = false, officerYears = 0) {
  const retirementIncome = calcRetirementIncome(
    severancePay, yearsOfService, isOfficer, officerYears
  );

  // 所得税（累進課税 × 復興特別所得税）
  const incomeTax = calcIncomeTax(retirementIncome);

  // 住民税（一律10%）
  const residentTax = Math.floor(retirementIncome * 0.10);

  const totalTax = incomeTax + residentTax;
  const takeHome = severancePay - totalTax;

  return {
    severancePay,
    deduction: calcRetirementDeduction(yearsOfService),
    retirementIncome,
    incomeTax,
    residentTax,
    totalTax,
    takeHome,
    effectiveRate: ((totalTax / severancePay) * 100).toFixed(2) + '%',
  };
}

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
      return Math.floor(taxableIncome * b.rate - b.deduction) * 1.021 | 0;
    }
  }
  return 0;
}

// 勤続30年、退職金2000万の場合
const result = calcRetirementTax(20000000, 30);
console.log(result);
// deduction: 1500万, retirementIncome: 250万
// incomeTax: 約15.6万, residentTax: 25万
// takeHome: 約1959.4万, effectiveRate: 約2.03%
```

退職金2,000万円に対して実効税率が約2%。1/2課税と大きな控除額の恩恵がよく分かる。

## 一括受取 vs 分割受取の比較

退職金を一括で受け取るか、年金として分割で受け取るか。税負担が大きく変わる。

```javascript
function compareLumpSumVsAnnuity(totalAmount, yearsOfService, annuityYears, annualOtherIncome) {
  // 一括受取
  const lumpSum = calcRetirementTax(totalAmount, yearsOfService);

  // 分割受取（公的年金等控除を適用）
  const annualAmount = Math.floor(totalAmount / annuityYears);
  let totalAnnuityTax = 0;

  for (let i = 0; i < annuityYears; i++) {
    // 公的年金等控除（65歳未満の場合）
    const pensionDeduction = calcPensionDeduction(annualAmount, false);
    const pensionIncome = Math.max(annualAmount - pensionDeduction, 0);

    // 他の所得と合算して総合課税
    const totalIncome = pensionIncome + annualOtherIncome;
    const taxableIncome = Math.max(totalIncome - 480000, 0); // 基礎控除
    const tax = calcIncomeTax(taxableIncome);
    const residentTax = Math.floor(Math.max(totalIncome - 430000, 0) * 0.10);
    totalAnnuityTax += tax + residentTax;
  }

  return {
    lumpSum: {
      takeHome: lumpSum.takeHome,
      totalTax: lumpSum.totalTax,
      effectiveRate: lumpSum.effectiveRate,
    },
    annuity: {
      takeHome: totalAmount - totalAnnuityTax,
      totalTax: totalAnnuityTax,
      effectiveRate: ((totalAnnuityTax / totalAmount) * 100).toFixed(2) + '%',
    },
    advantage: lumpSum.totalTax < totalAnnuityTax ? '一括受取が有利' : '分割受取が有利',
  };
}

function calcPensionDeduction(annualPension, isOver65) {
  if (isOver65) {
    if (annualPension <= 1100000) return annualPension;
    if (annualPension <= 3300000) return annualPension * 0.25 + 275000;
    if (annualPension <= 7700000) return annualPension * 0.15 + 685000;
    if (annualPension <= 10000000) return annualPension * 0.05 + 1455000;
    return 1955000;
  }
  // 65歳未満
  if (annualPension <= 700000) return annualPension;
  if (annualPension <= 1300000) return annualPension * 0.25 + 275000;
  if (annualPension <= 4100000) return annualPension * 0.15 + 685000;
  if (annualPension <= 7700000) return annualPension * 0.05 + 1455000;
  return 1955000;
}
```

## 小規模企業共済の退職所得控除

フリーランスが活用する小規模企業共済の解約金も退職所得扱いになる。掛金の納付年数が勤続年数に相当する。

```javascript
function calcSmallBizMutualRetirement(totalReceived, paymentYears) {
  // 共済金Aまたは共済金B → 退職所得扱い
  const result = calcRetirementTax(totalReceived, paymentYears);

  // 掛金累計との比較
  console.log(`受取額: ${(totalReceived / 10000).toFixed(0)}万円`);
  console.log(`控除額: ${(result.deduction / 10000).toFixed(0)}万円`);
  console.log(`手取り: ${(result.takeHome / 10000).toFixed(0)}万円`);
  console.log(`実効税率: ${result.effectiveRate}`);

  return result;
}

// 月額7万×20年 = 1,680万の掛金 → 受取約1,850万（共済金A）
calcSmallBizMutualRetirement(18500000, 20);
```

小規模企業共済の詳しいシミュレーションは [小規模企業共済 計算ツール](https://and-tools.net/tools/small-biz-mutual/)で試算できる。

## iDeCoとの退職所得控除の重複問題

iDeCoの一時金受取と退職金を同時に受け取ると、退職所得控除が重複して使えない場合がある。

```javascript
function calcOverlappingRetirement(retirement, ideco) {
  // 退職金の控除
  const retDeduction = calcRetirementDeduction(retirement.years);

  // iDeCoの控除
  const idecoDeduction = calcRetirementDeduction(ideco.years);

  // 重複期間がある場合、大きい方の控除を使い、差額分のみ追加
  const overlapYears = Math.min(retirement.years, ideco.years);

  if (overlapYears > 0) {
    // 先に受け取った方の控除をフルに使い、
    // 後から受け取る方は重複期間分を差し引く
    const firstDeduction = retDeduction; // 退職金を先に受取
    const overlapDeduction = calcRetirementDeduction(overlapYears);
    const secondDeduction = Math.max(idecoDeduction - overlapDeduction, 0);

    console.log(`退職金控除: ${(firstDeduction / 10000).toFixed(0)}万円`);
    console.log(`iDeCo控除: ${(secondDeduction / 10000).toFixed(0)}万円（重複調整後）`);
    console.log(`→ iDeCoは退職金受取の${19}年後に受け取ると控除をフル活用できる`);

    return { retDeduction: firstDeduction, idecoDeduction: secondDeduction };
  }

  return { retDeduction: retDeduction, idecoDeduction: idecoDeduction };
}
```

この重複問題を避けるには、退職金受取から一定期間（現行制度では19年）空ける必要がある。制度の詳細は[退職金手取り計算ツール](https://and-tools.net/tools/retirement-pay/)で確認できる。

## まとめ

退職所得の税計算で押さえるべきポイント:

1. **退職所得控除は2段階** — 20年以下は40万/年、20年超は70万/年
2. **1/2課税**で実効税率が大幅に下がる（勤続5年以下は制限あり）
3. **分離課税**なので他の所得とは合算しない
4. **一括 vs 分割**の損得は他の所得の多寡で変わる
5. **iDeCoとの重複**に注意 — 受取タイミングの設計が重要

退職金は人生で最大級の一時金。受取方法で手取りが数十万〜数百万変わる。[手取り計算シミュレーター](https://and-tools.net/tools/take-home-pay/)と合わせて使い、事前にシミュレーションしておくことを強く勧める。

---

[AND TOOLS](https://and-tools.net/) では、フリーランス・個人事業主向けの税金計算ツールを34個以上無料で公開しています。
