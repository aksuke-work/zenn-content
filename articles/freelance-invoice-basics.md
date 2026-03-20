---
title: "フリーランスの請求書に書くべき10項目 — インボイス制度対応版"
emoji: "📄"
type: "tech"
topics: ["フリーランス", "請求書", "インボイス", "個人事業主"]
published: false
---

請求書の記載ミスや漏れは、支払いトラブルや税務上の問題につながる。インボイス制度が始まってからは、記載要件がより厳格になった。

[AND TOOLS](https://and-tools.net/) でインボイス計算ツールを設計した経験から、フリーランスが作成する請求書に必要な10項目と実装例を解説する。

---

## インボイス制度とは（簡易まとめ）

2023年10月から始まった**適格請求書等保存方式（インボイス制度）**。仕入税額控除（消費税の仕入れ税額を差し引ける仕組み）を受けるために、クライアント側が**適格請求書（インボイス）**の保存を求められるようになった。

フリーランスへの影響：
- **免税事業者**のまま → インボイス発行不可 → クライアントが消費税仕入れ控除できない
- **課税事業者登録（インボイス登録）** → 適格請求書を発行できる → クライアントは仕入れ控除可能

---

## 請求書に書くべき10項目

### 通常の請求書（インボイス未登録・免税事業者）

免税事業者でも請求書は発行できる。インボイスの記載要件を満たさなくてよいが、基本項目は必要だ。

| No | 項目 | 説明 |
|----|------|------|
| 1 | 請求書のタイトル | 「請求書」と明記する |
| 2 | 発行日 | 請求書の作成日 |
| 3 | 請求書番号 | 管理用の連番（例: INV-2024-001） |
| 4 | 発行者情報 | 屋号・氏名・住所・連絡先 |
| 5 | 請求先情報 | 会社名・部署名・担当者名 |
| 6 | 請求金額（合計） | 税込・税抜を明記 |
| 7 | 明細 | 作業内容・単価・数量・金額 |
| 8 | 消費税額 | 税率と税額の明示 |
| 9 | 振込先情報 | 銀行名・支店名・口座種別・口座番号・口座名義 |
| 10 | 支払期限 | 具体的な日付（例：2024年3月31日） |

---

### 適格請求書（インボイス登録済みの場合）

インボイス登録をしている場合、上記10項目に加えて以下が必要だ。

| 追加項目 | 説明 |
|---------|------|
| 登録番号 | T + 13桁の数字（例: T1234567890123） |
| 税率ごとの税抜金額 | 10%対象・8%（軽減税率）対象を分けて記載 |
| 税率ごとの消費税額 | 10%分・8%分を分けて記載 |

---

## JavaScriptで請求書データを生成する

```javascript
// 請求書データの構造
class Invoice {
  constructor({
    invoiceNumber,
    issueDate,
    dueDate,
    issuer,
    client,
    items,
    isRegistered = false,
    registrationNumber = null,
  }) {
    this.invoiceNumber = invoiceNumber;
    this.issueDate = issueDate;
    this.dueDate = dueDate;
    this.issuer = issuer;
    this.client = client;
    this.items = items;
    this.isRegistered = isRegistered;
    this.registrationNumber = registrationNumber;
  }

  // 消費税の計算（税率ごとに集計）
  calcTax() {
    const taxGroups = {};

    this.items.forEach((item) => {
      const taxRate = item.taxRate || 0.1; // デフォルト10%
      const key = taxRate.toString();

      if (!taxGroups[key]) {
        taxGroups[key] = { taxRate, subtotal: 0, taxAmount: 0 };
      }

      const subtotal = item.unitPrice * item.quantity;
      const taxAmount = Math.floor(subtotal * taxRate);

      taxGroups[key].subtotal += subtotal;
      taxGroups[key].taxAmount += taxAmount;
    });

    return taxGroups;
  }

  // 合計金額の計算
  calcTotal() {
    const taxGroups = this.calcTax();
    let subtotal = 0;
    let totalTax = 0;

    Object.values(taxGroups).forEach((group) => {
      subtotal += group.subtotal;
      totalTax += group.taxAmount;
    });

    return {
      subtotal,
      totalTax,
      grandTotal: subtotal + totalTax,
    };
  }

  // 請求書テキストの生成
  toText() {
    const { subtotal, totalTax, grandTotal } = this.calcTotal();
    const taxGroups = this.calcTax();

    let text = `
===========================================
請求書
===========================================
請求書番号: ${this.invoiceNumber}
発行日: ${this.issueDate}
お支払期限: ${this.dueDate}
${this.isRegistered ? `登録番号: ${this.registrationNumber}` : ""}

【発行者】
${this.issuer.name}
${this.issuer.address}
${this.issuer.email}

【請求先】
${this.client.companyName} ${this.client.department || ""}
${this.client.contactName} 様

-------------------------------------------
【明細】
`;

    this.items.forEach((item) => {
      const amount = item.unitPrice * item.quantity;
      text += `${item.name}\n`;
      text += `  単価: ¥${item.unitPrice.toLocaleString()} × ${item.quantity} = ¥${amount.toLocaleString()} (税率${item.taxRate * 100}%)\n`;
    });

    text += `
-------------------------------------------
【税率別 小計・消費税】
`;
    Object.values(taxGroups).forEach((group) => {
      text += `${group.taxRate * 100}%対象: ¥${group.subtotal.toLocaleString()} / 消費税: ¥${group.taxAmount.toLocaleString()}\n`;
    });

    text += `
-------------------------------------------
小計: ¥${subtotal.toLocaleString()}
消費税合計: ¥${totalTax.toLocaleString()}
【合計（税込）: ¥${grandTotal.toLocaleString()}】
-------------------------------------------

【お振込先】
${this.issuer.bankInfo.bankName} ${this.issuer.bankInfo.branchName}
${this.issuer.bankInfo.accountType}：${this.issuer.bankInfo.accountNumber}
口座名義：${this.issuer.bankInfo.accountHolder}
===========================================
`;
    return text;
  }
}

// 使用例
const invoice = new Invoice({
  invoiceNumber: "INV-2024-001",
  issueDate: "2024-01-31",
  dueDate: "2024-02-29",
  issuer: {
    name: "山田 太郎（屋号: YT Works）",
    address: "東京都渋谷区〇〇 1-2-3",
    email: "taro@example.com",
    bankInfo: {
      bankName: "三菱UFJ銀行",
      branchName: "渋谷支店",
      accountType: "普通",
      accountNumber: "1234567",
      accountHolder: "ヤマダ タロウ",
    },
  },
  client: {
    companyName: "株式会社サンプル",
    department: "開発部",
    contactName: "鈴木 花子",
  },
  items: [
    {
      name: "Webシステム開発（1月分）",
      unitPrice: 500000,
      quantity: 1,
      taxRate: 0.1,
    },
    {
      name: "サーバー保守費",
      unitPrice: 30000,
      quantity: 1,
      taxRate: 0.1,
    },
  ],
  isRegistered: true,
  registrationNumber: "T1234567890123",
});

console.log(invoice.toText());
console.log("合計:", invoice.calcTotal());
```

---

## よくある記載ミス

### ミス1：消費税の計算が税率ごとになっていない

インボイスでは「10%対象の税抜金額・税額」「8%対象の税抜金額・税額」を分けて記載する必要がある。一括で「消費税10%」とだけ書くのはNGだ。

### ミス2：振込手数料の扱いを明示していない

「振込手数料は先方負担」なのか「こちら負担」なのか、最初の契約時に明示しておく。後で揉める原因になる。

### ミス3：請求書番号を連番管理していない

税務調査の際に「どの案件の請求書か」を追えるよう、連番管理が必要だ。

```javascript
// 請求書番号の自動生成
function generateInvoiceNumber(year, month, sequence) {
  const paddedSeq = String(sequence).padStart(3, "0");
  return `INV-${year}${String(month).padStart(2, "0")}-${paddedSeq}`;
}

console.log(generateInvoiceNumber(2024, 1, 1));  // → INV-202401-001
console.log(generateInvoiceNumber(2024, 1, 12)); // → INV-202401-012
```

### ミス4：源泉徴収額の記載漏れ

デザイン・ライティング・コンサルティング等の一定の業務は源泉徴収の対象だ。エンジニアの案件でも契約内容によっては源泉対象になる。

```javascript
// 源泉徴収の計算（報酬・料金等の場合）
function calcWithholdingTax(amount) {
  if (amount <= 1000000) {
    return Math.floor(amount * 0.1021);
  }
  return Math.floor(1000000 * 0.1021 + (amount - 1000000) * 0.2042);
}

const fee = 300000;
const withholding = calcWithholdingTax(fee);
const payable = fee - withholding;

console.log(`請求額: ¥${fee.toLocaleString()}`);
console.log(`源泉徴収税額: ¥${withholding.toLocaleString()}`);
console.log(`差引振込額: ¥${payable.toLocaleString()}`);
```

---

## インボイス未登録のフリーランスへのクライアント対応

インボイス未登録（免税事業者）の場合、クライアントは消費税の仕入れ控除ができない。ただし2029年9月30日まで経過措置があり、仕入れ控除の一定割合は認められている。

| 期間 | 控除可能割合 |
|------|-----------|
| 2023年10月〜2026年9月 | 80% |
| 2026年10月〜2029年9月 | 50% |
| 2029年10月〜 | 0% |

フリーランスとして登録すべきかの判断は、取引先の規模・税務上の影響・手続き負担を総合的に考える必要がある。[AND TOOLSのインボイス計算ツール](https://and-tools.net/tools/invoice/) で登録した場合の税負担を確認できる。

---

## まとめ

請求書の記載要件を押さえることは、支払い遅延の防止と税務上のトラブル回避につながる。

- 基本10項目を必ず記載する
- インボイス登録済みなら登録番号・税率別記載を追加する
- 連番管理で請求書を追跡可能にする
- 源泉徴収の有無を事前にクライアントと確認する

[AND TOOLS](https://and-tools.net/) では請求書関連の計算やフリーランス向け税金ツールを無料で提供している。請求書を作成する際の参考にしてほしい。
