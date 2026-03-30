---
title: "WASMで画像圧縮エンジン5つをブラウザに載せた話 — TinyPNGを超えるまで"
emoji: "🗜️"
type: "tech"
topics: ["wasm", "webassembly", "画像圧縮", "javascript", "webworker"]
published: false
---

## TinyPNGへの不満から始まった

Web制作の仕事をしていると、画像圧縮は避けて通れない。自分もずっとTinyPNGを使っていた。だが不満がある。

- **20枚/回の制限**。100枚の素材を処理するなら5回繰り返す
- **5MB/枚の制限**。一眼で撮った写真はまず弾かれる
- **画像がサーバーに送られる**。クライアントの機密画像を外部に送信するのは気持ち悪い
- **AVIF非対応**。2026年にもなってAVIF変換できない

「TinyPNGが使っているmozjpegもpngquantもOSSなんだから、ブラウザ上で動かせばいいのでは？」

そう思って作り始めた。結果、5つのWASMコーデックをブラウザに載せ、31MBのPNGを処理してもフリーズしないツールが出来上がった。しかもXserverのレンタルサーバー上で動いている。Node.jsもPHPも使っていない。

## なぜサーバーサイドではなくWASMか

画像圧縮をサーバーで処理するなら、VPSやクラウドに画像処理デーモンを立てるのが定石だ。だがこのアプローチには問題がある。

1. **プライバシー**: ユーザーの画像がサーバーに送信される。GDPRやクライアント機密を考えると避けたい
2. **サーバーコスト**: 画像処理はCPU負荷が高い。ユーザーが増えればインフラコストが青天井
3. **インフラ制約**: 自分のサイトはXserver（共用レンタルサーバー）で運用している。Node.jsプロセスは立てられないし、PHPで画像処理をやるのは現実的ではない

WASMなら全てユーザーのブラウザ内で完結する。サーバーには画像を1バイトも送らない。つまり**サーバーコスト0、プライバシー完全保護、レンタルサーバーでも動く**。

## アーキテクチャ概要

```
┌─────────────────────────────────────────────────────┐
│  Browser (Main Thread)                               │
│                                                       │
│  ユーザーがD&Dで画像を投入                              │
│       ↓                                               │
│  Canvas API で ImageData に変換                        │
│       ↓                                               │
│  Web Worker に postMessage (Transferable)             │
│       ↓                                               │
│  ┌─────────────────────────────────────────────┐      │
│  │  Web Worker (compress-worker.js)             │      │
│  │                                              │      │
│  │  format判定 → 対応コーデック呼び出し          │      │
│  │                                              │      │
│  │  JPEG → mozjpeg.wasm (246KB)                 │      │
│  │  PNG  → imagequant.wasm → oxipng.wasm        │      │
│  │  WebP → libwebp.wasm (275KB)                 │      │
│  │  AVIF → libaom.wasm (3.3MB, 遅延読込)       │      │
│  │                                              │      │
│  │  ← 圧縮バイナリを Transferable で返却         │      │
│  └─────────────────────────────────────────────┘      │
│       ↓                                               │
│  Blob URL生成 → ダウンロードリンク表示                  │
│  複数ファイルならJSZipでZIP化                           │
└─────────────────────────────────────────────────────┘

サーバー (Xserver): 静的ファイル配信のみ
  /wasm/*.js    ... esbuildバンドル済みESM
  /wasm/*.wasm  ... WASMバイナリ（.htaccessでMIME設定）
```

重要なのは、サーバーは**静的ファイルを配信するだけ**という点だ。圧縮処理は一切サーバーに到達しない。

## 各エンジンのWASM化

### 使ったもの: jSquash + imagequant

自前でEmscriptenビルドするのは最初に検討したが、メンテナンスコストが高すぎる。GoogleのSquooshプロジェクトからフォークされた[jSquash](https://github.com/nicholasgriffintn/jSquash)を使った。mozjpeg、oxipng、libwebp、libaomが既にWASMにコンパイルされてnpmパッケージとして配布されている。

PNG量子化（pngquant相当）には`@panda-ai/imagequant`を使った。libimagequantのwasm-bindgenビルドだ。

```
npm install @jsquash/jpeg @jsquash/png @jsquash/oxipng @jsquash/webp @jsquash/avif @panda-ai/imagequant
```

### CDNの罠と自サーバーホスト

最初はunpkg経由でCDN読み込みを試みた。結果は惨敗。

1. **unpkg**: CSPでブロックされる。`script-src`に追加しても、WASMのdynamic importが内部で別URLを叩くため連鎖的にブロック
2. **jsdelivr (+esm)**: ESMラッパーがWASMバイナリのパスを書き換え、`import.meta.url`ベースの解決が壊れる

教訓: **WASMコーデックはCDN経由で使うな。自サーバーにホストしろ。**

最終的に、esbuildで各パッケージの`index.js`をESMバンドルに変換し、WASMバイナリと一緒に`/wasm/`ディレクトリに配置した。

```bash
# esbuildでjSquashパッケージをプリバンドル
npx esbuild node_modules/@jsquash/jpeg/index.js \
  --bundle --format=esm --outfile=src/static/wasm/jpeg-encode.js

# WASMバイナリはそのままコピー
cp node_modules/@jsquash/jpeg/codec/enc/mozjpeg_enc.wasm src/static/wasm/
```

ここでハマったポイントがある。`encode.js`を直接バンドルすると`encode`関数がdefault exportになり、`module.encode`が`undefined`になる。**必ず`index.js`からバンドルする**こと。`index.js`経由なら`export { default as encode }`としてnamed exportが維持される。

### Xserverでの設定

レンタルサーバーでWASMを配信するには`.htaccess`で2つの設定が必要だ。

```apache
# WASM MIME type（これがないとContent-Type不正でロード失敗）
AddType application/wasm .wasm

# ブラウザキャッシュ1年（WASMバイナリは頻繁に変わらない）
<FilesMatch "\.wasm$">
    Header set Cache-Control "public, max-age=31536000, immutable"
</FilesMatch>
```

さらにCSPヘッダに`'wasm-unsafe-eval'`を追加しないと、WASMの`instantiate`がブロックされる。

```html
<meta http-equiv="Content-Security-Policy" content="
  script-src 'self' 'wasm-unsafe-eval' ...;
">
```

## PNG圧縮パイプライン: TinyPNGを再現する

TinyPNGの核心技術は**pngquant（カラー量子化） + oxipng（ロスレス最適化）の2段パイプライン**だ。これをブラウザ上で再現した。

```
原画像 (RGBA)
    ↓
imagequant: 24bit→8bit パレット量子化（256色）
    ↓  palette (Uint8Array) + indices (Uint8Array)
encodePngIndexed: インデックスPNG生成（PLTE+tRNS+IDAT）
    ↓  PNG ArrayBuffer
oxipng: ロスレス再圧縮（level 3）
    ↓
最終PNG
```

このパイプラインの中間ステップ――**インデックスPNGの生成**――が意外と面倒だった。imagequantはパレット配列とインデックス配列を返すだけで、PNGファイルは生成してくれない。自前でPNGエンコーダーを書く必要がある。

```javascript
// インデックスPNGの構造: Signature + IHDR + PLTE + tRNS + IDAT + IEND
async function encodePngIndexed(palette, indices, width, height) {
    // PLTEチャンク: RGB × numColors
    const plteData = new Uint8Array(numColors * 3);
    for (let i = 0; i < numColors; i++) {
        plteData[i * 3]     = palette[i * 4];
        plteData[i * 3 + 1] = palette[i * 4 + 1];
        plteData[i * 3 + 2] = palette[i * 4 + 2];
    }

    // tRNSチャンク: 透過情報（αチャンネル）
    // ← これを忘れると半透明PNGが真っ黒になる

    // IDATチャンク: フィルタバイト(0) + インデックス値を行ごとに格納
    // CompressionStreamでDeflate圧縮 → oxipngがさらに再圧縮する
    const cs = new CompressionStream('deflate');
    // ...
}
```

なお、oxipng単体（ロスレスのみ）では既に最適化されたPNGに対して効果がほぼない。macOSのスクリーンショットなどは元から最適化されているため、-0%〜-3%程度しか縮まない。pngquantの量子化ステップが入って初めてTinyPNG同等の圧縮率が出る。

## Web Worker化: 31MBでもフリーズしない

WASMの圧縮処理はCPUヘビーだ。メインスレッドで実行するとブラウザが完全にフリーズする。31MBのPNG画像なんかを食わせたら数十秒間操作不能になる。

解決策はWeb Worker。圧縮処理を別スレッドに逃がす。

```javascript
// compress-worker.js — Web Worker
self.onmessage = async function (e) {
    const { type, id, data } = e.data;

    if (type === 'init') {
        // 4つのコーデックを並列ロード
        const [mozjpeg, png, oxipng, webp] = await Promise.all([
            import('/wasm/jpeg-encode.js'),
            import('/wasm/png-encode.js'),
            import('/wasm/oxipng-optimise.js'),
            import('/wasm/webp-encode.js'),
        ]);
        self.postMessage({ type: 'ready' });
    }

    if (type === 'compress') {
        const result = await compressInWorker(data);
        // Transferable ObjectでArrayBufferをゼロコピー転送
        self.postMessage(
            { type: 'result', id, data: result },
            [result.buffer]
        );
    }
};
```

メインスレッド側ではWorkerの初期化に15秒のタイムアウトを設け、Workerが使えない環境ではCanvas APIにフォールバックする。

```javascript
// メインスレッド
compressWorker = new Worker('/wasm/compress-worker.js', { type: 'module' });

// 15秒以内にreadyが来なければCanvas APIフォールバック
setTimeout(() => {
    if (!loaded) {
        compressWorker.terminate();
        mode = 'standard'; // Canvas API
    }
}, 15000);
```

ポイントは**Transferable Object**の利用だ。`postMessage`の第2引数にArrayBufferを渡すと、メモリのコピーではなく所有権の移転が行われる。31MBの画像データをWorkerに送る際、コピーだと62MB+のメモリを消費するが、Transferableならオーバーヘッドはほぼゼロだ。

### AVIFは遅延読み込み

AVIFエンコーダー（libaom）のWASMバイナリは3.3MBある。他のコーデック（mozjpeg 246KB, libwebp 275KB）と比べて桁違いに大きい。初期ロードに含めると、ページ表示が遅くなる。

そのため、AVIFコーデックはユーザーがAVIFフォーマットを選択したタイミングで初めてロードする設計にした。

```javascript
// Worker内: AVIFは必要時にロード
async function loadAvifCodec() {
    if (avifCodec) return;
    avifCodec = await import('/wasm/avif-encode.js');
}
```

## パフォーマンス実測

| 入力 | フォーマット | 元サイズ | 圧縮後 | 削減率 |
|------|------------|---------|--------|--------|
| テスト用PNG (100x100) | PNG (pngquant+oxipng) | 12.3KB | 266B | **-98%** |
| スクリーンショットPNG | PNG (pngquant+oxipng) | 684KB | 232KB | **-66%** |
| 大型PNG (31.2MB) | WebP (libwebp) | 31.2MB | 841KB | **-97%** |

31MBのPNGをWebPに変換して97%削減。処理中もUIは完全にレスポンシブ。スクロールもクリックも問題なく動く。Web Workerの恩恵だ。

### TinyPNGとの比較

| 項目 | TinyPNG | 本ツール |
|------|---------|---------|
| JPEG圧縮エンジン | mozjpeg | mozjpeg (WASM) |
| PNG圧縮エンジン | pngquant + ? | imagequant + oxipng (WASM) |
| WebP | 対応 | libwebp (WASM) |
| AVIF | **非対応** | libaom (WASM) |
| ファイル数制限 | 20枚/回 | **無制限** |
| ファイルサイズ制限 | 5MB/枚 | **無制限** |
| 画像のサーバー送信 | あり | **なし（100%ブラウザ処理）** |
| API/有料プラン | あり（$25/月〜） | 不要 |
| オフライン動作 | 不可 | **可能（Service Worker）** |

## Gulp 5でハマった致命的バグ

これは本番で発覚したバグなので共有しておく。

Gulp 5はデフォルトのファイルエンコーディングが`utf8`に変更された。これが何を意味するか。**WASMバイナリがUTF-8として解釈されてバイト列が壊れる**。

開発中はDevToolsで見ても「WASMモード」と表示されていたから気付かなかった。実はCanvas APIフォールバックで動いていただけだった。デプロイ後にパフォーマンスが出ないのでログを見て初めて判明。

```javascript
// gulpfile の静的ファイルコピータスク
// BAD:  .pipe(dest('public')) — utf8でバイナリ破壊
// GOOD: .pipe(dest('public', { encoding: false })) — バイナリそのまま
```

Gulp 5を使っている人は、WASMに限らず画像やフォントなどのバイナリファイルをコピーするタスクで`encoding: false`を忘れずに。

## まとめ

- **mozjpeg, pngquant(imagequant), oxipng, libwebp, libaom** の5エンジンをWASMでブラウザに搭載
- **Web Worker + Transferable Object** でメインスレッドブロックゼロ
- **esbuildプリバンドル + 自サーバーホスト** でCDN依存ゼロ
- **CSP `wasm-unsafe-eval`** と **.htaccessの`AddType application/wasm .wasm`** がレンサバ運用の鍵
- **Gulp 5の`encoding: false`** を忘れるとWASMバイナリが壊れる（実体験）

全処理がブラウザ内で完結するため、レンタルサーバーの静的サイトでもTinyPNG同等以上の圧縮ツールが運用できる。サーバーコスト追加ゼロ、プライバシー問題ゼロ。

実際にこの構成で運用しているツールはこちら:
https://webatives.com/tools/image-compress/
