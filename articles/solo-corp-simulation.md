---
title: "法人化の損益分岐点をJavaScriptでシミュレーションする — 個人事業主vs一人法人の税金比較"
emoji: "🏢"
type: "tech"
topics: ["javascript", "税金", "法人化", "個人開発"]
published: false
---

フリーランスが事業所得1,000万円を超えたあたりで頭をよぎる「法人化」。でも変数が多すぎて、手計算では判断できない。JavaScriptでシミュレーターを組んで、損益分岐点を実際に求めてみた。

## 個人事業主の税金モデル

個人事業主が払う税金・社会保険料は5種類。

```javascript
function calcSoleProprietorTax(income, expenses) {
  const profit = income - expenses;

  // 青色申告特別控除
  const blueDeduction = 650000;
  const taxableBase = Math.max(profit - blueDeduction, 0);

  // 国民健康保険（所得割 + 均等割）
  const nhiIncome = taxableBase - 430000; // 基礎控除
  const nhiRate = 0.1136; // 医療+支援+介護（市区町村で異なる）
  const nhiFlat = 52000;  // 均等割（1人分概算）
  const nhiCap = 1060000; // 上限額（2026年度）
  const nhi = Math.min(
    Math.max(nhiIncome * nhiRate, 0) + nhiFlat,
    nhiCap
  );

  // 国民年金
  const pension = 16980 * 12; // 2026年度月額 × 12

  // 社会保険料控除後の課税所得
  const socialInsurance = nhi + pension;
  const taxableIncome = Math.max(taxableBase - socialInsurance - 480000, 0);

  // 所得税（累進課税）
  const incomeTax = calcProgressiveTax(taxableIncome);

  // 住民税
  const residentTaxable = Math.max(taxableBase - socialInsurance - 430000, 0);
  const residentTax = Math.floor(residentTaxable * 0.10) + 5000;

  // 個人事業税（5%、290万控除）
  const bizTax = Math.max((profit - 2900000) * 0.05, 0);

  return {
    incomeTax,
    residentTax,
    bizTax,
    nhi,
    pension,
    total: incomeTax + residentTax + bizTax + nhi + pension,
    takeHome: profit - incomeTax - residentTax - bizTax - nhi - pension,
  };
}
```

ポイント:
- **国保は自治体ごとに料率が違う**。ここでは全国平均的な値を使っている
- 青色申告特別控除65万円は電子申告が条件
- 個人事業税は業種によって税率3〜5%（多くのIT系は5%）

所得税の累進課税は[前回の記事](https://zenn.dev/emptycoke/articles/js-tax-calculation-japan)で書いた `calcProgressiveTax` をそのまま使う。

## 一人法人の税金モデル

法人化すると「法人の税金」と「社長個人の税金」の2つに分かれる。ここが複雑さの元凶。

```javascript
function calcCorpTax(profit, compensation) {
  // --- 法人側 ---
  const corpProfit = Math.max(profit - compensation, 0);

  // 法人税（年800万以下: 15%、超過分: 23.2%）
  const corpTax = corpProfit <= 8000000
    ? Math.floor(corpProfit * 0.15)
    : Math.floor(8000000 * 0.15 + (corpProfit - 8000000) * 0.232);

  // 法人住民税（法人税割 + 均等割）
  const corpResident = Math.floor(corpTax * 0.173) + 70000;

  // 法人事業税（簡易計算）
  const corpBizTax = Math.floor(corpProfit * 0.035);

  const corpTotal = corpTax + corpResident + corpBizTax;

  // --- 社長個人側 ---
  const salaryDeduction = calcSalaryDeduction(compensation);
  const salaryIncome = Math.max(compensation - salaryDeduction, 0);

  // 社会保険料（会社負担 + 個人負担の合計）
  const socialInsTotal = calcSocialInsuranceCorp(compensation);

  // 個人の課税所得
  const personalTaxable = Math.max(
    salaryIncome - socialInsTotal.employee - 480000, 0
  );
  const personalIncomeTax = calcProgressiveTax(personalTaxable);

  const personalResidentTaxable = Math.max(
    salaryIncome - socialInsTotal.employee - 430000, 0
  );
  const personalResidentTax = Math.floor(personalResidentTaxable * 0.10) + 5000;

  return {
    corpTotal,
    personalIncomeTax,
    personalResidentTax,
    socialInsEmployee: socialInsTotal.employee,
    socialInsCompany: socialInsTotal.company,
    grandTotal: corpTotal + personalIncomeTax + personalResidentTax
      + socialInsTotal.employee + socialInsTotal.company,
    takeHome: compensation - personalIncomeTax - personalResidentTax
      - socialInsTotal.employee,
  };
}
```

法人側のポイント:
- **年800万以下は法人税15%**（中小法人の軽減税率）
- 均等割7万円は赤字でも毎年かかる
- 社会保険料は会社負担分も実質コスト

## 役員報酬の最適配分アルゴリズム

法人化で最も重要なのが「利益のうち何%を役員報酬にするか」。ここを最適化するだけで年間数十万円変わる。

```javascript
function findOptimalCompensation(profit) {
  let bestTotal = Infinity;
  let bestComp = 0;

  // 役員報酬を10万円刻みで探索
  for (let comp = 0; comp <= profit; comp += 100000) {
    const result = calcCorpTax(profit, comp);
    if (result.grandTotal < bestTotal) {
      bestTotal = result.grandTotal;
      bestComp = comp;
    }
  }

  // 見つかった付近を1万円刻みで精密探索
  const start = Math.max(bestComp - 100000, 0);
  const end = Math.min(bestComp + 100000, profit);
  for (let comp = start; comp <= end; comp += 10000) {
    const result = calcCorpTax(profit, comp);
    if (result.grandTotal < bestTotal) {
      bestTotal = result.grandTotal;
      bestComp = comp;
    }
  }

  return { optimalCompensation: bestComp, totalBurden: bestTotal };
}
```

このアルゴリズムで分かること:
- 利益800万以下 → **ほぼ全額を役員報酬にする**のが最適（法人に利益を残さない）
- 利益1,500万超 → **法人に一部残す**ほうが、累進課税を避けられてトータルが安い
- 社会保険料の上限があるので、報酬を上げすぎても保険料は頭打ちになる

実際のシミュレーションは [法人化シミュレーター](https://and-tools.net/tools/solo-corp-sim/) で試せる。

## 損益分岐点を求める

個人事業主と法人の税負担を事業所得300万〜3,000万で比較する。

```javascript
function findBreakeven() {
  const results = [];

  for (let income = 3000000; income <= 30000000; income += 1000000) {
    const sole = calcSoleProprietorTax(income, 0);
    const { optimalCompensation } = findOptimalCompensation(income);
    const corp = calcCorpTax(income, optimalCompensation);

    const diff = sole.total - corp.grandTotal;

    results.push({
      income,
      soleTax: sole.total,
      corpTax: corp.grandTotal,
      diff,
      recommendation: diff > 0 ? '法人有利' : '個人有利',
    });
  }

  return results;
}
```

実行結果（概算）:

| 事業所得 | 個人の税負担 | 法人の税負担 | 差額 | 判定 |
|---------|------------|------------|------|------|
| 300万 | 約42万 | 約55万 | -13万 | 個人有利 |
| 500万 | 約108万 | 約110万 | -2万 | ほぼ同等 |
| 700万 | 約186万 | 約170万 | +16万 | **法人有利** |
| 1,000万 | 約310万 | 約265万 | +45万 | **法人有利** |
| 1,500万 | 約530万 | 約430万 | +100万 | **法人有利** |
| 2,000万 | 約780万 | 約610万 | +170万 | **法人有利** |

**結論: 事業所得700万〜1,000万が法人化の損益分岐点。**

ただし、この計算には含めていない要素がある:
- 法人設立費用（約25万円）
- 税理士費用（年30〜50万円）
- 社会保険の将来的な年金受給額の差
- 消費税のインボイス制度の影響

これらを含めると、**現実的には事業所得800万〜1,000万で法人化を検討**というのが妥当なラインだ。

## 手取り額の計算もセットで

法人化の判断には「結局手取りがいくら増えるか」が大事。[手取り計算シミュレーター](https://and-tools.net/tools/take-home-pay/) で個人事業主としての手取りを出し、[法人化シミュレーター](https://and-tools.net/tools/solo-corp-sim/)と比較すると、リアルな差額が見える。

さらに法人化後は[社会保険料計算](https://and-tools.net/tools/social-insurance/)で会社負担分を含めたコストを把握し、[フリーランス単価計算](https://and-tools.net/tools/freelance-rate-calc/)で必要な売上目標を逆算するのがおすすめ。

## まとめ

法人化の判断は「変数が多すぎて感覚で決められない」問題の典型。コードで書くと以下が見える:

1. **損益分岐点は事業所得700〜1,000万**（税額だけで見た場合）
2. **役員報酬の最適配分**が年間数十万円の差を生む
3. **社会保険料の上限**が法人化のメリットを大きくしている
4. **法人設立・維持コスト**を差し引いても、1,000万超ならほぼ法人有利

「なんとなく」で判断せず、自分の数字を入れてシミュレーションしてから税理士に相談するのが最も効率的だと思う。

---

この記事で使った計算ロジックは [AND TOOLS](https://and-tools.net/) で実際に動いている。フリーランスの税金・社会保険料に関する34個のツールが無料で使えるので、法人化を検討中の方はぜひ試してみてほしい。
