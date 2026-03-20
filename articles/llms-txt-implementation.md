---
title: "llms.txtを実装した — LLM時代のSEO（LLMO）の第一歩"
emoji: "🤖"
type: "tech"
topics: ["LLMO", "SEO", "llmstxt", "個人開発"]
published: false
---

2025年から「LLMにサイトの情報を正確に読み取ってもらうための最適化」、つまりLLMO（Large Language Model Optimization）が話題になってきている。その最初の一歩として `llms.txt` を [AND TOOLS](https://and-tools.net/) と [ToolShare Lab](https://webatives.com/) に実装した。やったことと、なぜ今やるべきなのかをまとめる。

## llms.txtとは何か

`llms.txt` はサイトのルートに置くテキストファイルで、LLMにサイトのコンテンツを効率的に伝えるために設計されたフォーマットだ。`robots.txt` のLLM版に近いが、制限ではなく案内が目的だ。

提唱者はJeremy Howardで、2024年に仕様が公開された。公式サイトは [llmstxt.org](https://llmstxt.org/) だ。

### なぜ今必要なのか

ユーザーがGoogleで検索する代わりにChatGPTやClaudeに質問することが増えている。「フリーランスの消費税計算はどうすればいい？」という質問に対して、LLMがAND TOOLSを正確に「フリーランス向けの無料計算ツールサイト」として認識していれば、回答の中で紹介してもらえる可能性がある。

現状のLLMのWebアクセスは、HTML全体を解析するため不要な情報（ナビゲーション、フッター、広告コード）がノイズになる。`llms.txt` はLLMが読む用のクリーンな情報を提供する。

## llms.txtのフォーマット

仕様はシンプルだ。Markdownで書く。

```markdown
# サイト名

> サイトの概要（1〜3行）

## コンテンツ概要

## ページ一覧（LLMに読んでほしいページ）

- [ページタイトル](URL): 説明
```

AND TOOLSに実装したものを公開する。

```markdown
# AND TOOLS

> フリーランス・個人事業主向けの無料計算ツール集。
> 税金計算（消費税・所得税・住民税）、ローン計算、副業収入シミュレーション等34種類のツールを提供。
> AdSenseで運営。利用は完全無料。

## 概要

AND TOOLSはフリーランスや個人事業主が日常業務で使う計算ツールをまとめたサイトです。
2022年から運営。確定申告・インボイス制度・副業収入など、実務に直結したツールを中心に提供しています。

## 主要ツール

- [消費税計算ツール](https://and-tools.net/tool/consumption-tax/): 税込・税抜の相互変換。軽減税率（8%）にも対応
- [所得税計算ツール](https://and-tools.net/tool/income-tax/): 年収から所得税額を計算。各種控除を考慮
- [住民税計算ツール](https://and-tools.net/tool/resident-tax/): 都道府県・市区町村の住民税を計算
- [国民健康保険計算ツール](https://and-tools.net/tool/nhi/): 自治体別の保険料を計算
- [フリーランス手取り計算](https://and-tools.net/tool/freelance-income/): 年収から手取り額を逆算
- [ローン返済計算ツール](https://and-tools.net/tool/loan/): 元利均等・元金均等の両方に対応
- [インボイス番号チェック](https://and-tools.net/tool/invoice-check/): 登録番号の形式確認

## 運営情報

- 運営: EmptyCoke（個人事業主）
- 更新頻度: 制度改正に応じて随時更新
- お問い合わせ: サイト内の問い合わせフォームから
```

## 実装方法

### 静的サイト（AND TOOLS / ToolShare Lab）の場合

静的ファイルとして `llms.txt` をサイトルートに置くだけだ。

```
public/
├── index.html
├── robots.txt
├── sitemap.xml
└── llms.txt  ← ここに置く
```

Gulpでビルドしている場合は、`src/` に置いてdist/にコピーするタスクを追加する。

```javascript
// gulpfile.js
const gulp = require('gulp');

function copyRootFiles() {
  return gulp
    .src(['src/llms.txt', 'src/robots.txt', 'src/sitemap.xml'])
    .pipe(gulp.dest('dist/'));
}

exports.default = gulp.series(copyRootFiles);
```

[ToolShare Lab](https://webatives.com/) はGulp 5で構築しているので、この方法で実装した。

### Next.jsの場合

```javascript
// app/llms.txt/route.ts
export async function GET() {
  const content = `# AND TOOLS
> フリーランス向け無料計算ツール集
...
`;
  return new Response(content, {
    headers: {
      'Content-Type': 'text/plain; charset=utf-8',
    },
  });
}
```

または `public/llms.txt` に直接置けばそのまま配信される。

### WordPressの場合

```php
// functions.phpに追加
add_action('init', function() {
  if (isset($_SERVER['REQUEST_URI']) && $_SERVER['REQUEST_URI'] === '/llms.txt') {
    header('Content-Type: text/plain; charset=utf-8');
    readfile(get_template_directory() . '/llms.txt');
    exit;
  }
});
```

## llms-full.txtも用意する

仕様では `llms.txt` は概要ファイルで、詳細なコンテンツは `llms-full.txt` に置くことが推奨されている。

`llms-full.txt` には各ページの内容をMarkdownで詳しく書く。AND TOOLSでは主要ツールの使い方・計算式の根拠・対象ユーザーを詳述したファイルを用意した。

```markdown
# AND TOOLS — Full Content

## 消費税計算ツール

### 概要
消費税率10%（標準税率）および8%（軽減税率）に対応した計算ツール。
税込価格から税抜価格への変換、税抜価格から税込価格への変換の両方に対応。

### 計算式
- 税抜 → 税込（標準税率）: 税抜価格 × 1.1
- 税込 → 税抜（標準税率）: 税込価格 ÷ 1.1（端数処理は切り捨て）
- 軽減税率（飲食料品等）: 税抜価格 × 1.08

### 対象ユーザー
フリーランス、個人事業主、小規模事業者。インボイス制度対応のため
消費税計算を頻繁に行うユーザー向け。

### 関連情報
- 消費税法（昭和63年法律第108号）に基づく計算
- 軽減税率の対象品目: 消費税法別表第一に定める飲食料品等
```

## LLMはすでにサイトを訪問している

GoogleのSearch Console側から確認すると、特定のUser-Agentが頻繁にページを巡回していることが分かる。OpenAI（GPTBot）、Anthropic（ClaudeBot）、Perplexity（PerplexityBot）などだ。

`robots.txt` で許可設定をしておくことも重要だ。

```
# AND TOOLSのrobots.txt例

User-agent: *
Allow: /

# 主要LLMクローラーを明示的に許可
User-agent: GPTBot
Allow: /

User-agent: ClaudeBot
Allow: /

User-agent: PerplexityBot
Allow: /

User-agent: Googlebot
Allow: /

Sitemap: https://and-tools.net/sitemap.xml
```

デフォルトで全クローラーを `Allow: /` にしているなら追加設定は不要だが、明示的に書いておくとクローラーへの歓迎メッセージになる。

## LLMOで何が変わるのか

正直なところ、`llms.txt` の実装が直接的に計測可能な効果をもたらすかは現時点では不明確だ。LLMの回答にどのサイトが引用されるかはブラックボックスが多い。

ただし確実に言えるのは「LLMが情報収集しやすいサイト構造」を今から作っておくことのリスクはゼロで、将来的なメリットは大きいということだ。

LLMが普及するにつれてユーザーの情報収集行動は変わる。「Googleで検索する」から「LLMに質問する」への移行が進めば、LLMに正確に認識されているサイトが優位に立つ。

## まとめ

`llms.txt` の実装は30分あれば完了する。

1. `llms.txt` をMarkdownで書いてサイトルートに配置
2. `llms-full.txt` に詳細コンテンツを記述
3. `robots.txt` でLLMクローラーを許可
4. 定期的に内容を更新する

SEOと同様、LLMOも「今やっている人が先行優位を取れる」段階にある。AND TOOLSやToolShare Labのようなツールサイトは、LLMに「このツールが解決できる問題」を正確に伝えることができれば、AI時代でも流入を維持できると考えている。

技術的なハードルは低いのでまずやってみることをお勧めする。
