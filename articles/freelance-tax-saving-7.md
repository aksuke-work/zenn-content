---
title: "フリーランスの節税対策7選 — やるかやらないかで年50万変わる"
emoji: "💡"
type: "tech"
topics: ["フリーランス", "節税", "確定申告", "個人事業主"]
published: false
---

フリーランスエンジニアが節税を意識するかどうかで、年間50万円以上の差が出ることは珍しくない。「税金は仕方ない」と諦めている人が多いが、合法的な節税手段はいくつもある。

[AND TOOLS](https://and-tools.net/) の税金計算ツールで様々なパターンの税額を計算してきた経験から、効果が高い順に7つの節税対策をまとめる。

---

## 前提：フリーランスの税負担の構造

```javascript
// フリーランスの税負担シミュレーション
function calcTotalTaxBurden(annualRevenue) {
  const expenses = annualRevenue * 0.15; // 経費15%
  const businessIncome = annualRevenue - expenses;

  // 主な控除
  const blueReturn = 650000;   // 青色申告65万控除
  const basicDeduction = 480000; // 基礎控除
  const pension = 16980 * 12;  // 国民年金
  const healthIns = annualRevenue < 5000000 ? 500000 : 700000; // 国保（概算）

  const totalDeduction = blueReturn + basicDeduction + pension + healthIns;
  const taxableIncome = Math.max(businessIncome - totalDeduction, 0);

  // 所得税（概算）
  function incomeTax(income) {
    if (income <= 1950000) return income * 0.05;
    if (income <= 3300000) return income * 0.1 - 97500;
    if (income <= 6950000) return income * 0.2 - 427500;
    if (income <= 9000000) return income * 0.23 - 636000;
    return income * 0.33 - 1536000;
  }

  const tax = incomeTax(taxableIncome);
  const residentTax = taxableIncome * 0.1;
  const totalTax = tax + residentTax + pension + healthIns;

  console.log(`年収: ¥${annualRevenue.toLocaleString()} / 税・社保合計: ¥${Math.floor(totalTax).toLocaleString()} / 負担率: ${(totalTax / annualRevenue * 100).toFixed(1)}%`);
}

[3000000, 5000000, 8000000, 12000000].forEach(calcTotalTaxBurden);
```

---

## 節税対策1：青色申告で65万円控除

**節税効果: 年間13〜20万円（税率20〜30%の場合）**

青色申告の65万円控除は最も基本的で効果が高い節税だ。

| 申告方法 | 控除額 | 条件 |
|---------|--------|------|
| 白色申告 | 0円 | - |
| 青色10万円 | 10万円 | 青色申告承認申請書の提出のみ |
| 青色55万円 | 55万円 | 複式簿記 + 書面提出 |
| 青色65万円 | 65万円 | 複式簿記 + **e-Tax提出** |

```javascript
// 65万控除 vs 10万控除の差額
function calcBlueReturnBenefit(taxRate) {
  const difference = 650000 - 100000; // 55万円の差
  const taxSaving = difference * taxRate;
  console.log(`税率${taxRate * 100}%の場合: 年間¥${taxSaving.toLocaleString()}の節税`);
}

calcBlueReturnBenefit(0.20); // → 年間¥110,000の節税
calcBlueReturnBenefit(0.30); // → 年間¥165,000の節税
```

**アクション**: freee・弥生・MFクラウドなどの会計ソフトを使い、e-Taxで提出する。

---

## 節税対策2：小規模企業共済（最大70,000円/月）

**節税効果: 年間12〜28万円（税率20〜40%の場合）**

掛金が全額「小規模企業共済等掛金控除」になる。受け取りは廃業・退職時で退職金として扱われる。

```javascript
// 小規模企業共済の節税効果
function calcShokiboEffect(monthlyContribution, taxRate) {
  const yearlyContribution = monthlyContribution * 12;
  const taxSaving = yearlyContribution * taxRate;
  const effectiveCost = yearlyContribution - taxSaving;

  console.log(`掛金: 月¥${monthlyContribution.toLocaleString()} / 年¥${yearlyContribution.toLocaleString()}`);
  console.log(`節税額（税率${taxRate * 100}%）: 年¥${taxSaving.toLocaleString()}`);
  console.log(`実質負担: 年¥${effectiveCost.toLocaleString()}`);
}

calcShokiboEffect(70000, 0.30);
// → 掛金: 月7万 / 年84万 / 節税: 25.2万 / 実質負担: 58.8万
```

[AND TOOLSの小規模企業共済シミュレーター](https://and-tools.net/tools/mutual-aid/) で節税額を確認できる。

**アクション**: 中小機構（smrj.go.jp）から申込書を取り寄せて金融機関で手続き。

---

## 節税対策3：iDeCo（最大68,000円/月）

**節税効果: 年間8〜27万円（税率20〜40%の場合）**

掛金が全額所得控除になり、かつ運用益が非課税。

```javascript
// iDeCoの節税効果
function calcIdecoEffect(monthlyContribution, taxRate, years, expectedReturn = 0.04) {
  const yearlyContribution = monthlyContribution * 12;
  const taxSaving = yearlyContribution * taxRate;

  // 複利運用の効果
  let total = 0;
  for (let i = 0; i < years; i++) {
    total = (total + yearlyContribution) * (1 + expectedReturn);
  }

  const totalContribution = yearlyContribution * years;
  const totalTaxSaving = taxSaving * years;
  const investmentGain = total - totalContribution;

  console.log(`期間: ${years}年間`);
  console.log(`年間拠出: ¥${yearlyContribution.toLocaleString()}`);
  console.log(`年間節税: ¥${taxSaving.toLocaleString()}`);
  console.log(`${years}年間節税合計: ¥${totalTaxSaving.toLocaleString()}`);
  console.log(`運用益（想定年利4%）: ¥${Math.floor(investmentGain).toLocaleString()}`);
  console.log(`受取概算: ¥${Math.floor(total).toLocaleString()}`);
}

// 月3万円・税率25%・20年間
calcIdecoEffect(30000, 0.25, 20);
```

**注意**: 60歳まで引き出せない。受取時に課税される（退職所得控除で大幅軽減可能）。

**アクション**: SBI証券・楽天証券など手数料が安い機関でiDeCoを開設。

---

## 節税対策4：経費の漏れをなくす

**節税効果: 年間5〜20万円（経費計上漏れの解消）**

計上漏れが多い経費TOP5：

```javascript
const commonlyMissedExpenses = [
  {
    item: "自宅家賃の家事按分",
    example: "月12万の家賃 × 20%按分 = 月2.4万円の経費",
    yearlyAmount: 288000,
  },
  {
    item: "通信費の按分（スマホ）",
    example: "月7,000円 × 70% = 月4,900円の経費",
    yearlyAmount: 58800,
  },
  {
    item: "電気代の按分",
    example: "月8,000円 × 20% = 月1,600円の経費",
    yearlyAmount: 19200,
  },
  {
    item: "書籍・Udemy・サブスク",
    example: "技術書・オンライン学習費用",
    yearlyAmount: 50000,
  },
  {
    item: "開業費（開業前の準備費用）",
    example: "PC・ドメイン・書籍等の開業前費用",
    yearlyAmount: 200000, // 一時的
  },
];

const totalMissed = commonlyMissedExpenses.reduce((sum, e) => sum + e.yearlyAmount, 0);
console.log(`計上漏れ経費の合計: ¥${totalMissed.toLocaleString()}`);
// → ¥616,000 / 税率20%の場合: 約123,200円の節税チャンス

commonlyMissedExpenses.forEach((e) => {
  console.log(`・${e.item}: ${e.example}`);
});
```

[AND TOOLSの経費率判定ツール](https://and-tools.net/tools/expense-rate/) で経費率をチェックし、業種平均と比較して計上漏れを発見できる。経費を計上した後の粗利は [粗利計算ツール](https://webatives.com/tools/gross-profit-calc/) で手早く確認できる。

---

## 節税対策5：付加年金（月400円でコスパ最強）

**節税効果: 年間960円の追加控除 + 将来の年金増額**

月400円の追加で付加年金を積み立てられる。保険料は社会保険料控除として全額控除される上、年金として受け取る金額も増える。

```javascript
// 付加年金の費用対効果
const fukanenkin = {
  monthlyPremium: 400,
  yearlyPremium: 4800,

  // 税率20%での年間節税
  taxSaving: Math.floor(4800 * 0.20), // 960円

  // 将来の月額追加受給（20年払いの場合: 200円×240ヶ月=4.8万円/月）
  addedMonthlyBenefit: 200 * 240, // 48,000円/月の追加

  // 回収期間: 総支払額 ÷ 月額追加受給
  recoveryMonths: Math.ceil((4800 * 20) / 48000), // 約2ヶ月
};

console.log(`付加年金 - 年間追加保険料: ¥${fukanenkin.yearlyPremium.toLocaleString()}`);
console.log(`年間節税: ¥${fukanenkin.taxSaving.toLocaleString()}`);
console.log(`20年後の月額追加受給: ¥${fukanenkin.addedMonthlyBenefit.toLocaleString()}`);
console.log(`回収期間: ${fukanenkin.recoveryMonths}ヶ月`);
```

**アクション**: 市区町村の国民年金担当窓口で申請。iDeCoと別に加入可能（ただしiDeCoの掛金上限に影響する場合あり）。

---

## 節税対策6：ふるさと納税

**節税効果: 実質負担2,000円で返礼品がもらえる**

住民税・所得税から控除されるため、フリーランスにとって最もコスパの高い節税の一つだ。

```javascript
// ふるさと納税の控除上限計算（概算）
function calcFurusato(annualIncome, socialInsurance, deductions = 0) {
  // 住民税の所得割額を計算
  const taxableIncome = annualIncome - socialInsurance - deductions - 480000; // 基礎控除
  const residentTaxBase = Math.max(taxableIncome, 0);
  const residentTaxAmount = residentTaxBase * 0.1; // 住民税10%

  // 上限額 = (住民税所得割額 × 20%) / (90% - 所得税率) + 2,000円
  // 簡易計算（所得税率10%の場合）
  const incomeTaxRate = taxableIncome <= 6950000 ? 0.1 : 0.2;
  const maxContribution = Math.floor(
    (residentTaxAmount * 0.2) / (0.9 - incomeTaxRate) + 2000
  );

  return {
    estimatedMax: maxContribution,
    selfBurden: 2000, // 自己負担は一律2,000円
    benefit: maxContribution - 2000,
  };
}

// 年収700万円・社保120万円の場合
const furusato = calcFurusato(7000000, 1200000);
console.log(`ふるさと納税の概算上限: ¥${furusato.estimatedMax.toLocaleString()}`);
console.log(`自己負担: ¥${furusato.selfBurden.toLocaleString()}`);
console.log(`実質的なメリット: ¥${furusato.benefit.toLocaleString()}分の返礼品`);
```

**注意**: 確定申告をする人はワンストップ特例が使えない。確定申告でふるさと納税を申告する必要がある。

---

## 節税対策7：法人化のタイミング検討

**節税効果: 年収1,000万円超なら年50〜200万円規模**

個人事業主と法人では税率構造が異なる。一定の年収を超えると法人化で大幅な節税になる。

```javascript
// 法人化の損益分岐点（簡易版）
function calcCorpBenefit(annualRevenue) {
  // 個人事業主の税負担（概算）
  const personalExpenseRate = 0.20;
  const personalIncome = annualRevenue * (1 - personalExpenseRate);

  function personalTax(income) {
    const deductions = 650000 + 480000 + 200000; // 青色65万+基礎+社保概算
    const taxable = Math.max(income - deductions, 0);
    if (taxable <= 1950000) return taxable * 0.05;
    if (taxable <= 3300000) return taxable * 0.1 - 97500;
    if (taxable <= 6950000) return taxable * 0.2 - 427500;
    if (taxable <= 9000000) return taxable * 0.23 - 636000;
    return taxable * 0.33 - 1536000;
  }

  const personalTaxAmount = personalTax(personalIncome);
  const personalResidentTax = personalIncome * 0.1;
  const personalTotal = personalTaxAmount + personalResidentTax;

  // 法人の税負担（概算：法人税等23.2%）
  // 役員報酬を設定し、給与所得控除を活用
  const corpTaxRate = 0.232; // 法人税・地方法人税・事業税等
  const directorSalary = Math.min(annualRevenue * 0.6, 8000000); // 役員報酬60%設定
  const corpProfit = annualRevenue - directorSalary - annualRevenue * personalExpenseRate;
  const corpTax = Math.max(corpProfit * corpTaxRate, 0);

  // 役員の所得税（給与所得控除が使える）
  function salaryDeduction(salary) {
    if (salary <= 1625000) return 550000;
    if (salary <= 1800000) return salary * 0.4;
    if (salary <= 3600000) return salary * 0.3 + 180000;
    if (salary <= 6600000) return salary * 0.2 + 540000;
    return salary * 0.1 + 1200000;
  }
  const directorIncome = directorSalary - salaryDeduction(directorSalary);
  const directorTax = directorIncome > 0 ? directorIncome * 0.15 : 0; // 簡略化

  const corpTotal = corpTax + directorTax;
  const savings = personalTotal - corpTotal;

  console.log(`年収: ¥${annualRevenue.toLocaleString()}`);
  console.log(`個人事業税合計: ¥${Math.floor(personalTotal).toLocaleString()}`);
  console.log(`法人化後税合計（概算）: ¥${Math.floor(corpTotal).toLocaleString()}`);
  console.log(`節税額: ¥${Math.floor(savings).toLocaleString()}`);
  console.log(savings > 0 ? "→ 法人化が有利" : "→ 個人事業のまま有利");
  console.log("---");
}

[8000000, 10000000, 15000000].forEach(calcCorpBenefit);
```

法人化の詳細は [AND TOOLSの法人化シミュレーター](https://and-tools.net/tools/solo-corp/) で試算できる。

---

## 節税効果のまとめ

| 節税対策 | 年間節税効果（税率20%） | 実施の難易度 |
|---------|---------------------|-----------|
| 1. 青色申告65万控除 | 11〜17万円 | 低（会計ソフトを使う） |
| 2. 小規模企業共済 | 最大16.8万円 | 低（申込書を提出） |
| 3. iDeCo | 最大16.3万円 | 低（証券口座で開設） |
| 4. 経費計上漏れの解消 | 5〜20万円 | 低（帳簿の見直し） |
| 5. 付加年金 | 960円 + 年金増額 | 低（窓口で申請） |
| 6. ふるさと納税 | 実質2,000円で返礼品 | 低（サイトで申し込み） |
| 7. 法人化 | 年収1,000万超で50万〜 | 高（専門家に相談推奨） |

まず1〜6を全部実施して、年収が1,000万円を超えたら7を検討する、というのが現実的な順序だ。

[AND TOOLS](https://and-tools.net/) では [所得税計算](https://and-tools.net/tools/income-tax/)・[手取り計算](https://and-tools.net/tools/take-home/)・[法人化シミュレーター](https://and-tools.net/tools/solo-corp/) など、節税効果を数字で確認できる無料ツールを提供している。
