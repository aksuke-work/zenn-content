---
title: "SHA-256が一致しない理由を14種類のバイト列で確かめる"
emoji: "🧪"
type: "tech"
topics: ["sha256", "unicode", "python", "javascript"]
published: true
---

同じ`hello`を使ったはずなのに、SHA-256が一致しない。そんなときは、文字の見た目と実際のバイト列を分けて確認すると原因を探しやすくなります。

改行、空白、文字コード、BOM、Unicodeの正規化を変えた14例を計算しました。入力はすべてこの記事のコードにある公開ダミーデータです。実ファイルや秘密情報は使っていません。

Webatives（ToolShare Lab）の運営側で、ハッシュ生成・照合ツールを作っています。この記事はその検証から、手元でも再現できる例をまとめたものです。

## helloに改行が1つ付くだけで変わる

以下はUTF-8です。`LF`は`0a`、`CRLF`は`0d 0a`のバイトを付けています。

```text
hello（5バイト）
2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824

hello + LF（6バイト）
5891b5b522d5df086d0ff0b110fbd9d21bb4fc7163af34d08286a2e846f6be03
```

SHA-256は文字の見た目を比較する処理ではありません。[SubtleCrypto.digest()](https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto/digest)にも、入力としてArrayBufferやTypedArrayなどを渡します。同じ方式に同じバイト列を渡せば同じ値になります。

## 14例の結果

Pythonの`hashlib`とNode.jsの`crypto`で、バイト列を別々に作って照合しました。14例すべてでバイト数・入力hex・SHA-256が一致しています。速度や安全性のベンチマークではありません。

表のハッシュは読みやすさのため**先頭16桁だけ**です。実際に照合するときは64桁すべてを使ってください。下のコードは省略しない値を出力します。

| 入力条件 | バイト数 | 入力バイト（16進数） | SHA-256先頭16桁 |
|---|---:|---|---|
| hello / UTF-8 | 5 | `68656c6c6f` | `2cf24dba5fb0a30e` |
| hello! / UTF-8 | 6 | `68656c6c6f21` | `ce06092fb948d9ff` |
| hello + LF | 6 | `68656c6c6f0a` | `5891b5b522d5df08` |
| hello + CRLF | 7 | `68656c6c6f0d0a` | `cd2eca3535741f27` |
| hello + 末尾スペース | 6 | `68656c6c6f20` | `5e3235a8346e5a45` |
| 先頭スペース + hello | 6 | `2068656c6c6f` | `1c9c535c80b32988` |
| UTF-8 BOM + hello | 8 | `efbbbf68656c6c6f` | `7489ebbcc2a00056` |
| hello / UTF-16 LE・BOMなし | 10 | `680065006c006c006f00` | `06e44dc1b95c469f` |
| hello / UTF-16 BE・BOMなし | 10 | `00680065006c006c006f` | `2bbdfe2be1db5af1` |
| é / NFC・UTF-8 | 2 | `c3a9` | `4a99557e4033c353` |
| é / NFD・UTF-8 | 3 | `65cc81` | `bf12767b0f2a56b2` |
| が / NFC・UTF-8 | 3 | `e3818c` | `e3b21956ce958d25` |
| が / NFD・UTF-8 | 6 | `e3818be38299` | `f62b28f746dd7512` |
| 空ファイル | 0 | 空のバイト列 | `e3b0c44298fc1c14` |

特に見落としやすいのが、Unicodeの表現です。NFCの`é`と、NFDの`e`＋結合アクセントは同じように表示されても、上の例ではUTF-8のバイト列が異なります。日本語の`が`にも同じ種類の例があります。[Unicode正規化の仕様](https://www.unicode.org/reports/tr15/)で扱われる違いです。

ただし、不一致を見てすぐ正規化するのは避けてください。配布ファイルのチェックサムを照合するなら、配布元と同じファイルのバイト列を計算します。勝手な改行変換や正規化を挟むと、比較対象そのものが変わります。

## 手元で再現するコード

Python 3の標準ライブラリだけで動きます。ファイルへの読み書きや通信はしません。

```python
import hashlib
import unicodedata

cases = {
    "hello": b"hello",
    "hello!": b"hello!",
    "hello+LF": b"hello\n",
    "hello+CRLF": b"hello\r\n",
    "hello+space": b"hello ",
    "space+hello": b" hello",
    "UTF8-BOM+hello": b"\xef\xbb\xbfhello",
    "hello-UTF16LE": "hello".encode("utf-16-le"),
    "hello-UTF16BE": "hello".encode("utf-16-be"),
    "e-NFC": unicodedata.normalize("NFC", "e\u0301").encode("utf-8"),
    "e-NFD": unicodedata.normalize("NFD", "\u00e9").encode("utf-8"),
    "ga-NFC": unicodedata.normalize("NFC", "\u304b\u3099").encode("utf-8"),
    "ga-NFD": unicodedata.normalize("NFD", "\u304c").encode("utf-8"),
    "empty-file": b"",
}

for name, data in cases.items():
    print(name, len(data), data.hex(), hashlib.sha256(data).hexdigest())
```

ブラウザの[TextEncoder](https://developer.mozilla.org/en-US/docs/Web/API/TextEncoder)はUTF-8にエンコードします。UTF-16・CRLF・BOMの例をブラウザで確認する場合は、対象のバイト列を持ったファイルとして扱う必要があります。テキスト欄がすべての文字コードを切り替えてくれるわけではありません。

## ツールで試す場合

[hello＋LFの例をハッシュツールに読み込む](https://webatives.com/tools/hash/?utm_source=zenn&utm_medium=referral&utm_campaign=sha256_byte_vectors_20261103#sample-hello-lf)と、公開ダミー文字列と期待値が入ります。「期待値と照合する」で一致を確認し、その後に文字を変えて不一致を確かめられます。

このツールのテキスト入力はUTF-8で、空白や改行の自動除去・Unicode正規化はしません。空のテキスト欄では結果を消します。空ファイルの計算はファイル選択で行う別の操作です。

学習には公開ダミーを使ってください。コピー時は入力の先頭最大80文字、またはファイル名とハッシュ値が端末内の履歴に保存されます。期待値の欄はそこへ追加保存しません。

一致したことだけでは、ファイルの無害性や配布者の本人性は確認できません。期待値は信頼できる配布元から取得し、ファイル名・版・アルゴリズムも確認してください。ハッシュ値の不一致だけで、どの違いが原因かを特定できるわけでもありません。
