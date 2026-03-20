---
title: "スライダーとテキスト入力を双方向同期するUIパターン"
emoji: "🎚️"
type: "tech"
topics: ["javascript", "ui", "ux", "個人開発"]
published: false
---

[AND TOOLS](https://and-tools.net/) の計算ツールでは、ほぼすべての入力項目に**スライダー（range）とテキスト入力（number）の双方向同期UI**を採用している。直感的に操作でき、かつ正確な値も入力できる。この実装パターンを解説する。

## 基本の双方向バインディング

最小限の実装はこうなる。

```javascript
function setupSync(sliderId, inputId) {
  const slider = document.getElementById(sliderId);
  const input = document.getElementById(inputId);

  slider.addEventListener('input', () => {
    input.value = slider.value;
  });

  input.addEventListener('input', () => {
    slider.value = input.value;
  });
}
```

HTMLはこう:

```html
<input type="range" id="salarySlider" min="0" max="20000000" step="10000" value="5000000">
<input type="number" id="salaryInput" min="0" max="20000000" step="10000" value="5000000">
```

これだけで動くが、実用レベルにするにはいくつかの課題がある。

## 課題1: min/max制約の同期

テキスト入力に範囲外の値を入れられると、スライダーがおかしな位置に飛ぶ。

```javascript
function setupSyncWithClamp(sliderId, inputId, options = {}) {
  const slider = document.getElementById(sliderId);
  const input = document.getElementById(inputId);
  const { min = 0, max = 20000000, step = 10000, onChange } = options;

  function clamp(value) {
    const num = parseInt(value, 10);
    if (isNaN(num)) return min;
    return Math.max(min, Math.min(max, num));
  }

  function roundToStep(value) {
    return Math.round(value / step) * step;
  }

  slider.addEventListener('input', () => {
    const value = clamp(slider.value);
    input.value = value;
    if (onChange) onChange(value);
  });

  input.addEventListener('input', () => {
    const value = clamp(input.value);
    slider.value = value;
    if (onChange) onChange(value);
  });

  // フォーカスを外したときにstepに丸める
  input.addEventListener('blur', () => {
    const value = roundToStep(clamp(input.value));
    input.value = value;
    slider.value = value;
    if (onChange) onChange(value);
  });
}
```

ポイント:
- `clamp` で範囲外の値を制約内に収める
- `blur` イベントでstep単位に丸める（入力中は自由に打てるようにする）
- `isNaN` チェックで空文字やゴミ値を防ぐ

## 課題2: 日本円フォーマット表示

数値そのままだと `5000000` と表示され、パッと見でいくらかわからない。カンマ区切り + 「円」表示が必要だ。

```javascript
function formatYen(value) {
  return value.toLocaleString('ja-JP') + '円';
}

function setupSyncWithFormat(sliderId, inputId, displayId, options = {}) {
  const slider = document.getElementById(sliderId);
  const input = document.getElementById(inputId);
  const display = document.getElementById(displayId);
  const { min = 0, max = 20000000, step = 10000, onChange } = options;

  function update(value) {
    const clamped = Math.max(min, Math.min(max, parseInt(value, 10) || min));
    slider.value = clamped;
    input.value = clamped;
    display.textContent = formatYen(clamped);
    if (onChange) onChange(clamped);
  }

  slider.addEventListener('input', () => update(slider.value));
  input.addEventListener('input', () => update(input.value));
  input.addEventListener('blur', () => {
    const rounded = Math.round((parseInt(input.value, 10) || min) / step) * step;
    update(rounded);
  });

  // 初期表示
  update(slider.value);
}
```

HTML:

```html
<div class="inputGroup">
  <input type="range" id="salarySlider" min="0" max="20000000" step="10000" value="5000000">
  <input type="number" id="salaryInput" min="0" max="20000000" step="10000" value="5000000">
  <span id="salaryDisplay">5,000,000円</span>
</div>
```

## 課題3: スライダーの非線形スケール

年収の入力で `min=0` `max=20000000` だと、0〜200万のあたりが極端に狭くなる。100万円単位で微調整したいのに、スライダーが小さな範囲に密集してしまう。

対数スケールを使うと低い値の操作性が上がる。

```javascript
function setupLogScaleSync(sliderId, inputId, displayId, options = {}) {
  const slider = document.getElementById(sliderId);
  const input = document.getElementById(inputId);
  const display = document.getElementById(displayId);
  const { min = 100000, max = 20000000, step = 10000, onChange } = options;

  // 対数変換: スライダー値(0-1000) → 実際の値
  const logMin = Math.log(min);
  const logMax = Math.log(max);
  const SLIDER_MAX = 1000;

  function sliderToValue(sliderVal) {
    const logVal = logMin + (logMax - logMin) * (sliderVal / SLIDER_MAX);
    const raw = Math.exp(logVal);
    return Math.round(raw / step) * step;
  }

  function valueToSlider(value) {
    const logVal = Math.log(Math.max(value, min));
    return Math.round((logVal - logMin) / (logMax - logMin) * SLIDER_MAX);
  }

  function update(value) {
    const clamped = Math.max(min, Math.min(max, value));
    slider.value = valueToSlider(clamped);
    input.value = clamped;
    display.textContent = formatYen(clamped);
    if (onChange) onChange(clamped);
  }

  slider.addEventListener('input', () => {
    update(sliderToValue(parseInt(slider.value, 10)));
  });

  input.addEventListener('input', () => {
    update(parseInt(input.value, 10) || min);
  });

  // スライダーのmin/maxを対数スケール用に設定
  slider.min = 0;
  slider.max = SLIDER_MAX;
  slider.step = 1;

  update(parseInt(input.value, 10) || min);
}
```

対数スケールにすると:
- 年収100万〜500万の範囲がスライダーの広い領域を使える
- 年収1,000万〜2,000万は狭くなるが、このあたりはテキスト入力で調整
- ユーザーは低い年収でも微調整しやすい

## 課題4: モバイルでの操作性

スマホでのスライダー操作には固有の問題がある。

```javascript
function setupMobileOptimized(sliderId, inputId, displayId, options = {}) {
  const slider = document.getElementById(sliderId);
  const input = document.getElementById(inputId);
  const display = document.getElementById(displayId);
  const { min = 0, max = 20000000, step = 10000, onChange } = options;

  function clamp(val) {
    return Math.max(min, Math.min(max, parseInt(val, 10) || min));
  }

  function update(value) {
    const clamped = clamp(value);
    slider.value = clamped;
    input.value = clamped;
    display.textContent = formatYen(clamped);
    if (onChange) onChange(clamped);
  }

  // スライダー: inputイベント（リアルタイム更新）
  slider.addEventListener('input', () => update(slider.value));

  // テキスト入力: changeイベント（確定時のみ更新）
  input.addEventListener('change', () => update(input.value));

  // モバイル: タッチ中はスクロールを防止
  slider.addEventListener('touchstart', (e) => {
    e.stopPropagation();
    document.body.style.overflow = 'hidden';
  }, { passive: false });

  slider.addEventListener('touchend', () => {
    document.body.style.overflow = '';
  });

  // テキスト入力: フォーカス時に全選択（上書きしやすくする）
  input.addEventListener('focus', () => {
    input.select();
  });

  update(slider.value);
}
```

モバイルのポイント:
- **スクロールとスライダーの操作が競合**する。`touchstart` でスクロールを一時停止
- テキスト入力では `input` ではなく `change` を使う（キーボードで打つたびに更新すると不便）
- `focus` 時に全選択すると、既存の値を消す手間が省ける

## 汎用クラスとして整理

実際のプロジェクトではクラスにまとめている。

```javascript
class SliderInputSync {
  constructor(container, options = {}) {
    this.slider = container.querySelector('[data-role="slider"]');
    this.input = container.querySelector('[data-role="input"]');
    this.display = container.querySelector('[data-role="display"]');

    this.min = options.min ?? 0;
    this.max = options.max ?? 10000000;
    this.step = options.step ?? 10000;
    this.onChange = options.onChange ?? null;

    this._bindEvents();
    this._update(parseInt(this.input.value, 10) || this.min);
  }

  _clamp(value) {
    const num = parseInt(value, 10);
    if (isNaN(num)) return this.min;
    return Math.max(this.min, Math.min(this.max, num));
  }

  _update(value) {
    const clamped = this._clamp(value);
    this.slider.value = clamped;
    this.input.value = clamped;
    if (this.display) {
      this.display.textContent = clamped.toLocaleString('ja-JP') + '円';
    }
    if (this.onChange) this.onChange(clamped);
  }

  _bindEvents() {
    this.slider.addEventListener('input', () => this._update(this.slider.value));
    this.input.addEventListener('input', () => this._update(this.input.value));
    this.input.addEventListener('blur', () => {
      const rounded = Math.round(this._clamp(this.input.value) / this.step) * this.step;
      this._update(rounded);
    });
    this.input.addEventListener('focus', () => this.input.select());
  }

  setValue(value) {
    this._update(value);
  }

  getValue() {
    return this._clamp(this.input.value);
  }
}

// 使い方
const container = document.querySelector('.salaryInput');
const sync = new SliderInputSync(container, {
  min: 0,
  max: 20000000,
  step: 10000,
  onChange: (value) => recalculate(value),
});
```

## まとめ

- **基本**は `input` イベントの双方向リスナーだけ
- **実用化**には clamp / step丸め / formatYen / モバイル対応が必要
- **非線形スケール**で低い値域の操作性を改善できる
- クラスにまとめておくと、ツールを量産するときに便利

## 関連ツール・サイト

- [AND TOOLS](https://and-tools.net/) — 34本の計算ツールすべてでこのUIパターンを使用
- [ToolShare Lab](https://webatives.com/tools/) — 別サイトのツール集でも同様のUIを採用

スライダー + テキスト入力の双方向同期は「計算ツール」のUXを大きく左右する。実装は地味だが、ここを丁寧に作ると使い勝手が段違いになる。
