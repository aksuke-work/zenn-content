---
title: "ゲージバーUIをCSS+JSで実装する — 経費率の5段階判定表示"
emoji: "📏"
type: "tech"
topics: ["css", "javascript", "ui", "個人開発"]
published: false
---

数値データを直感的に伝えるUIとして、ゲージバー（プログレスバー）は定番。[AND TOOLS](https://and-tools.net/) の経費率判定ツールで実装した、CSS linear-gradient による5段階色分け + JS による動的制御 + アニメーション付きの値変化を解説する。

## 完成イメージ

経費率（0%〜100%）を入力すると、ゲージバーの位置と色が変わり、「適正」「やや高め」「要注意」などの判定ラベルが表示される。

- 0-20%: 青（低め）
- 20-40%: 緑（適正）
- 40-60%: 黄（やや高め）
- 60-80%: オレンジ（高め）
- 80-100%: 赤（要注意）

## CSSでゲージバーの土台を作る

```css
.gaugeBar {
  position: relative;
  width: 100%;
  height: 28px;
  border-radius: 14px;
  background: linear-gradient(
    to right,
    #3b82f6 0%,      /* 青: 低め */
    #22c55e 25%,      /* 緑: 適正 */
    #eab308 50%,      /* 黄: やや高め */
    #f97316 75%,      /* オレンジ: 高め */
    #ef4444 100%      /* 赤: 要注意 */
  );
  overflow: visible;
  box-shadow: inset 0 2px 4px rgba(0, 0, 0, 0.1);
}

.gaugeBar__track {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  border-radius: 14px;
}

.gaugeBar__marker {
  position: absolute;
  top: 50%;
  transform: translate(-50%, -50%);
  width: 20px;
  height: 36px;
  background: #1e293b;
  border: 3px solid #fff;
  border-radius: 6px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.25);
  transition: left 0.5s cubic-bezier(0.34, 1.56, 0.64, 1);
  z-index: 2;
}

.gaugeBar__valueLabel {
  position: absolute;
  top: -32px;
  transform: translateX(-50%);
  font-size: 14px;
  font-weight: 700;
  white-space: nowrap;
  transition: left 0.5s cubic-bezier(0.34, 1.56, 0.64, 1),
              color 0.3s ease;
  z-index: 3;
}

.gaugeBar__judgmentLabel {
  position: absolute;
  bottom: -28px;
  transform: translateX(-50%);
  font-size: 13px;
  font-weight: 600;
  padding: 2px 10px;
  border-radius: 4px;
  white-space: nowrap;
  transition: left 0.5s cubic-bezier(0.34, 1.56, 0.64, 1),
              background-color 0.3s ease;
  z-index: 3;
}
```

`cubic-bezier(0.34, 1.56, 0.64, 1)` はバウンスのようなオーバーシュートを持つイージング。マーカーの移動が気持ちいい動きになる。

## 目盛りラベルの追加

ゲージバーの下に目盛りラベルを配置して、基準値をわかりやすくする。

```html
<div class="gaugeBar">
  <div class="gaugeBar__track"></div>
  <div class="gaugeBar__marker"></div>
  <span class="gaugeBar__valueLabel">35%</span>
  <span class="gaugeBar__judgmentLabel">適正</span>

  <div class="gaugeBar__scale">
    <span class="gaugeBar__scaleItem" style="left: 0%">0%</span>
    <span class="gaugeBar__scaleItem" style="left: 25%">25%</span>
    <span class="gaugeBar__scaleItem" style="left: 50%">50%</span>
    <span class="gaugeBar__scaleItem" style="left: 75%">75%</span>
    <span class="gaugeBar__scaleItem" style="left: 100%">100%</span>
  </div>
</div>
```

```css
.gaugeBar__scale {
  position: absolute;
  bottom: -52px;
  left: 0;
  width: 100%;
}

.gaugeBar__scaleItem {
  position: absolute;
  transform: translateX(-50%);
  font-size: 11px;
  color: #94a3b8;
}
```

## JSで値に応じた動的制御

```javascript
class GaugeBar {
  constructor(el) {
    this.el = el;
    this.marker = el.querySelector('.gaugeBar__marker');
    this.valueLabel = el.querySelector('.gaugeBar__valueLabel');
    this.judgmentLabel = el.querySelector('.gaugeBar__judgmentLabel');
  }

  update(rate, industryKey) {
    const position = this.calcPosition(rate);
    const judgment = this.judge(rate, industryKey);

    // マーカー位置
    this.marker.style.left = `${position}%`;
    this.valueLabel.style.left = `${position}%`;
    this.judgmentLabel.style.left = `${position}%`;

    // 値ラベル
    this.valueLabel.textContent = `${rate}%`;
    this.valueLabel.style.color = judgment.color;

    // 判定ラベル
    this.judgmentLabel.textContent = judgment.label;
    this.judgmentLabel.style.backgroundColor = judgment.color;
    this.judgmentLabel.style.color = '#fff';
  }

  calcPosition(rate) {
    // 0-100%の範囲にクランプ
    return Math.max(0, Math.min(100, rate));
  }

  judge(rate, industryKey) {
    // 業種別の判定閾値
    const thresholds = this.getThresholds(industryKey);

    if (rate <= thresholds.low) {
      return { level: 1, label: 'やや低め', color: '#3b82f6' };
    }
    if (rate <= thresholds.typical) {
      return { level: 2, label: '適正', color: '#22c55e' };
    }
    if (rate <= thresholds.high) {
      return { level: 3, label: 'やや高め', color: '#eab308' };
    }
    if (rate <= thresholds.warning) {
      return { level: 4, label: '高め', color: '#f97316' };
    }
    return { level: 5, label: '要注意', color: '#ef4444' };
  }

  getThresholds(industryKey) {
    const presets = {
      webEngineer:  { low: 15, typical: 30, high: 50, warning: 60 },
      webDesigner:  { low: 20, typical: 35, high: 55, warning: 65 },
      writer:       { low: 10, typical: 25, high: 45, warning: 55 },
      photographer: { low: 25, typical: 45, high: 65, warning: 75 },
      restaurant:   { low: 50, typical: 65, high: 80, warning: 85 },
    };
    return presets[industryKey] || presets.webEngineer;
  }
}
```

## 入力と連動させる

フォームの入力値が変わるたびにゲージバーを更新する。

```javascript
document.addEventListener('DOMContentLoaded', () => {
  const gaugeEl = document.querySelector('.gaugeBar');
  const gauge = new GaugeBar(gaugeEl);

  const revenueInput = document.getElementById('revenue');
  const expensesInput = document.getElementById('expenses');
  const industrySelect = document.getElementById('industry');

  function onInputChange() {
    const revenue = parseFloat(revenueInput.value) || 0;
    const expenses = parseFloat(expensesInput.value) || 0;
    const industry = industrySelect.value;

    if (revenue <= 0) return;

    const rate = Math.round((expenses / revenue) * 1000) / 10;
    gauge.update(rate, industry);
  }

  revenueInput.addEventListener('input', onInputChange);
  expensesInput.addEventListener('input', onInputChange);
  industrySelect.addEventListener('change', onInputChange);
});
```

## 数値のカウントアップアニメーション

値が変わったとき、数字がカウントアップ（ダウン）するアニメーションを付ける。

```javascript
function animateValue(el, from, to, duration = 500) {
  const startTime = performance.now();
  const diff = to - from;

  function tick(currentTime) {
    const elapsed = currentTime - startTime;
    const progress = Math.min(elapsed / duration, 1);

    // easeOutCubic
    const eased = 1 - Math.pow(1 - progress, 3);
    const current = from + diff * eased;

    el.textContent = `${current.toFixed(1)}%`;

    if (progress < 1) {
      requestAnimationFrame(tick);
    }
  }

  requestAnimationFrame(tick);
}
```

`GaugeBar.update` に組み込む:

```javascript
update(rate, industryKey) {
  const position = this.calcPosition(rate);
  const judgment = this.judge(rate, industryKey);

  this.marker.style.left = `${position}%`;
  this.valueLabel.style.left = `${position}%`;
  this.judgmentLabel.style.left = `${position}%`;

  // カウントアップアニメーション
  const prevRate = parseFloat(this.valueLabel.textContent) || 0;
  animateValue(this.valueLabel, prevRate, rate);

  this.valueLabel.style.color = judgment.color;
  this.judgmentLabel.textContent = judgment.label;
  this.judgmentLabel.style.backgroundColor = judgment.color;
  this.judgmentLabel.style.color = '#fff';
}
```

## レスポンシブ対応

モバイルではゲージバーの高さを小さくし、ラベル位置を調整する。

```css
@media (max-width: 640px) {
  .gaugeBar {
    height: 20px;
    border-radius: 10px;
  }

  .gaugeBar__marker {
    width: 16px;
    height: 28px;
    border-radius: 4px;
  }

  .gaugeBar__valueLabel {
    top: -28px;
    font-size: 12px;
  }

  .gaugeBar__judgmentLabel {
    bottom: -24px;
    font-size: 11px;
    padding: 1px 8px;
  }

  .gaugeBar__scaleItem {
    font-size: 10px;
  }

  .gaugeBar__scale {
    bottom: -44px;
  }
}
```

## 業種に応じたグラデーション範囲の変更

業種によって「適正範囲」が異なるため、グラデーションの色分け位置を動的に変更する。

```javascript
updateGradient(industryKey) {
  const thresholds = this.getThresholds(industryKey);

  // 閾値を0-100%のパーセンテージに変換
  const max = thresholds.warning * 1.3; // 余裕を持たせる
  const lowPct = (thresholds.low / max * 100).toFixed(1);
  const typicalPct = (thresholds.typical / max * 100).toFixed(1);
  const highPct = (thresholds.high / max * 100).toFixed(1);
  const warningPct = (thresholds.warning / max * 100).toFixed(1);

  this.el.style.background = `linear-gradient(
    to right,
    #3b82f6 0%,
    #3b82f6 ${lowPct}%,
    #22c55e ${lowPct}%,
    #22c55e ${typicalPct}%,
    #eab308 ${typicalPct}%,
    #eab308 ${highPct}%,
    #f97316 ${highPct}%,
    #f97316 ${warningPct}%,
    #ef4444 ${warningPct}%,
    #ef4444 100%
  )`;
}
```

これにより、ITエンジニア（適正30%）と飲食業（適正65%）で色分けの位置が変わる。「緑の範囲がどこまでか」が一目でわかるUIになる。

## アクセシビリティ

ゲージバーは視覚的なUIなので、スクリーンリーダー向けのテキスト情報も必ず付ける。

```html
<div class="gaugeBar"
     role="meter"
     aria-valuenow="35"
     aria-valuemin="0"
     aria-valuemax="100"
     aria-label="経費率ゲージ: 35%（判定: 適正）">
  <!-- 視覚的なUI -->
</div>
```

```javascript
update(rate, industryKey) {
  // ... 既存の処理 ...

  // アクセシビリティ属性の更新
  this.el.setAttribute('aria-valuenow', rate);
  this.el.setAttribute('aria-label',
    `経費率ゲージ: ${rate}%（判定: ${judgment.label}）`
  );
}
```

## まとめ

1. **CSS linear-gradient** でゲージバーの色分けを実現。滑らかなグラデーションと明確な段階分けの2パターン
2. **cubic-bezierイージング**でマーカー移動に気持ちいいバウンスを追加
3. **カウントアップアニメーション**で数値変化を視覚的にフィードバック
4. **業種に応じたグラデーション範囲の変更**で「自分の業種での適正位置」が直感的にわかる
5. **`role="meter"` + `aria-valuenow`** でスクリーンリーダー対応も忘れずに

---

## 関連サイト

ゲージバーUIの実装例はこちらで確認できます。

- [経費率 判定ツール](https://and-tools.net/tools/expense-rate/) — 業種別の5段階判定ゲージバーが実際に稼働
- [AND TOOLS](https://and-tools.net/) — フリーランス向け無料計算ツール集
- [ToolShare Lab](https://webatives.com/tools/) — 150個以上の無料ツールを公開中
