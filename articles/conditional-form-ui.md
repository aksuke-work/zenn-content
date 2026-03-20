---
title: "条件分岐フォームの設計 — ラジオボタンで入力項目を動的切替"
emoji: "🔀"
type: "tech"
topics: ["javascript", "form", "ui", "webdev"]
published: false
---

「会社員」を選んだら給与入力欄を表示し、「フリーランス」を選んだら経費入力欄を表示する。こうした条件分岐フォームは税金計算ツールでは必須のパターン。[AND TOOLS](https://and-tools.net/) の副業税金シミュレーターで3モード切替、インボイス計算ツールで2モード切替を実装した経験から、設計と実装を解説する。

## 基本パターン — ラジオボタンで表示切替

最もシンプルな実装から始める。

```html
<form class="condForm">
  <fieldset class="condForm__modeSelect">
    <legend class="condForm__legend">働き方を選択</legend>
    <label class="condForm__radioLabel">
      <input type="radio" name="workStyle" value="employee"
             class="condForm__radio js-modeSwitch" checked>
      会社員
    </label>
    <label class="condForm__radioLabel">
      <input type="radio" name="workStyle" value="freelance"
             class="condForm__radio js-modeSwitch">
      フリーランス
    </label>
    <label class="condForm__radioLabel">
      <input type="radio" name="workStyle" value="sidejob"
             class="condForm__radio js-modeSwitch">
      会社員＋副業
    </label>
  </fieldset>

  <!-- 会社員モード -->
  <div class="condForm__section js-modeSection" data-mode="employee">
    <div class="condForm__inputGroup">
      <label class="condForm__label" for="salary">給与年収</label>
      <input class="condForm__input" type="number" id="salary" placeholder="5,000,000">
    </div>
    <div class="condForm__inputGroup">
      <label class="condForm__label" for="bonus">賞与</label>
      <input class="condForm__input" type="number" id="bonus" placeholder="1,000,000">
    </div>
  </div>

  <!-- フリーランスモード -->
  <div class="condForm__section js-modeSection" data-mode="freelance" hidden>
    <div class="condForm__inputGroup">
      <label class="condForm__label" for="revenue">売上</label>
      <input class="condForm__input" type="number" id="revenue" placeholder="8,000,000">
    </div>
    <div class="condForm__inputGroup">
      <label class="condForm__label" for="expenses">経費</label>
      <input class="condForm__input" type="number" id="expenses" placeholder="2,000,000">
    </div>
    <div class="condForm__inputGroup">
      <label class="condForm__label" for="blueDeduction">青色申告控除</label>
      <select class="condForm__select" id="blueDeduction">
        <option value="0">なし</option>
        <option value="100000">10万円</option>
        <option value="550000">55万円</option>
        <option value="650000" selected>65万円</option>
      </select>
    </div>
  </div>

  <!-- 会社員＋副業モード -->
  <div class="condForm__section js-modeSection" data-mode="sidejob" hidden>
    <div class="condForm__inputGroup">
      <label class="condForm__label" for="mainSalary">本業の給与年収</label>
      <input class="condForm__input" type="number" id="mainSalary" placeholder="5,000,000">
    </div>
    <div class="condForm__inputGroup">
      <label class="condForm__label" for="sideRevenue">副業の売上</label>
      <input class="condForm__input" type="number" id="sideRevenue" placeholder="1,000,000">
    </div>
    <div class="condForm__inputGroup">
      <label class="condForm__label" for="sideExpenses">副業の経費</label>
      <input class="condForm__input" type="number" id="sideExpenses" placeholder="200,000">
    </div>
  </div>
</form>
```

## display切替 vs classList — どちらを使うか

表示・非表示の切替には2つのアプローチがある。

### パターン1: hidden属性（推奨）

```javascript
function switchMode(mode) {
  const sections = document.querySelectorAll('.js-modeSection');

  sections.forEach(section => {
    if (section.dataset.mode === mode) {
      section.removeAttribute('hidden');
    } else {
      section.setAttribute('hidden', '');
    }
  });
}
```

メリット:
- `hidden` 属性は **アクセシビリティツリーからも除外** される
- 非表示セクション内の `required` 属性がバリデーションに影響しない
- セマンティックに正しい

### パターン2: classList

```javascript
function switchMode(mode) {
  const sections = document.querySelectorAll('.js-modeSection');

  sections.forEach(section => {
    section.classList.toggle(
      'condForm__section_state_hidden',
      section.dataset.mode !== mode
    );
  });
}
```

```css
.condForm__section_state_hidden {
  display: none;
}
```

こちらはアニメーションを付けたい場合に使う（後述）。

**結論**: アニメーション不要なら `hidden` 属性を使う。アニメーション付きなら `classList` で CSS transition と組み合わせる。

## JS実装 — イベントハンドリング

```javascript
class ConditionalForm {
  constructor(formEl) {
    this.form = formEl;
    this.switches = formEl.querySelectorAll('.js-modeSwitch');
    this.sections = formEl.querySelectorAll('.js-modeSection');

    this.init();
  }

  init() {
    this.switches.forEach(radio => {
      radio.addEventListener('change', (e) => {
        this.switchMode(e.target.value);
      });
    });

    // 初期状態の設定
    const checked = this.form.querySelector('.js-modeSwitch:checked');
    if (checked) {
      this.switchMode(checked.value);
    }
  }

  switchMode(mode) {
    this.sections.forEach(section => {
      const isTarget = section.dataset.mode === mode;

      if (isTarget) {
        this.showSection(section);
      } else {
        this.hideSection(section);
      }
    });

    // 非表示セクション内のinputをリセット（任意）
    this.clearHiddenInputs();
  }

  showSection(section) {
    section.removeAttribute('hidden');
    section.setAttribute('aria-hidden', 'false');

    // フォーカスを最初の入力欄に移動
    const firstInput = section.querySelector('input, select');
    if (firstInput) {
      setTimeout(() => firstInput.focus(), 100);
    }
  }

  hideSection(section) {
    section.setAttribute('hidden', '');
    section.setAttribute('aria-hidden', 'true');
  }

  clearHiddenInputs() {
    this.sections.forEach(section => {
      if (section.hasAttribute('hidden')) {
        const inputs = section.querySelectorAll('input, select');
        inputs.forEach(input => {
          if (input.type === 'number' || input.type === 'text') {
            input.value = '';
          }
        });
      }
    });
  }
}

// 初期化
document.addEventListener('DOMContentLoaded', () => {
  const form = document.querySelector('.condForm');
  if (form) new ConditionalForm(form);
});
```

## アニメーション付きの展開/折りたたみ

`hidden` 属性ではアニメーションが付かないので、CSS transition を使う。

```css
.condForm__section {
  max-height: 0;
  opacity: 0;
  overflow: hidden;
  transition: max-height 0.4s ease, opacity 0.3s ease, padding 0.3s ease;
  padding: 0 20px;
}

.condForm__section_state_active {
  max-height: 500px; /* 十分な値を設定 */
  opacity: 1;
  padding: 20px;
}
```

JS側:

```javascript
showSection(section) {
  // hidden属性を削除（アニメーション用）
  section.removeAttribute('hidden');

  // 次のフレームでクラスを付与（transition発火のため）
  requestAnimationFrame(() => {
    section.classList.add('condForm__section_state_active');
  });

  section.setAttribute('aria-hidden', 'false');
}

hideSection(section) {
  section.classList.remove('condForm__section_state_active');
  section.setAttribute('aria-hidden', 'true');

  // transitionend後にhiddenを付与
  section.addEventListener('transitionend', function handler() {
    if (!section.classList.contains('condForm__section_state_active')) {
      section.setAttribute('hidden', '');
    }
    section.removeEventListener('transitionend', handler);
  });
}
```

`max-height` のトランジションは `height: auto` が使えない問題の定番ワークアラウンド。`500px` は内容が収まる十分な値を設定する。値が小さすぎると内容が切れ、大きすぎるとアニメーション速度が不自然になる。

## フォームバリデーションとの連携

条件分岐フォームでのバリデーションの注意点。

### 問題: 非表示セクションの required

```html
<!-- フリーランスモードが非表示なのに required がある -->
<div hidden>
  <input type="number" id="revenue" required>
</div>
```

`hidden` 属性が付いた要素の中の `required` はブラウザのバリデーションをブロックしない（送信を妨げない）。しかし **カスタムバリデーション** を使う場合は明示的にスキップする必要がある。

```javascript
function validateForm(formEl) {
  const errors = [];
  const activeSection = formEl.querySelector('.js-modeSection:not([hidden])');

  if (!activeSection) return errors;

  // アクティブなセクション内のinputだけバリデーション
  const inputs = activeSection.querySelectorAll('input[required], select[required]');

  inputs.forEach(input => {
    if (!input.value || input.value.trim() === '') {
      errors.push({
        field: input.id,
        message: `${input.previousElementSibling?.textContent || input.id}を入力してください`,
      });
    }

    if (input.type === 'number' && input.value) {
      const num = parseFloat(input.value);
      if (isNaN(num) || num < 0) {
        errors.push({
          field: input.id,
          message: `${input.previousElementSibling?.textContent || input.id}は0以上の数値を入力してください`,
        });
      }
    }
  });

  return errors;
}
```

### エラー表示

```javascript
function showErrors(errors) {
  // 既存のエラーをクリア
  document.querySelectorAll('.condForm__error').forEach(el => el.remove());
  document.querySelectorAll('.condForm__input_state_error').forEach(el => {
    el.classList.remove('condForm__input_state_error');
  });

  errors.forEach(error => {
    const input = document.getElementById(error.field);
    if (!input) return;

    input.classList.add('condForm__input_state_error');

    const errorEl = document.createElement('span');
    errorEl.className = 'condForm__error';
    errorEl.textContent = error.message;
    errorEl.setAttribute('role', 'alert');
    input.parentNode.appendChild(errorEl);
  });
}
```

```css
.condForm__input_state_error {
  border-color: #ef4444;
  background-color: #fef2f2;
}

.condForm__error {
  display: block;
  margin-top: 4px;
  font-size: 13px;
  color: #ef4444;
}
```

## トグルスイッチでの切替

ラジオボタンの代わりにトグルスイッチを使う場合。2モード切替に適している。

```html
<div class="toggleSwitch js-toggle" role="radiogroup" aria-label="計算モード">
  <button class="toggleSwitch__option toggleSwitch__option_state_active"
          role="radio" aria-checked="true" data-mode="before">
    インボイス登録前
  </button>
  <button class="toggleSwitch__option"
          role="radio" aria-checked="false" data-mode="after">
    インボイス登録後
  </button>
  <div class="toggleSwitch__slider"></div>
</div>
```

```css
.toggleSwitch {
  display: inline-flex;
  position: relative;
  background: #e2e8f0;
  border-radius: 8px;
  padding: 4px;
}

.toggleSwitch__option {
  position: relative;
  z-index: 1;
  padding: 8px 20px;
  border: none;
  background: none;
  font-size: 14px;
  font-weight: 600;
  color: #64748b;
  cursor: pointer;
  transition: color 0.3s ease;
}

.toggleSwitch__option_state_active {
  color: #1e293b;
}

.toggleSwitch__slider {
  position: absolute;
  top: 4px;
  left: 4px;
  width: calc(50% - 4px);
  height: calc(100% - 8px);
  background: #fff;
  border-radius: 6px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  transition: transform 0.3s cubic-bezier(0.34, 1.56, 0.64, 1);
}

/* 2番目のオプションが選択された場合 */
.toggleSwitch_state_right .toggleSwitch__slider {
  transform: translateX(100%);
}
```

```javascript
class ToggleSwitch {
  constructor(el, onSwitch) {
    this.el = el;
    this.options = el.querySelectorAll('.toggleSwitch__option');
    this.onSwitch = onSwitch;

    this.options.forEach(option => {
      option.addEventListener('click', () => this.select(option));
    });
  }

  select(option) {
    const mode = option.dataset.mode;
    const index = [...this.options].indexOf(option);

    // aria-checked更新
    this.options.forEach(opt => {
      opt.setAttribute('aria-checked', 'false');
      opt.classList.remove('toggleSwitch__option_state_active');
    });
    option.setAttribute('aria-checked', 'true');
    option.classList.add('toggleSwitch__option_state_active');

    // スライダー移動
    if (index > 0) {
      this.el.classList.add('toggleSwitch_state_right');
    } else {
      this.el.classList.remove('toggleSwitch_state_right');
    }

    // コールバック
    this.onSwitch(mode);
  }
}
```

## 実際のツールでの使用例

### 副業税金シミュレーター（3モード切替）

[AND TOOLSの副業税金シミュレーター](https://and-tools.net/tools/side-job-tax/) では「給与所得のみ」「事業所得のみ」「給与＋副業」の3モードを切り替える。各モードで必要な入力項目が異なり、計算ロジックも変わる。

### インボイス移行計算ツール（2モード切替）

[インボイス移行シミュレーター](https://and-tools.net/tools/invoice-transition/) では「登録前」「登録後」の2モードをトグルスイッチで切り替える。消費税の計算方法が変わるため、フォーム項目だけでなく結果表示のレイアウトも連動して変化する。

## まとめ

1. **表示切替は `hidden` 属性を基本に**。アクセシビリティツリーからも除外される
2. **アニメーション付きなら `classList` + CSS transition**。`max-height` のワークアラウンドで `height: auto` 問題を解決
3. **非表示セクションのバリデーションをスキップ**。アクティブなセクション内の入力だけ検証する
4. **2モード切替はトグルスイッチ**、3モード以上はラジオボタンが自然
5. **`aria-checked`、`aria-hidden`、`role="radiogroup"`** でアクセシビリティ担保

---

## 関連サイト

条件分岐フォームが実際に動いているサイトはこちら。

- [副業税金シミュレーター](https://and-tools.net/tools/side-job-tax/) — 3モード切替の実例。給与・事業・副業で入力項目と計算ロジックが変わる
- [インボイス移行シミュレーター](https://and-tools.net/tools/invoice-transition/) — 2モード切替のトグルスイッチ実装
- [AND TOOLS](https://and-tools.net/) — フリーランス向け無料計算ツール集
- [ToolShare Lab](https://webatives.com/) — 150個以上の無料ツールを公開中
