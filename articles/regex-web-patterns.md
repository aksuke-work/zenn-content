---
title: "Web制作で使う正規表現パターン20選 — バリデーション編"
emoji: "🔍"
type: "tech"
topics: ["javascript", "regex", "frontend", "html"]
published: false
---

フリーランス向けの計算ツールサイト [AND TOOLS](https://and-tools.net/) には、入力フォームが多い。メールアドレス、電話番号、郵便番号——それぞれのバリデーションを実装するたびに正規表現を調べ直していた。

この記事では、Web制作でよく使う正規表現パターンを20個まとめた。JavaScriptとHTML5のパターン属性の両方で使えるように解説する。

---

## 正規表現の基本確認

```javascript
// テスト
const pattern = /^[a-z]+$/;
pattern.test('hello'); // true
pattern.test('Hello'); // false

// マッチ
'hello world'.match(/\w+/g); // ['hello', 'world']

// 置換
'foo bar'.replace(/\s+/g, '-'); // 'foo-bar'
```

---

## メール・認証系

### 1. メールアドレス（実用的）

```javascript
const emailPattern = /^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$/;

// テスト
emailPattern.test('user@example.com');      // true
emailPattern.test('user+tag@example.co.jp'); // true
emailPattern.test('invalid@');               // false
```

```html
<input type="email" pattern="[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}">
```

### 2. パスワード（英数字8文字以上、大文字・小文字・数字各1文字以上）

```javascript
const passwordPattern = /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)[a-zA-Z\d!@#$%^&*]{8,}$/;

passwordPattern.test('Password1');   // true
passwordPattern.test('password');    // false（大文字・数字なし）
passwordPattern.test('Pass1');       // false（8文字未満）
```

### 3. パスワード（より厳格：記号必須）

```javascript
const strongPasswordPattern = /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[!@#$%^&*])[a-zA-Z\d!@#$%^&*]{10,}$/;
```

### 4. ユーザー名（英数字・アンダースコア・ハイフン、3〜20文字）

```javascript
const usernamePattern = /^[a-zA-Z0-9_\-]{3,20}$/;

usernamePattern.test('user_name'); // true
usernamePattern.test('ab');       // false（短すぎ）
usernamePattern.test('user name'); // false（スペース不可）
```

---

## 電話・郵便番号系

### 5. 日本の電話番号（固定・携帯）

```javascript
// ハイフンあり・なし両対応
const phonePattern = /^(0\d{1,4}[\-\s]?\d{1,4}[\-\s]?\d{4}|0[789]0[\-\s]?\d{4}[\-\s]?\d{4})$/;

phonePattern.test('090-1234-5678'); // true
phonePattern.test('0312345678');    // true
phonePattern.test('0312-3456-78');  // false
```

```javascript
// シンプル版（形式の厳密チェックを緩める）
const phonePatterSimple = /^[0-9\-]{10,13}$/;
```

### 6. 携帯電話番号のみ

```javascript
const mobilePattern = /^0[789]0[\-\s]?\d{4}[\-\s]?\d{4}$/;

mobilePattern.test('090-1234-5678'); // true
mobilePattern.test('03-1234-5678');  // false
```

### 7. 日本の郵便番号

```javascript
// ハイフンあり・なし
const zipPattern = /^\d{3}[\-]?\d{4}$/;

zipPattern.test('123-4567'); // true
zipPattern.test('1234567');  // true
zipPattern.test('123456');   // false
```

---

## URL・ドメイン系

### 8. URL（http/https）

```javascript
const urlPattern = /^https?:\/\/(www\.)?[-a-zA-Z0-9@:%._+~#=]{1,256}\.[a-zA-Z0-9()]{1,6}\b([-a-zA-Z0-9()@:%_+.~#?&//=]*)$/;

urlPattern.test('https://example.com');          // true
urlPattern.test('https://example.com/path?q=1'); // true
urlPattern.test('ftp://example.com');            // false
```

### 9. ドメイン名のみ

```javascript
const domainPattern = /^[a-zA-Z0-9]([a-zA-Z0-9\-]{0,61}[a-zA-Z0-9])?(\.[a-zA-Z0-9]([a-zA-Z0-9\-]{0,61}[a-zA-Z0-9])?)*\.[a-zA-Z]{2,}$/;

domainPattern.test('example.com');     // true
domainPattern.test('sub.example.com'); // true
domainPattern.test('example');         // false
```

---

## 数値・金額系

### 10. 整数（正の数のみ）

```javascript
const positiveIntPattern = /^[1-9]\d*$/;

positiveIntPattern.test('123');  // true
positiveIntPattern.test('0');    // false（0は含めない）
positiveIntPattern.test('-5');   // false
```

### 11. 0以上の整数

```javascript
const nonNegativeIntPattern = /^\d+$/;

nonNegativeIntPattern.test('0');   // true
nonNegativeIntPattern.test('123'); // true
nonNegativeIntPattern.test('-1');  // false
```

### 12. 金額（カンマ区切り対応）

```javascript
const currencyPattern = /^¥?[\d,]+$/;

// 変換関数
function parseCurrency(str) {
  return parseInt(str.replace(/[¥,]/g, ''), 10);
}

parseCurrency('¥1,234,567'); // 1234567
```

### 13. 小数点2桁まで

```javascript
const decimalPattern = /^\d+(\.\d{1,2})?$/;

decimalPattern.test('100');    // true
decimalPattern.test('100.5');  // true
decimalPattern.test('100.55'); // true
decimalPattern.test('100.555'); // false
```

---

## 日付・時刻系

### 14. 日付（YYYY-MM-DD）

```javascript
const datePattern = /^\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])$/;

datePattern.test('2026-03-15'); // true
datePattern.test('2026-13-01'); // false
```

### 15. 日付（日本形式 YYYY年MM月DD日）

```javascript
const jpDatePattern = /^\d{4}年(0?[1-9]|1[0-2])月(0?[1-9]|[12]\d|3[01])日$/;

jpDatePattern.test('2026年3月15日'); // true
```

### 16. 時刻（HH:MM）

```javascript
const timePattern = /^([01]\d|2[0-3]):[0-5]\d$/;

timePattern.test('09:30'); // true
timePattern.test('24:00'); // false
```

---

## テキスト・コンテンツ系

### 17. カタカナのみ（全角）

```javascript
const katakanaPattern = /^[ァ-ヶー]+$/;

katakanaPattern.test('アイウエオ'); // true
katakanaPattern.test('あいうえお'); // false
```

### 18. ひらがなのみ（全角）

```javascript
const hiraganaPattern = /^[ぁ-ん]+$/;

hiraganaPattern.test('あいうえお'); // true
hiraganaPattern.test('アイウエオ'); // false
```

### 19. 全角文字のみ

```javascript
const fullwidthPattern = /^[^\x00-\x7F]+$/;
```

### 20. HTMLタグを除去

```javascript
const htmlTagPattern = /<[^>]+>/g;

'<p>Hello <strong>world</strong></p>'.replace(htmlTagPattern, '');
// 'Hello world'
```

---

## JavaScriptでのバリデーション実装パターン

### フォームバリデーション関数

```javascript
const validators = {
  email: {
    pattern: /^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$/,
    message: '正しいメールアドレスを入力してください'
  },
  phone: {
    pattern: /^[0-9\-]{10,13}$/,
    message: '正しい電話番号を入力してください'
  },
  zip: {
    pattern: /^\d{3}[\-]?\d{4}$/,
    message: '正しい郵便番号を入力してください'
  }
};

function validate(type, value) {
  const v = validators[type];
  if (!v) return { valid: true };
  const valid = v.pattern.test(value);
  return { valid, message: valid ? '' : v.message };
}

// 使い方
const result = validate('email', 'test@example.com');
if (!result.valid) {
  console.error(result.message);
}
```

### リアルタイムバリデーション

```javascript
document.querySelectorAll('[data-validate]').forEach(input => {
  const type = input.dataset.validate;
  const errorEl = document.querySelector(`[data-error-for="${input.id}"]`);

  input.addEventListener('blur', () => {
    const result = validate(type, input.value);
    if (errorEl) {
      errorEl.textContent = result.message;
    }
    input.setAttribute('aria-invalid', String(!result.valid));
  });
});
```

---

## 正規表現を扱うときの注意点

**1. `test()` は `lastIndex` をリセットする**

`g` フラグと `test()` を繰り返し呼ぶと `lastIndex` がずれる。

```javascript
// NG
const re = /\d+/g;
re.test('abc123'); // false → true
re.test('abc123'); // false（lastIndexがずれている）

// OK
const re = /\d+/; // gフラグなし
re.test('abc123'); // false
re.test('abc123'); // false（毎回同じ）
```

**2. ユーザー入力をそのままRegExpに使わない（ReDos対策）**

```javascript
// NG
const userInput = '(((((((((((((((((((((((';
new RegExp(userInput); // エラー or ReDos

// OK
function escapeRegex(str) {
  return str.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
}
```

---

## まとめ

正規表現は覚えるより「コピペして使うもの」と割り切るほうが実用的だ。この記事のパターンをチームの共有ファイルやスニペットに登録しておくと、都度ゼロから書く手間を省ける。

[ToolShare Lab](https://webatives.com/) では正規表現テストツールなど、Web制作に役立つツールを無料公開している。フリーランスの業務効率化ツールは [AND TOOLS](https://and-tools.net/) でまとめて利用できる。
