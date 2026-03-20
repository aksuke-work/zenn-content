---
title: "計算結果をコピーする機能の実装 — Clipboard APIとフォールバック"
emoji: "📋"
type: "tech"
topics: ["javascript", "clipboard", "ui", "webdev"]
published: false
---

Webツールの計算結果をワンクリックでコピーできる機能は、ユーザー体験を大きく左右する。[AND TOOLS](https://and-tools.net/) の税金計算ツール群で実装した Clipboard API + フォールバック + トースト通知の仕組みを解説する。

## Clipboard API の基本

モダンブラウザでは `navigator.clipboard.writeText()` でクリップボードに書き込める。

```javascript
async function copyToClipboard(text) {
  try {
    await navigator.clipboard.writeText(text);
    return true;
  } catch (err) {
    console.error('Clipboard API failed:', err);
    return false;
  }
}
```

シンプルだが、**この API には制約がある**。

### HTTPS必須問題

`navigator.clipboard` は **Secure Context（HTTPS）** でないと使えない。`http://localhost` での開発時は動くが、本番が HTTP のままだとエラーになる。

```javascript
// Secure Contextかどうかの判定
if (window.isSecureContext) {
  // Clipboard API が使える
} else {
  // フォールバックが必要
}
```

また、ユーザーのアクション（クリックイベント等）の中で呼ばないと `NotAllowedError` が発生する。非同期のタイマー内や、ページ読み込み時に自動でコピーすることはできない。

## document.execCommand('copy') フォールバック

古いブラウザや HTTP 環境のために、`document.execCommand('copy')` を使ったフォールバックを用意する。非推奨 API だが、動作するブラウザは多い。

```javascript
function fallbackCopy(text) {
  const textarea = document.createElement('textarea');

  // 画面に見えないように配置（display:noneだと選択できない）
  textarea.value = text;
  textarea.style.position = 'fixed';
  textarea.style.left = '-9999px';
  textarea.style.top = '-9999px';
  textarea.style.opacity = '0';

  document.body.appendChild(textarea);
  textarea.focus();
  textarea.select();

  let success = false;
  try {
    success = document.execCommand('copy');
  } catch (err) {
    console.error('execCommand copy failed:', err);
  }

  document.body.removeChild(textarea);
  return success;
}
```

ポイント:

- `display: none` だとテキスト選択ができない。`opacity: 0` + 画面外配置にする
- `textarea` を使うのは複数行テキストにも対応するため。`input[type="text"]` だと改行が消える
- 処理後に要素を必ず削除する

## 統合関数 — API + フォールバック

Clipboard API を優先し、失敗時にフォールバックする統合関数を作る。

```javascript
async function copyText(text) {
  // 1. Clipboard API を試す
  if (navigator.clipboard && window.isSecureContext) {
    try {
      await navigator.clipboard.writeText(text);
      return true;
    } catch (err) {
      // Permission denied 等 → フォールバックへ
    }
  }

  // 2. execCommand フォールバック
  return fallbackCopy(text);
}
```

この関数はどの環境でも安全にコピーを試みる。

## コピーボタンの実装

計算結果の横にコピーボタンを設置する実装。

```javascript
function setupCopyButtons() {
  const buttons = document.querySelectorAll('.js-copyButton');

  buttons.forEach(button => {
    button.addEventListener('click', async () => {
      const targetId = button.dataset.copyTarget;
      const targetEl = document.getElementById(targetId);
      if (!targetEl) return;

      // 計算結果のテキストを取得（カンマ区切り等のフォーマット済み）
      const text = targetEl.textContent.trim();
      const success = await copyText(text);

      if (success) {
        showToast('コピーしました');
        animateCopyButton(button);
      } else {
        showToast('コピーに失敗しました', 'error');
      }
    });
  });
}
```

HTML側:

```html
<div class="resultCard">
  <span class="resultCard__label">手取り年収</span>
  <span class="resultCard__value" id="takeHomePay">4,832,500円</span>
  <button class="resultCard__copyButton js-copyButton"
          data-copy-target="takeHomePay"
          aria-label="手取り年収をコピー">
    <svg width="16" height="16" viewBox="0 0 16 16" fill="currentColor">
      <path d="M4 2a2 2 0 0 1 2-2h6a2 2 0 0 1 2 2v8a2 2 0 0 1-2 2H6a2 2 0 0 1-2-2V2z"/>
      <path d="M2 4a2 2 0 0 0-2 2v8a2 2 0 0 0 2 2h6a2 2 0 0 0 2-2v-1H6a3 3 0 0 1-3-3V4H2z"/>
    </svg>
  </button>
</div>
```

## トースト通知UI

コピー成功・失敗をユーザーに伝えるトースト通知を実装する。

```javascript
function showToast(message, type = 'success') {
  // 既存のトーストがあれば削除
  const existing = document.querySelector('.toast');
  if (existing) existing.remove();

  const toast = document.createElement('div');
  toast.className = `toast toast_type_${type}`;
  toast.textContent = message;
  toast.setAttribute('role', 'status');
  toast.setAttribute('aria-live', 'polite');

  document.body.appendChild(toast);

  // アニメーション: 表示 → 2秒待ち → フェードアウト → 削除
  requestAnimationFrame(() => {
    toast.classList.add('toast_state_visible');
  });

  setTimeout(() => {
    toast.classList.add('toast_state_hiding');
    toast.addEventListener('transitionend', () => toast.remove());
  }, 2000);
}
```

```css
.toast {
  position: fixed;
  bottom: 24px;
  left: 50%;
  transform: translateX(-50%) translateY(20px);
  padding: 12px 24px;
  border-radius: 8px;
  font-size: 14px;
  font-weight: 600;
  color: #fff;
  opacity: 0;
  transition: opacity 0.3s ease, transform 0.3s ease;
  z-index: 9999;
  pointer-events: none;
}

.toast_type_success {
  background: #27ae60;
}

.toast_type_error {
  background: #e74c3c;
}

.toast_state_visible {
  opacity: 1;
  transform: translateX(-50%) translateY(0);
}

.toast_state_hiding {
  opacity: 0;
  transform: translateX(-50%) translateY(-10px);
}
```

## コピーボタンのフィードバックアニメーション

ボタン自体にもコピー成功のフィードバックを付ける。アイコンが一瞬チェックマークに変わる演出。

```javascript
function animateCopyButton(button) {
  const originalHTML = button.innerHTML;

  // チェックマークに切替
  button.innerHTML = `
    <svg width="16" height="16" viewBox="0 0 16 16" fill="currentColor">
      <path d="M13.78 4.22a.75.75 0 0 1 0 1.06l-7.25 7.25a.75.75 0 0 1-1.06 0L2.22 9.28a.75.75 0 0 1 1.06-1.06L6 10.94l6.72-6.72a.75.75 0 0 1 1.06 0z"/>
    </svg>
  `;
  button.classList.add('resultCard__copyButton_state_copied');

  // 1.5秒後に元に戻す
  setTimeout(() => {
    button.innerHTML = originalHTML;
    button.classList.remove('resultCard__copyButton_state_copied');
  }, 1500);
}
```

```css
.resultCard__copyButton {
  background: none;
  border: 1px solid #ddd;
  border-radius: 4px;
  padding: 4px 8px;
  cursor: pointer;
  color: #666;
  transition: color 0.2s, border-color 0.2s;
}

.resultCard__copyButton:hover {
  color: #333;
  border-color: #999;
}

.resultCard__copyButton_state_copied {
  color: #27ae60;
  border-color: #27ae60;
}
```

## 複数項目の一括コピー

税金計算ツールでは、複数の計算結果をまとめてコピーしたいケースがある。

```javascript
function copyAllResults() {
  const results = document.querySelectorAll('.resultCard__value');
  const lines = [];

  results.forEach(el => {
    const label = el.previousElementSibling?.textContent?.trim() || '';
    const value = el.textContent.trim();
    if (label && value) {
      lines.push(`${label}: ${value}`);
    }
  });

  const text = lines.join('\n');
  return copyText(text);
}

// 出力例:
// 年収: 6,000,000円
// 所得税: 232,500円
// 住民税: 355,000円
// 社会保険料: 580,000円
// 手取り: 4,832,500円
```

このテキスト形式なら、チャットやメモアプリに貼り付けてもそのまま読める。

## ブラウザ対応状況

| ブラウザ | Clipboard API | execCommand |
|---------|--------------|-------------|
| Chrome 66+ | OK | OK |
| Firefox 63+ | OK | OK |
| Safari 13.1+ | OK | OK |
| Edge 79+ | OK | OK |
| IE 11 | NG | OK |
| HTTP環境 | NG | OK |

2026年現在、Clipboard API だけでもカバー率は十分高い。ただし HTTP 環境を考慮するなら execCommand のフォールバックは残しておいたほうが安全。

## まとめ

1. **Clipboard API を第一選択**に。`navigator.clipboard.writeText()` は Promise ベースで扱いやすい
2. **HTTPS 必須**を忘れない。HTTP 環境では `execCommand('copy')` でフォールバック
3. **コピー成功のフィードバック**が重要。トースト通知 + ボタンのアイコン変化で確実に伝える
4. **`aria-live="polite"`** でスクリーンリーダーにもコピー結果を伝達
5. **一括コピー**で複数の計算結果をまとめて取得可能

---

## 関連サイト

この記事のコピー機能は以下のサイトで実際に稼働しています。

- [AND TOOLS](https://and-tools.net/) — フリーランス向け無料計算ツール集。各ツールの計算結果にコピーボタンを設置
- [手取り計算シミュレーター](https://and-tools.net/tools/take-home-pay/) — 年収から手取りを計算。結果の一括コピー機能付き
- [ToolShare Lab](https://webatives.com/tools/) — 150個以上の無料ツールを公開中
