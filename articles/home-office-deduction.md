---
title: "自宅兼事務所の家賃按分 — 面積比と時間比、どちらが有利か"
emoji: "🏠"
type: "tech"
topics: ["フリーランス", "経費", "家賃按分", "節税"]
published: false
---

在宅ワーク中心のフリーランスにとって「自宅家賃を経費にする」は最も身近な節税の一つだ。ただし按分の計算方法で経費額が大きく変わることを知らないまま、損をしているケースが多い。

[AND TOOLS](https://and-tools.net/) で税金計算ツールを開発してきた経験から、面積比と時間比のどちらが有利かを数値で検証する。

---

## 家賃按分の基本

自宅を事業用に使っている場合、「事業に使った割合」を家賃・光熱費から経費として計上できる。これを**家事按分（かじあんぶん）**という。

按分の方法は主に2種類ある。

| 按分方法 | 計算式 |
|---------|--------|
| **面積按分** | 事業専用スペース面積 ÷ 自宅総面積 |
| **時間按分** | 1日の業務時間 ÷ 24時間 |

どちらを使うかは納税者が選択できる。ただし税務署から「合理性」を求められることがあるため、実態に沿った計算方法を選ぶ必要がある。

---

## 面積按分の計算

### 計算例

```javascript
// 面積按分の計算
function calcAreaRatio(officeArea, totalArea) {
  const ratio = officeArea / totalArea;
  return {
    ratio: ratio,
    percentage: (ratio * 100).toFixed(1) + "%",
  };
}

// 設定例：2LDK 60㎡、書斎スペース8㎡
const result = calcAreaRatio(8, 60);
console.log(result);
// → { ratio: 0.1333, percentage: "13.3%" }

// 月家賃12万円の場合
const monthlyRent = 120000;
const expensePerMonth = Math.floor(monthlyRent * result.ratio);
console.log(`月の家賃経費：${expensePerMonth.toLocaleString()}円`);
// → 月の家賃経費：15,999円

const expensePerYear = expensePerMonth * 12;
console.log(`年間の家賃経費：${expensePerYear.toLocaleString()}円`);
// → 年間の家賃経費：191,988円
```

### 面積按分が有利なケース

- **書斎・作業部屋が広い**場合
- **一人暮らしで全体の面積が狭い**場合（相対的に比率が上がる）
- **事業専用スペースが明確**な場合（混用スペースは按分対象にしにくい）

### 注意点

- 事業専用スペースでなければ面積按分は難しい
- リビングで作業している場合、「事業専用」とは言えないため按分率が低くなる
- LDKをそのまま作業スペースとするのは税務署に否認されるリスクがある

---

## 時間按分の計算

### 計算例

```javascript
// 時間按分の計算
function calcTimeRatio(workHoursPerDay, daysPerWeek = 5) {
  const weeklyWorkHours = workHoursPerDay * daysPerWeek;
  const weeklyTotalHours = 24 * 7;
  const ratio = weeklyWorkHours / weeklyTotalHours;

  return {
    ratio: ratio,
    percentage: (ratio * 100).toFixed(1) + "%",
    weeklyWorkHours,
    weeklyTotalHours,
  };
}

// 1日8時間・週5日勤務の場合
const result = calcTimeRatio(8, 5);
console.log(result);
// → { ratio: 0.238, percentage: "23.8%", weeklyWorkHours: 40, weeklyTotalHours: 168 }

// 月家賃12万円の場合
const monthlyRent = 120000;
const expensePerMonth = Math.floor(monthlyRent * result.ratio);
console.log(`月の家賃経費：${expensePerMonth.toLocaleString()}円`);
// → 月の家賃経費：28,571円

const expensePerYear = expensePerMonth * 12;
console.log(`年間の家賃経費：${expensePerYear.toLocaleString()}円`);
// → 年間の家賃経費：342,852円
```

### 時間按分が有利なケース

- **作業スペースが専用部屋でない**（リビング等での作業中心）
- **業務時間が長い**（フリーランスの実態として8〜10時間/日は多い）
- **狭い1Kや1Rに住んでいる**（面積では按分率が高くならない）

---

## 面積比 vs 時間比 — どちらが有利か

実際に比較してみる。

```javascript
// 2つの按分方法を比較
function compareMethods(config) {
  const {
    monthlyRent,
    officeArea,
    totalArea,
    workHoursPerDay,
    workDaysPerWeek,
  } = config;

  // 面積按分
  const areaRatio = officeArea / totalArea;
  const areaExpense = monthlyRent * areaRatio * 12;

  // 時間按分
  const weeklyWorkHours = workHoursPerDay * workDaysPerWeek;
  const timeRatio = weeklyWorkHours / (24 * 7);
  const timeExpense = monthlyRent * timeRatio * 12;

  console.log("=== 按分比較 ===");
  console.log(`家賃: 月${monthlyRent.toLocaleString()}円`);
  console.log(`面積按分: ${(areaRatio * 100).toFixed(1)}% → 年${areaExpense.toLocaleString()}円`);
  console.log(`時間按分: ${(timeRatio * 100).toFixed(1)}% → 年${timeExpense.toLocaleString()}円`);
  console.log(`差額: ${Math.abs(timeExpense - areaExpense).toLocaleString()}円`);
  console.log(`有利な方法: ${timeExpense > areaExpense ? "時間按分" : "面積按分"}`);
}

// ケース1: 1K（25㎡）、書斎なし、リビング作業
compareMethods({
  monthlyRent: 80000,
  officeArea: 5,   // ちょっとしたデスクスペース
  totalArea: 25,
  workHoursPerDay: 8,
  workDaysPerWeek: 5,
});
// → 面積: 20.0%（192,000円/年）時間: 23.8%（228,571円/年）→ 時間按分が有利

// ケース2: 3LDK（80㎡）、専用書斎20㎡
compareMethods({
  monthlyRent: 150000,
  officeArea: 20,
  totalArea: 80,
  workHoursPerDay: 8,
  workDaysPerWeek: 5,
});
// → 面積: 25.0%（450,000円/年）時間: 23.8%（428,571円/年）→ 面積按分が有利
```

### 判断のポイント

| 状況 | 有利な方法 |
|------|-----------|
| 専用書斎あり・広い | **面積按分** |
| 1K・1R・ワンルーム | **時間按分** |
| リビングで作業 | **時間按分**（面積を事業専用とは言えない） |
| 業務時間が週40時間以上 | **時間按分**が高くなりやすい |
| 事業専用スペースが全体の25%以上 | **面積按分**が有利 |

---

## 実務上の注意点

### 1. 「事業専用」の証明

面積按分を使う場合、そのスペースが「主に事業に使っている」ことが前提だ。書斎として使っているが、プライベートの読書もしているような場合は按分率を下げる必要がある。

### 2. 混用スペースの扱い

リビングで仕事もプライベートも行っている場合：
- 面積按分：「事業専用」とは言い難いため、按分率を大きく下げるか計上しない
- 時間按分：業務時間の割合で経費計上できる

### 3. 持ち家（住宅ローン）の場合

持ち家の場合、家賃の代わりに**減価償却費**を按分の対象にする。住宅ローンの元本は経費にならないが、利息部分は経費になる。

```javascript
// 持ち家の場合の按分対象
const homeExpenses = {
  depreciationAnnual: 500000,    // 建物の減価償却費（年額）
  mortgageInterestAnnual: 80000, // ローン利息（年額）
  propertyTax: 150000,           // 固定資産税（年額）
  fireInsurance: 30000,          // 火災保険料（年額）
};

const officeRatio = 0.2; // 面積按分20%

const totalDeductible = Object.values(homeExpenses).reduce((a, b) => a + b, 0);
const officeExpense = totalDeductible * officeRatio;

console.log(`按分前合計: ${totalDeductible.toLocaleString()}円`);
console.log(`事業経費分: ${officeExpense.toLocaleString()}円`);
// → 按分前合計: 760,000円 / 事業経費分: 152,000円
```

### 4. 光熱費も按分できる

家賃だけでなく、電気代・ガス代・水道代も按分できる。電気代は家賃より業務との関連が説明しやすい（PCの電力消費として）。

---

## 電気代の按分

電気代は「業務用機器の使用時間」という明確な按分根拠を作りやすい。

```javascript
// 電気代の按分（業務時間ベース）
function calcElectricityExpense(monthlyBill, workHoursPerDay, workDays) {
  const monthlyWorkHours = workHoursPerDay * workDays;
  const monthlyTotalHours = 24 * 30;
  const ratio = monthlyWorkHours / monthlyTotalHours;

  return {
    ratio,
    percentage: (ratio * 100).toFixed(1) + "%",
    expensePerMonth: Math.floor(monthlyBill * ratio),
    expensePerYear: Math.floor(monthlyBill * ratio * 12),
  };
}

// 月の電気代8,000円、1日8時間・月20日勤務
const elec = calcElectricityExpense(8000, 8, 20);
console.log(elec);
// → ratio: 0.222, percentage: "22.2%", expensePerMonth: 1,777円, expensePerYear: 21,333円
```

---

## まとめ

家賃按分の最適な方法はケースによって異なる。

- **専用書斎あり・面積が広い** → 面積按分
- **ワンルーム・リビング作業** → 時間按分
- **どちらか計算して高い方を選ぶ**のが原則

[AND TOOLSの手取り計算ツール](https://and-tools.net/tools/take-home/) で家賃按分後の経費額を反映させた手取りを試算すると、実際にどれだけ節税効果があるかを確認できる。

按分方法は自由に変更できるが、**年度をまたいで一貫した方法を使う**ことが重要だ。毎年方法を変えると税務調査時に説明が難しくなる。
