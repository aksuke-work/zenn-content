---
title: "フリーランスの保険を比較した — 国保vs文美国保vsITフリーランス"
emoji: "🏥"
type: "tech"
topics: ["フリーランス", "国民健康保険", "社会保険", "個人事業主"]
published: false
---

フリーランスになると健康保険を自分で選ぶ必要がある。「とりあえず国民健康保険」で済ませている人が多いが、条件によっては別の選択肢の方が大幅に安くなるケースがある。

[AND TOOLS](https://and-tools.net/) の手取り計算ツールで社会保険料の計算設計をした経験から、フリーランスが選べる健康保険の選択肢を比較する。

---

## フリーランスが選べる健康保険の選択肢

| 保険 | 加入条件 | 特徴 |
|------|---------|------|
| **国民健康保険（国保）** | 誰でも | 前年所得に応じた保険料 |
| **文芸美術国民健康組合（文美国保）** | 文芸・美術・デザイン等の職種 | 所得に関わらず定額 |
| **ITS（情報サービス産業健保）** | IT企業の社員・特定の組合員 | 個人では入りにくい |
| **フリーランス協会の保険** | 一般社団法人への入会 | 賠償責任保険がセット |
| **健康保険の任意継続** | 退職後2年間 | 前職の健保を継続 |

---

## 1. 国民健康保険（国保）

最も一般的な選択肢。保険料は前年の所得に応じて変わる。

### 計算方法

市区町村によって計算式が異なるが、おおよそ以下の構成だ。

```javascript
// 東京都の国保保険料（概算計算 - 2024年度参考値）
function calcNationalHealthInsurance(annualIncome, age, dependents = 0) {
  // 所得割・均等割・平等割・資産割（自治体によって異なる）
  // 東京都の参考値（各区で差あり）

  const INCOME_DEDUCTION = 430000; // 基礎控除相当
  const taxableIncome = Math.max(annualIncome - INCOME_DEDUCTION, 0);

  // 医療分
  const medicalIncome = taxableIncome * 0.0739;   // 所得割
  const medicalHeadCount = 48300 * (1 + dependents); // 均等割

  // 支援分
  const supportIncome = taxableIncome * 0.0241;
  const supportHeadCount = 15000 * (1 + dependents);

  // 介護分（40〜64歳のみ）
  let careIncome = 0;
  let careHeadCount = 0;
  if (age >= 40 && age < 65) {
    careIncome = taxableIncome * 0.0207;
    careHeadCount = 17000 * (1 + dependents);
  }

  const annual = medicalIncome + medicalHeadCount + supportIncome + supportHeadCount + careIncome + careHeadCount;

  // 上限（2024年度）
  const MAX_MEDICAL = 920000;
  const MAX_SUPPORT = 240000;
  const MAX_CARE = 170000;
  const cappedAnnual = Math.min(medicalIncome + medicalHeadCount, MAX_MEDICAL)
    + Math.min(supportIncome + supportHeadCount, MAX_SUPPORT)
    + Math.min(careIncome + careHeadCount, MAX_CARE);

  return {
    estimatedAnnual: Math.floor(cappedAnnual),
    estimatedMonthly: Math.floor(cappedAnnual / 12),
  };
}

// 年収別の概算保険料
const incomes = [3000000, 5000000, 8000000, 12000000];
incomes.forEach((income) => {
  const result = calcNationalHealthInsurance(income, 35);
  console.log(
    `年収${(income / 10000).toFixed(0)}万円 → 年間¥${result.estimatedAnnual.toLocaleString()} / 月¥${result.estimatedMonthly.toLocaleString()}`
  );
});
```

国保の特徴：
- 所得が上がるほど保険料が高くなる
- 年収1,000万円超から上限に達する（年間約100〜130万円）
- 扶養概念がない（家族全員分の保険料がかかる）

---

## 2. 文芸美術国民健康組合（文美国保）

デザイナー・イラストレーター・ライター・写真家・映像クリエイターなどが加入できる職種別の国保組合。

### 特徴

```
月額保険料（2024年度）:
├── 40歳未満: 22,940円/月（年額: 275,280円）
├── 40〜64歳: 28,040円/月（年額: 336,480円）
└── 家族1人追加: +8,000円/月程度
```

### 国保との比較

```javascript
// 文美国保 vs 国保の比較
function compareInsurance(annualIncome, age) {
  // 国保（概算）
  const nationalResult = calcNationalHealthInsurance(annualIncome, age);

  // 文美国保（定額）
  const fumbiMonthly = age >= 40 ? 28040 : 22940;
  const fumbiAnnual = fumbiMonthly * 12;

  const diff = nationalResult.estimatedAnnual - fumbiAnnual;
  const savings = diff > 0 ? `文美国保が年間¥${diff.toLocaleString()}安い` : `国保が年間¥${Math.abs(diff).toLocaleString()}安い`;

  return {
    national: nationalResult.estimatedAnnual,
    fumbi: fumbiAnnual,
    recommendation: savings,
  };
}

// 年収500万円・35歳
const result35 = compareInsurance(5000000, 35);
console.log(`年収500万・35歳: ${result35.recommendation}`);

// 年収800万円・35歳
const result800 = compareInsurance(8000000, 35);
console.log(`年収800万・35歳: ${result800.recommendation}`);
```

### 文美国保が有利になるライン

所得が高くなるほど国保保険料も上がるため、年収が高い人ほど文美国保が有利になる傾向がある。おおよそ**年収500万円以上**から文美国保の方が安くなるケースが多い（地域・年齢・家族構成によって変わる）。

### 加入条件の確認が必要

文美国保はクリエイティブ職の組合であり、**エンジニアは原則として加入対象外**の場合が多い。ただし、UI/UXデザイン・映像制作などのクリエイティブ要素が強い業務の場合は申請できることもある。公式サイトで最新の加入条件を確認する。

---

## 3. 健康保険の任意継続

会社を退職後、2年間に限り前職の健康保険を継続できる。

### 任意継続が有利なケース

```javascript
// 任意継続の保険料計算
function calcNininkei(previousMonthlyStandardWage) {
  // 任意継続は以前の標準報酬月額または上限（2024年度: 30万円）の低い方
  const baseWage = Math.min(previousMonthlyStandardWage, 300000);

  // 健保組合によって異なるが、協会けんぽ（東京）の参考値
  const healthInsuranceRate = 0.1022; // 10.22%（東京都・2024年度）
  const nursingRate = 0.018; // 介護保険（40歳以上のみ）

  const healthMonthly = Math.floor(baseWage * healthInsuranceRate);
  const nursingMonthly = Math.floor(baseWage * nursingRate);

  return {
    healthMonthly,
    nursingMonthly,
    totalMonthly: healthMonthly + nursingMonthly,
    totalAnnual: (healthMonthly + nursingMonthly) * 12,
  };
}

// 以前の月給40万円の場合
const result = calcNininkei(400000);
console.log(`任意継続月額（標準報酬30万円上限）: ¥${result.totalMonthly.toLocaleString()}`);
```

任意継続は**開業直後に所得が低い場合**や**前職の健保組合が充実している場合**に有利になることがある。ただし2年間という期間制限があり、開業後に所得が増えた場合でも途中変更は原則できない。

---

## 4. フリーランス協会の保険

一般社団法人「フリーランス協会」の有料会員（年間10,780円）になることで、以下の保険が付帯する。

- **バイヤーズガイド会員特典**: 賠償責任保険（対人・対物・情報漏洩等）
- **健康保険**: フリーランス協会の団体保険

賠償責任保険が単体で入ると月3,000〜5,000円程度かかるため、コスパが良い。エンジニアはとくに「情報漏洩」リスクがあるため検討する価値がある。

---

## 選択の判断フロー

```
フリーランスになった
│
├── 退職直後で国保より安い可能性あり
│   └── → 任意継続を2年間利用
│
├── クリエイティブ職で年収500万以上
│   └── → 文美国保を検討
│
├── ITエンジニア・年収300〜500万
│   └── → 国保（+ 保険料軽減の申請を活用）
│
└── 国保 + フリーランス協会で賠償責任保険をカバー
```

---

## 保険料を安くする方法（国保の場合）

### 前納割引

国保の前納（一括払い）を選ぶと割引になる自治体がある（約1〜2%）。資金に余裕がある年は活用する。

### 開業初年度の軽減申請

会社を退職した場合、「退職者軽減」として保険料が軽減される場合がある。市区町村の国保窓口で確認する。

### 収入が激減した年の減額申請

前年と比べて収入が大幅に下がった場合、「保険料の減額・免除申請」ができる自治体がある。

---

## 実際の計算で比較する

```javascript
// 3種類の保険料を一括比較する関数
function compareAllInsurance(annualIncome, age, previousWage = null) {
  console.log(`\n=== 年収${(annualIncome / 10000).toFixed(0)}万円・${age}歳の保険比較 ===`);

  // 国保
  const national = calcNationalHealthInsurance(annualIncome, age);
  console.log(`国民健康保険: 年¥${national.estimatedAnnual.toLocaleString()}`);

  // 文美国保（クリエイティブ職の場合）
  const fumbiMonthly = age >= 40 ? 28040 : 22940;
  const fumbiAnnual = fumbiMonthly * 12;
  console.log(`文美国保（クリエイティブ職）: 年¥${fumbiAnnual.toLocaleString()}`);

  // 任意継続（前職があれば）
  if (previousWage) {
    const nininkei = calcNininkei(previousWage);
    console.log(`任意継続（前月給${(previousWage / 10000).toFixed(0)}万）: 年¥${nininkei.totalAnnual.toLocaleString()}`);
  }
}

compareAllInsurance(5000000, 35, 350000);
compareAllInsurance(8000000, 42);
```

---

## まとめ

フリーランスの健康保険選択は「年収・職種・年齢」で答えが変わる。

- **開業直後**：任意継続を2年間活用（前職の健保の方が安い場合が多い）
- **クリエイティブ職・年収500万超**：文美国保を真剣に検討
- **エンジニア中心**：国保 + フリーランス協会で賠償責任保険をカバー

[AND TOOLSの手取り計算ツール](https://and-tools.net/tools/take-home/) では、保険料込みの実質手取りを試算できる。どの保険を選んだ場合の手取りが最大化されるか、数字で確認してから決断してほしい。
