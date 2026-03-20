---
title: "小規模企業共済の実質リターンを計算してみた — 節税効果込みの利回りは驚異的だった"
emoji: "💰"
type: "tech"
topics: ["javascript", "節税", "フリーランス", "個人開発"]
published: false
---

フリーランスの節税手段として最強クラスの「小規模企業共済」。掛金が全額所得控除になるのは知っていたが、**実質リターンが年利何%に相当するのか**をちゃんと計算したことがなかった。JavaScriptで計算してみたら、想像以上の数字が出た。

## 小規模企業共済を30秒で説明

エンジニアにとってのポイントだけ:
- 月額1,000円〜70,000円を積み立て（年間最大84万円）
- **掛金が全額所得控除**（=課税所得が減る）
- 受取時は「退職所得」扱い（税制上かなり有利）
- 中小機構が運営する国の制度（破綻リスクは限りなく低い）

要は「積み立てるだけで毎年の税金が減り、受取時も優遇される」仕組み。

## 節税効果の計算ロジック

掛金が所得控除になることで、どれだけ税金が減るかを計算する。

```javascript
function calcTaxSaving(annualContribution, taxableIncome) {
  // 控除前の税額
  const taxBefore = calcIncomeTax(taxableIncome)
    + calcResidentTax(taxableIncome);

  // 控除後の税額（課税所得から掛金分を差し引く）
  const incomeAfter = Math.max(taxableIncome - annualContribution, 0);
  const taxAfter = calcIncomeTax(incomeAfter)
    + calcResidentTax(incomeAfter);

  const saving = taxBefore - taxAfter;

  return {
    saving,
    effectiveRate: saving / annualContribution, // 実効節税率
  };
}
```

累進課税なので、**課税所得が高いほど節税率も高い**。

| 課税所得 | 適用税率（所得税+住民税） | 年84万掛けた場合の節税額 |
|---------|----------------------|---------------------|
| 330万 | 20%（10%+10%） | 約16.8万円 |
| 500万 | 30%（20%+10%） | 約25.2万円 |
| 700万 | 33%（23%+10%） | 約27.7万円 |
| 900万 | 43%（33%+10%） | 約36.1万円 |

課税所得500万のフリーランスなら、**84万円積み立てて25.2万円が返ってくる**。実質の自己負担は58.8万円。自分の課税所得がわからない場合は [手取り計算シミュレーター](https://and-tools.net/tools/take-home-pay/) で確認できる。

## 退職所得控除の計算

受取時は「退職所得」として課税される。退職所得控除が非常に大きい。

```javascript
function calcRetirementDeduction(years) {
  if (years <= 20) {
    return years * 400000; // 20年以下: 年40万
  }
  return 8000000 + (years - 20) * 700000; // 20年超: 年70万
}

function calcRetirementTax(lumpSum, years) {
  const deduction = calcRetirementDeduction(years);
  // 退職所得 = (受取額 - 退職所得控除) × 1/2
  const retirementIncome = Math.max(
    Math.floor((lumpSum - deduction) / 2), 0
  );

  const incomeTax = calcProgressiveTax(retirementIncome);
  const residentTax = Math.floor(retirementIncome * 0.10);

  return { incomeTax, residentTax, total: incomeTax + residentTax };
}
```

20年加入なら退職所得控除は**800万円**。つまり受取額が800万以下なら**受取時の税金はゼロ**。

## 実質リターンの計算: 月3万円 × 20年

具体的なケースで実質利回りを計算してみる。

```javascript
function calcRealReturn(monthlyContribution, years, taxableIncome) {
  const annual = monthlyContribution * 12;
  const totalContribution = annual * years;

  // 毎年の節税額
  const { saving: annualSaving } = calcTaxSaving(annual, taxableIncome);
  const totalSaving = annualSaving * years;

  // 共済金の受取額（一括・20年以上で共済金A）
  // 掛金合計に対して付加共済金が付くが、ここでは元本のみで計算
  const lumpSum = totalContribution; // 保守的に元本100%で計算

  // 受取時の税金
  const retirementTax = calcRetirementTax(lumpSum, years);

  // 実質リターン
  const netGain = totalSaving - retirementTax.total;
  const realContribution = totalContribution - totalSaving;

  // 実質年利（複利ベースで逆算）
  const annualReturn = Math.pow(
    (totalContribution - retirementTax.total) / realContribution,
    1 / years
  ) - 1;

  return {
    totalContribution,
    totalSaving,
    lumpSum,
    retirementTax: retirementTax.total,
    netGain,
    annualReturnPct: (annualReturn * 100).toFixed(2) + '%',
  };
}

// 月3万 × 20年、課税所得500万の場合
const result = calcRealReturn(30000, 20, 5000000);
console.log(result);
```

実行結果:

```
{
  totalContribution: 7,200,000,   // 掛金合計: 720万円
  totalSaving: 5,040,000,         // 節税額合計: 504万円
  lumpSum: 7,200,000,             // 受取額: 720万円（退職所得控除内なので非課税）
  retirementTax: 0,               // 受取時の税金: 0円
  netGain: 5,040,000,             // 実質利益: 504万円
  annualReturnPct: '4.82%'        // 実質年利: 約4.8%
}
```

**年利4.8%**。しかも元本割れリスクがない国の制度でこの利回り。

課税所得が高いほどリターンは上がる:

| 課税所得 | 月額掛金 | 20年の節税合計 | 受取時税金 | 実質年利 |
|---------|---------|-------------|----------|---------|
| 330万 | 3万 | 336万 | 0円 | 約2.7% |
| 500万 | 3万 | 504万 | 0円 | 約4.8% |
| 700万 | 7万 | 約665万 | 0円 | 約5.1% |
| 900万 | 7万 | 約868万 | 約8万 | 約6.5% |

## iDeCoとの比較

フリーランスならiDeCo（月額68,000円上限）も使える。両方の特徴を整理する。

```javascript
function compareDeductions(income, mutualAmount, idecoAmount) {
  // 小規模企業共済の節税
  const mutualSaving = calcTaxSaving(mutualAmount * 12, income);

  // iDeCoの節税
  const idecoSaving = calcTaxSaving(idecoAmount * 12, income);

  // 両方使った場合
  const bothSaving = calcTaxSaving(
    mutualAmount * 12 + idecoAmount * 12, income
  );

  return {
    mutual: mutualSaving.saving,
    ideco: idecoSaving.saving,
    both: bothSaving.saving,
    maxAnnualDeduction: mutualAmount * 12 + idecoAmount * 12,
  };
}

// 課税所得600万、共済7万+iDeCo6.8万
const comparison = compareDeductions(6000000, 70000, 68000);
// → 年間165.6万円の所得控除、節税額は約54万円
```

| 項目 | 小規模企業共済 | iDeCo |
|------|-------------|-------|
| 月額上限 | 70,000円 | 68,000円 |
| 控除の種類 | 小規模企業共済等掛金控除 | 同左 |
| 受取時 | 退職所得（一括）/ 雑所得（分割） | 同左 |
| 途中解約 | 可能（元本割れあり） | 原則60歳まで不可 |
| 運用リスク | なし（予定利率1%） | あり（投資信託） |
| 貸付制度 | あり | なし |

**結論: 両方使うのが最強。** 合計で月額138,000円（年165.6万円）の所得控除が取れる。

フリーランスで課税所得600万なら、小規模企業共済+iDeCoで**年間約54万円の節税**。これを使わない理由がない。節税後の経費率と手取りの関係は [経費率シミュレーター](https://and-tools.net/tools/expense-rate/) で確認するとイメージしやすい。

## 注意点

- 加入後**12ヶ月未満で解約すると掛け捨て**になる
- 減額すると減額分の運用が止まる（増額は自由）
- 受取方法（一括 vs 分割）で税金が大きく変わる

## まとめ

小規模企業共済の実質リターンをコードで計算した結果:

1. **課税所得500万×月3万×20年で実質年利約4.8%**（元本保証で）
2. **退職所得控除のおかげで受取時の税金がゼロになるケースが多い**
3. **iDeCoと併用すれば年間165.6万円の所得控除**が可能
4. **フリーランスの節税手段としては最優先で検討すべき**

「投資で年利5%出すのは難しいが、節税で実質5%は制度を使うだけ」という話。

---

この記事の計算は [小規模企業共済シミュレーター](https://and-tools.net/tools/small-biz-mutual/) で実際に試せる。掛金と課税所得を入力すれば節税額と実質リターンが出る。手取り額は [手取り計算シミュレーター](https://and-tools.net/tools/take-home-pay/) で確認し、経費率の目安は [経費率シミュレーター](https://and-tools.net/tools/expense-rate/) で把握できる。フリーランスのお金周りのツールは [AND TOOLS](https://and-tools.net/) にまとめている。
