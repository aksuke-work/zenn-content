---
title: "目標手取りから必要年収を逆算するアルゴリズム — 二分探索で税金の逆問題を解く"
emoji: "🔍"
type: "tech"
topics: ["javascript", "algorithm", "フリーランス", "個人開発"]
published: false
---

「手取りで500万欲しいんだけど、年収いくら必要？」

フリーランスや転職を考えている人からよく聞く質問だ。[AND TOOLS](https://and-tools.net/) の逆算ツールを作るときに、この問題が意外と難しいことに気づいた。

## なぜ単純な逆算ができないのか

手取りの計算式はざっくりこう:

```
手取り = 年収 - 所得税 - 住民税 - 社会保険料
```

「じゃあ逆に解けばいいじゃん」と思うが、**累進課税は非線形関数**なので代数的に逆関数を求めるのが面倒。

具体的には:
- 所得税は7段階の累進税率（どのブラケットに入るか事前にわからない）
- 社会保険料は標準報酬月額テーブルへのマッピング（階段関数）
- 給与所得控除も年収帯で計算式が変わる（区分関数）

これらが組み合わさった合成関数の逆関数を閉じた形で書くのは現実的ではない。

## 二分探索（バイナリサーチ）で解く

発想を変える。**「年収 → 手取り」の順方向関数は既にある**。であれば、二分探索で年収を探せばいい。

```javascript
function calcTakeHome(annualIncome) {
  // 順方向計算: 年収 → 手取り
  const salaryDeduction = calcSalaryDeduction(annualIncome);
  const salaryIncome = annualIncome - salaryDeduction;
  const socialInsurance = calcSocialInsurance(annualIncome);
  const taxableIncome = Math.max(0, salaryIncome - socialInsurance - 580000);
  const incomeTax = calcIncomeTax(taxableIncome);
  const residentTax = calcResidentTax(taxableIncome);

  return annualIncome - incomeTax - residentTax - socialInsurance;
}
```

この関数は**単調増加**。年収が上がれば手取りも上がる（税率が上がっても手取りが減ることはない）。単調増加関数なら二分探索が使える。

```javascript
function reverseCalcIncome(targetTakeHome) {
  // 探索範囲: 手取りの1.0倍〜2.0倍
  let low = targetTakeHome;
  let high = targetTakeHome * 2;
  const tolerance = 1000; // 精度 ±1,000円

  while (high - low > tolerance) {
    const mid = Math.floor((low + high) / 2);
    const takeHome = calcTakeHome(mid);

    if (takeHome < targetTakeHome) {
      low = mid;
    } else {
      high = mid;
    }
  }

  return high;
}
```

実装: [手取りから年収を逆算するツール](https://and-tools.net/tools/reverse-salary/)

## 収束回数の考察

探索範囲が `[targetTakeHome, targetTakeHome * 2]` のとき、最大探索幅は `targetTakeHome`。精度 ±1,000円に収束するまでの反復回数は:

```
log2(targetTakeHome / 1000)
```

手取り500万の場合:
- 探索幅: 500万
- 反復回数: log2(5,000,000 / 1,000) = log2(5000) ≒ **13回**

たった13回のループで±1,000円の精度が出る。O(log n) の威力。

```javascript
// 収束の様子を可視化
function reverseCalcWithLog(targetTakeHome) {
  let low = targetTakeHome;
  let high = targetTakeHome * 2;
  let iteration = 0;

  while (high - low > 1000) {
    const mid = Math.floor((low + high) / 2);
    const takeHome = calcTakeHome(mid);
    const diff = takeHome - targetTakeHome;
    iteration++;

    console.log(
      `#${iteration}: 年収=${mid.toLocaleString()}円`,
      `手取り=${takeHome.toLocaleString()}円`,
      `差=${diff > 0 ? '+' : ''}${diff.toLocaleString()}円`,
      `範囲=${(high - low).toLocaleString()}円`
    );

    if (takeHome < targetTakeHome) low = mid;
    else high = mid;
  }

  return high;
}

// 実行結果の例（手取り500万）:
// #1:  年収=7,500,000円 手取り=5,693,xxx円 差=+693,xxx円 範囲=2,500,000円
// #2:  年収=6,250,000円 手取り=4,821,xxx円 差=-178,xxx円 範囲=1,250,000円
// ...
// #13: 年収=6,732,000円 手取り=5,000,xxx円 差=+xxx円    範囲=976円
```

## フリーランス版 — 単価からの逆算

フリーランスの場合、「手取り目標」から「月額単価」を逆算したいことが多い。

```javascript
function reverseCalcFreelanceRate(targetTakeHome, options = {}) {
  const {
    workingDays = 20,      // 月稼働日数
    expenseRate = 0.3,     // 経費率
    isFreeFromInvoice = false, // 免税事業者か
  } = options;

  // Step 1: 手取り → 必要売上（年間）
  let low = targetTakeHome;
  let high = targetTakeHome * 3; // フリーランスは税負担が大きい

  while (high - low > 1000) {
    const mid = Math.floor((low + high) / 2);
    const expenses = mid * expenseRate;
    const income = mid - expenses;
    const tax = calcFreelanceTax(income, { isFreeFromInvoice });
    const takeHome = mid - expenses - tax;

    if (takeHome < targetTakeHome) low = mid;
    else high = mid;
  }

  // Step 2: 年間売上 → 月額単価
  const annualRevenue = high;
  const monthlyRate = Math.ceil(annualRevenue / 12 / workingDays / 1000) * 1000;

  return {
    annualRevenue,
    monthlyRate: Math.ceil(annualRevenue / 12),
    dailyRate: monthlyRate,
  };
}
```

実装: [フリーランス単価計算ツール](https://and-tools.net/tools/freelance-rate-calc/)

## 探索範囲の初期値をどう決めるか

二分探索の性能は初期範囲に依存する。範囲が広すぎると無駄な反復が増え、狭すぎると解が範囲外になる。

```javascript
function estimateSearchRange(targetTakeHome) {
  // 実効税率の概算から初期範囲を絞る
  // 年収300万: 実効負担率 ≒ 20% → 手取り率80%
  // 年収600万: 実効負担率 ≒ 25% → 手取り率75%
  // 年収1000万: 実効負担率 ≒ 30% → 手取り率70%

  const estimatedLow = Math.floor(targetTakeHome / 0.85);   // 保守的な下限
  const estimatedHigh = Math.ceil(targetTakeHome / 0.60);    // 余裕のある上限

  return { low: estimatedLow, high: estimatedHigh };
}
```

ただし、実際のプロダクションコードでは**安全マージンを取って広めに設定**している。13回が16回になっても体感速度は変わらない。

## ニュートン法は使えるか？

数値解法としてニュートン法も候補に上がる。収束が2次なので、二分探索より速い。

```javascript
function reverseCalcNewton(targetTakeHome) {
  let x = targetTakeHome * 1.3; // 初期推定
  const h = 100; // 数値微分の刻み幅

  for (let i = 0; i < 20; i++) {
    const fx = calcTakeHome(x) - targetTakeHome;
    const fxh = calcTakeHome(x + h) - targetTakeHome;
    const derivative = (fxh - fx) / h;

    if (Math.abs(derivative) < 1e-10) break;
    x = x - fx / derivative;

    if (Math.abs(fx) < 1000) break;
  }

  return Math.ceil(x);
}
```

結論: **二分探索のほうが安全**。理由は:
- 階段関数（標準報酬月額テーブル）があるので、微分が不連続点でゼロになる
- 収束保証がある（二分探索は必ず収束するが、ニュートン法は発散のリスクがある）
- 13回で十分な精度が出るので、速度の差は無視できる

実際に [AND TOOLSの手取り計算ツール](https://and-tools.net/tools/take-home-pay/) では二分探索を採用している。

## まとめ

1. **税金の逆算は非線形方程式の求根問題**。閉じた形の逆関数は書けない
2. **二分探索で解くのが最適解**。単調増加の性質を利用する
3. **13回程度の反復で±1,000円の精度**が出る。計算量は O(log n)
4. **ニュートン法は不連続点の問題**があるので、二分探索のほうが安全
5. **初期範囲は広めに取って問題ない**。数回の無駄は誤差の範囲

## AND TOOLSについて

[AND TOOLS](https://and-tools.net/) では、フリーランス・個人事業主向けの税金計算ツールを34個以上無料で公開しています。この記事で紹介した計算ロジックを実際に試せます。
