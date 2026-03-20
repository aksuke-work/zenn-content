---
title: "マイクロ法人で社会保険料を最適化する方法"
emoji: "🏢"
type: "tech"
topics: ["マイクロ法人", "社会保険", "節税", "法人化"]
published: false
---

「マイクロ法人」という言葉を聞いたことがあるだろうか。個人事業主のまま法人を設立し、社会保険料を最適化する戦略だ。うまく活用すれば社会保険料を年間数十万円削減できる。

[AND TOOLS](https://and-tools.net/) の法人化シミュレーターを設計する過程で調査した内容を解説する。

---

## マイクロ法人とは

**マイクロ法人** = 自分が唯一の代表取締役社員（従業員なし）である小さな法人。法律上の正式な名称ではなく、こうした形態の法人を指す俗称だ。

### 活用される理由

個人事業主の社会保険は「国民健康保険 + 国民年金」だが、法人の役員になると「健康保険 + 厚生年金」に加入できる。

| 保険種別 | 個人事業主 | 法人役員（マイクロ法人） |
|---------|---------|---------------------|
| 健康保険 | 国民健康保険（所得比例） | 健康保険（標準報酬月額比例） |
| 年金 | 国民年金（月16,980円） | 厚生年金（報酬比例） |
| 扶養 | なし（家族も各自加入） | あり（扶養家族は追加負担なし） |

---

## 社会保険料の仕組みと最適化

### 法人の社会保険料の計算

```javascript
// 社会保険料の計算（法人役員として最低限の報酬を設定した場合）
function calcCorporateSocialInsurance(monthlyStandardWage) {
  // 健康保険料率（協会けんぽ・東京・40歳未満・2024年度）
  const healthInsuranceRate = 0.0999; // 9.99%（労使折半）
  // 厚生年金保険料率
  const pensionRate = 0.1830; // 18.30%（労使折半）

  // 本人負担（半額）
  const healthPersonal = Math.floor(monthlyStandardWage * healthInsuranceRate / 2);
  const pensionPersonal = Math.floor(monthlyStandardWage * pensionRate / 2);

  // 会社負担（法人が払うが、実質自分の費用）
  const healthCorporate = Math.floor(monthlyStandardWage * healthInsuranceRate / 2);
  const pensionCorporate = Math.floor(monthlyStandardWage * pensionRate / 2);

  const totalPersonal = healthPersonal + pensionPersonal;
  const totalCorporate = healthCorporate + pensionCorporate;
  const totalAll = totalPersonal + totalCorporate;

  return {
    monthlyStandardWage,
    personalMonthly: totalPersonal,
    corporateMonthly: totalCorporate,
    totalMonthly: totalAll,
    personalAnnual: totalPersonal * 12,
    totalAnnual: totalAll * 12,
  };
}

// 最低水準の標準報酬月額（58,000円）で設定した場合
const result58k = calcCorporateSocialInsurance(58000);
console.log("=== 標準報酬月額5.8万円（最低クラス）===");
console.log(`個人負担（月）: ¥${result58k.personalMonthly.toLocaleString()}`);
console.log(`法人負担（月）: ¥${result58k.corporateMonthly.toLocaleString()}`);
console.log(`合計（月）: ¥${result58k.totalMonthly.toLocaleString()}`);
console.log(`個人負担（年）: ¥${result58k.personalAnnual.toLocaleString()}`);
```

---

## 国民健康保険との比較

```javascript
// 国保（個人事業主）vs 健康保険（マイクロ法人）の比較
function compareInsurance(annualIncome, age = 35) {
  // 国保（東京・概算）
  const incomeDeduction = 430000;
  const taxableBase = Math.max(annualIncome - incomeDeduction, 0);
  const nationalHealthAnnual = Math.min(
    taxableBase * 0.0739 + 48300 + taxableBase * 0.0241 + 15000,
    920000 + 240000
  );
  const nationalPensionAnnual = 16980 * 12; // 国民年金
  const nationalTotal = nationalHealthAnnual + nationalPensionAnnual;

  // マイクロ法人（標準報酬月額8.8万円で設定）
  const corporateInsurance = calcCorporateSocialInsurance(88000);
  // 法人負担分も実質的なコストとして含める
  const corporateTotal = corporateInsurance.totalAnnual;

  const savings = nationalTotal - corporateInsurance.personalAnnual;

  console.log(`年収: ¥${annualIncome.toLocaleString()}`);
  console.log(`【個人事業主】国保+国民年金（年）: ¥${Math.floor(nationalTotal).toLocaleString()}`);
  console.log(`【マイクロ法人】健保+厚生年金・個人負担（年）: ¥${corporateInsurance.personalAnnual.toLocaleString()}`);
  console.log(`【マイクロ法人】法人負担分（年）: ¥${corporateInsurance.corporateMonthly * 12}（法人の経費）`);
  console.log(`個人の手出し比較での差額: ¥${Math.floor(savings).toLocaleString()}`);
  console.log("---");
}

compareInsurance(8000000, 35);
compareInsurance(12000000, 35);
```

---

## マイクロ法人の具体的な設計

### 役員報酬の最適設定

マイクロ法人では、役員報酬を**社会保険料が最も低い水準**に設定するのが基本戦略だ。

```javascript
// 最適な役員報酬の設定
const optimalDesign = {
  // 標準報酬月額の最低水準（等級1: 5.8万円）に対応する月額報酬
  monthlyDirectorSalary: 58000,  // 役員報酬（最低水準）
  annualDirectorSalary: 58000 * 12, // = 696,000円/年

  // 個人事業主部分の収入はそのまま事業所得
  freelanceIncome: "個人事業主として継続（事業所得）",

  benefits: [
    "健康保険の扶養制度が使える（家族を扶養に入れると保険料が増えない）",
    "厚生年金に加入できる（国民年金より将来の受給額が増える）",
    "傷病手当金が受け取れる（病気・怪我で働けない期間の収入保障）",
    "出産手当金が受け取れる（出産の場合）",
  ],
};

console.log("=== マイクロ法人の設計例 ===");
console.log(`法人からの役員報酬: 月¥${optimalDesign.monthlyDirectorSalary.toLocaleString()}`);
console.log(`年間役員報酬: ¥${optimalDesign.annualDirectorSalary.toLocaleString()}`);
optimalDesign.benefits.forEach((b) => console.log(`・${b}`));
```

---

## 標準報酬月額の等級と社会保険料

標準報酬月額は報酬の金額によって「等級」が決まる。マイクロ法人では等級を低くすることで保険料を抑える。

| 等級 | 標準報酬月額 | 健保（個人負担） | 厚生年金（個人負担） | 合計（月） |
|------|-----------|--------------|-------------------|---------|
| 1 | 58,000円 | 約2,898円 | 約5,307円 | 約8,205円 |
| 2 | 68,000円 | 約3,397円 | 約6,222円 | 約9,619円 |
| 5 | 98,000円 | 約4,896円 | 約8,967円 | 約13,863円 |
| 10 | 150,000円 | 約7,493円 | 約13,725円 | 約21,218円 |

月8,205円（年約98,460円）の個人負担は、国保+国民年金（年間40〜60万円）と比べて大幅に低い。

---

## 設立コストと維持コスト

マイクロ法人を設立・維持するコストも把握する必要がある。

```javascript
const microCorpCosts = {
  setup: {
    registrationFee: 60000,     // 法人登記費用（司法書士費用含む）
    governmentFee: 150000,       // 登録免許税（株式会社の場合）
    total: 210000,
  },
  annual: {
    socialInsurancePersonal: 98000,  // 個人負担（概算）
    socialInsuranceCorporate: 98000, // 法人負担（概算、法人の経費）
    accountingSoftware: 30000,       // 会計ソフト
    taxAccountant: 150000,           // 税理士（法人決算）
    residualTax: 70000,              // 法人住民税均等割（最低）
    total: 446000,
  },
};

console.log("=== マイクロ法人のコスト ===");
console.log(`設立費用: ¥${microCorpCosts.setup.total.toLocaleString()}`);
console.log(`年間維持費（概算）: ¥${microCorpCosts.annual.total.toLocaleString()}`);
```

**重要**: 法人は売上がゼロでも**法人住民税の均等割（約7万円/年）**がかかる。また法人の確定申告は個人より複雑で、税理士費用が増えることも考慮する。

---

## マイクロ法人が有効なパターン

```javascript
const effectivePatterns = [
  {
    pattern: "フリーランス年収1,000〜2,000万円",
    reason: "国保保険料が年間80〜100万円超になるため、マイクロ法人の健保が圧倒的に有利",
    estimatedSaving: "年50〜100万円",
  },
  {
    pattern: "家族（配偶者・子供）を扶養に入れたい",
    reason: "健康保険には扶養制度があり、家族を無料で扶養できる（国保は家族分も個別に保険料がかかる）",
    estimatedSaving: "年20〜40万円（扶養人数による）",
  },
  {
    pattern: "傷病手当金・出産手当金が必要",
    reason: "国民健康保険にはこれらの給付がない",
    estimatedSaving: "傷病時の収入保障効果",
  },
];

effectivePatterns.forEach((p) => {
  console.log(`【${p.pattern}】`);
  console.log(`理由: ${p.reason}`);
  console.log(`節約効果: ${p.estimatedSaving}`);
  console.log("---");
});
```

---

## 注意点とリスク

### 税務的な注意点

- **役員報酬は期中変更不可**：期首（決算から3ヶ月以内）に決めたら1年間変更できない
- **法人住民税の均等割**：赤字でもかかる（約7万円/年）
- **法人税の申告**：個人より複雑。税理士への依頼が現実的

### 最低賃金との関係

役員は労働基準法の適用外のため、役員報酬に最低賃金の制限はない。ただし社会保険料の観点から不自然に低すぎる報酬は避ける。

### 「節税目的のみ」の法人設立

実態のない法人設立は税務調査で問題になる可能性がある。事業活動の実態を持たせることが重要だ。

---

## まとめ

マイクロ法人は、年収1,000万円を超えてきたフリーランスが「社会保険料の重さ」を感じ始めたタイミングで検討するのが適切だ。

設立・維持コストを考慮しても、年収1,000万円以上なら十分な節税効果が期待できる。

[AND TOOLSの法人化シミュレーター](https://and-tools.net/tools/solo-corp/) で、個人事業主のままの場合とマイクロ法人設立後の税・社保負担を比較できる。また [手取り計算ツール](https://and-tools.net/tools/take-home/) で現状の手取りを把握した上で判断してほしい。

実際の設立は税理士・社会保険労務士との相談を強く推奨する。
