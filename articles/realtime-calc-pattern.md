---
title: "リアルタイム計算UIの設計パターン — debounceは必要か？"
emoji: "⚡"
type: "tech"
topics: ["javascript", "performance", "ui", "個人開発"]
published: false
---

[AND TOOLS](https://and-tools.net/) の計算ツールは、スライダーを動かすと**即座に結果が更新**される。入力のたびに税金計算・社会保険料計算・ローン計算が走る。このリアルタイム計算UIの設計判断を解説する。

## 入力即計算 vs ボタン押下計算

計算UIには大きく2つのパターンがある。

| パターン | メリット | デメリット |
|---------|---------|----------|
| 入力即計算 | 直感的、探索しやすい | 重い計算だとカクつく |
| ボタン押下 | 実装がシンプル | 毎回ボタンを押す手間 |

AND TOOLSでは**入力即計算**を採用している。税金計算ツールの主な使い方は「年収を少しずつ変えて手取りの変化を見る」こと。ボタンを毎回押すのは体験として悪い。

## 即計算の基本実装

```javascript
function setupRealtimeCalc() {
  const inputs = document.querySelectorAll('.calcInput');
  const resultArea = document.getElementById('result');

  function calculate() {
    const salary = parseInt(document.getElementById('salary').value, 10) || 0;
    const age = parseInt(document.getElementById('age').value, 10) || 30;
    const dependents = parseInt(document.getElementById('dependents').value, 10) || 0;

    const result = calcTakeHomePay(salary, age, dependents);
    renderResult(resultArea, result);
  }

  inputs.forEach(input => {
    input.addEventListener('input', calculate);
  });

  // 初期計算
  calculate();
}
```

これで十分動く。では debounce は不要なのか？

## debounce が必要なケースと不要なケース

### 不要: 計算が軽い場合

AND TOOLSの税金計算は**1回あたり0.01ms以下**で完了する。毎秒60回（スライダーの更新頻度）呼んでも余裕。

```javascript
// パフォーマンス計測
function benchmark() {
  const start = performance.now();
  for (let i = 0; i < 10000; i++) {
    calcTakeHomePay(5000000 + i * 100, 35, 0);
  }
  const end = performance.now();
  console.log(`10,000回の計算: ${(end - start).toFixed(2)}ms`);
  console.log(`1回あたり: ${((end - start) / 10000).toFixed(4)}ms`);
}

benchmark();
// 10,000回の計算: 15.23ms
// 1回あたり: 0.0015ms
```

この程度なら debounce は**オーバーエンジニアリング**。遅延が入るとスライダーの追従が遅く感じ、むしろ体験が悪化する。

### 必要: DOM更新が重い場合

計算自体は軽くても、結果の**DOM更新が重い**場合は debounce が有効。

```javascript
// NG: 毎回DOMを大量に書き換える
function renderResultHeavy(container, result) {
  // innerHTML で全体を書き換え → レイアウト再計算が走る
  container.innerHTML = `
    <div class="resultCard">
      <h3>手取り額</h3>
      <p class="amount">${result.takeHome.toLocaleString()}円</p>
      <div class="breakdown">
        <div>所得税: ${result.incomeTax.toLocaleString()}円</div>
        <div>住民税: ${result.residentTax.toLocaleString()}円</div>
        <div>社会保険料: ${result.socialInsurance.toLocaleString()}円</div>
      </div>
      <!-- さらに20項目の詳細... -->
    </div>
  `;
}

// OK: textContentだけ更新する
function renderResultLight(result) {
  document.getElementById('takeHome').textContent =
    result.takeHome.toLocaleString() + '円';
  document.getElementById('incomeTax').textContent =
    result.incomeTax.toLocaleString() + '円';
  document.getElementById('residentTax').textContent =
    result.residentTax.toLocaleString() + '円';
  document.getElementById('socialIns').textContent =
    result.socialInsurance.toLocaleString() + '円';
}
```

`textContent` の更新はレイアウト再計算を最小限に抑えられる。`innerHTML` の全書き換えより**10倍以上速い**。

## debounce / throttle の実装

必要になった場合の実装。

```javascript
function debounce(fn, delay) {
  let timer = null;
  return function (...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
  };
}

function throttle(fn, interval) {
  let lastTime = 0;
  let timer = null;
  return function (...args) {
    const now = Date.now();
    const remaining = interval - (now - lastTime);

    clearTimeout(timer);

    if (remaining <= 0) {
      lastTime = now;
      fn.apply(this, args);
    } else {
      timer = setTimeout(() => {
        lastTime = Date.now();
        fn.apply(this, args);
      }, remaining);
    }
  };
}
```

使い分けの判断基準:

| 手法 | 挙動 | 向いている場面 |
|------|------|--------------|
| debounce | 入力が止まってからN ms後に実行 | テキスト入力（タイピング完了を待つ） |
| throttle | N msに1回だけ実行 | スライダー（定期的に更新したい） |
| なし | 毎回実行 | 計算が軽い場合（推奨） |

```javascript
// テキスト入力にはdebounce
const debouncedCalc = debounce(calculate, 300);
textInput.addEventListener('input', debouncedCalc);

// スライダーにはthrottleまたは生のinput
slider.addEventListener('input', calculate); // 軽ければそのまま
```

## 重い計算でのWeb Worker活用

ローンのシミュレーションなど、**数千回のループ計算**が必要な場合はWeb Workerが選択肢に入る。

```javascript
// worker.js
self.addEventListener('message', (e) => {
  const { principal, rate, years } = e.data;

  // 重い計算: 毎月の返済シミュレーション
  const monthly = calcMonthlyPayment(principal, rate, years);
  const schedule = [];
  let remaining = principal;

  for (let month = 1; month <= years * 12; month++) {
    const interest = Math.floor(remaining * rate / 12);
    const principalPart = monthly - interest;
    remaining = Math.max(remaining - principalPart, 0);
    schedule.push({ month, payment: monthly, interest, principalPart, remaining });
  }

  self.postMessage({ monthly, schedule });
});

function calcMonthlyPayment(principal, annualRate, years) {
  const monthlyRate = annualRate / 12;
  const payments = years * 12;
  return Math.floor(
    principal * monthlyRate * Math.pow(1 + monthlyRate, payments)
    / (Math.pow(1 + monthlyRate, payments) - 1)
  );
}
```

```javascript
// メインスレッド
const worker = new Worker('worker.js');

function calculateWithWorker(principal, rate, years) {
  worker.postMessage({ principal, rate, years });
}

worker.addEventListener('message', (e) => {
  const { monthly, schedule } = e.data;
  renderResult(monthly, schedule);
});

// スライダーの入力 → Worker に投げる
slider.addEventListener('input', () => {
  calculateWithWorker(
    parseInt(principalInput.value, 10),
    parseFloat(rateInput.value),
    parseInt(yearsInput.value, 10)
  );
});
```

Web Workerのメリット:
- メインスレッドがブロックされない → スライダーがカクつかない
- 計算中もUIが反応する
- 計算が重いほど効果が大きい

デメリット:
- DOM操作ができない（結果をpostMessageで返す必要あり）
- ファイルが分離される（bundlerの設定が必要）
- 軽い計算には過剰

## requestAnimationFrame による最適化

もうひとつのアプローチは `requestAnimationFrame` で更新を画面描画に同期させること。

```javascript
function setupRAFCalc() {
  let pendingUpdate = false;
  let latestValues = {};

  function scheduleUpdate() {
    if (!pendingUpdate) {
      pendingUpdate = true;
      requestAnimationFrame(() => {
        calculate(latestValues);
        pendingUpdate = false;
      });
    }
  }

  document.querySelectorAll('.calcInput').forEach(input => {
    input.addEventListener('input', () => {
      latestValues = gatherInputValues();
      scheduleUpdate();
    });
  });
}
```

この方式なら:
- 画面のリフレッシュレート（通常60fps）に同期
- 1フレーム内に複数回inputイベントが来ても計算は1回だけ
- throttle + debounce の中間的な動作

## AND TOOLSでの実際の判断

```
計算ロジック: 0.01ms以下 → debounce不要
DOM更新: textContentのみ → innerHTML避ける
入力方式: スライダー中心 → 即時更新がUXに直結
結論: debounceなしの即時計算
```

ローンシミュレーターだけ、返済スケジュール表（数百行）のDOM生成が重いため、`requestAnimationFrame` で更新を間引いている。

## まとめ

- **計算が軽いなら debounce は不要**。スライダーの追従性を殺す
- **DOM更新を軽くする**ほうが debounce より効果的（innerHTML → textContent）
- 重い計算は**Web Worker**、DOM更新の間引きは **requestAnimationFrame**
- 判断基準: 1回の計算+描画が **16ms以下**なら即時更新で問題ない

## 関連ツール・サイト

- [AND TOOLS](https://and-tools.net/) — 34本すべて即時計算UI
- [ToolShare Lab](https://webatives.com/tools/) — 同様のリアルタイム計算パターンを採用

「ボタンを押す」という1アクションの省略が、ツールの使い勝手を大きく変える。
