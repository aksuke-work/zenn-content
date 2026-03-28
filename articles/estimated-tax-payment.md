---
title: "予定納税とは何か — 突然届く納付書に慌てないために"
emoji: "📬"
type: "tech"
topics: ["予定納税", "確定申告", "フリーランス", "税金"]
published: true
---

フリーランス2年目の夏に突然届く「所得税及び復興特別所得税の予定納税の納付書」。初めて見た人の多くが「これは何？払わないといけないの？」と慌てる。

[AND TOOLS](https://and-tools.net/) の税金計算ツールを開発する中で、予定納税に関する相談は特に多かった。この記事で仕組みと対応方法を完全に理解する。

---

## 予定納税とは

**前年の確定申告での所得税額が15万円以上**の場合、翌年の税金を「前払い」させる制度だ。

```
通常の所得税の流れ（会社員）:
毎月の給与から源泉徴収 → 年末調整で精算

フリーランスの所得税の流れ（予定納税あり）:
7月に前払い（第1期）→ 11月に前払い（第2期）→ 翌年3月に確定申告で精算
```

税務署からすると「大きな所得があるなら、翌年の税金も前払いしてください」という仕組みだ。

---

## 発生条件と計算方法

### 発生条件

前年の「予定納税基準額」が15万円以上の場合に自動的に発生する。

```javascript
// 予定納税基準額の計算
function calcYoteiNozei(previousYearTax) {
  // 予定納税基準額 = 前年の課税所得に対する所得税
  // （源泉徴収税額や配当控除等を差し引いた後の金額）

  const threshold = 150000; // 15万円が発生条件

  if (previousYearTax < threshold) {
    return {
      required: false,
      message: "予定納税は発生しません",
      previousYearTax,
    };
  }

  // 各期の納税額 = 基準額の1/3
  const perPeriod = Math.floor(previousYearTax / 3);

  return {
    required: true,
    previousYearTax,
    firstPeriod: {
      amount: perPeriod,
      deadline: "7月31日",
    },
    secondPeriod: {
      amount: perPeriod,
      deadline: "11月30日",
    },
    prepaidTotal: perPeriod * 2,
    message: `前払い合計: ¥${(perPeriod * 2).toLocaleString()}（各期¥${perPeriod.toLocaleString()}）`,
  };
}

// 例：前年の所得税が30万円だった場合
const result300 = calcYoteiNozei(300000);
console.log(result300);
// → 第1期: 100,000円（7月31日）/ 第2期: 100,000円（11月30日）/ 前払い合計: 200,000円

// 予定納税なしの例
const result100 = calcYoteiNozei(100000);
console.log(result100);
// → 予定納税は発生しません
```

### 実際の納付額

前年の所得税の3分の1ずつを2回（7月・11月）に分けて納付する。残りは翌年3月の確定申告で精算される。

```
前年所得税: 90万円の場合
├── 第1期（7月31日）: 30万円
├── 第2期（11月30日）: 30万円
└── 確定申告精算（翌年3月）: 残り（増減に応じて追納 or 還付）
```

---

## 予定納税通知の見方

毎年6月中旬頃に「所得税及び復興特別所得税の予定納税額の通知書」が届く。

通知書の記載内容：
- **第1期分**: 7月31日納付期限の金額
- **第2期分**: 11月30日納付期限の金額
- **合計**: 前払いする総額

この通知書に記載された金額をそのまま納付する。

---

## 予定納税の納付方法

```
1. e-Tax（オンライン）  ← 推奨
   └── 銀行口座からダイレクト納付
   └── クレジットカード（手数料かかる）

2. 振替納税
   └── 口座振替の登録が必要
   └── 確定申告の振替納税と同様

3. コンビニ納付（QRコード）
   └── 確定申告後に発行されるQRコードを使用

4. 金融機関・郵便局の窓口
   └── 納付書（納付書が届いた場合）持参
```

---

## 予定納税を減額する方法

今年の所得が前年より大幅に下がる見込みの場合、予定納税の**減額申請**ができる。

### 減額申請の流れ

1. **申請期限**：
   - 第1期・第2期ともに減額したい場合 → **7月15日まで**
   - 第2期のみ減額したい場合 → **11月15日まで**

2. **申請書の提出**：税務署に「予定納税額の減額申請書」を提出

3. **承認・却下**：提出後、税務署が審査し通知が来る

```javascript
// 減額申請の検討基準
function shouldApplyForReduction(previousTax, estimatedCurrentTax) {
  const scheduledPrepayment = Math.floor(previousTax / 3) * 2;

  if (estimatedCurrentTax < scheduledPrepayment) {
    const reduction = scheduledPrepayment - estimatedCurrentTax;
    return {
      shouldApply: true,
      reason: "今年の税額が前払い額より少ない見込み",
      scheduledPrepayment,
      estimatedCurrentTax,
      potentialReduction: reduction,
      message: `減額申請で¥${reduction.toLocaleString()}の支払いを減らせる可能性あり`,
    };
  }

  return {
    shouldApply: false,
    reason: "前払い額以上の税金が見込まれるため申請不要",
  };
}

// 前年税額90万円・今年の見込み税額30万円
const check = shouldApplyForReduction(900000, 300000);
console.log(check);
// → 減額申請で¥300,000の支払いを減らせる可能性あり
```

### 減額申請が有効なケース

- 今年は**大きな経費（機器購入等）**がある
- **収入が前年より大幅に減少**した
- **iDeCo・小規模企業共済**を大幅に増額した

---

## 予定納税の資金計画

予定納税を踏まえた資金計画を立てる。

```javascript
// 予定納税を考慮した年間資金計画
function planCashFlow(monthlyRevenue, taxRate = 0.25) {
  const monthlyTaxReserve = monthlyRevenue * taxRate;

  // 予定納税が発生する前年の所得税を概算
  const annualRevenue = monthlyRevenue * 12;
  const estimatedAnnualTax = annualRevenue * taxRate * 0.6; // 粗い概算

  const yoteiNozei = calcYoteiNozei(estimatedAnnualTax);

  console.log("=== 年間資金計画（予定納税考慮）===");
  console.log(`月収: ¥${monthlyRevenue.toLocaleString()}`);
  console.log(`毎月積み立て推奨: ¥${Math.floor(monthlyTaxReserve).toLocaleString()}`);
  console.log("\n主な支出タイミング:");
  console.log(`3月: 確定申告・所得税納付`);

  if (yoteiNozei.required) {
    console.log(`7月: 予定納税第1期 ¥${yoteiNozei.firstPeriod.amount.toLocaleString()}`);
    console.log(`11月: 予定納税第2期 ¥${yoteiNozei.secondPeriod.amount.toLocaleString()}`);
  }

  console.log(`6〜8月: 住民税・個人事業税（4回払い）`);
}

planCashFlow(700000); // 月収70万円
```

---

## 延滞した場合のペナルティ

予定納税を期限内に納付しないと**延滞税**が発生する。

```javascript
// 延滞税の概算計算
function calcEnntaizei(principalTax, delinquencyDays) {
  // 納期限の翌日から2ヶ月以内: 年2.4%（2024年）
  const rate1 = 0.024;
  // 納期限の翌日から2ヶ月超: 年8.7%（2024年）
  const rate2 = 0.087;

  const daysInYear = 365;

  if (delinquencyDays <= 60) {
    const enntaizei = Math.floor(principalTax * rate1 * delinquencyDays / daysInYear);
    return { enntaizei, details: `${delinquencyDays}日間の延滞（2ヶ月以内・年2.4%）` };
  }

  const first60 = Math.floor(principalTax * rate1 * 60 / daysInYear);
  const remaining = Math.floor(principalTax * rate2 * (delinquencyDays - 60) / daysInYear);
  return {
    enntaizei: first60 + remaining,
    details: `最初60日: ¥${first60.toLocaleString()} + 残り${delinquencyDays - 60}日: ¥${remaining.toLocaleString()}`,
  };
}

// 50万円を90日延滞した場合
const penalty = calcEnntaizei(500000, 90);
console.log(`延滞税: ¥${penalty.enntaizei.toLocaleString()}`);
console.log(penalty.details);
```

延滞税は利率が高い。7月と11月の期限を必ずカレンダーに入れておく。

---

## 予定納税と確定申告の関係

予定納税で前払いした金額は、翌年の確定申告で精算される。

```
【確定申告での精算】

本来の所得税（確定申告で計算した税額）
├── 予定納税額（前払い分）を差し引く
│   ├── 差額がプラス → 追納（3月15日まで）
│   └── 差額がマイナス → 還付
└── 源泉徴収税額も差し引く
```

つまり、予定納税は「税金の前払い」であり、最終的な税額が変わるわけではない。今年の収入が前年より低ければ、前払いしすぎた分が3月の申告後に還付される。

[AND TOOLSの所得税計算ツール](https://and-tools.net/tools/income-tax/) で今年の所得税額を概算し、予定納税との差額を把握しておくと資金計画が立てやすい。

---

## まとめ

- 予定納税は**前年の所得税が15万円以上**の場合に発生
- **7月31日**と**11月30日**に前年所得税の1/3ずつを前払い
- 収入が減った場合は**減額申請**（7月15日・11月15日まで）で対応できる
- 最終的な税額は翌年3月の確定申告で精算される

突然届く納付書に慌てないために、毎月の収入から税金積立を行い、[AND TOOLS](https://and-tools.net/) で税額を試算しながら資金計画を立てておくのが最善策だ。
