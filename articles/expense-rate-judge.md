---
title: "フリーランスの経費率を業種別データで自動判定する — 税務調査リスクの可視化"
emoji: "📋"
type: "tech"
topics: ["javascript", "フリーランス", "確定申告", "個人開発"]
published: false
---

「経費率どのくらいが適正？」はフリーランスなら一度は悩む問題。[AND TOOLS](https://and-tools.net/) の経費率判定ツールを作った経験から、業種別の適正経費率データの設計と5段階判定ロジックの実装を解説する。

## 経費率とは

```
経費率 = 経費 ÷ 売上 × 100
```

シンプルな割り算だが、この数字は確定申告での**税務調査リスクの指標**になる。業種ごとに「普通はこのくらい」という相場があり、大きく外れると税務署の目に留まりやすい。

```javascript
function calcExpenseRate(revenue, expenses) {
  if (revenue <= 0) return 0;
  return Math.round((expenses / revenue) * 1000) / 10; // 小数第1位まで
}
```

## 12業種のプリセットデータ

国税庁の統計や業界データをもとに、12業種の適正経費率レンジを設計した。

```javascript
const INDUSTRY_PRESETS = {
  webEngineer: {
    label: 'ITエンジニア・プログラマー',
    typical: 30,      // 典型的な経費率
    low: 15,           // 低め（在宅中心）
    high: 50,          // 高め（外注あり）
    warning: 60,       // 要注意ライン
    items: ['通信費', 'PC・機材', 'ソフトウェア', '書籍', 'クラウドサービス'],
  },
  webDesigner: {
    label: 'Webデザイナー',
    typical: 35,
    low: 20,
    high: 55,
    warning: 65,
    items: ['Adobe CC', 'フォント', 'ストックフォト', 'PC・ディスプレイ', '通信費'],
  },
  writer: {
    label: 'ライター・編集者',
    typical: 25,
    low: 10,
    high: 45,
    warning: 55,
    items: ['取材費', '書籍', '通信費', 'PC', '交通費'],
  },
  consultant: {
    label: 'コンサルタント',
    typical: 30,
    low: 10,
    high: 50,
    warning: 60,
    items: ['交通費', '会議費', '書籍', '通信費', 'セミナー費'],
  },
  photographer: {
    label: 'フォトグラファー',
    typical: 45,
    low: 25,
    high: 65,
    warning: 75,
    items: ['機材', 'スタジオ', '交通費', '現像・プリント', '保険'],
  },
  youtuber: {
    label: 'YouTuber・動画クリエイター',
    typical: 40,
    low: 20,
    high: 60,
    warning: 70,
    items: ['撮影機材', '編集ソフト', 'スタジオ', '企画費', '外注費'],
  },
  illustrator: {
    label: 'イラストレーター',
    typical: 25,
    low: 10,
    high: 45,
    warning: 55,
    items: ['ペンタブ', 'ソフトウェア', 'PC', '資料費', '通信費'],
  },
  realEstate: {
    label: '不動産業',
    typical: 70,
    low: 50,
    high: 85,
    warning: 90,
    items: ['物件費', '修繕費', '管理費', '広告費', '保険料'],
  },
  restaurant: {
    label: '飲食業',
    typical: 65,
    low: 50,
    high: 80,
    warning: 85,
    items: ['原材料費', '家賃', '人件費', '光熱費', '消耗品'],
  },
  retailOnline: {
    label: 'ネットショップ・EC',
    typical: 60,
    low: 40,
    high: 75,
    warning: 85,
    items: ['仕入原価', '送料', '梱包材', 'プラットフォーム手数料', '広告費'],
  },
  coach: {
    label: 'コーチ・講師業',
    typical: 20,
    low: 5,
    high: 40,
    warning: 50,
    items: ['会場費', '教材費', '交通費', '通信費', '広告費'],
  },
  construction: {
    label: '建設・内装業',
    typical: 60,
    low: 40,
    high: 75,
    warning: 85,
    items: ['材料費', '工具', '車両費', '外注費', '保険料'],
  },
};
```

## 5段階判定ロジック

経費率を業種のレンジと照合して、5段階で判定する。

```javascript
function judgeExpenseRate(rate, industryKey) {
  const preset = INDUSTRY_PRESETS[industryKey];
  if (!preset) return { level: 'unknown', message: '業種を選択してください' };

  const { typical, low, high, warning } = preset;

  if (rate <= low) {
    return {
      level: 1,
      label: 'やや低め',
      color: '#3498db',
      message: `${preset.label}の平均(${typical}%)より大幅に低いです。経費の計上漏れがないか確認してください。`,
      risk: 'low',
    };
  }

  if (rate <= typical) {
    return {
      level: 2,
      label: '適正',
      color: '#27ae60',
      message: `${preset.label}の標準的な範囲内です。問題ありません。`,
      risk: 'none',
    };
  }

  if (rate <= high) {
    return {
      level: 3,
      label: '適正（やや高め）',
      color: '#f39c12',
      message: `${preset.label}の平均(${typical}%)よりやや高めですが、業務内容によっては妥当です。`,
      risk: 'low',
    };
  }

  if (rate <= warning) {
    return {
      level: 4,
      label: '高め',
      color: '#e67e22',
      message: `${preset.label}の平均(${typical}%)を大きく上回っています。経費の内訳を整理し、説明できるようにしてください。`,
      risk: 'medium',
    };
  }

  return {
    level: 5,
    label: '要注意',
    color: '#e74c3c',
    message: `${preset.label}としてはかなり高い経費率です。税務調査で指摘される可能性があります。`,
    risk: 'high',
  };
}
```

実装: [経費率 判定ツール](https://and-tools.net/tools/expense-rate/)

## ゲージバーのUI実装

判定結果を視覚的にわかりやすくするため、ゲージバーを実装する。

```css
.gauge {
  width: 100%;
  height: 24px;
  border-radius: 12px;
  background: linear-gradient(
    to right,
    #3498db 0%,
    #27ae60 25%,
    #f39c12 50%,
    #e67e22 75%,
    #e74c3c 100%
  );
  position: relative;
}

.gauge__marker {
  position: absolute;
  top: -8px;
  width: 4px;
  height: 40px;
  background: #2c3e50;
  border-radius: 2px;
  transform: translateX(-50%);
  transition: left 0.3s ease;
}

.gauge__label {
  position: absolute;
  top: 40px;
  transform: translateX(-50%);
  font-size: 14px;
  font-weight: bold;
  white-space: nowrap;
}
```

```javascript
function renderGauge(rate, industryKey) {
  const preset = INDUSTRY_PRESETS[industryKey];
  const maxRate = preset.warning * 1.2;
  const position = Math.min((rate / maxRate) * 100, 100);

  const marker = document.querySelector('.gauge__marker');
  const label = document.querySelector('.gauge__label');

  marker.style.left = `${position}%`;
  label.style.left = `${position}%`;
  label.textContent = `${rate}%`;

  const judgment = judgeExpenseRate(rate, industryKey);
  label.style.color = judgment.color;
}
```

## 業種別の経費内訳テーブルの動的切替

業種を選択すると、その業種の主要経費項目と目安金額が表示される。

```javascript
function getExpenseBreakdown(industryKey, revenue) {
  const breakdowns = {
    webEngineer: [
      { item: 'PC・周辺機器（減価償却）', ratio: 0.05, note: '3年で按分' },
      { item: '通信費（回線・サーバー）', ratio: 0.03, note: '按分率に注意' },
      { item: 'クラウドサービス', ratio: 0.05, note: 'AWS, GitHub等' },
      { item: '書籍・教材', ratio: 0.02, note: '技術書、Udemy等' },
      { item: '家賃（按分）', ratio: 0.10, note: '作業スペースの面積比' },
      { item: '水道光熱費（按分）', ratio: 0.02, note: '使用時間比' },
      { item: '消耗品', ratio: 0.01, note: 'キーボード、マウス等' },
      { item: '交通費', ratio: 0.02, note: '打合せ移動等' },
    ],
    // ... 他の業種も同様に定義
  };

  const items = breakdowns[industryKey] || [];
  return items.map(item => ({
    ...item,
    estimatedAmount: Math.round(revenue * item.ratio),
  }));
}

// 使用例: 年商600万のITエンジニア
const breakdown = getExpenseBreakdown('webEngineer', 6000000);
// → PC: 30万, 通信費: 18万, クラウド: 30万, 書籍: 12万, 家賃: 60万...
// → 合計: 約180万（経費率 30%）
```

## アドバイスロジック — 高い場合 / 低い場合

経費率に応じた具体的なアドバイスを出す。

```javascript
function getAdvice(rate, industryKey) {
  const judgment = judgeExpenseRate(rate, industryKey);
  const preset = INDUSTRY_PRESETS[industryKey];
  const advices = [];

  if (judgment.level <= 1) {
    // 経費率が低い場合のアドバイス
    advices.push(
      '家賃の按分を見直してください。自宅作業なら面積比で計上できます。',
      'PCやスマホの通信費は事業使用比率で按分可能です。',
      '業務に必要な書籍・セミナー費を経費計上していますか？',
      '小規模企業共済やiDeCoの控除も活用しましょう。'
    );
  }

  if (judgment.level >= 4) {
    // 経費率が高い場合のアドバイス
    advices.push(
      '経費の証拠書類（レシート・領収書）を必ず保管してください。',
      '各経費と事業の関連性を説明できるように整理しましょう。',
      'プライベートとの按分比率が適切か見直してください。',
      `${preset.label}の業界平均は${preset.typical}%です。大幅に超える場合は理由を明確に。`
    );
  }

  if (judgment.level >= 5) {
    advices.push(
      '税理士に相談することを強くお勧めします。',
      '税務調査で否認された場合、追徴課税＋延滞税が発生します。'
    );
  }

  return advices;
}
```

実装: [フリーランス単価計算ツール](https://and-tools.net/tools/freelance-rate-calc/)

## 経費率が税金に与えるインパクト

経費率を変えたときの手取りの変化をシミュレーションする。

```javascript
function simulateExpenseImpact(revenue, rateFrom, rateTo) {
  const expenses1 = revenue * rateFrom / 100;
  const expenses2 = revenue * rateTo / 100;
  const income1 = revenue - expenses1;
  const income2 = revenue - expenses2;

  const tax1 = calcFreelanceTotalTax(income1);
  const tax2 = calcFreelanceTotalTax(income2);

  const takeHome1 = revenue - expenses1 - tax1;
  const takeHome2 = revenue - expenses2 - tax2;

  return {
    expenseDiff: expenses2 - expenses1,
    taxDiff: tax2 - tax1,
    takeHomeDiff: takeHome2 - takeHome1,
    note: takeHome2 > takeHome1
      ? `経費率を${rateFrom}%→${rateTo}%にすると手取りが${(takeHome2-takeHome1).toLocaleString()}円増加`
      : `経費率を${rateFrom}%→${rateTo}%にすると手取りが${(takeHome1-takeHome2).toLocaleString()}円減少`,
  };
}

// 年商800万円で経費率を30%→40%にした場合
const impact = simulateExpenseImpact(8000000, 30, 40);
// → 経費80万増、税金約24万減、手取り約56万減
// （経費が実際に発生していないのに計上するのは脱税なので絶対NG）
```

重要: 「経費率を上げれば手取りが増える」は**大きな誤解**。経費は実際に支出した金額しか計上できない。架空経費は脱税であり、税務調査で発覚すれば重加算税の対象になる。

[AND TOOLSの手取り計算ツール](https://and-tools.net/tools/take-home-pay/)では、経費率を変えたときの手取りの変化を視覚的に確認できる。

## まとめ

1. **経費率の適正値は業種によって大きく異なる**（IT: 30% vs 飲食: 65%）
2. **5段階判定**で「低め/適正/やや高め/高め/要注意」を自動分類
3. **ゲージバーUI**で直感的にリスクを可視化
4. **経費が低すぎる場合も要チェック** — 計上漏れで損している可能性
5. **架空経費は絶対NG** — ツールはあくまで適正値を知るためのもの

## AND TOOLSについて

[AND TOOLS](https://and-tools.net/) では、フリーランス・個人事業主向けの税金計算ツールを34個以上無料で公開しています。この記事で紹介した計算ロジックを実際に試せます。
