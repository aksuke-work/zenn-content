---
title: "ボットトラフィックの見分け方 — GAの数字が信用できない時の対処法"
emoji: "🤖"
type: "tech"
topics: ["アクセス解析", "GoogleAnalytics", "セキュリティ", "個人開発"]
published: false
---

Google Analytics 4のデータを見ていて「なんかおかしい」と感じたことはないか。突然のPVスパイク、妙に高い直帰率、セッション時間が0秒のユーザーが大量に来る——これらはボットトラフィックのサインだ。[AND TOOLS](https://and-tools.net/) の運営でこの問題に直面して対処した経験をまとめる。

## ボットトラフィックとは

ボット（クローラー・スパイダー・スクレイパー等）が生成するアクセスで、実際のユーザーではないものを指す。種類によって目的が異なる：

**善意のボット:**
- Googlebot、Bingbot（検索エンジンのクロール）
- ClaudeBot、GPTBot（LLMの学習クロール）
- Pingdom、GTmetrixなどの監視ツール

**悪意・不必要なボット:**
- スパムリファラー（GAデータを汚染する）
- コンテンツスクレイパー（サイトコンテンツを無断コピー）
- DDoS攻撃ボット
- 脆弱性スキャナー

Google Analytics 4はデフォルトで「既知のボットと思われるアクセスを除外」する設定があるが、完全ではない。特にスパムリファラーや巧妙なボットはGAデータに紛れ込む。

## ボットトラフィックの見分け方

### サイン1: セッション時間が異常に短い

GA4の「エンゲージメント時間」が0秒〜1秒のセッションが大量にある場合はボットを疑う。人間が1秒以内にページを読んで離脱することはほとんどない。

確認方法（GA4）:
1. 探索 → 自由形式レポート
2. ディメンション: セッションのデフォルトチャネルグループ
3. 指標: セッション + 平均エンゲージメント時間

エンゲージメント時間が0秒のチャネルはボットの可能性が高い。

### サイン2: 見知らぬ参照元からのトラフィック

GA4 → 集客 → トラフィック獲得 で参照元(Referral)を確認する。

**ボットの典型的な参照元パターン:**
- `semalt.com`, `buttons-for-website.com` 等のスパムドメイン
- ランダムな文字列のドメイン
- 参照元がロシア・東欧のドメインで中身がギャンブル・アダルト系

これらを見つけたらGAのフィルタリングで除外する。

### サイン3: 突然の大量アクセス

AND TOOLSで一度、深夜に通常の10倍のPVが記録されたことがあった。翌日確認するとサーバーログに同一IPからの大量リクエストがあり、コンテンツスクレイパーだった。

**サーバーログを確認する（Xserverの場合）:**

Xserverの管理パネルから生のアクセスログをダウンロードできる。

```bash
# アクセスログを解析
# 同一IPのアクセス数を集計
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -20

# 特定のUser-Agentを確認
grep "python-requests" access.log | head -20
```

スクレイパーはしばしば `python-requests`、`Go-http-client`、`curl` といったUser-Agentを使う。

### サイン4: リアルタイムレポートに大量のユーザー

GA4のリアルタイムレポートで、どのページも数秒で大量のユーザーが来ては消える場合、スパムボットがGAのメジャメントプロトコルを直接叩いている可能性がある。

## GA4でボットを除外する

### 設定1: ボットフィルタリングをONにする

GA4では管理 → データストリーム → 詳細設定で「Googleが識別したボットやスパイダーからのイベントを除外する」オプションを確認する。デフォルトでONのはずだが、確認する。

### 設定2: IPアドレスの除外

自分のIPアドレス（自宅・会社）からのアクセスを除外する。

GA4 → 管理 → データストリーム → タグの設定を構成 → 内部トラフィックの定義

```javascript
// gtag.jsでの内部トラフィック設定例
gtag('config', 'G-XXXXXXXXXX', {
  'traffic_type': 'internal'
});
```

### 設定3: フィルタの作成

スパムリファラーが判明している場合はGA4でフィルタを作成する。

GA4 → 管理 → データフィルタ → フィルタを作成

「参照元」が特定ドメインの場合に除外するフィルタを設定できる。ただしGA4のフィルタは遡及的には適用されない（過去データは変わらない）。

## サーバー側でボットをブロックする

GAでの除外だけでなく、サーバー側でボットをブロックすることも重要だ。サーバーリソースの無駄遣いを防げる。

### .htaccessでUser-Agentをブロック

AND TOOLSはXserverで運営しているため `.htaccess` でブロックできる。

```apache
# .htaccess
<IfModule mod_rewrite.c>
  RewriteEngine On

  # 悪質なUser-Agentをブロック
  RewriteCond %{HTTP_USER_AGENT} (semalt|buttons-for-website|priceg|domainscan) [NC]
  RewriteRule .* - [F,L]

  # 空のUser-Agentをブロック
  RewriteCond %{HTTP_USER_AGENT} ^$
  RewriteRule .* - [F,L]
</IfModule>
```

**注意:** LLMのクローラー（ClaudeBot、GPTBot）は許可しておく。SEOにはGooglebotだけでなくLLMのクローラーへの対応も重要になってきている。

### rate-limitingの設定

同一IPからの過度なアクセスをブロックする。

```apache
# .htaccessでmod_evasive的な設定（Xserverは設定が限られる）
# より厳密には管理パネルのセキュリティ設定を活用
```

Xserverの場合はサーバーパネルの「WAF設定」を有効にすることで、基本的なDDoS・スパムをブロックできる。

## Cloudflareを挟む

より本格的にボット対策をする場合はCloudflareのFreeプランが効果的だ。AND TOOLSはXserverで運用しているが、DNSをCloudflareに向けることで：

- ボットフィルタリング
- DDoS保護
- 悪質なIPのブロック
- ブラウザチャレンジ（人間かどうか確認）

が無料で利用できる。

```
Cloudflare無料プランで使える機能:
- ボット管理（基本）
- WAF（基本ルール）
- DDoS保護（無制限）
- SSL/TLS
- キャッシュ
```

ただしCloudflareを挟むとGooglebotへの影響を懸念する声もある。Cloudflareは検索エンジンクローラーを識別して通過させる仕組みを持っているが、設定を確認する必要がある。

## GAのデータをクリーンに保つ方法

### BigQueryへのエクスポートで元データを保全

GA4はBigQueryと連携してデータをエクスポートできる（Google Cloud Freeティアで可能）。フィルタリングが適用される前の生データがBigQueryに保存されるので、後から除外条件を変えて分析できる。

```sql
-- BigQueryでセッション時間が0秒のセッションを除外
SELECT
  event_date,
  COUNT(DISTINCT session_id) as sessions
FROM `project.dataset.events_*`
WHERE
  event_name = 'session_start'
  AND user_engagement_time_msec > 1000 -- 1秒以上
GROUP BY 1
ORDER BY 1
```

### Looker Studioでフィルタリング済みレポートを作る

GA4のデータをLooker Studio（旧Googleデータポータル）に繋いで、ボットっぽいセッションを除外した独自ダッシュボードを作る。

```
除外条件の例:
- セッション継続時間 < 1秒
- 直帰率 = 100% かつ PV = 1
- 特定の参照元ドメイン
```

## [ToolShare Lab](https://and-and.net/) での対策事例

ToolShare Labは150以上のツールを持つ静的サイトだ。大量のページがあるためスクレイパーのターゲットになりやすい。実施した対策：

1. **Cloudflare導入**: DNSをCloudflareに向けてボット管理を有効化
2. **robots.txtの整備**: 許可するクローラーを明示
3. **サーバーログ定期チェック**: 月1回アクセスログを確認
4. **GA4フィルタ**: スパムリファラーを除外するフィルタを複数設定

## まとめ

GAのデータが「信用できない」と感じたら：

1. **GA4でボットフィルタリングを確認**する
2. **参照元レポート**でスパムドメインを探す
3. **サーバーログ**で異常なUser-AgentやIPを特定する
4. **GAフィルタ**でスパムを除外する
5. 大規模対策は**Cloudflare**を挟む

データは戦略の根拠になる。ボットで汚染されたデータで判断すると、間違った施策に時間を使うことになる。定期的なデータ品質チェックを習慣化することを勧める。
