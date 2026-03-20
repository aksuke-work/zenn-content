---
title: "静的サイトのデプロイ自動化 — rsync+SSH鍵でワンコマンドデプロイ"
emoji: "🚀"
type: "tech"
topics: ["rsync", "ssh", "gulp", "デプロイ"]
published: false
---

[ToolShare Lab](https://webatives.com/) はXserverの共用サーバーで動いている。Vercel や Netlify は使っておらず、`rsync` + SSH鍵でワンコマンドデプロイしている。「プラットフォームに依存しないデプロイ」の設計をまとめる。

## なぜrsyncなのか

静的サイトのデプロイには様々な選択肢がある。

| 方法 | メリット | デメリット |
|------|---------|-----------|
| Vercel / Netlify | 設定ゼロ、CDN自動 | プラットフォーム依存、独自ドメインSLAに注意 |
| GitHub Actions + rsync | 自動化、履歴管理 | Actions の設定が必要 |
| rsync直接 | シンプル、速い、どこにでも置ける | 手動実行（自動化は別途） |
| FTP | 直感的 | 遅い、差分転送なし |

[ToolShare Lab](https://webatives.com/) でrsyncを選んだ理由:
- Xserverの共用サーバーで動いている（Vercel等は使えない）
- **差分転送**なので変更ファイルだけ転送、速い
- `--delete` オプションでサーバーの不要ファイルを自動削除
- 150ページのビルド + 転送で **30秒以内**

## SSH鍵の設定

まずSSH鍵でパスワードなしログインできるようにする。

```bash
# ローカルでSSH鍵生成（Ed25519を推奨）
ssh-keygen -t ed25519 -C "deploy@toolsharelab" -f ~/.ssh/toolsharelab_deploy

# 公開鍵をサーバーに登録
ssh-copy-id -i ~/.ssh/toolsharelab_deploy.pub user@server.example.com

# または手動でサーバーの authorized_keys に追記
cat ~/.ssh/toolsharelab_deploy.pub | ssh user@server.example.com 'mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys'
```

`~/.ssh/config` で鍵を設定しておくと省略できる:

```
# ~/.ssh/config
Host toolsharelab
  HostName server.example.com
  User username
  IdentityFile ~/.ssh/toolsharelab_deploy
  IdentitiesOnly yes
```

接続テスト:

```bash
ssh toolsharelab "echo 'SSH connection OK'"
```

## deploy.sh の設計

```bash
#!/bin/bash
# deploy.sh

set -e  # エラーで即停止
set -o pipefail  # パイプ内のエラーも検出

# ===== 設定 =====
REMOTE_HOST="toolsharelab"  # ~/.ssh/config のHost名
REMOTE_PATH="/home/username/public_html/webatives/"
LOCAL_PATH="./dist/"
BACKUP_DIR="/home/username/backups/"

# ===== カラー出力 =====
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

echo -e "${YELLOW}🏗  Building...${NC}"
npx gulp build

echo -e "${YELLOW}📦  Creating backup on server...${NC}"
ssh "$REMOTE_HOST" "cp -r $REMOTE_PATH $BACKUP_DIR/backup_$(date +%Y%m%d_%H%M%S) 2>/dev/null || true"

echo -e "${YELLOW}🚀  Deploying...${NC}"
rsync -avz \
  --delete \
  --checksum \
  --exclude '.git' \
  --exclude '.DS_Store' \
  --exclude 'node_modules' \
  --exclude '*.map' \
  "$LOCAL_PATH" "$REMOTE_HOST:$REMOTE_PATH"

echo -e "${GREEN}✅  Deploy completed!${NC}"
echo -e "${GREEN}🌐  https://webatives.com/${NC}"
```

## rsyncのオプション解説

```bash
rsync -avz \
  --delete \
  --checksum \
  --exclude '.git' \
  --exclude '.DS_Store' \
  --exclude 'node_modules' \
  --exclude '*.map' \
  ./dist/ user@server:/path/to/public_html/
```

| オプション | 意味 |
|-----------|------|
| `-a` | アーカイブモード（パーミッション・タイムスタンプ保持） |
| `-v` | 転送ファイル一覧を表示（verbose） |
| `-z` | 転送時にgzip圧縮（帯域節約） |
| `--delete` | ローカルにないファイルをサーバーから削除 |
| `--checksum` | タイムスタンプではなくチェックサムで差分判定 |
| `--exclude` | 転送しないファイル/ディレクトリ |

**`--checksum` の重要性**: デフォルトの差分判定はタイムスタンプ。ビルドし直すとタイムスタンプが更新されて「変更なし」のファイルも転送される。`--checksum` にするとファイル内容のハッシュで判定するため、本当に変わったファイルだけ転送できる。

## npm scripts への組み込み

```json
// package.json
{
  "scripts": {
    "build": "NODE_ENV=production gulp build",
    "dev": "gulp dev",
    "deploy": "bash deploy.sh",
    "deploy:dry": "bash deploy.sh --dry-run"
  }
}
```

dry-run 対応版 deploy.sh:

```bash
#!/bin/bash
set -e

DRY_RUN=false
if [[ "$1" == "--dry-run" ]]; then
  DRY_RUN=true
  echo "=== DRY RUN MODE ==="
fi

RSYNC_OPTS="-avz --delete --checksum --exclude '.git' --exclude '.DS_Store'"

if [ "$DRY_RUN" = true ]; then
  RSYNC_OPTS="$RSYNC_OPTS --dry-run"
fi

# ビルド（dry-runでもビルドは実行）
npx gulp build

# デプロイ
rsync $RSYNC_OPTS \
  ./dist/ \
  toolsharelab:/home/username/public_html/webatives/

if [ "$DRY_RUN" = true ]; then
  echo "=== DRY RUN 完了: 実際には転送していません ==="
fi
```

実行方法:

```bash
# dry-run（何が転送されるか確認）
npm run deploy:dry

# 本番デプロイ
npm run deploy
```

## Gulp タスクとして統合する

Gulp タスクに組み込むとビルド + デプロイをシームレスに実行できる:

```javascript
// gulpfile.mjs
import { execSync } from 'child_process';

function deploy(done) {
  try {
    execSync('bash deploy.sh', { stdio: 'inherit' });
    done();
  } catch (err) {
    done(err);
  }
}

// gulp deploy でビルド + デプロイ
export { deploy };
```

```bash
npx gulp deploy
```

## デプロイ前チェックリスト

毎回手動で確認するのは面倒なので、スクリプトに組み込む:

```bash
#!/bin/bash
# pre-deploy-check.sh

ERRORS=0

# distディレクトリが存在するか
if [ ! -d "./dist" ]; then
  echo "ERROR: dist ディレクトリがありません。先にビルドしてください。"
  ERRORS=$((ERRORS + 1))
fi

# index.html が存在するか
if [ ! -f "./dist/index.html" ]; then
  echo "ERROR: dist/index.html がありません。"
  ERRORS=$((ERRORS + 1))
fi

# CSS が存在するか
if [ ! -f "./dist/css/style.css" ]; then
  echo "ERROR: dist/css/style.css がありません。"
  ERRORS=$((ERRORS + 1))
fi

# SSHの疎通確認
if ! ssh -o ConnectTimeout=5 toolsharelab "echo ok" > /dev/null 2>&1; then
  echo "ERROR: SSH接続に失敗しました。"
  ERRORS=$((ERRORS + 1))
fi

if [ $ERRORS -gt 0 ]; then
  echo "デプロイ前チェックに失敗しました。($ERRORS 件のエラー)"
  exit 1
fi

echo "デプロイ前チェック OK"
```

## サーバー側の確認

デプロイ後にHTTPSでアクセスできるか確認:

```bash
# デプロイ後の疎通確認
curl -I https://webatives.com/ | head -5
curl -I https://webatives.com/tools/income-tax/ | head -5
```

```
HTTP/2 200
server: nginx
content-type: text/html; charset=UTF-8
```

## .envでサーバー情報を管理

```bash
# .env（gitにコミットしない）
REMOTE_HOST=toolsharelab
REMOTE_PATH=/home/username/public_html/webatives/
LOCAL_PATH=./dist/
```

```bash
# deploy.sh から読み込む
source .env

rsync -avz --delete \
  "$LOCAL_PATH" \
  "$REMOTE_HOST:$REMOTE_PATH"
```

`.gitignore` に `.env` を追加するのを忘れずに。

## まとめ

rsync + SSH鍵によるデプロイの要点:

1. **SSH鍵**でパスワードなしログイン → スクリプトから自動実行できる
2. **`~/.ssh/config`** にホスト設定 → コマンドがシンプルになる
3. **`--delete`** でサーバーとローカルを同期 → 削除したファイルがサーバーに残らない
4. **`--checksum`** で正確な差分判定 → 本当に変わったファイルだけ転送
5. **dry-run** で事前確認 → 本番デプロイのミスを防ぐ

[ToolShare Lab](https://webatives.com/) は150ページを毎回フルビルド + rsync転送しても30秒以内。Vercelのようなプラットフォームに依存せず、Xserverの共用サーバーで安定稼働している。シンプルで手堅いデプロイ方法として、静的サイトには十分すぎる構成だ。
