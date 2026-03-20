---
title: "バニラJSでテーブルソートを実装する — ライブラリ不要の軽量版"
emoji: "📊"
type: "tech"
topics: ["javascript", "table", "sort", "webdev"]
published: false
---

テーブルのヘッダーをクリックしたら昇順/降順でソートする、よくあるUIをバニラJSだけで実装する。50行以下のテーブルならライブラリは不要。[AND TOOLS](https://and-tools.net/) の年収別壁テーブルで使っている実装をベースに解説する。

## 完成形のデモ

こういうテーブルを想定する。

```html
<table class="sortTable js-sortTable">
  <thead>
    <tr>
      <th data-sort="string">年収の壁</th>
      <th data-sort="number">金額（万円）</th>
      <th data-sort="string">影響する税・保険</th>
      <th data-sort="number">手取り減少額</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>103万の壁</td>
      <td>103</td>
      <td>所得税</td>
      <td>0</td>
    </tr>
    <tr>
      <td>106万の壁</td>
      <td>106</td>
      <td>社会保険</td>
      <td>150,000</td>
    </tr>
    <tr>
      <td>130万の壁</td>
      <td>130</td>
      <td>社会保険（扶養）</td>
      <td>200,000</td>
    </tr>
    <tr>
      <td>150万の壁</td>
      <td>150</td>
      <td>配偶者特別控除</td>
      <td>0</td>
    </tr>
    <tr>
      <td>201万の壁</td>
      <td>201</td>
      <td>配偶者特別控除消滅</td>
      <td>380,000</td>
    </tr>
  </tbody>
</table>
```

`data-sort` 属性で列のデータ型（`number` or `string`）を指定する。

## ソートのコアロジック

```javascript
class TableSorter {
  constructor(tableEl) {
    this.table = tableEl;
    this.thead = tableEl.querySelector('thead');
    this.tbody = tableEl.querySelector('tbody');
    this.headers = [...this.thead.querySelectorAll('th[data-sort]')];
    this.sortState = { colIndex: -1, direction: 'asc' };

    this.init();
  }

  init() {
    this.headers.forEach((th, index) => {
      th.style.cursor = 'pointer';
      th.setAttribute('role', 'columnheader');
      th.setAttribute('aria-sort', 'none');
      th.addEventListener('click', () => this.sort(index));
    });
  }

  sort(colIndex) {
    const direction = (this.sortState.colIndex === colIndex && this.sortState.direction === 'asc')
      ? 'desc'
      : 'asc';

    const sortType = this.headers[colIndex].dataset.sort;
    const rows = [...this.tbody.querySelectorAll('tr')];

    rows.sort((a, b) => {
      const cellA = a.cells[colIndex].textContent.trim();
      const cellB = b.cells[colIndex].textContent.trim();
      return this.compare(cellA, cellB, sortType, direction);
    });

    // DOM再配置
    rows.forEach(row => this.tbody.appendChild(row));

    // 状態更新
    this.sortState = { colIndex, direction };
    this.updateHeaderIcons(colIndex, direction);
  }

  compare(a, b, type, direction) {
    let result;

    if (type === 'number') {
      const numA = this.parseNumber(a);
      const numB = this.parseNumber(b);
      result = numA - numB;
    } else {
      result = a.localeCompare(b, 'ja');
    }

    return direction === 'desc' ? -result : result;
  }

  parseNumber(str) {
    // カンマ、円、万 などを除去して数値化
    const cleaned = str.replace(/[,、円万¥%]/g, '');
    const num = parseFloat(cleaned);
    return isNaN(num) ? 0 : num;
  }

  updateHeaderIcons(activeIndex, direction) {
    this.headers.forEach((th, index) => {
      // 既存のアイコンを削除
      const existingIcon = th.querySelector('.sortTable__sortIcon');
      if (existingIcon) existingIcon.remove();

      if (index === activeIndex) {
        const icon = document.createElement('span');
        icon.className = 'sortTable__sortIcon';
        icon.textContent = direction === 'asc' ? ' ▲' : ' ▼';
        icon.setAttribute('aria-hidden', 'true');
        th.appendChild(icon);
        th.setAttribute('aria-sort', direction === 'asc' ? 'ascending' : 'descending');
      } else {
        th.setAttribute('aria-sort', 'none');
      }
    });
  }
}
```

## 初期化

```javascript
document.addEventListener('DOMContentLoaded', () => {
  const tables = document.querySelectorAll('.js-sortTable');
  tables.forEach(table => new TableSorter(table));
});
```

これだけでページ内のすべての `.js-sortTable` がソート可能になる。

## 数値の判定 — カンマ・単位付きに対応

テーブルの数値が `1,500,000` や `150万円` のようにフォーマットされていることは多い。`parseNumber` メソッドでそれらを正しく数値化する。

```javascript
parseNumber(str) {
  // 「万」が含まれる場合は10,000倍
  if (str.includes('万')) {
    const cleaned = str.replace(/[^0-9.]/g, '');
    return parseFloat(cleaned) * 10000 || 0;
  }

  // カンマ、円記号、%を除去
  const cleaned = str.replace(/[,、円¥%\s]/g, '');
  const num = parseFloat(cleaned);
  return isNaN(num) ? 0 : num;
}
```

テストケース:

```javascript
parseNumber('1,500,000')   // → 1500000
parseNumber('150万円')     // → 1500000
parseNumber('¥3,000')      // → 3000
parseNumber('45.5%')       // → 45.5
parseNumber('N/A')         // → 0
```

## ソート方向のアイコン表示

CSSでアイコンのスタイリングを整える。

```css
.sortTable th[data-sort] {
  cursor: pointer;
  user-select: none;
  white-space: nowrap;
  transition: background-color 0.15s ease;
}

.sortTable th[data-sort]:hover {
  background-color: rgba(0, 0, 0, 0.04);
}

.sortTable__sortIcon {
  display: inline-block;
  font-size: 0.75em;
  margin-left: 4px;
  color: #2563eb;
  vertical-align: middle;
}
```

もう少し凝るなら、CSS疑似要素でソート前の「↕」アイコンも表示できる。

```css
.sortTable th[data-sort]::after {
  content: ' ↕';
  font-size: 0.7em;
  color: #ccc;
  transition: color 0.15s;
}

.sortTable th[data-sort]:hover::after {
  color: #999;
}

/* ソート中はCSS疑似要素を消す（JS側のアイコンに任せる） */
.sortTable th[aria-sort="ascending"]::after,
.sortTable th[aria-sort="descending"]::after {
  content: '';
}
```

## 50行以下ならライブラリ不要という判断

テーブルソートのライブラリは多い（DataTables、Tablesort、List.js 等）。しかし以下の条件ならバニラJS実装のほうが適切。

| 条件 | ライブラリ | バニラJS |
|------|----------|---------|
| 行数 50行以下 | オーバースペック | 最適 |
| ソートだけでいい | 他の機能が無駄 | 最適 |
| バンドルサイズ重視 | 20-80KB追加 | 約2KB |
| ページネーション要 | DataTables推奨 | 自前は面倒 |
| フィルタリング要 | List.js推奨 | 自前は面倒 |
| 1000行以上 | 仮想スクロール必要 | 性能不足 |

AND TOOLS の年収の壁テーブルは5-10行なので、ライブラリは完全に不要。

## ソート安定性の考慮

JavaScriptの `Array.prototype.sort()` は **ES2019以降安定ソートが保証** されている。つまり同じ値の行は元の順序を維持する。

ただし古いブラウザ（IE含む）では不安定ソートの可能性があるため、安定ソートを保証したい場合は元のインデックスを持たせる。

```javascript
sort(colIndex) {
  const sortType = this.headers[colIndex].dataset.sort;
  const rows = [...this.tbody.querySelectorAll('tr')];

  // 元のインデックスを保持（安定ソート保証）
  const indexed = rows.map((row, i) => ({ row, originalIndex: i }));

  const direction = (this.sortState.colIndex === colIndex && this.sortState.direction === 'asc')
    ? 'desc'
    : 'asc';

  indexed.sort((a, b) => {
    const cellA = a.row.cells[colIndex].textContent.trim();
    const cellB = b.row.cells[colIndex].textContent.trim();
    const result = this.compare(cellA, cellB, sortType, direction);
    // 同値の場合は元の順序を維持
    return result !== 0 ? result : a.originalIndex - b.originalIndex;
  });

  indexed.forEach(({ row }) => this.tbody.appendChild(row));
  this.sortState = { colIndex, direction };
  this.updateHeaderIcons(colIndex, direction);
}
```

## 日本語ソートの注意点

`String.prototype.localeCompare()` に `'ja'` ロケールを渡すと、ひらがな・カタカナ・漢字を正しい順序でソートできる。

```javascript
// 五十音順でソートされる
'か'.localeCompare('あ', 'ja') // → 正の数（か > あ）
'ア'.localeCompare('ア', 'ja') // → 0
```

ただし漢字のソート順は音読み/訓読みが混在するため完全ではない。業務データでは「漢字列でソートする」より「コード番号列でソートする」ほうが実用的。

## まとめ

1. **50行以下のテーブルソートにライブラリは不要**。バニラJS約60行で実装可能
2. **`data-sort` 属性**で列のデータ型を宣言。`number` と `string` で比較ロジックを分岐
3. **カンマ・単位付き数値**は `parseNumber` で正規化してからソート
4. **`aria-sort` 属性**でアクセシビリティ対応
5. **`localeCompare('ja')`** で日本語の五十音順ソートに対応

---

## 関連サイト

テーブルソートが実際に動いているサイトはこちら。

- [年収の壁 完全ガイド](https://and-tools.net/tools/178-wall/) — 年収別の壁をテーブルで比較。ソート機能で金額順・影響度順に並び替え可能
- [AND TOOLS](https://and-tools.net/) — フリーランス向け無料計算ツール集。テーブル出力がある各ツールでソートを活用
- [ToolShare Lab](https://webatives.com/tools/) — 150個以上の無料ツール。比較テーブルを多用
