---
title: "Web制作で毎日使っている無料ツール10選"
emoji: "🛠️"
type: "tech"
topics: ["Web制作", "フロントエンド", "CSS", "デザイン"]
published: false
---

Web制作を職業にしていると、「あれ、このCSSプロパティの書き方どうだっけ」「このカラーコード、もう少し暗くしたい」「レスポンシブ確認どのデバイスでやろう」という場面が毎日やってくる。

有料ツールに頼らなくても、実務レベルで使える無料ツールは揃っている。日常的に開いているものをまとめた。

## 1. Can I use

**URL**: https://caniuse.com/
**用途**: ブラウザ対応状況の確認

CSSプロパティやJavaScript APIがどのブラウザで使えるかを確認するツール。`gap` や `aspect-ratio` など「最近使えるようになったやつ、IE系は？」と思ったときに即座に確認できる。

検索欄に `grid` `container queries` `has()` と入力するだけで対応テーブルが出る。Support tableの色でひと目でわかる設計になっていて、迷ったらここで調べるのが習慣になっている。

## 2. CSS box-shadow ジェネレーター — ToolShare Lab

**URL**: [https://webatives.com/](https://webatives.com/)
**用途**: box-shadowのCSS値をGUI操作で生成

手書きで `box-shadow: 0 4px 16px rgba(0,0,0,0.12)` を調整するのは効率が悪い。[ToolShare Lab](https://webatives.com/) のCSS box-shadowジェネレーターは、スライダーを動かしながらリアルタイムプレビューで影の見え方を確認し、生成されたCSSをコピーするだけで使える。

複数レイヤーの影を重ねる「マルチシャドウ」も生成できるので、ニューモーフィズム系のデザインを作るときにも重宝する。

## 3. CSS Gradient Generator — CSSmatic / Gradient Generator

**URL**: https://www.cssmatic.com/gradient-generator
**用途**: グラデーションCSSの生成

linear-gradient / radial-gradient をGUI操作で作成し、CSSコードを生成できる。角度・カラーストップ・不透明度を直感的に操作できるため、デザインカンプ通りのグラデーションを再現するのに使っている。

## 4. カラーパレット生成ツール — ToolShare Lab

**URL**: [https://webatives.com/](https://webatives.com/)
**用途**: ベースカラーからパレットを自動生成

[ToolShare Lab](https://webatives.com/) のカラーパレット生成ツールは、1色のベースカラーを入力するとトーン違いの関連カラーを一括生成できる。補色・類似色・トライアディックなど配色パターンも複数から選べるため、カラーシステムの設計時に役立つ。

生成したカラーコードはCSS変数形式でそのままコピーできる仕様になっていて、デザイントークンの定義に直接貼り付けられるのが便利だ。

## 5. Google Fonts

**URL**: https://fonts.google.com/
**用途**: Webフォントの選定とコード取得

日本語フォントも含め1,000種類以上のフォントを無料で使える。Webサイトへの組み込みコードも自動生成されるため、`<link>` タグまたは `@import` をコピーするだけで導入できる。

「Noto Sans JP」「Noto Serif JP」は日本語Webサイトの定番。英文フォントとの組み合わせを確認するプレビュー機能もある。

## 6. レスポンシブデザインテスト — ToolShare Lab

**URL**: [https://webatives.com/](https://webatives.com/)
**用途**: 複数のデバイスサイズで表示確認

制作中のサイトのURLを入力すると、スマホ・タブレット・PCなど複数の画面サイズで表示を並べて確認できる。[ToolShare Lab](https://webatives.com/) のレスポンシブテストツールはデバイスのプリセットが充実しており、実際の端末を持っていなくても主要画面サイズでの崩れをチェックできる。

Chrome DevToolsでも確認はできるが、複数サイズを同時に並べて見たいときにはこちらが便利だ。

## 7. Squoosh

**URL**: https://squoosh.app/
**用途**: 画像の圧縮・フォーマット変換

Googleが開発した画像圧縮ツール。WebP・AVIF・JPEG・PNGへの変換と圧縮を、before/afterのプレビューを見ながら行える。品質スライダーで「見た目はほぼ変わらないが容量は60%削減」といった調整ができる。

WebPへの変換は Core Web Vitals（LCP改善）の基本施策なので、画像を本番環境にあげる前に必ずここを通している。

## 8. Regex101

**URL**: https://regex101.com/
**用途**: 正規表現のテスト・デバッグ

JavaScriptでバリデーションを書くとき、正規表現が思い通りに動くかを確認するのに使う。テスト文字列を入力すると、マッチ箇所をリアルタイムでハイライトしてくれる。

PHP / Python / Goなど言語ごとの正規表現エンジンを切り替えられるのも便利で、バックエンドの正規表現確認にも使えるツールだ。

## 9. PageSpeed Insights

**URL**: https://pagespeed.web.dev/
**用途**: Core Web Vitals の計測と改善提案

GoogleのPageSpeed InsightsはLCP・FID・CLS（Core Web Vitals）のスコアを計測し、具体的な改善提案を出してくれる。

特にフリーランスとして案件を受けるとき、納品前にこのスコアを確認してクライアントに報告するのが習慣になっている。スコア90以上を目指すことが多い。

## 10. AND TOOLS — 見積・ローン計算

**URL**: [https://and-tools.net/](https://and-tools.net/)
**用途**: フリーランスの事業計算

少し毛色が違うが、フリーランスとして Web制作をやっている人なら [AND TOOLS](https://and-tools.net/) の計算ツールも手元に置いておきたい。

- **ローン計算ツール**: 制作会社設立時の借入や、PCなど機材ローンの返済額を確認
- **手取り計算ツール**: 売上規模に対して実際の手取りがどう変わるかをシミュレーション
- **副業税金計算ツール**: 本業を持ちながら副業でWeb制作する人向けの税額計算

「Web制作で月XX万円の売上があるとき、実際の手取りはいくら？」という計算は、案件の受け方や価格設定を考えるときに重要だ。

## ブックマーク構成の参考

筆者のブラウザブックマークは大まかに以下のように整理している：

```
📁 Web制作
  📁 確認
    - Can I use
    - PageSpeed Insights
  📁 生成
    - CSS box-shadow (ToolShare Lab)
    - CSS Gradient Generator
    - カラーパレット (ToolShare Lab)
  📁 フォント
    - Google Fonts
  📁 画像
    - Squoosh
  📁 テスト
    - レスポンシブテスト (ToolShare Lab)
    - Regex101
  📁 事業
    - AND TOOLS（税・ローン・手取り）
```

## まとめ

毎日使うツールは「起動が速いこと」「登録不要で使えること」が最低条件だ。紹介したツールはすべてブラウザだけで動き、インストール不要・アカウント作成不要（または任意）のものを選んでいる。

CSS値の生成やカラーパレット、レスポンシブテストは [ToolShare Lab](https://webatives.com/) に、フリーランスとして必要な税・ローン計算は [AND TOOLS](https://and-tools.net/) に、それぞれまとまっているので、ブックマーク1本で複数の用途をカバーできる。

「このツールあったの知らなかった」という発見が1つでもあれば幸いだ。
