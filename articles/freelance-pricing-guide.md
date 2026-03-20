---
title: "フリーランスエンジニアの単価設定ガイド — 時給3,000円は安すぎる"
emoji: "💰"
type: "tech"
topics: ["フリーランス", "単価設定", "エンジニア", "個人事業主"]
published: false
---

「時給3,000円で月契約=約48万円」を見て「悪くない」と思っているなら、計算が足りていない。社会保険・税金・有給相当を差し引くと、会社員換算で年収350万円以下になるケースもある。

[AND TOOLS](https://and-tools.net/) の手取り計算ツールを設計する過程でフリーランスの実質コストを正確に計算した。この記事では「適正単価」の計算方法を具体的に解説する。

---

## 会社員との実質コスト比較

### 会社員年収500万円の会社負担コスト

会社員を1人雇うとき、会社が負担するコストは年収以上だ。

```javascript
// 会社員を雇うコスト（年収500万円の場合）
function calcEmployerCost(annualSalary) {
  // 会社負担の社会保険料
  const healthInsuranceRate = 0.0511;  // 健康保険（東京・2024年）
  const pensionRate = 0.0915;          // 厚生年金
  const employmentInsuranceRate = 0.0095; // 雇用保険
  const accidentInsuranceRate = 0.003;   // 労災保険（業種平均）

  const socialInsurance = annualSalary * (
    healthInsuranceRate + pensionRate + employmentInsuranceRate + accidentInsuranceRate
  );

  // 有給休暇（20日間）
  const dailyWage = annualSalary / 240; // 年間稼働日数
  const paidLeave = dailyWage * 20;

  // 交通費・研修費等（概算）
  const otherCosts = 300000;

  const totalCost = annualSalary + socialInsurance + paidLeave + otherCosts;

  return {
    annualSalary,
    socialInsurance: Math.floor(socialInsurance),
    paidLeave: Math.floor(paidLeave),
    otherCosts,
    totalCost: Math.floor(totalCost),
    monthlyEquivalent: Math.floor(totalCost / 12),
  };
}

const result = calcEmployerCost(5000000);
console.log("=== 会社員年収500万の会社負担コスト ===");
console.log(`年収: ¥${result.annualSalary.toLocaleString()}`);
console.log(`社会保険（会社負担）: ¥${result.socialInsurance.toLocaleString()}`);
console.log(`有給相当: ¥${result.paidLeave.toLocaleString()}`);
console.log(`その他（交通費・研修等）: ¥${result.otherCosts.toLocaleString()}`);
console.log(`合計コスト: ¥${result.totalCost.toLocaleString()}`);
console.log(`月換算: ¥${result.monthlyEquivalent.toLocaleString()}`);
// → 合計コスト: 約700万円 / 月換算: 約58万円
```

**年収500万円の会社員を雇うコストは約700万円/年**。フリーランスの月単価は、この「会社が払う総コスト」を基準に設定するのが適切だ。

---

## フリーランスの実質手取り計算

月単価50万円（時給3,000円×稼働160時間）で実際にいくら手元に残るか。

```javascript
// フリーランスの実質手取り計算
function calcFreelanceNetIncome(monthlyRevenue) {
  const annualRevenue = monthlyRevenue * 12;

  // 経費（仮: 年収の15%）
  const expenses = annualRevenue * 0.15;
  const taxableIncome = annualRevenue - expenses;

  // 社会保険料（国民年金 + 国保）
  const nationalPension = 16980 * 12;       // 国民年金
  const nationalHealthIns = monthlyRevenue > 500000
    ? 130000 * 12  // 年収600万超の国保目安（東京）
    : 80000 * 12;  // 年収400万程度の国保目安

  const totalSocialIns = nationalPension + nationalHealthIns;

  // 青色申告特別控除
  const blueReturnDeduction = 650000;
  const basicDeduction = 480000;

  const totalDeduction = totalSocialIns + blueReturnDeduction + basicDeduction;
  const taxableBase = Math.max(taxableIncome - totalDeduction, 0);

  // 所得税（概算）
  function calcIncomeTax(income) {
    if (income <= 1950000) return income * 0.05;
    if (income <= 3300000) return income * 0.10 - 97500;
    if (income <= 6950000) return income * 0.20 - 427500;
    if (income <= 9000000) return income * 0.23 - 636000;
    if (income <= 18000000) return income * 0.33 - 1536000;
    return income * 0.40 - 2796000;
  }

  const incomeTax = calcIncomeTax(taxableBase);
  const residentTax = taxableBase * 0.10; // 住民税10%（概算）

  const totalTax = incomeTax + residentTax;
  const netIncome = annualRevenue - expenses - totalSocialIns - totalTax;

  return {
    annualRevenue,
    expenses,
    totalSocialIns,
    totalTax: Math.floor(totalTax),
    netIncome: Math.floor(netIncome),
    netMonthly: Math.floor(netIncome / 12),
    netRate: ((netIncome / annualRevenue) * 100).toFixed(1) + "%",
  };
}

// 月単価50万円
const result50 = calcFreelanceNetIncome(500000);
console.log("=== 月単価50万円の実質手取り ===");
console.log(`年間収入: ¥${result50.annualRevenue.toLocaleString()}`);
console.log(`経費: ¥${result50.expenses.toLocaleString()}`);
console.log(`社会保険料: ¥${result50.totalSocialIns.toLocaleString()}`);
console.log(`税金: ¥${result50.totalTax.toLocaleString()}`);
console.log(`実質手取り（年）: ¥${result50.netIncome.toLocaleString()}`);
console.log(`実質手取り（月）: ¥${result50.netMonthly.toLocaleString()}`);
console.log(`手取り率: ${result50.netRate}`);
```

月単価50万円でも、実質の手取りは月28〜32万円程度になることが多い。会社員年収500万（月手取り約30万円）と同等か下回るレベルだ。

---

## 適正単価の逆算計算

「月手取りXX万円を確保したい」から逆算する。

```javascript
// 目標手取りから必要な月単価を逆算
function calcRequiredRate(targetMonthlyNet) {
  const targetAnnualNet = targetMonthlyNet * 12;

  // 手取りは収入の約55〜65%が目安（税率・経費率による）
  // 保守的に55%として逆算
  const NET_RATE = 0.55;
  const requiredAnnualRevenue = targetAnnualNet / NET_RATE;
  const requiredMonthlyRevenue = requiredAnnualRevenue / 12;

  // 稼働時間の計算
  const workingDays = 20; // 月稼働日数
  const workingHours = workingDays * 8; // 月稼働時間

  const hourlyRate = requiredMonthlyRevenue / workingHours;

  return {
    requiredMonthlyRevenue: Math.ceil(requiredMonthlyRevenue / 10000) * 10000,
    requiredHourlyRate: Math.ceil(hourlyRate / 100) * 100,
    targetMonthlyNet,
  };
}

// 目標手取り40万円の場合
const needed40 = calcRequiredRate(400000);
console.log(`目標月手取り¥40万 → 必要月単価: ¥${needed40.requiredMonthlyRevenue.toLocaleString()}`);
console.log(`必要時給: ¥${needed40.requiredHourlyRate.toLocaleString()}`);

// 目標手取り60万円の場合
const needed60 = calcRequiredRate(600000);
console.log(`目標月手取り¥60万 → 必要月単価: ¥${needed60.requiredMonthlyRevenue.toLocaleString()}`);
console.log(`必要時給: ¥${needed60.requiredHourlyRate.toLocaleString()}`);
```

---

## スキル別の市場相場（2024年参考値）

### 常駐型フリーランス（月額単価）

| スキル | 経験年数 | 月単価相場 |
|--------|---------|-----------|
| Webフロントエンド（React） | 3〜5年 | 60〜80万円 |
| バックエンド（Go/Rust） | 3〜5年 | 70〜90万円 |
| インフラ・SRE | 5年以上 | 80〜100万円 |
| フルスタック | 5年以上 | 75〜95万円 |
| PM・テックリード | 8年以上 | 90〜120万円 |
| モバイル（iOS/Android） | 3〜5年 | 65〜85万円 |

### リモート・プロジェクト型

リモート案件はやや単価が下がる傾向があるが、交通費・時間コストが減るため実質的には有利なことが多い。

---

## 単価交渉のポイント

### 1. 「市場価格を知る」から始める

最低でも3〜5社のエージェントに登録し、「自分のスキルで今いくらの案件があるか」を把握する。

### 2. 更新時に値上げ交渉する

契約更新のタイミングで5〜10%の値上げを要求する。6ヶ月または1年ごとの交渉が現実的だ。

```javascript
// 値上げ交渉のシミュレーション
function calcRateIncreaseImpact(currentMonthlyRate, increaseRate, months = 12) {
  const newMonthlyRate = Math.floor(currentMonthlyRate * (1 + increaseRate));
  const monthlyIncrease = newMonthlyRate - currentMonthlyRate;
  const totalIncrease = monthlyIncrease * months;

  return {
    currentMonthlyRate,
    newMonthlyRate,
    monthlyIncrease,
    totalIncrease,
    increaseRateLabel: `${(increaseRate * 100).toFixed(0)}%アップ`,
  };
}

// 月単価70万円から10%アップ
const negotiation = calcRateIncreaseImpact(700000, 0.1);
console.log(`交渉前: ¥${negotiation.currentMonthlyRate.toLocaleString()}`);
console.log(`交渉後: ¥${negotiation.newMonthlyRate.toLocaleString()}`);
console.log(`月額増分: ¥${negotiation.monthlyIncrease.toLocaleString()}`);
console.log(`年間追加収入: ¥${negotiation.totalIncrease.toLocaleString()}`);
// → 月額増分: 70,000円 / 年間追加: 840,000円
```

### 3. スキルアップで単価を上げる

市場価値の高いスキルを習得することで、単価の上昇幅が変わる。

| スキル追加 | 単価への影響 |
|----------|-----------|
| TypeScript → Rust習得 | +10〜20万円 |
| フロント → フルスタック | +10〜15万円 |
| 開発のみ → PL/PM | +20〜40万円 |
| 国内 → グローバル対応 | +15〜30万円 |

---

## 「時給3,000円」の本当の問題

時給3,000円・月160時間 = 月48万円は、フリーランスとして考えると：

```javascript
const hourlyRate = 3000;
const monthlyHours = 160;
const monthlyRevenue = hourlyRate * monthlyHours; // 480,000円

// 手取りは約55%
const netMonthly = Math.floor(monthlyRevenue * 0.55);
console.log(`月手取り概算: ¥${netMonthly.toLocaleString()}`);
// → ¥264,000

// 会社員換算（手取り26万円は年収約430万円相当）
console.log("会社員換算: 年収430万円相当");
```

時給3,000円は会社員年収430万円相当に過ぎない。フリーランスには有給・退職金・福利厚生がないため、会社員と同等の生活水準を維持するには**最低でも時給4,500〜5,000円以上**が必要だ。

正確な手取り計算は [AND TOOLSの手取り計算ツール](https://and-tools.net/tools/take-home/) で試算できる。月単価を変えながら実質手取りを確認し、交渉の根拠にしてほしい。

---

## まとめ

フリーランスエンジニアの適正単価は「目標手取り ÷ 0.55」から逆算する。

- **月手取り40万円** → 月単価73万円前後が必要
- **月手取り60万円** → 月単価109万円前後が必要
- 時給3,000円（月48万円）は会社員年収430万円相当で「安すぎる」

[AND TOOLS](https://and-tools.net/) では手取り計算・所得税計算・法人化シミュレーターなど、フリーランスが単価設定・キャリア判断に使えるツールを提供している。数字を根拠に交渉・判断する習慣をつけてほしい。
