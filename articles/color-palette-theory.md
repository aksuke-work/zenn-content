---
title: "配色の60-30-10ルール — Webデザインで失敗しない色選び"
emoji: "🎯"
type: "tech"
topics: ["webdesign", "css", "design", "frontend"]
published: false
---

フリーランス向けの計算ツールサイト [AND TOOLS](https://and-tools.net/) のリデザインを行ったとき、最も悩んだのが配色だった。センスや経験に頼らず、理論的に「失敗しない配色」を選ぶ方法を調べていくうちに「60-30-10ルール」にたどり着いた。

この記事では、60-30-10ルールをWebデザインに適用する方法を具体的なCSS実装とともに解説する。

---

## 60-30-10ルールとは

インテリアデザインの世界から来た配色の法則。空間を「支配色60%・補助色30%・アクセント色10%」の比率で構成すると、視覚的に安定した印象を作れる。

| 役割 | 比率 | 用途 |
|---|---|---|
| **ドミナントカラー（支配色）** | 60% | 背景、大きなエリア |
| **サブカラー（補助色）** | 30% | カード、サイドバー、セクション |
| **アクセントカラー** | 10% | ボタン、リンク、ハイライト |

---

## なぜ10%だけ使うのか

アクセントカラーを多用すると、どこに視線を誘導したいかが曖昧になる。10%に絞ることで「この色 = クリックしてほしい」「この色 = 重要な情報」という視覚的なシグナルが際立つ。

コンバージョン設計の観点でも重要で、**CTAボタンだけにアクセントカラーを使う**と自然に目が行く。

---

## カラーパレット設計の手順

### Step 1：ドミナントカラーを決める

ブランドイメージから選ぶ。Webサイトでは多くの場合、白またはオフホワイトが支配色になる。

```css
:root {
  /* ドミナントカラー（60%） */
  --color-bg:        #ffffff;
  --color-bg-muted:  #f8fafc;
  --color-bg-subtle: #f1f5f9;
}
```

ダークモードの場合：

```css
:root[data-theme="dark"] {
  --color-bg:        #0f0f1a;
  --color-bg-muted:  #1a1a2e;
  --color-bg-subtle: #16213e;
}
```

### Step 2：サブカラーを選ぶ

ドミナントカラーと調和する落ち着いた色。テキスト色、ボーダー、カードの背景などに使う。

```css
:root {
  /* サブカラー（30%） */
  --color-surface:   #f0f4ff;
  --color-border:    #e2e8f0;
  --color-text:      #1e293b;
  --color-text-muted:#64748b;
}
```

### Step 3：アクセントカラーを決める

ブランドのプライマリーカラーをここに置く。1〜2色に絞る。

```css
:root {
  /* アクセントカラー（10%） */
  --color-primary:       #4f46e5;
  --color-primary-hover: #4338ca;
  --color-primary-light: rgba(79, 70, 229, 0.1);
}
```

---

## 実際のカラーシステム設計

### パターン1：クリーン系（SaaS・ツールサイト向け）

```css
:root {
  /* ドミナント */
  --bg-base:    #ffffff;
  --bg-muted:   #f8fafc;
  --bg-subtle:  #f1f5f9;

  /* サブ */
  --text-body:  #1e293b;
  --text-muted: #64748b;
  --border:     #e2e8f0;
  --surface:    #ffffff;

  /* アクセント */
  --accent:     #4f46e5;
  --accent-alt: #7c3aed;
}
```

```css
/* 実際の使い方 */
body        { background: var(--bg-base); color: var(--text-body); }
.card       { background: var(--surface); border: 1px solid var(--border); }
.btn-primary{ background: var(--accent); color: #fff; }
.link       { color: var(--accent); }
```

### パターン2：プロフェッショナル系（コーポレート向け）

```css
:root {
  /* ドミナント */
  --bg-base:    #fafafa;
  --bg-muted:   #f5f5f5;
  --bg-subtle:  #eeeeee;

  /* サブ */
  --text-body:  #212121;
  --text-muted: #757575;
  --border:     #e0e0e0;

  /* アクセント */
  --accent:     #1976d2;
  --accent-alt: #1565c0;
}
```

### パターン3：ダーク系（開発者向け・ポートフォリオ）

```css
:root {
  /* ドミナント */
  --bg-base:    #0f0f1a;
  --bg-muted:   #1a1a2e;
  --bg-subtle:  #16213e;

  /* サブ */
  --text-body:  #e2e8f0;
  --text-muted: #94a3b8;
  --border:     #2d3748;

  /* アクセント */
  --accent:     #818cf8;
  --accent-alt: #6366f1;
}
```

### パターン4：ウォーム系（飲食・ライフスタイル向け）

```css
:root {
  /* ドミナント */
  --bg-base:    #fefdf9;
  --bg-muted:   #fdf8ef;
  --bg-subtle:  #faf0dc;

  /* サブ */
  --text-body:  #2d1b00;
  --text-muted: #8b6914;
  --border:     #e8d5a3;

  /* アクセント */
  --accent:     #d97706;
  --accent-alt: #b45309;
}
```

---

## 色の組み合わせ：カラーハーモニー

アクセントカラーを選ぶとき、ドミナントカラーとの関係性が重要になる。

### 補色（Complementary）

色相環の反対側の色。コントラストが強く、力強い印象を与える。

```css
/* ブルー × オレンジ */
--dominant: #1e40af;
--accent:   #f59e0b;
```

### 類似色（Analogous）

色相環で隣り合う色。統一感があり穏やかな印象。

```css
/* ブルー × パープル × インディゴ */
--dominant: #3b82f6;
--sub:      #6366f1;
--accent:   #8b5cf6;
```

### トライアド（Triadic）

色相環を3等分した位置にある色の組み合わせ。

```css
/* 赤 × 黄 × 青（原色トライアド） */
--color-a: #ef4444;
--color-b: #eab308;
--color-c: #3b82f6;
```

---

## アクセシビリティ：コントラスト比の確認

WCAGのガイドラインでは以下のコントラスト比が要求される。

| レベル | 通常テキスト | 大きいテキスト（18pt+） |
|---|---|---|
| AA（最低基準） | 4.5:1 | 3:1 |
| AAA（推奨）  | 7:1   | 4.5:1 |

JavaScriptでコントラスト比を計算する関数：

```javascript
function getContrastRatio(hex1, hex2) {
  const getLuminance = (hex) => {
    const rgb = parseInt(hex.slice(1), 16);
    const r = ((rgb >> 16) & 0xff) / 255;
    const g = ((rgb >>  8) & 0xff) / 255;
    const b = ((rgb >>  0) & 0xff) / 255;

    const toLinear = c => c <= 0.03928
      ? c / 12.92
      : Math.pow((c + 0.055) / 1.055, 2.4);

    return 0.2126 * toLinear(r) + 0.7152 * toLinear(g) + 0.0722 * toLinear(b);
  };

  const l1 = getLuminance(hex1);
  const l2 = getLuminance(hex2);
  const lighter = Math.max(l1, l2);
  const darker  = Math.min(l1, l2);

  return (lighter + 0.05) / (darker + 0.05);
}

// 使い方
const ratio = getContrastRatio('#4f46e5', '#ffffff');
console.log(ratio.toFixed(2)); // 例: 5.04（AA合格）
```

---

## よくある配色の失敗パターン

**1. アクセントカラーを使いすぎる**

ボタン・リンク・見出し・バッジすべてにプライマリーカラーを使ってしまうケース。視線の導線が壊れる。

解決策：ボタン（CTA）以外はアクセントカラーを控える。見出しは `--text-body` を使う。

**2. コントラストが不足する**

薄いグレー背景に薄いグレーのテキストを置くケース。ライトモード時に問題ないと思っていても、直射日光下のスマホでは読めないことが多い。

解決策：テキストとその背景のコントラスト比を必ず4.5:1以上確認する。

**3. 多色使いすぎ**

5色以上を使うと統一感がない印象になる。60-30-10ルールを守り、基本3色（+ バリエーション）に絞る。

---

## ダークモード対応のトークン設計

CSS変数で設計していれば、ダークモード対応が容易になる。

```css
:root {
  --color-bg:      #ffffff;
  --color-text:    #1e293b;
  --color-accent:  #4f46e5;
  --color-border:  #e2e8f0;
}

@media (prefers-color-scheme: dark) {
  :root {
    --color-bg:      #0f0f1a;
    --color-text:    #e2e8f0;
    --color-accent:  #818cf8;  /* 少し明るく調整 */
    --color-border:  #2d3748;
  }
}
```

アクセントカラーはダークモードで少し明るくすることでコントラストを保つ。

---

## まとめ

60-30-10ルールはシンプルだが、デザインの判断軸として非常に強力だ。迷ったら「今どの色を何に使っているか」を3分類に当てはめてみると、問題のある箇所が見えてくる。

[ToolShare Lab](https://and-and.net/) では、カラーパレット生成などWebデザイン支援ツールを公開している。フリーランスの業務に役立つ各種計算ツールは [AND TOOLS](https://and-tools.net/) でまとめて利用できる。

色は感覚で選ぶものではなく、ルールと目的から導き出せる。まず3色のトークンを設計するところから始めてほしい。
