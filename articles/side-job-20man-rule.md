---
title: "副業の20万円ルールを正確に理解する — 所得税と住民税の違い"
emoji: "📊"
type: "tech"
topics: ["副業", "確定申告", "住民税", "税金"]
published: false
---

「副業の収入が20万円以下なら確定申告しなくていい」という話を聞いたことがあるだろう。これは半分正しく、半分は誤解だ。20万円ルールは**所得税の確定申告義務**の話であって、**住民税の申告義務とは別**になっている。

[AND TOOLS](https://and-tools.net/) の副業税金計算ツールを設計した際に、この混同が最も多かった。ここで正確に整理する。

---

## 20万円ルールとは

所得税法第121条に基づくルールだ。

**適用条件**：
- 給与収入が2,000万円以下
- 給与を1か所からのみもらっている
- 副業（給与以外）の所得が年間**20万円以下**

この条件をすべて満たす場合、**所得税の確定申告が不要**になる。

```javascript
// 20万円ルールの適用判定
function needsTaxReturn(config) {
  const {
    salaryIncome,
    numberOfEmployers,
    otherIncome, // 副業等の所得（収入ではなく所得）
  } = config;

  // 条件1: 給与収入が2000万超なら申告必須
  if (salaryIncome > 20000000) {
    return { required: true, reason: "給与収入2,000万円超" };
  }

  // 条件2: 複数の雇用主から給与をもらっていたら申告必須
  if (numberOfEmployers > 1) {
    return { required: true, reason: "複数の給与支払元あり" };
  }

  // 条件3: 副業所得が20万超なら申告必須
  if (otherIncome > 200000) {
    return { required: true, reason: "副業所得20万円超" };
  }

  return {
    required: false,
    reason: "所得税の確定申告は不要（ただし住民税の申告は別途確認）",
  };
}

// 例1: 副業所得18万円
console.log(needsTaxReturn({ salaryIncome: 5000000, numberOfEmployers: 1, otherIncome: 180000 }));
// → 不要

// 例2: 副業所得22万円
console.log(needsTaxReturn({ salaryIncome: 5000000, numberOfEmployers: 1, otherIncome: 220000 }));
// → 必要（20万円超）
```

---

## 重要：住民税の申告は別ルール

ここが最大の落とし穴だ。**住民税（市区町村民税・都道府県民税）には20万円ルールが存在しない**。

### 住民税の申告義務

所得税の確定申告をしない場合でも、副業所得が**1円でもある**場合は住民税の申告が必要だ（市区町村の窓口に申告書を提出する）。

```
【正確な理解】
副業所得19万円の場合:
├── 所得税の確定申告: 不要（20万円以下なので）
└── 住民税の申告: 必要（1円以上の所得があるので）
```

### なぜ重要か

住民税を申告しないと：

1. **脱税になる**（故意でなくても税務上の問題）
2. **後から追徴課税される**（延滞税も発生）
3. **バレる可能性がある**（クライアント企業が支払調書を税務署に提出するため）

---

## 「所得」と「収入」の違いに注意

20万円ルールの「20万円」は**所得**であって**収入（売上）ではない**。

```javascript
// 所得の計算
function calcSideIncome(revenue, expenseType = "simple") {
  // 給与所得の場合（バイト・副業社員等）
  function calcSalaryIncome(salary) {
    // 給与所得控除の計算
    if (salary <= 1625000) return Math.max(salary - 550000, 0);
    if (salary <= 1800000) return salary * 0.6 - 100000;
    if (salary <= 3600000) return salary * 0.7 - 280000;
    if (salary <= 6600000) return salary * 0.8 - 640000;
    if (salary <= 8500000) return salary * 0.9 - 1100000;
    return salary - 1950000;
  }

  // 事業所得・雑所得の場合（フリーランス・業務委託）
  function calcBusinessIncome(revenue, expenses) {
    return Math.max(revenue - expenses, 0);
  }

  if (expenseType === "salary") {
    const income = calcSalaryIncome(revenue);
    return { revenue, income, taxableAmount: income };
  }

  // 業務委託（経費30%の場合）
  const expenses = revenue * 0.3;
  const income = calcBusinessIncome(revenue, expenses);
  return { revenue, expenses, income };
}

// 副業フリーランス売上30万円（経費10万円）の場合
const sideJob = calcSideIncome(300000, "business");
console.log(`副業収入: ¥${sideJob.revenue.toLocaleString()}`);
console.log(`経費: ¥${sideJob.expenses.toLocaleString()}`);
console.log(`副業所得: ¥${sideJob.income.toLocaleString()}`);
console.log(`20万円ルール適用: ${sideJob.income <= 200000 ? "確定申告不要（住民税は要申告）" : "確定申告必要"}`);
// → 副業所得: 210,000円 → 確定申告必要
```

売上30万円でも経費が多ければ所得は20万円以下になる場合がある。

---

## 副業の種類による「所得区分」の違い

副業の所得は、内容によって区分が変わる。

| 副業の種類 | 所得区分 | 特徴 |
|----------|---------|------|
| 業務委託・フリーランス | **事業所得 or 雑所得** | 経費を差し引ける |
| アルバイト | **給与所得** | 給与所得控除あり |
| 株・FX | **譲渡所得 or 雑所得** | 特定口座なら源泉分離課税 |
| YouTube・ブログ収益 | **雑所得 or 事業所得** | 経費を差し引ける |
| 不動産賃貸 | **不動産所得** | 経費を差し引ける |

事業所得と雑所得の違いは「継続性・独立性・営利目的」の有無で判断される。副業を本格的に行っている場合は事業所得として申告できるが、税務署の判断基準は厳格になっている（2022年度以降）。

---

## 副業がバレるのはなぜか

会社員の副業禁止規則に違反するリスクがある場合、「副業がバレないか」を気にする人がいる。実際にどうバレるかを理解しておく。

### バレる主なルート

1. **住民税の増額**：副業所得が増えると翌年の住民税が増える。会社の給与担当者が「住民税の天引き額が増えた」と気づく
2. **支払調書**：クライアントが税務署に支払調書を提出するため、税務署が把握している
3. **SNS・SNSの特定**：副業実績をSNSに投稿して特定される

### バレにくくする方法（合法）

確定申告の際に「普通徴収」を選択すると、副業分の住民税を自分で納付することになり、会社への通知額に副業分が含まれにくくなる。

```
確定申告書の「住民税に関する事項」欄:
├── 特別徴収（給与から天引き）← デフォルト・バレやすい
└── 普通徴収（自分で納付）← 副業分を分離できる
```

ただし、これは合法的な選択であり「副業を隠す」行為ではない。税務署への申告は確実に必要だ。

---

## 副業の税金を計算する

[AND TOOLSの副業税金計算ツール](https://and-tools.net/tools/side-job-tax/) で正確な税額を計算できる。以下はロジックの概要だ。

```javascript
// 副業による追加税金の概算計算
function calcSideJobTax(mainSalary, sideJobIncome) {
  // 主たる給与の所得を計算
  function calcSalaryDeduction(salary) {
    if (salary <= 1625000) return 550000;
    if (salary <= 1800000) return salary * 0.4;
    if (salary <= 3600000) return salary * 0.3 + 180000;
    if (salary <= 6600000) return salary * 0.2 + 540000;
    if (salary <= 8500000) return salary * 0.1 + 1200000;
    return 1950000;
  }

  const mainIncome = mainSalary - calcSalaryDeduction(mainSalary);
  const totalTaxableIncome = mainIncome + sideJobIncome;

  // 基礎控除
  const basicDeduction = 480000;

  // 社会保険料控除（概算：給与の15%）
  const socialInsDeduction = mainSalary * 0.15;

  const taxableBase = Math.max(totalTaxableIncome - basicDeduction - socialInsDeduction, 0);

  // 所得税率の適用
  function getIncomeTax(income) {
    if (income <= 1950000) return income * 0.05;
    if (income <= 3300000) return income * 0.1 - 97500;
    if (income <= 6950000) return income * 0.2 - 427500;
    if (income <= 9000000) return income * 0.23 - 636000;
    return income * 0.33 - 1536000;
  }

  // 副業なしの税額
  const taxableWithout = Math.max(mainIncome - basicDeduction - socialInsDeduction, 0);
  const taxWithout = getIncomeTax(taxableWithout);

  // 副業ありの税額
  const taxWith = getIncomeTax(taxableBase);

  // 追加税額
  const additionalTax = Math.floor(taxWith - taxWithout);
  const additionalResidentTax = Math.floor(sideJobIncome * 0.1); // 住民税10%

  return {
    sideJobIncome,
    additionalIncomeTax: additionalTax,
    additionalResidentTax,
    totalAdditionalTax: additionalTax + additionalResidentTax,
    netSideIncome: sideJobIncome - additionalTax - additionalResidentTax,
  };
}

// 年収500万円・副業所得50万円の場合
const result = calcSideJobTax(5000000, 500000);
console.log(`副業所得: ¥${result.sideJobIncome.toLocaleString()}`);
console.log(`追加所得税: ¥${result.additionalIncomeTax.toLocaleString()}`);
console.log(`追加住民税: ¥${result.additionalResidentTax.toLocaleString()}`);
console.log(`追加税合計: ¥${result.totalAdditionalTax.toLocaleString()}`);
console.log(`実質手取り: ¥${result.netSideIncome.toLocaleString()}`);
```

---

## まとめ：20万円ルールの正確な理解

| 状況 | 所得税の確定申告 | 住民税の申告 |
|------|---------------|-----------|
| 副業所得0円 | 不要 | 不要 |
| 副業所得1〜19万円 | 不要 | **必要** |
| 副業所得20万円以上 | **必要** | **必要** |

「確定申告しなくていい」=「住民税も払わなくていい」ではない。副業所得が1円でも発生したら住民税の申告は必要だ。

副業の税金計算は [AND TOOLSの副業税金計算ツール](https://and-tools.net/tools/side-job-tax/) で試算できる。また [AND TOOLS](https://and-tools.net/) には所得税計算・手取り計算など、確定申告に役立つ無料ツールを揃えている。
