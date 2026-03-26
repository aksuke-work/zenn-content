---
title: "給与所得控除の計算テーブルをJSで実装する — 年収別6段階の控除額"
emoji: "💴"
type: "tech"
topics: ["javascript", "tax", "給与", "個人開発"]
published: true
---

会社員の「年収」と「所得」は違う。この差を埋めるのが**給与所得控除**だ。フリーランスの経費に相当する概念で、年収に応じて6段階で控除額が変わる。

[AND TOOLS](https://and-tools.net/) の[手取り計算ツール](https://and-tools.net/tools/take-home-pay/)を開発する際に実装したロジックを解説する。

## 給与所得控除の6段階テーブル

2020年（令和2年）の税制改正で現在の形になった。以前より一律10万円引き下げられ、上限も220万円から195万円に下がっている。

```javascript
function calcSalaryDeduction(annualIncome) {
  if (annualIncome <= 0) return 0;

  if (annualIncome <= 1625000) return 550000;
  if (annualIncome <= 1800000) return annualIncome * 0.4 - 100000;
  if (annualIncome <= 3600000) return annualIncome * 0.3 + 80000;
  if (annualIncome <= 6600000) return annualIncome * 0.2 + 440000;
  if (annualIncome <= 8500000) return annualIncome * 0.1 + 1100000;

  return 1950000; // 上限195万円
}
```

テーブルにすると以下の通り。

| 年収 | 控除額の計算式 |
|------|-------------|
| 〜162.5万 | 55万円（最低保証額） |
| 162.5万〜180万 | 年収 × 40% − 10万 |
| 180万〜360万 | 年収 × 30% + 8万 |
| 360万〜660万 | 年収 × 20% + 44万 |
| 660万〜850万 | 年収 × 10% + 110万 |
| 850万超 | 195万円（上限） |

## 最低保証額55万円の意味

年収が162.5万円以下の場合、一律55万円が控除される。つまり、年収55万円以下なら**課税所得がゼロ**になる。これに基礎控除48万円（住民税は43万円）が加わるため、年収103万円以下なら所得税がかからない — いわゆる「103万円の壁」の正体がこれだ。

```javascript
function calc103Wall(annualIncome) {
  const salaryDeduction = calcSalaryDeduction(annualIncome); // 55万
  const basicDeduction = 480000; // 基礎控除48万

  const taxableIncome = annualIncome - salaryDeduction - basicDeduction;
  return Math.max(taxableIncome, 0);
}

// 103万円の場合
console.log(calc103Wall(1030000));
// → 0（所得税ゼロ）

// 104万円の場合
console.log(calc103Wall(1040000));
// → 10000（1万円に課税）
```

2025年度改正で基礎控除が58万円に引き上げられたため、実質的なボーダーは**123万円**に移行している。詳しくは [178万円の壁 シミュレーション](https://and-tools.net/tools/178-wall/)で試算できる。

## 年収850万超の所得金額調整控除

年収850万円を超えると控除額は195万円で頭打ちになる。ただし、以下の条件に該当する場合は**所得金額調整控除**が適用される。

- 本人が特別障害者
- 23歳未満の扶養親族がいる
- 特別障害者の同一生計配偶者・扶養親族がいる

```javascript
function calcIncomeAdjustment(annualIncome, hasQualifyingDependents) {
  if (annualIncome <= 8500000 || !hasQualifyingDependents) return 0;

  // (年収 - 850万) × 10%、上限15万円
  const base = Math.min(annualIncome, 10000000) - 8500000;
  return Math.floor(base * 0.1);
}
```

上限は年収1,000万円で計算するため、最大15万円の追加控除になる。

## 給与所得の計算と端数処理

実際の給与所得を計算するとき、年収が660万円未満の場合は**所得税法別表第五**のテーブルを使って4,000円刻みで計算する。ただし概算であれば上記の計算式で十分だ。

```javascript
function calcSalaryIncome(annualIncome) {
  const deduction = calcSalaryDeduction(annualIncome);
  const income = annualIncome - deduction;
  return Math.max(income, 0);
}

// テスト
const testCases = [
  { income: 3000000, expected: 2020000 },  // 300万 → 202万
  { income: 5000000, expected: 3560000 },  // 500万 → 356万
  { income: 7000000, expected: 5200000 },  // 700万 → 520万
  { income: 10000000, expected: 8050000 }, // 1000万 → 805万
];

testCases.forEach(({ income, expected }) => {
  const result = calcSalaryIncome(income);
  console.log(
    `年収${income / 10000}万 → 給与所得${result / 10000}万`,
    result === expected ? '✓' : `✗ (expected ${expected})`
  );
});
```

## フリーランスとの比較

フリーランスには給与所得控除がない代わりに、実際の経費を計上できる。どちらが有利かは経費率に依存する。

```javascript
function compareDeduction(revenue) {
  const salaryDeduction = calcSalaryDeduction(revenue);
  const deductionRate = (salaryDeduction / revenue * 100).toFixed(1);

  console.log(`年収: ${(revenue / 10000).toFixed(0)}万円`);
  console.log(`給与所得控除: ${(salaryDeduction / 10000).toFixed(0)}万円（${deductionRate}%）`);
  console.log(`→ 経費率${deductionRate}%以上なら、フリーランスが有利`);
}

compareDeduction(4000000);
// 年収: 400万円
// 給与所得控除: 124万円（31.0%）
// → 経費率31.0%以上なら、フリーランスが有利

compareDeduction(8000000);
// 年収: 800万円
// 給与所得控除: 190万円（23.8%）
// → 経費率23.8%以上なら、フリーランスが有利
```

フリーランスの経費率の目安は [経費率 判定ツール](https://and-tools.net/tools/expense-rate/)で確認できる。

## まとめ

給与所得控除の実装ポイント:

1. **6段階の条件分岐**で、年収に応じた控除額を計算する
2. **最低保証55万円** + 基礎控除で「103万円の壁」が決まる
3. **上限195万円**（年収850万超で頭打ち）
4. 年収660万未満は本来4,000円刻みのテーブル参照だが、概算では計算式で十分

給与所得控除は手取り計算の入口。ここから社会保険料控除、所得税、住民税の計算へと続く。[手取り計算シミュレーター](https://and-tools.net/tools/take-home-pay/)では、これらを一括で計算できる。

---

[AND TOOLS](https://and-tools.net/) では、フリーランス・個人事業主向けの税金計算ツールを34個以上無料で公開しています。
