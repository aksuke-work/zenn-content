---
title: "青色申告特別控除65万円の適用条件をコードで判定する"
emoji: "📘"
type: "tech"
topics: ["javascript", "確定申告", "青色申告", "個人開発"]
published: false
---

青色申告特別控除は65万円・55万円・10万円の3段階があるが、どの段階が適用されるかの判定条件は意外と複雑だ。[AND TOOLS](https://and-tools.net/) の[手取り計算ツール](https://and-tools.net/tools/take-home-pay/)でフリーランスの税金計算を実装する際に整理した判定ロジックを解説する。

## 3段階の控除額と適用条件

```javascript
function judgeBlueReturnDeduction(conditions) {
  const {
    hasBlueReturn,       // 青色申告承認を受けているか
    hasDoubleEntry,      // 複式簿記で記帳しているか
    hasBalanceSheet,     // 貸借対照表を添付しているか
    hasProfitLossSheet,  // 損益計算書を添付しているか
    isETax,              // e-Taxで申告するか
    hasElectronicBooks,  // 電子帳簿保存をしているか
    businessType,        // 'business' | 'realEstate' | 'forestry'
  } = conditions;

  // 青色申告でなければ控除なし
  if (!hasBlueReturn) {
    return { amount: 0, reason: '青色申告の承認を受けていない' };
  }

  // 不動産所得の場合、事業的規模（5棟10室基準）が必要
  // ここでは簡略化して businessType で判定

  // 10万円控除の最低条件: 青色申告 + 損益計算書
  if (!hasProfitLossSheet) {
    return { amount: 0, reason: '損益計算書の添付がない' };
  }

  // 複式簿記 + 貸借対照表がない → 10万円
  if (!hasDoubleEntry || !hasBalanceSheet) {
    return {
      amount: 100000,
      reason: '複式簿記または貸借対照表が不足 → 10万円控除',
      missing: [
        ...(!hasDoubleEntry ? ['複式簿記'] : []),
        ...(!hasBalanceSheet ? ['貸借対照表'] : []),
      ],
    };
  }

  // 複式簿記 + 貸借対照表あり
  // e-Tax または 電子帳簿保存 → 65万円
  if (isETax || hasElectronicBooks) {
    return {
      amount: 650000,
      reason: `複式簿記 + 貸借対照表 + ${isETax ? 'e-Tax申告' : '電子帳簿保存'} → 65万円控除`,
    };
  }

  // 複式簿記 + 貸借対照表あるが、e-Taxでも電子帳簿保存でもない → 55万円
  return {
    amount: 550000,
    reason: '複式簿記 + 貸借対照表あるが、e-Taxまたは電子帳簿保存が未対応 → 55万円控除',
    suggestion: 'e-Taxで申告すれば65万円に引き上げ可能（+10万円の節税効果）',
  };
}
```

## 判定フローをテストする

```javascript
// テストケース
const testCases = [
  {
    name: '白色申告',
    conditions: {
      hasBlueReturn: false,
      hasDoubleEntry: false,
      hasBalanceSheet: false,
      hasProfitLossSheet: false,
      isETax: false,
      hasElectronicBooks: false,
    },
    expected: 0,
  },
  {
    name: '青色10万円（簡易簿記）',
    conditions: {
      hasBlueReturn: true,
      hasDoubleEntry: false,
      hasBalanceSheet: false,
      hasProfitLossSheet: true,
      isETax: false,
      hasElectronicBooks: false,
    },
    expected: 100000,
  },
  {
    name: '青色55万円（紙で申告）',
    conditions: {
      hasBlueReturn: true,
      hasDoubleEntry: true,
      hasBalanceSheet: true,
      hasProfitLossSheet: true,
      isETax: false,
      hasElectronicBooks: false,
    },
    expected: 550000,
  },
  {
    name: '青色65万円（e-Tax）',
    conditions: {
      hasBlueReturn: true,
      hasDoubleEntry: true,
      hasBalanceSheet: true,
      hasProfitLossSheet: true,
      isETax: true,
      hasElectronicBooks: false,
    },
    expected: 650000,
  },
  {
    name: '青色65万円（電子帳簿保存）',
    conditions: {
      hasBlueReturn: true,
      hasDoubleEntry: true,
      hasBalanceSheet: true,
      hasProfitLossSheet: true,
      isETax: false,
      hasElectronicBooks: true,
    },
    expected: 650000,
  },
];

testCases.forEach(({ name, conditions, expected }) => {
  const result = judgeBlueReturnDeduction(conditions);
  const pass = result.amount === expected;
  console.log(`${pass ? '✓' : '✗'} ${name}: ${result.amount.toLocaleString()}円`);
  console.log(`  → ${result.reason}`);
  if (result.suggestion) console.log(`  💡 ${result.suggestion}`);
});
```

## 65万円控除の節税効果を計算する

65万円の控除が実際にいくらの節税になるかは、課税所得によって変わる。

```javascript
function calcBlueReturnSavings(income, expenses, socialInsurance) {
  const deductions = [0, 100000, 550000, 650000];
  const results = [];

  deductions.forEach(blueDeduction => {
    const taxableIncome = Math.max(
      income - expenses - socialInsurance - 480000 - blueDeduction,
      0
    );

    // 所得税
    const incomeTax = calcIncomeTax(taxableIncome);

    // 住民税
    const residentTaxable = Math.max(
      income - expenses - socialInsurance - 430000 - blueDeduction,
      0
    );
    const residentTax = Math.floor(residentTaxable * 0.10) + 5000;

    // 国保（所得割ベースの概算）
    const nhiBase = Math.max(income - expenses - 430000 - blueDeduction, 0);
    const nhi = Math.min(Math.floor(nhiBase * 0.12), 890000);

    results.push({
      blueDeduction,
      taxableIncome,
      incomeTax,
      residentTax,
      nhi,
      totalBurden: incomeTax + residentTax + nhi,
    });
  });

  // 控除なしとの差額を計算
  const baseline = results[0].totalBurden;
  results.forEach(r => {
    r.savings = baseline - r.totalBurden;
    console.log(
      `控除${(r.blueDeduction / 10000).toFixed(0)}万 → ` +
      `所得税${(r.incomeTax / 10000).toFixed(1)}万 + ` +
      `住民税${(r.residentTax / 10000).toFixed(1)}万 + ` +
      `国保${(r.nhi / 10000).toFixed(1)}万 = ` +
      `合計${(r.totalBurden / 10000).toFixed(1)}万 ` +
      `(節税${(r.savings / 10000).toFixed(1)}万)`
    );
  });

  return results;
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

  if (taxableIncome <= 0) return 0;

  for (const b of brackets) {
    if (taxableIncome <= b.limit) {
      return Math.floor((taxableIncome * b.rate - b.deduction) * 1.021);
    }
  }
  return 0;
}

// 売上600万、経費120万、社保60万の場合
console.log('=== 売上600万のケース ===');
calcBlueReturnSavings(6000000, 1200000, 600000);
```

65万円控除の実効節税額は、所得税率20%帯なら約**20万円**（所得税13万+住民税6.5万+国保の削減）になる。

## 不動産所得の事業的規模判定

青色申告控除は不動産所得でも適用されるが、65万円控除を受けるには「事業的規模」が必要。

```javascript
function judgeRealEstateScale(properties) {
  // 5棟10室基準
  const buildings = properties.filter(p => p.type === 'building').length;
  const rooms = properties.filter(p => p.type === 'room').length;

  // 棟は2室換算
  const equivalentRooms = buildings * 2 + rooms;

  const isBusinessScale = buildings >= 5 || rooms >= 10 || equivalentRooms >= 10;

  return {
    isBusinessScale,
    buildings,
    rooms,
    equivalentRooms,
    maxDeduction: isBusinessScale ? 650000 : 100000,
    reason: isBusinessScale
      ? '事業的規模（5棟10室基準）を満たす → 65万円控除可能'
      : '事業的規模に該当しない → 10万円控除のみ',
  };
}
```

## 期限後申告のペナルティ

確定申告の期限（3/15）を過ぎると、65万円/55万円控除は**10万円に減額**される。

```javascript
function judgeDeductionWithDeadline(conditions, filingDate) {
  const result = judgeBlueReturnDeduction(conditions);

  const deadline = new Date(filingDate.getFullYear(), 2, 15); // 3月15日

  if (filingDate > deadline) {
    return {
      ...result,
      amount: Math.min(result.amount, 100000),
      originalAmount: result.amount,
      penalty: true,
      reason: `期限後申告のため ${result.amount.toLocaleString()}円 → 10万円に減額`,
      lostAmount: result.amount - 100000,
    };
  }

  return { ...result, penalty: false };
}

// 期限後申告の影響
const conditions = {
  hasBlueReturn: true,
  hasDoubleEntry: true,
  hasBalanceSheet: true,
  hasProfitLossSheet: true,
  isETax: true,
  hasElectronicBooks: false,
};

const onTime = judgeDeductionWithDeadline(conditions, new Date(2026, 2, 10)); // 3/10
const late = judgeDeductionWithDeadline(conditions, new Date(2026, 2, 20));   // 3/20

console.log(`期限内: ${onTime.amount.toLocaleString()}円`); // 650,000円
console.log(`期限後: ${late.amount.toLocaleString()}円 (損失: ${late.lostAmount.toLocaleString()}円)`); // 100,000円
```

期限後申告で55万円の控除を失うのは非常に痛い。[経費率 判定ツール](https://and-tools.net/tools/expense-rate/)で事前に必要な経費データを整理し、余裕を持って申告することが重要だ。

## まとめ

青色申告特別控除の判定ポイント:

1. **65万円**: 複式簿記 + 貸借対照表 + e-Tax（または電子帳簿保存）
2. **55万円**: 複式簿記 + 貸借対照表（紙で申告）
3. **10万円**: 簡易簿記（または期限後申告）
4. **実効節税額**は所得税率帯によって変わる（所得税+住民税+国保の3重効果）
5. 不動産所得は**事業的規模（5棟10室）**でないと10万円控除のみ

55万→65万の差は「e-Taxを使うかどうか」だけ。マイナンバーカードがあればe-Taxは無料で使えるので、やらない理由はない。[フリーランス単価計算ツール](https://and-tools.net/tools/freelance-rate-calc/)で、65万円控除後の実質的な手取りを確認することを勧める。

---

[AND TOOLS](https://and-tools.net/) では、フリーランス・個人事業主向けの税金計算ツールを34個以上無料で公開しています。
