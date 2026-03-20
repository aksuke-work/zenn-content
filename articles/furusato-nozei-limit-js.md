---
title: "ふるさと納税の控除上限額を正確に計算するアルゴリズム"
emoji: "🍚"
type: "tech"
topics: ["javascript", "ふるさと納税", "tax", "個人開発"]
published: false
---

ふるさと納税の「控除上限額」を正確に計算するのは意外と難しい。ざっくり「年収の1%」ではなく、住民税所得割の20%と所得税率から逆算する必要がある。[AND TOOLS](https://and-tools.net/) の[住民税計算ツール](https://and-tools.net/tools/resident-tax/)で実装した控除上限の計算ロジックを解説する。

## ふるさと納税の控除の仕組み

ふるさと納税の控除は3つの要素で構成される。

```javascript
function calcFurusatoDeduction(donationAmount, taxableIncome, residentTaxIncome) {
  const selfBurden = 2000; // 自己負担額
  const donation = donationAmount - selfBurden;

  // ① 所得税からの控除（所得控除）
  const incomeTaxRate = getIncomeTaxRate(taxableIncome);
  const incomeTaxDeduction = Math.floor(donation * incomeTaxRate * 1.021);

  // ② 住民税からの控除（基本分）: 寄附金 × 10%
  const residentBasic = Math.floor(donation * 0.10);

  // ③ 住民税からの控除（特例分）: 住民税所得割の20%が上限
  const residentIncomeTax = Math.floor(residentTaxIncome * 0.10);
  const specialRate = 1 - 0.10 - incomeTaxRate * 1.021;
  const residentSpecial = Math.min(
    Math.floor(donation * specialRate),
    Math.floor(residentIncomeTax * 0.20) // ← ここが上限
  );

  return {
    incomeTaxDeduction,
    residentBasic,
    residentSpecial,
    totalDeduction: incomeTaxDeduction + residentBasic + residentSpecial,
    selfBurden,
    isWithinLimit: Math.floor(donation * specialRate) <= Math.floor(residentIncomeTax * 0.20),
  };
}
```

3つの控除の合計が `寄附額 - 2,000円` に一致すれば、自己負担が2,000円で済む。③の特例分が上限（住民税所得割の20%）を超えると、超過分は自己負担になる。

## 控除上限額の逆算

「自己負担2,000円で済む最大寄附額はいくらか」を逆算する。

```javascript
function calcFurusatoLimit(income, socialInsurance, otherDeductions = 0) {
  // 所得税の課税所得
  const incomeTaxTaxable = Math.max(
    income - socialInsurance - otherDeductions - 480000, // 基礎控除48万
    0
  );

  // 住民税の課税所得
  const residentTaxTaxable = Math.max(
    income - socialInsurance - otherDeductions - 430000, // 基礎控除43万
    0
  );

  // 住民税所得割
  const residentIncomeTax = Math.floor(residentTaxTaxable * 0.10);

  // 所得税の限界税率
  const taxRate = getIncomeTaxRate(incomeTaxTaxable);

  // 上限額の計算式:
  // 上限 = 住民税所得割 × 20% ÷ (100% - 住民税10% - 所得税率 × 1.021) + 2,000
  const denominator = 1 - 0.10 - taxRate * 1.021;

  if (denominator <= 0) return 2000; // 安全策

  const limit = Math.floor(residentIncomeTax * 0.20 / denominator) + 2000;

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

## 税率のブラケット境界問題

ふるさと納税自体が所得控除になるため、寄附額を増やすと課税所得が下がり、税率のブラケットが変わる可能性がある。この循環参照を二分探索で解く。

```javascript
function calcFurusatoLimitPrecise(income, socialInsurance, otherDeductions = 0) {
  let low = 2000;
  let high = income; // 最大でも年収まで

  while (high - low > 100) {
    const mid = Math.floor((low + high) / 2);

    // ふるさと納税を所得控除として考慮した課税所得
    const adjustedTaxable = Math.max(
      income - socialInsurance - otherDeductions - 480000 - (mid - 2000),
      0
    );

    // この課税所得での税率
    const taxRate = getIncomeTaxRate(adjustedTaxable);

    // 住民税の課税所得（ふるさと納税の所得控除分を反映）
    const residentTaxTaxable = Math.max(
      income - socialInsurance - otherDeductions - 430000,
      0
    );
    const residentIncomeTax = Math.floor(residentTaxTaxable * 0.10);

    // この条件での上限額
    const denominator = 1 - 0.10 - taxRate * 1.021;
    if (denominator <= 0) {
      high = mid;
      continue;
    }

    const limit = Math.floor(residentIncomeTax * 0.20 / denominator) + 2000;

    if (mid <= limit) {
      low = mid;
    } else {
      high = mid;
    }
  }

  return low;
}

// テスト: 年収別の控除上限額（独身・社保15%概算）
const testIncomes = [3000000, 4000000, 5000000, 6000000, 7000000, 8000000, 10000000];
testIncomes.forEach(income => {
  const si = Math.floor(income * 0.15);
  const limit = calcFurusatoLimit(income, si);
  console.log(`年収${income / 10000}万 → 上限約${(limit / 10000).toFixed(1)}万円`);
});
```

## 会社員 vs フリーランスの控除上限の違い

フリーランスは給与所得控除がないため、同じ売上でも控除上限額が異なる。

```javascript
function compareEmployeeVsFreelance(revenue) {
  // 会社員: 給与所得控除あり
  const salaryDeduction = calcSalaryDeduction(revenue);
  const employeeIncome = revenue - salaryDeduction;
  const employeeSI = Math.floor(revenue * 0.15);
  const employeeLimit = calcFurusatoLimit(employeeIncome, employeeSI);

  // フリーランス: 経費率30%と仮定、国保+年金
  const expenses = Math.floor(revenue * 0.3);
  const freelanceIncome = revenue - expenses;
  const freelanceSI = Math.floor(freelanceIncome * 0.14) + 16980 * 12; // 国保+年金概算
  const freelanceLimit = calcFurusatoLimit(freelanceIncome, freelanceSI);

  console.log(`売上/年収: ${revenue / 10000}万円`);
  console.log(`  会社員の上限: ${(employeeLimit / 10000).toFixed(1)}万円`);
  console.log(`  フリーランスの上限: ${(freelanceLimit / 10000).toFixed(1)}万円`);
}

function calcSalaryDeduction(income) {
  if (income <= 1625000) return 550000;
  if (income <= 1800000) return income * 0.4 - 100000;
  if (income <= 3600000) return income * 0.3 + 80000;
  if (income <= 6600000) return income * 0.2 + 440000;
  if (income <= 8500000) return income * 0.1 + 1100000;
  return 1950000;
}
```

## ワンストップ特例 vs 確定申告

ワンストップ特例を使うと所得控除（所得税分）がなくなり、住民税から全額控除される。控除の上限額は変わらないが、計算過程が変わる。

```javascript
function calcOneStopVsTaxReturn(donation, taxableIncome, residentTaxIncome) {
  const selfBurden = 2000;
  const amount = donation - selfBurden;
  const taxRate = getIncomeTaxRate(taxableIncome);

  // 確定申告の場合
  const taxReturn = {
    incomeTaxRefund: Math.floor(amount * taxRate * 1.021),
    residentBasic: Math.floor(amount * 0.10),
    residentSpecial: Math.floor(amount * (1 - 0.10 - taxRate * 1.021)),
  };
  taxReturn.total = taxReturn.incomeTaxRefund + taxReturn.residentBasic + taxReturn.residentSpecial;

  // ワンストップ特例の場合（住民税から全額控除）
  const oneStop = {
    incomeTaxRefund: 0,
    residentBasic: Math.floor(amount * 0.10),
    // 特例分に所得税相当分が上乗せされる
    residentSpecial: Math.floor(amount * (1 - 0.10)),
  };
  oneStop.total = oneStop.residentBasic + oneStop.residentSpecial;

  console.log(`確定申告: 所得税還付${taxReturn.incomeTaxRefund}円 + 住民税減額${taxReturn.residentBasic + taxReturn.residentSpecial}円`);
  console.log(`ワンストップ: 住民税減額${oneStop.total}円`);
  console.log(`控除合計はどちらも${amount}円（自己負担2,000円を除く）`);
}
```

フリーランスはワンストップ特例が使えず、必ず確定申告になる。[所得税 計算ツール](https://and-tools.net/tools/income-tax/)で確定申告時の税額を事前に確認するとよい。

## 注意: 住宅ローン控除との併用

住宅ローン控除がある場合、ふるさと納税の控除上限額が実質的に下がる。住宅ローン控除で所得税がゼロになっていると、ふるさと納税の所得税控除分が消失するためだ。

```javascript
function calcLimitWithHousingLoan(income, socialInsurance, housingLoanCredit) {
  const incomeTaxTaxable = Math.max(income - socialInsurance - 480000, 0);
  const incomeTax = calcIncomeTax(incomeTaxTaxable);

  // 住宅ローン控除で所得税がゼロになるケース
  const remainingIncomeTax = Math.max(incomeTax - housingLoanCredit, 0);

  if (remainingIncomeTax === 0) {
    console.log('⚠ 住宅ローン控除で所得税ゼロ → ふるさと納税の所得税控除が使えない');
    console.log('→ ワンストップ特例を使えば住民税から全額控除されるので影響が小さい');
  }

  // 通常の上限計算
  const normalLimit = calcFurusatoLimit(income, socialInsurance);

  return normalLimit; // 上限自体は変わらないが、確定申告だと控除しきれない場合がある
}
```

この点は[手取り計算シミュレーター](https://and-tools.net/tools/take-home-pay/)で住宅ローン控除とふるさと納税の両方を入力して確認できる。

## まとめ

ふるさと納税の控除上限額計算のポイント:

1. **住民税所得割の20%**が特例控除の上限
2. 上限額 = 住民税所得割 × 20% / (1 - 住民税率10% - 所得税率 × 1.021) + 2,000円
3. **税率ブラケットの境界**では二分探索で正確な上限を求める
4. **フリーランスは給与所得控除がない**分、所得が大きくなり上限も変わる
5. 住宅ローン控除との併用時は確定申告 vs ワンストップ特例の選択に注意

「だいたいの上限」ではなく正確な計算をしないと、自己負担が2,000円を超えてしまう。[住民税 計算ツール](https://and-tools.net/tools/resident-tax/)で住民税所得割を正確に出してから上限を算出することを勧める。

---

[AND TOOLS](https://and-tools.net/) では、フリーランス・個人事業主向けの税金計算ツールを34個以上無料で公開しています。
