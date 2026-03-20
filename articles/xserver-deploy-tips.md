---
title: "Xserverで静的サイトを運用するTips — SSH+rsync+SSL設定"
emoji: "🖥️"
type: "tech"
topics: ["Xserver", "デプロイ", "SSH", "静的サイト"]
published: false
---

AND TOOLS（[and-tools.net](https://and-tools.net/)）をXserverで2年以上運用してきた。最初は「FTPでファイルをアップロード」という原始的な方法だったが、今はSSH + rsyncで自動デプロイが動いている。Xserver特有のつまずきポイントと解決策をまとめる。

## Xserverで静的サイトを運用する前提

Xserverは共有ホスティングだ。VPSではないため制約がある：

- root権限なし（sudo不可）
- インストールできるソフトウェアに制限あり
- Node.jsのバージョン管理は自前で行う
- SSHはデフォルトで有効（設定が必要）

反面、個人サイト運営に必要な機能は揃っている：

- SSH（ポート22 or 10022）
- rsync（最初から入っている）
- Let's Encrypt（無料SSL）
- cronジョブ（定期実行）

## SSH接続の設定

### SSHキーの登録

1. Xserverのサーバーパネルにログイン
2. SSH設定 → SSHキー登録
3. 公開鍵を貼り付けて登録

```bash
# ローカルでSSHキーを生成
ssh-keygen -t ed25519 -C "your-email@example.com"
cat ~/.ssh/id_ed25519.pub
```

生成された公開鍵（`.pub`ファイルの内容）をXserverのパネルに貼り付ける。

### ~/.ssh/configを設定する

毎回オプションを指定するのは面倒なのでconfigに書いておく。

```
# ~/.ssh/config
Host xserver-and-tools
  HostName sv12345.xserver.jp  # サーバーパネルで確認
  User xs123456                 # ユーザー名
  Port 10022                    # Xserverは10022を使う
  IdentityFile ~/.ssh/id_ed25519
  ServerAliveInterval 60
  ServerAliveCountMax 3
```

```bash
# 接続テスト
ssh xserver-and-tools
```

## rsyncでデプロイする

FTPよりrsyncが圧倒的に速くて安全だ。変更されたファイルだけを転送するので、100ファイルあっても変更が3ファイルなら3ファイルだけ転送する。

### 基本のrsyncコマンド

```bash
rsync -avz --delete \
  --exclude='.git/' \
  --exclude='.DS_Store' \
  --exclude='node_modules/' \
  ./dist/ \
  xserver-and-tools:~/and-tools.net/public_html/
```

**オプション解説:**
- `-a`: アーカイブモード（パーミッション・タイムスタンプを保持）
- `-v`: 詳細表示
- `-z`: 転送時に圧縮
- `--delete`: リモートにしか存在しないファイルを削除（ミラーリング）
- `--exclude`: 除外パターン

### Gulpと組み合わせる

[AND TOOLS](https://and-tools.net/) はGulp + 静的HTMLで構築しているため、Gulpのタスクにrsyncを組み込んでいる。

```javascript
// gulpfile.js
const { exec } = require('child_process');
const gulp = require('gulp');

function deploy(cb) {
  const command = [
    'rsync',
    '-avz',
    '--delete',
    '--exclude=\'.git/\'',
    '--exclude=\'.DS_Store\'',
    './dist/',
    'xserver-and-tools:~/and-tools.net/public_html/'
  ].join(' ');

  exec(command, (err, stdout, stderr) => {
    if (err) {
      console.error('デプロイ失敗:', stderr);
      cb(err);
      return;
    }
    console.log(stdout);
    cb();
  });
}

exports.deploy = gulp.series(
  'build', // 先にビルドを実行
  deploy
);
```

```bash
# ビルド + デプロイを一発で実行
gulp deploy
```

### ドライランで事前確認

実際に転送する前に何が変わるか確認できる。

```bash
# --dry-runで変更内容を確認（実際には転送しない）
rsync -avz --delete --dry-run \
  ./dist/ \
  xserver-and-tools:~/and-tools.net/public_html/
```

本番デプロイ前にドライランを確認する習慣をつけることをお勧めする。

## SSL証明書（Let's Encrypt）の設定

### Xserverの独自SSL

Xserverはサーバーパネルから無料SSL（Let's Encrypt）を設定できる。

1. サーバーパネル → SSL設定
2. ドメインを選択
3. 「無料独自SSL設定を追加する」をクリック

設定後、HTTPSでアクセスできるようになるが、HTTPからHTTPSへのリダイレクトは別途必要だ。

### .htaccessでHTTPSリダイレクト

```apache
# .htaccess（public_htmlのルートに配置）
<IfModule mod_rewrite.c>
  RewriteEngine On

  # HTTPからHTTPSにリダイレクト
  RewriteCond %{HTTPS} off
  RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [R=301,L]

  # wwwなしにリダイレクト（任意）
  RewriteCond %{HTTP_HOST} ^www\.(.+)$ [NC]
  RewriteRule ^(.*)$ https://%1/$1 [R=301,L]
</IfModule>
```

301リダイレクトはSEOの評価を引き継ぐ恒久的なリダイレクト。302（一時的）ではなく301を使う。

### SSL証明書の自動更新確認

Let's Encryptの証明書は90日で有効期限が切れる。Xserverは自動更新してくれるが、メール通知を確認する設定も入れておく。

```bash
# 証明書の有効期限を確認（SSH接続後）
openssl s_client -connect and-tools.net:443 -servername and-tools.net 2>/dev/null | openssl x509 -noout -dates
```

## Xserver特有の落とし穴

### .htaccessのApacheバージョン差異

XserverのApacheバージョンによって `.htaccess` の書き方が変わることがある。`mod_rewrite` の条件式で意図しない動作をする場合は、Xserverのサポートページでバージョンを確認する。

### Node.jsのバージョン管理

AND TOOLSのビルド（Gulp）はローカルで実行してdistをrsyncで転送する方式のため、サーバー側でNode.jsを実行する必要はない。

もしサーバー側でNode.jsを実行したい場合は、Xserverのサーバーパネルで「Node.js設定」からバージョンを選択できる（最近追加された機能）。

### ファイルパーミッション

rsyncで転送したファイルのパーミッションがサーバー側で正しく設定されているか確認する。

```bash
# SSH接続後にパーミッションを確認
ls -la ~/and-tools.net/public_html/

# ファイルは644、ディレクトリは755が基本
find ~/and-tools.net/public_html/ -type f -exec chmod 644 {} \;
find ~/and-tools.net/public_html/ -type d -exec chmod 755 {} \;
```

### キャッシュの問題

Xserverの標準機能にはキャッシュ機能がある（設定による）。デプロイ後に変更が反映されない場合は、ブラウザキャッシュとサーバーキャッシュの両方を確認する。

開発中は `Cache-Control: no-cache` ヘッダーを使って確認する：

```bash
curl -H "Cache-Control: no-cache" https://and-tools.net/ -I
```

## cronジョブで定期タスクを自動化

Xserverのcronジョブを使って定期的なタスクを自動化できる。

サーバーパネル → cronジョブ設定

```bash
# sitemap.xmlを毎日0時に更新するcron例
0 0 * * * /usr/bin/php ~/and-tools.net/public_html/generate-sitemap.php
```

PHPファイルを実行して動的にsitemap.xmlを生成・更新する使い方が一般的だ。

## デプロイパイプラインの完成形

AND TOOLSのデプロイフローをまとめる：

```
ローカル開発
  ↓
git commit（変更を記録）
  ↓
gulp build（静的ファイルを生成）
  ↓
rsync --dry-run（変更内容を確認）
  ↓
rsync（Xserverに転送）
  ↓
ブラウザで動作確認
  ↓
Search Consoleに更新URLを送信（必要な場合）
```

GitHub Actionsを使ったCI/CDも構築できる。pushすると自動でXserverにデプロイされるようにすることも可能だ。

```yaml
# .github/workflows/deploy.yml
name: Deploy to Xserver

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm install
      - run: npm run build
      - name: Deploy via rsync
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519
          echo "${{ secrets.SSH_KNOWN_HOSTS }}" > ~/.ssh/known_hosts
          rsync -avz --delete dist/ \
            ${{ secrets.XSERVER_USER }}@${{ secrets.XSERVER_HOST }}:~/and-tools.net/public_html/
```

## [ToolShare Lab](https://webatives.com/) での運用

ToolShare Labも同様のGulp + rsync構成でXserverに デプロイしている。150以上のツールページを持つ静的サイトだが、rsyncの差分転送のおかげでデプロイ時間は通常10〜30秒で完了する。

## まとめ

Xserverで静的サイトを快適に運用するためのポイント：

1. **SSH設定**: 公開鍵認証 + `~/.ssh/config` で快適なSSH環境
2. **rsync**: FTPは捨ててrsyncに切り替える
3. **SSL**: サーバーパネルからLet's Encryptを設定 + .htaccessでリダイレクト
4. **ドライラン**: 本番デプロイ前にrsync --dry-runで確認
5. **GitHub Actions**: pushで自動デプロイを構築してさらに効率化

Xserverは制約はあるが、個人サイト運営に必要なものは十分に揃っている。設定さえ整えれば快適な運用環境ができる。
