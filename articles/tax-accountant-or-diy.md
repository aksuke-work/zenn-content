---
title: "税理士に頼むvs自分でやる — 年商いくらが分岐点？"
emoji: "⚖️"
type: "tech"
topics: ["フリーランス", "税理士", "確定申告", "個人事業主"]
published: false
---

「確定申告は自分でやっているけど、そろそろ税理士に頼むべき？」という疑問はフリーランスなら必ず持つ。費用対効果・リスク・時間コストを数値化して判断する方法を解説する。

[AND TOOLS](https://and-tools.net/) で税金計算ツールを運用する中で、「税理士費用を払う価値があるか」を判断するための指標を設計した経験から整理する。

---

## 税理士費用の相場

まず費用を把握する。

```javascript
// 税理士費用の相場（個人事業主・フリーランス）
const taxAccountantFees = {
  // 確定申告のみ依頼（記帳は自分でやる）
  taxReturnOnly: {
    range: "30,000〜80,000円/年",
    typical: 50000,
    notes: "記帳データを自分で用意する前提",
  },

  // 記帳代行 + 確定申告
  withBookkeeping: {
    range: "80,000〜200,000円/年",
    typical: 120000,
    notes: "月次レビューあり、資料を渡すだけでOK",
  },

  // 顧問契約（月次）
  monthlyAdvisory: {
    range: "15,000〜30,000円/月（180,000〜360,000円/年）",
    typical: 240000,
    notes: "節税アドバイス・経営相談・税務調査対応含む",
  },

  // 法人の場合（参考）
  corporation: {
    range: "200,000〜500,000円/年",
    typical: 300000,
    notes: "法人決算が複雑なため個人より高い",
  },
};

Object.entries(taxAccountantFees).forEach(([key, value]) => {
  console.log(`【${key}】${value.range} / 目安: ¥${value.typical.toLocaleString()}`);
});
```

---

## 「自分でやる」のコストを計算する

税理士費用だけ見て「高い」と思うのは片側だけを見ている。自分でやる場合のコストも計算する。

```javascript
// DIYの時間コストと機会損失
function calcDIYCost(hourlyRate, hoursSpent) {
  const opportunityCost = hourlyRate * hoursSpent;

  console.log(`時給: ¥${hourlyRate.toLocaleString()}`);
  console.log(`確定申告に費やす時間: ${hoursSpent}時間`);
  console.log(`機会コスト: ¥${opportunityCost.toLocaleString()}`);
  console.log(`（この時間に仕事をすれば¥${opportunityCost.toLocaleString()}の収入になる）`);

  return opportunityCost;
}

// 時給5,000円のエンジニアが20時間かける場合
const diy20h = calcDIYCost(5000, 20);   // → ¥100,000
// 時給8,000円のエンジニアが15時間かける場合
const diy15h = calcDIYCost(8000, 15);   // → ¥120,000
```

### 「自分でやる」場合の実際の時間

```javascript
const taxReturnHours = {
  "帳簿の整理（年次）": 10,
  "確定申告書の作成": 5,
  "e-Taxの操作・送信": 2,
  "不明点の調査・勉強": 8,
  "税務署への問い合わせ": 2,
  "ミス修正・修正申告（年によっては）": 5,
  total: 32,
};

const total = Object.values(taxReturnHours).reduce((a, b) => a + b, 0);
console.log(`確定申告の年間総時間: 約${total}時間`);
```

1年目・2年目は特に時間がかかる。慣れてきても年間15〜20時間程度は必要だ。

---

## 費用対効果の分析

```javascript
// 税理士費用の費用対効果
function analyzeROI(config) {
  const {
    annualRevenue,
    hourlyRate,
    diyHours,
    accountantFee,
    estimatedTaxSaving,    // 税理士のアドバイスで節税できる見込み額
  } = config;

  // DIYのコスト
  const diyOpportunityCost = hourlyRate * diyHours;

  // 税理士を使う場合のトータルコスト（機会損失は少ない）
  const accountantTotalCost = accountantFee + hourlyRate * 3; // 資料準備3時間程度

  // 税理士を使うことによる純メリット
  const diyNetCost = diyOpportunityCost; // DIYの機会損失
  const accountantNetCost = accountantTotalCost - estimatedTaxSaving; // 費用 - 節税効果

  const roi = (diyNetCost - accountantNetCost);

  return {
    annualRevenue,
    diyOpportunityCost,
    accountantTotalCost,
    estimatedTaxSaving,
    roi,
    recommendation: roi > 0 ? "税理士を使う方が有利" : "自分でやる方が有利",
  };
}

// ケース1: 年収500万・時給5,000円・DIY20時間・税理士5万円・節税効果なし
const case1 = analyzeROI({
  annualRevenue: 5000000,
  hourlyRate: 5000,
  diyHours: 20,
  accountantFee: 50000,
  estimatedTaxSaving: 0,
});
console.log(`ケース1: DIYコスト¥${case1.diyOpportunityCost.toLocaleString()} vs 税理士¥${case1.accountantTotalCost.toLocaleString()} → ${case1.recommendation}`);

// ケース2: 年収1000万・時給8,000円・DIY25時間・税理士10万円・節税効果15万
const case2 = analyzeROI({
  annualRevenue: 10000000,
  hourlyRate: 8000,
  diyHours: 25,
  accountantFee: 100000,
  estimatedTaxSaving: 150000,
});
console.log(`ケース2: DIYコスト¥${case2.diyOpportunityCost.toLocaleString()} vs 税理士¥${case2.accountantTotalCost.toLocaleString()} / 節税¥${case2.estimatedTaxSaving.toLocaleString()} → ${case2.recommendation}`);
```

---

## 年商別の判断基準

```javascript
const criteria = [
  {
    revenueRange: "〜年商300万円",
    recommendation: "自分でやる",
    reasons: [
      "税額が低いため節税効果が小さい",
      "会計ソフト（freee等）で十分対応できる",
      "税理士費用の回収が難しい",
    ],
  },
  {
    revenueRange: "年商300〜800万円",
    recommendation: "確定申告のみ税理士に依頼（年5〜8万円）",
    reasons: [
      "節税アドバイスの効果が出始める",
      "自分の時間を本業に集中させる価値が出る",
      "ミスのリスクが増える税額帯",
    ],
  },
  {
    revenueRange: "年商800万〜1500万円",
    recommendation: "記帳代行 + 確定申告依頼（年10〜20万円）",
    reasons: [
      "経費・節税の複雑さが増す",
      "iDeCo・小規模企業共済の最適化アドバイスが重要になる",
      "予定納税・個人事業税の管理も必要",
    ],
  },
  {
    revenueRange: "年商1500万円超 or 法人化検討",
    recommendation: "顧問契約（月2〜3万円）",
    reasons: [
      "法人化の判断・設立サポートが必要",
      "消費税の課税事業者・インボイス対応",
      "社会保険・マイクロ法人の設計",
      "税務調査のリスクが無視できなくなる",
    ],
  },
];

criteria.forEach((c) => {
  console.log(`【${c.revenueRange}】${c.recommendation}`);
  c.reasons.forEach((r) => console.log(`  ・${r}`));
});
```

---

## 「税理士に頼む」価値がある場面

### 1. 税務調査のリスクが高まってきた

- 売上1,000万円を超えると消費税の申告が必要になる
- 売上と経費の規模が大きくなると申告ミスの影響も大きくなる
- 税務調査は申告から3〜7年さかのぼれる

### 2. 大きな出来事があった年

```javascript
const taxablEvents = [
  "不動産の売却",
  "法人化（個人事業廃業 + 法人設立）",
  "マイクロ法人の設立",
  "相続・贈与",
  "海外取引の発生",
  "副業収入が大幅増加",
  "投資（株・FX）で大きな利益 or 損失",
];

console.log("税理士への相談を強く勧めるイベント:");
taxablEvents.forEach((e) => console.log(`・${e}`));
```

### 3. 節税の最適化に自信がない

iDeCo・小規模企業共済・青色申告の活用・経費の按分・家事按分を適切に組み合わせることで、年間50万円以上の節税差が出る場合がある。「自分でできているつもり」でも、税理士のレビューで見落としが発見されることは多い。

[AND TOOLSの所得税計算ツール](https://and-tools.net/tools/income-tax/) で現在の税額を把握し、最適化の余地があるか確認することから始めるのも一つの方法だ。

---

## 良い税理士の見つけ方

```javascript
const goodTaxAccountantCriteria = {
  必須条件: [
    "フリーランス・個人事業主の対応実績が豊富",
    "クラウド会計ソフト（freee・弥生・MFC）に対応",
    "料金が明確（追加費用の条件が明示されている）",
    "レスポンスが早い（メール・チャット対応可）",
  ],
  望ましい条件: [
    "IT・Web業界の経験がある（業種特有の経費判断ができる）",
    "節税アドバイスに積極的（申告書を作るだけでなく提案してくれる）",
    "税務調査の対応実績がある",
    "初回相談が無料",
  ],
  避けるべき: [
    "料金体系が不透明",
    "「全部任せてください」だけで節税提案がない",
    "年1回しか会わない・連絡が取れない",
  ],
};
```

### 探す方法

1. **紹介**: 同業フリーランスからの紹介が最も信頼性が高い
2. **税理士ドットコム**: エリア・業種・料金で絞り込める
3. **クラウド会計ソフトの提携税理士**: freee・弥生が紹介しているパートナー税理士

---

## 自分でやる場合のツール選定

| ツール | 月額 | 特徴 | 向いている人 |
|--------|------|------|------------|
| freee | 1,980円〜 | UI最良・銀行連携強 | 初心者 |
| 弥生会計オンライン | 1,100円〜 | 安定・実績あり | 乗り換えたくない人 |
| MFクラウド会計 | 1,298円〜 | 仕訳自動化が優秀 | データ連携重視 |
| Excel | 0円 | 自由度高い | 複式簿記の知識がある人のみ |

Excelでの確定申告は青色65万円控除の要件（複式簿記）を満たすのが難しく、現実的でない。

---

## まとめ：分岐点の目安

| 年商 | 推奨 | 理由 |
|------|------|------|
| 〜300万円 | DIY + 会計ソフト | コスト回収が難しい |
| 300〜800万円 | 確定申告だけ税理士に | 時間コスト > 税理士費用になるラインを超えてくる |
| 800万〜1500万円 | 記帳代行込みで依頼 | 節税効果 > 費用が明確になる |
| 1500万円超 | 顧問契約 | 法人化・税務調査対策が必要 |

[AND TOOLS](https://and-tools.net/) では [所得税計算](https://and-tools.net/tools/income-tax/)・[手取り計算](https://and-tools.net/tools/take-home/)・[法人化シミュレーター](https://and-tools.net/tools/solo-corp/) を無料で提供している。現在の税負担を正確に把握した上で、税理士費用の費用対効果を判断してほしい。
