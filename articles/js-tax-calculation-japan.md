---
title: "JavaScriptで日本の税金を正確に計算する — 累進課税・社会保険料・住民税の実装"
emoji: "🧮"
type: "tech"
topics: ["javascript", "tax", "webdev", "個人開発"]
published: false
---

フリーランス向けの税金計算ツールサイト [AND TOOLS](https://and-tools.net/) を作っている。34個の計算ツールの裏側にある、日本の税制をJavaScriptで実装するときのポイントをまとめる。

## 1. 累進課税（所得税）の計算

日本の所得税は7段階の累進課税。よくある間違いは「年収500万だから税率20%で100万」と計算してしまうこと。実際は**各ブラケットごとに税率が違う**。

```javascript
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

  for (const b of brackets) {
    if (taxableIncome <= b.limit) {
      const tax = taxableIncome * b.rate - b.deduction;
      // 復興特別所得税 2.1% を加算（2037年末まで）
      return Math.floor(tax * 1.021);
    }
  }
}
```

ポイント:
- `deduction`（控除額）を使う速算表方式が正式な計算方法
- **復興特別所得税 2.1%** を忘れがち。`tax * 1.021` で加算する
- `Math.floor` で1円未満切捨て（国税の原則）

実装: [所得税 計算ツール](https://and-tools.net/tools/income-tax/)

## 2. 給与所得控除の計算

会社員の「必要経費」に相当する控除。年収に応じて段階的に計算する。

```javascript
function calcSalaryDeduction(income) {
  if (income <= 1625000) return 550000;
  if (income <= 1800000) return income * 0.4 - 100000;
  if (income <= 3600000) return income * 0.3 + 80000;
  if (income <= 6600000) return income * 0.2 + 440000;
  if (income <= 8500000) return income * 0.1 + 1100000;
  return 1950000; // 上限
}
```

2025年度の税制改正で基礎控除が48万→58万に引き上げられている点に注意。こういう改正に追従するのが地味に大変。

## 3. 社会保険料の計算（標準報酬月額）

社会保険料の計算が一番厄介。月額報酬を「標準報酬月額」のテーブルにマッピングする必要がある。

```javascript
function getStandardMonthly(monthly) {
  // 標準報酬月額テーブル（抜粋）
  const table = [
    { lower: 0,      upper: 63000,   standard: 58000 },
    { lower: 63000,  upper: 73000,   standard: 68000 },
    { lower: 73000,  upper: 83000,   standard: 78000 },
    // ... 50等級まで続く
    { lower: 1355000, upper: Infinity, standard: 1390000 },
  ];

  for (const row of table) {
    if (monthly >= row.lower && monthly < row.upper) {
      return row.standard;
    }
  }
}

function calcSocialInsurance(standardMonthly, age) {
  const healthRate = 0.10000; // 協会けんぽ全国平均（2026年度）
  const pensionRate = 0.18300; // 厚生年金保険料率（固定）
  const nursingRate = age >= 40 ? 0.01600 : 0; // 介護保険（40歳以上）

  const healthFee = Math.floor(standardMonthly * (healthRate + nursingRate) / 2);
  const pensionFee = Math.floor(standardMonthly * pensionRate / 2);

  // 厚生年金の上限: 標準報酬月額 65万円
  const pensionCap = Math.floor(650000 * pensionRate / 2);

  return {
    health: healthFee,
    pension: Math.min(pensionFee, pensionCap),
    total: healthFee + Math.min(pensionFee, pensionCap),
  };
}
```

注意点:
- 健康保険料率は**都道府県ごとに違う**（協会けんぽの場合）
- 厚生年金は標準報酬月額 **65万円が上限**
- 会社負担と個人負担で折半（`/ 2`）
- 40歳以上は介護保険料が加算される

実装: [社会保険料 計算ツール](https://and-tools.net/tools/social-insurance/)

## 4. 住民税の計算

住民税は所得税より単純。税率は一律10%。

```javascript
function calcResidentTax(taxableIncome) {
  const incomeRate = Math.floor(taxableIncome * 0.10); // 所得割 10%
  const adjustDeduction = 2500; // 調整控除（基本パターン）
  const equalRate = 5000; // 均等割（森林環境税含む）

  return Math.max(incomeRate - adjustDeduction, 0) + equalRate;
}
```

ポイント:
- 住民税の基礎控除は **43万円**（所得税は48万円/58万円）。この差が調整控除で補正される
- 均等割は2024年度から森林環境税1,000円が加わり **5,000円**

実装: [住民税 計算ツール](https://and-tools.net/tools/resident-tax/)

## 5. 源泉徴収税の逆算

フリーランスが「手取り○○円欲しい」ときに必要な報酬額を逆算する。これが意外と難しい。

```javascript
function reverseWithholding(netAmount) {
  if (netAmount <= 0) return 0;

  // 手取り ≤ 100万×(1-10.21%) = 897,900円 の場合
  if (netAmount <= 897900) {
    return Math.ceil(netAmount / 0.8979);
  }

  // 100万超: Math.ceil((netAmount - 897900) / 0.7958 + 1000000)
  return Math.ceil((netAmount - 897900) / 0.7958 + 1000000);
}
```

ここでハマったのは、100万円のボーダー付近で計算がズレるケース。二分探索で解くのが一番安全。

```javascript
function reverseWithholdingSafe(targetNet) {
  let low = targetNet;
  let high = targetNet * 1.3;

  while (high - low > 1) {
    const mid = Math.floor((low + high) / 2);
    const tax = calcWithholding(mid);
    const net = mid - tax;

    if (net < targetNet) low = mid;
    else high = mid;
  }

  return high;
}
```

実装: [源泉徴収税 計算ツール](https://and-tools.net/tools/tax-withholding/)

## 6. 手取り額の総合計算

最終的に「年収から手取り」を出すには、上記を全部組み合わせる。

```
手取り = 年収 - 所得税 - 住民税 - 社会保険料（健保 + 厚年 + 雇用）
```

計算順序が重要:
1. 給与所得控除 → 給与所得
2. 社会保険料を概算（年収 × 約15%）
3. 課税所得 = 給与所得 - 社会保険料控除 - 基礎控除
4. 所得税 = 累進課税(課税所得) × 1.021
5. 住民税 = 課税所得(住民税用) × 10% + 均等割
6. 手取り = 年収 - 所得税 - 住民税 - 社会保険料

実装: [手取り計算シミュレーター](https://and-tools.net/tools/take-home-pay/)

## まとめ

日本の税制をコードで表現すると、エッジケースの多さに驚く。特に:

- **税制改正への追従** — 毎年何かが変わる（2025年は基礎控除引き上げ）
- **端数処理のルール** — 国税は切捨て、地方税も基本切捨て、でも場所によって違う
- **標準報酬月額のテーブル** — 50等級のマッピングを間違えると全部ズレる
- **複数の控除の組み合わせ** — 配偶者控除、扶養控除、iDeCo控除...条件分岐が膨大

全34ツールは [AND TOOLS](https://and-tools.net/) で実際に動いているので、興味があれば触ってみてほしい。
