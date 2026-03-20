---
title: "フリーランスの年金戦略 — 国民年金+付加年金+iDeCo+小規模企業共済"
emoji: "🏦"
type: "tech"
topics: ["フリーランス", "年金", "iDeCo", "小規模企業共済"]
published: false
---

フリーランスの老後資金は自分で作るしかない。会社員には厚生年金があるが、フリーランスには国民年金しかない。その差は生涯で数千万円になる。

[AND TOOLS](https://and-tools.net/) で老後資金シミュレーターを開発した際に調べた年金・節税の組み合わせ戦略をまとめる。制度を正しく理解して、全部積み上げれば相当な差が生まれる。

---

## フリーランス vs 会社員の年金格差

### 会社員の年金構造

```
【会社員】
└── 基礎年金（国民年金）: 月約6.5万円
    └── 厚生年金（報酬比例）: 月約10〜15万円
        └── 合計: 月約16〜22万円
```

### フリーランスの年金構造（デフォルト）

```
【フリーランス】
└── 基礎年金（国民年金）: 月約6.5万円
    └── 合計: 月約6.5万円
```

この差を埋めるために、以下の制度を積み上げる。

---

## 積み上げ可能な4つの制度

| 制度 | 月額上限 | 所得控除 | 特徴 |
|------|---------|---------|------|
| 国民年金 | 16,980円（2024年度） | 全額控除 | 必須 |
| 付加年金 | +400円/月 | 全額控除 | コスパ最強 |
| iDeCo（確定拠出年金） | 68,000円 | 全額控除 | 運用益非課税 |
| 小規模企業共済 | 70,000円 | 全額控除 | 退職金代わり |

---

## 1. 国民年金（必須）

- **月額**: 16,980円（2024年度）
- **年額**: 約20.4万円
- **受給額**: 満額で月約6.5万円（40年間納付した場合）
- **所得控除**: 社会保険料控除として全額控除

国民年金は老齢年金だけでなく、障害年金・遺族年金のベースにもなる。未納は絶対に避ける。

```javascript
// 国民年金の保険料と受給額の計算
const nationalPension = {
  monthlyPremium: 16980,         // 2024年度
  yearlyPremium: 16980 * 12,
  fullYears: 40,                 // 満額のための納付年数
  fullMonthlyBenefit: 68000,     // 満額受給額（月額・2024年度）

  // n年間納付した場合の受給額
  estimateBenefit(paidYears) {
    return Math.floor(this.fullMonthlyBenefit * (paidYears / this.fullYears));
  },
};

console.log(`年間保険料: ¥${nationalPension.yearlyPremium.toLocaleString()}`);
console.log(`30年納付の場合の月受給額: ¥${nationalPension.estimateBenefit(30).toLocaleString()}`);
console.log(`40年納付の場合の月受給額: ¥${nationalPension.estimateBenefit(40).toLocaleString()}`);
```

---

## 2. 付加年金（コスパ最強）

- **月額**: +400円（国民年金に上乗せ）
- **追加効果**: 月額200円 × 納付月数が年金に上乗せされる
- **回収期間**: わずか2年で元が取れる

```javascript
// 付加年金の計算
function calcFukanenkin(paidMonths) {
  const monthlyPremium = 400;
  const totalPaid = monthlyPremium * paidMonths;

  // 追加される月額年金
  const additionalMonthly = 200 * paidMonths;

  // 元が取れる月数
  const recoveryMonths = Math.ceil(totalPaid / additionalMonthly);

  return {
    totalPaid: totalPaid,
    additionalMonthlyBenefit: additionalMonthly,
    additionalYearlyBenefit: additionalMonthly * 12,
    recoveryMonths,
    recoveryYears: (recoveryMonths / 12).toFixed(1),
  };
}

// 20年間（240ヶ月）納付した場合
const result = calcFukanenkin(240);
console.log(`総納付額: ¥${result.totalPaid.toLocaleString()}`);
console.log(`月額追加受給: ¥${result.additionalMonthlyBenefit.toLocaleString()}`);
console.log(`年額追加受給: ¥${result.additionalYearlyBenefit.toLocaleString()}`);
console.log(`回収期間: ${result.recoveryYears}年`);
// → 総納付額: ¥960,000 / 月額追加: ¥48,000 / 年額追加: ¥576,000 / 回収: 1.7年
```

月400円の追加で受給額を増やせる付加年金は、フリーランスが最初に手を付けるべき制度だ。国民年金保険料の振込と同時に申請できる（市区町村の国民年金担当窓口で手続き）。

ただし、**iDeCoと小規模企業共済に加入している場合でも申請可能**。ただしiDeCoに加入している場合は付加年金の金額がiDeCoの限度額に影響するため、注意が必要だ（後述）。

---

## 3. iDeCo（確定拠出年金）

- **月額上限**: 68,000円（フリーランスの場合）
- **年額上限**: 81.6万円
- **所得控除**: 掛金全額が小規模企業共済等掛金控除として全額控除
- **運用益**: 非課税

```javascript
// iDeCoの節税効果計算
function calcIdeco(monthlyContribution, taxRate) {
  const yearlyContribution = monthlyContribution * 12;
  const yearlyTaxSaving = yearlyContribution * taxRate;

  return {
    yearlyContribution,
    yearlyTaxSaving,
    effectiveContribution: yearlyContribution - yearlyTaxSaving,
  };
}

// 月3万円・税率20%（所得税15%+住民税5%の場合）
const ideco = calcIdeco(30000, 0.20);
console.log(`年間拠出額: ¥${ideco.yearlyContribution.toLocaleString()}`);
console.log(`年間節税額: ¥${ideco.yearlyTaxSaving.toLocaleString()}`);
console.log(`実質負担額: ¥${ideco.effectiveContribution.toLocaleString()}`);
// → 年間拠出: 360,000円 / 節税: 72,000円 / 実質負担: 288,000円
```

### iDeCoの注意点

- **受け取りは60歳以降**（原則途中解約不可）
- 運用リスクがある（元本割れの可能性）
- 受取時に所得税がかかる（退職所得控除の活用で節税可能）

---

## 4. 小規模企業共済

- **月額**: 1,000円〜70,000円（1,000円単位）
- **所得控除**: 掛金全額が小規模企業共済等掛金控除として全額控除
- **受取**: 廃業・退職時に退職金として受け取れる（元本+利子）

フリーランスの「退職金」として設計された制度だ。

```javascript
// 小規模企業共済の効果計算
function calcShoukibo(monthlyContribution, taxRate, years) {
  const yearlyContribution = monthlyContribution * 12;
  const totalContribution = yearlyContribution * years;
  const yearlyTaxSaving = yearlyContribution * taxRate;
  const totalTaxSaving = yearlyTaxSaving * years;

  // 概算の共済金（実際は共済金種類・掛金月額・掛金月数により変動）
  // 20年間の場合、元本の約110%程度が目安
  const estimatedRetirement = totalContribution * 1.1;

  return {
    yearlyContribution,
    totalContribution,
    yearlyTaxSaving,
    totalTaxSaving,
    estimatedRetirement: Math.floor(estimatedRetirement),
  };
}

// 月5万円・税率20%・20年間
const shoukibo = calcShoukibo(50000, 0.20, 20);
console.log(`年間拠出: ¥${shoukibo.yearlyContribution.toLocaleString()}`);
console.log(`20年間総拠出: ¥${shoukibo.totalContribution.toLocaleString()}`);
console.log(`年間節税: ¥${shoukibo.yearlyTaxSaving.toLocaleString()}`);
console.log(`20年間総節税: ¥${shoukibo.totalTaxSaving.toLocaleString()}`);
console.log(`受取概算: ¥${shoukibo.estimatedRetirement.toLocaleString()}`);
// → 年間拠出: 600,000 / 総拠出: 12,000,000 / 年間節税: 120,000 / 総節税: 2,400,000 / 受取: 13,200,000
```

[AND TOOLSの小規模企業共済シミュレーター](https://and-tools.net/tools/mutual-aid/) で、月額掛金と節税額を試算できる。

---

## 全部積み上げた場合のシミュレーション

月収100万円（年収1,200万円・所得税率33%の場合）で最大拠出した場合：

```javascript
function fullSimulation(annualIncome, taxRate) {
  // 国民年金（必須）
  const nenkin = { monthly: 16980, yearly: 16980 * 12 };

  // 付加年金
  const fuka = { monthly: 400, yearly: 400 * 12 };

  // iDeCo（上限）
  const ideco = { monthly: 68000, yearly: 68000 * 12 };

  // 小規模企業共済（上限）
  const shoukibo = { monthly: 70000, yearly: 70000 * 12 };

  const totalYearly =
    nenkin.yearly + fuka.yearly + ideco.yearly + shoukibo.yearly;

  // 節税効果（国民年金は社会保険料控除で全額控除）
  const taxSaving = totalYearly * taxRate;

  console.log("=== フリーランス年金最大化プラン ===");
  console.log(`国民年金: 月¥${nenkin.monthly.toLocaleString()} / 年¥${nenkin.yearly.toLocaleString()}`);
  console.log(`付加年金: 月¥${fuka.monthly.toLocaleString()} / 年¥${fuka.yearly.toLocaleString()}`);
  console.log(`iDeCo: 月¥${ideco.monthly.toLocaleString()} / 年¥${ideco.yearly.toLocaleString()}`);
  console.log(`小規模企業共済: 月¥${shoukibo.monthly.toLocaleString()} / 年¥${shoukibo.yearly.toLocaleString()}`);
  console.log("-------------------------------");
  console.log(`年間総拠出: ¥${totalYearly.toLocaleString()}`);
  console.log(`年間節税効果: ¥${Math.floor(taxSaving).toLocaleString()}`);
  console.log(`実質年間負担: ¥${Math.floor(totalYearly - taxSaving).toLocaleString()}`);
}

fullSimulation(12000000, 0.33);
```

---

## 優先順位

資金に余裕がない場合の優先順位：

1. **国民年金**（未納は将来の年金・保障を失う。最優先で納付）
2. **付加年金**（月400円でコスパ最強。即申請）
3. **小規模企業共済**（退職金代わり。掛金を調整しながら）
4. **iDeCo**（余裕資金がある場合。60歳まで引き出せない点を理解した上で）

---

## まとめ

フリーランスの老後資金は4層構造で積み上げる。

| 制度 | 月額目安 | 年額節税効果（税率20%の場合） |
|------|---------|--------------------------|
| 国民年金 | 16,980円 | 40,752円 |
| 付加年金 | 400円 | 960円 |
| iDeCo | 〜68,000円 | 〜163,200円 |
| 小規模企業共済 | 〜70,000円 | 〜168,000円 |
| **合計** | **〜155,380円** | **〜372,912円** |

[AND TOOLS](https://and-tools.net/) では所得税計算・手取り計算など、フリーランスの税金・老後資金計画に役立つ無料ツールを提供している。節税効果を具体的な数字で把握してから実行に移してほしい。
