---
name: vite-patterns
description: Viteビルドツールのパターン（設定、プラグイン、HMR、環境変数、プロキシ設定、SSR、ライブラリモード、依存関係プリバンドル、ビルド最適化）。vite.config.ts、Viteプラグイン、またはViteベースのプロジェクトを扱う際に有効化する。
origin: ECC
---

# Vite パターン

Vite 8+ プロジェクト向けのビルドツール・開発サーバーパターン集。設定、環境変数、プロキシ設定、ライブラリモード、依存関係プリバンドル、および本番環境でよくある落とし穴を網羅する。

## 使用するタイミング

- `vite.config.ts` または `vite.config.js` の設定
- 環境変数や `.env` ファイルのセットアップ
- APIバックエンド向けの開発サーバープロキシ設定
- ビルド出力の最適化（チャンク、ミニファイ、アセット）
- `build.lib` を用いたライブラリの公開
- 依存関係プリバンドルや CJS/ESM 相互運用の問題解決
- HMR、開発サーバー、またはビルドエラーのデバッグ
- Vite プラグインの選定・順序付け

## 仕組み

- **開発モード**ではソースファイルをネイティブ ESM として提供し、バンドルは行わない。変換はモジュールリクエストごとにオンデマンドで実行されるため、コールドスタートが速く HMR が精密に行われる。
- **ビルドモード**では Rolldown（v7+）または Rollup（v5〜v6）を使用してアプリを本番向けにバンドルし、ツリーシェイキング、コード分割、Oxc ベースのミニファイを適用する。
- **依存関係プリバンドル**では esbuild を用いて CJS/UMD の依存関係を ESM へ一度だけ変換し、結果を `node_modules/.vite` にキャッシュするため、以降の起動ではその処理をスキップする。
- **プラグイン**は開発とビルドにわたって統一されたインターフェースを共有する——同一のプラグインオブジェクトが、開発サーバーのオンデマンド変換と本番パイプラインの両方で機能する。
- **環境変数**はビルド時に静的にインライン化される。`VITE_` プレフィックス付き変数はバンドル内の公開定数となり、プレフィックスなしのものはクライアントコードには見えない。

## 例

### 設定の構造

#### 基本設定

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: { '@': new URL('./src', import.meta.url).pathname },
  },
})
```

#### 条件付き設定

```typescript
// vite.config.ts
import { defineConfig, loadEnv } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig(({ command, mode }) => {
  const env = loadEnv(mode, process.cwd())   // VITE_ prefixed only (safe)

  return {
    plugins: [react()],
    server: command === 'serve' ? { port: 3000 } : undefined,
    define: {
      __API_URL__: JSON.stringify(env.VITE_API_URL),
    },
  }
})
```

#### 主要な設定オプション

| キー | デフォルト | 説明 |
|-----|---------|-------------|
| `root` | `'.'` | プロジェクトルート（`index.html` の場所） |
| `base` | `'/'` | デプロイされたアセットの公開ベースパス |
| `envPrefix` | `'VITE_'` | クライアントに公開する環境変数のプレフィックス |
| `build.outDir` | `'dist'` | 出力ディレクトリ |
| `build.minify` | `'oxc'` | ミニファイアー（`'oxc'`、`'terser'`、または `false`） |
| `build.sourcemap` | `false` | `true`、`'inline'`、または `'hidden'` |

### プラグイン

#### 主要プラグイン

プラグインのニーズの大半は、メンテナンスが行き届いた少数のパッケージで満たせる。独自のプラグインを書く前にこれらを検討すること。

| プラグイン | 目的 | 使用タイミング |
|--------|---------|-------------|
| `@vitejs/plugin-react-swc` | SWC による React HMR + Fast Refresh | React アプリのデフォルト（Babel 版より高速） |
| `@vitejs/plugin-react` | Babel による React HMR + Fast Refresh | Babel プラグインが必要な場合のみ（emotion、MobX デコレータ） |
| `@vitejs/plugin-vue` | Vue 3 SFC サポート | Vue アプリ |
| `vite-plugin-checker` | HMR オーバーレイ付きワーカースレッドで `tsc` + ESLint を実行 | **TypeScript アプリ全般**——`vite build` 中は型チェックを行わない |
| `vite-tsconfig-paths` | `tsconfig.json` の `paths` エイリアスを使用 | すでに `tsconfig.json` にエイリアスがある場合 |
| `vite-plugin-dts` | ライブラリモードで `.d.ts` ファイルを生成 | TypeScript ライブラリの公開時 |
| `vite-plugin-svgr` | SVG を React コンポーネントとしてインポート | SVG をコンポーネントとして使用する React アプリ |
| `rollup-plugin-visualizer` | バンドルのツリーマップ/サンバーストレポート | 定期的なバンドルサイズ監査（`enforce: 'post'` を使用） |
| `vite-plugin-pwa` | ゼロ設定 PWA + Workbox | オフライン対応アプリ |

**重要な注意点:** `vite build` はトランスパイルのみを行い、型チェックは**行わない**。`vite-plugin-checker` を追加するか CI で `tsc --noEmit` を実行しない限り、型エラーはサイレントに本番環境へ届いてしまう。

#### カスタムプラグインの作成

作成が必要になるケースは稀で、既存プラグインで大半のニーズは満たせる。必要な場合は `vite.config.ts` にインラインで記述し始め、再利用する場合にのみ抽出すること。

```typescript
// vite.config.ts — minimal inline plugin
function myPlugin(): Plugin {
  return {
    name: 'my-plugin',                       // required, must be unique
    enforce: 'pre',                           // 'pre' | 'post' (optional)
    apply: 'build',                           // 'build' | 'serve' (optional)
    transform(code, id) {
      if (!id.endsWith('.custom')) return
      return { code: transformCustom(code), map: null }
    },
  }
}
```

**主要フック:** `transform`（ソース変換）、`resolveId` + `load`（仮想モジュール）、`transformIndexHtml`（HTML への注入）、`configureServer`（開発ミドルウェア追加）、`hotUpdate`（カスタム HMR——v7+ で非推奨の `handleHotUpdate` を置き換え）。

**仮想モジュール**には `\0` プレフィックスの規約を使用する——`resolveId` は `'\0virtual:my-id'` を返し、他のプラグインがスキップするようにする。ユーザーコードは `'virtual:my-id'` としてインポートする。

プラグイン API の全容は [vite.dev/guide/api-plugin](https://vite.dev/guide/api-plugin) を参照。開発中に変換パイプラインをデバッグするには `vite-plugin-inspect` を使用する。

### HMR API

フレームワークプラグイン（`@vitejs/plugin-react`、`@vitejs/plugin-vue` 等）は HMR を自動的に処理する。カスタム状態ストア、開発ツール、または更新をまたいで状態を保持する必要があるフレームワーク非依存のユーティリティを構築する場合にのみ `import.meta.hot` を直接使用すること。

```typescript
// src/store.ts — manual HMR for a vanilla module
if (import.meta.hot) {
  // Persist state across updates (must MUTATE, never reassign .data)
  import.meta.hot.data.count = import.meta.hot.data.count ?? 0

  // Cleanup side effects before module is replaced
  import.meta.hot.dispose((data) => clearInterval(data.intervalId))

  // Accept this module's own updates
  import.meta.hot.accept()
}
```

`import.meta.hot` のコードはすべて本番ビルドでツリーシェイクされるため、ガード削除は不要。

### 環境変数

Vite は `.env`、`.env.local`、`.env.[mode]`、`.env.[mode].local` をその順番で読み込み（後の方が優先）、`*.local` ファイルは gitignore され、ローカルシークレット用として使われる。

#### クライアントサイドアクセス

`VITE_` プレフィックス付き変数のみがクライアントコードに公開される:

```typescript
import.meta.env.VITE_API_URL   // string
import.meta.env.MODE            // 'development' | 'production' | custom
import.meta.env.BASE_URL        // base config value
import.meta.env.DEV             // boolean
import.meta.env.PROD            // boolean
import.meta.env.SSR             // boolean
```

#### 設定内での環境変数の使用

```typescript
// vite.config.ts
import { defineConfig, loadEnv } from 'vite'

export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd())          // VITE_ prefixed only (safe)
  return {
    define: {
      __API_URL__: JSON.stringify(env.VITE_API_URL),
    },
  }
})
```

### セキュリティ

#### `VITE_` プレフィックスはセキュリティ境界ではない

`VITE_` プレフィックスの付いた変数はすべて**ビルド時にクライアントバンドルへ静的にインライン化**される。ミニファイ、base64 エンコード、ソースマップの無効化によっては隠せない。意図的な攻撃者は配布された JavaScript から任意の `VITE_` 変数を抽出できる。

**ルール:** 公開値（API URL、フィーチャーフラグ、公開鍵）のみを `VITE_` 変数に入れる。シークレット（API トークン、データベース URL、秘密鍵）は必ずサーバーサイドの API またはサーバーレス関数の裏側に置くこと。

#### `loadEnv('')` の罠

```typescript
// BAD: passing '' as the third arg loads ALL env vars — including server secrets —
// and makes them available to inline into client code via `define`.
const env = loadEnv(mode, process.cwd(), '')

// GOOD: explicit prefix list
const env = loadEnv(mode, process.cwd(), ['VITE_', 'APP_'])
```

#### 本番環境のソースマップ

本番ソースマップは元のソースコードを漏洩させる。エラートラッカー（Sentry、Bugsnag）にアップロードしてローカルから削除する場合を除き、無効化すること:

```typescript
build: {
  sourcemap: false,                                  // default — keep it this way
}
```

#### `.gitignore` チェックリスト

- `.env.local`、`.env.*.local` — ローカルシークレットの上書き
- `dist/` — ビルド出力
- `node_modules/.vite` — プリバンドルキャッシュ（古いエントリが幻のエラーを引き起こす）

### サーバープロキシ

```typescript
// vite.config.ts — server.proxy
server: {
  proxy: {
    '/foo': 'http://localhost:4567',                    // string shorthand

    '/api': {
      target: 'http://localhost:8080',
      changeOrigin: true,                               // needed for virtual-hosted backends
      rewrite: (path) => path.replace(/^\/api/, ''),
    },
  },
}
```

WebSocket プロキシには、ルート設定に `ws: true` を追加する。

### ビルド最適化

#### マニュアルチャンク

```typescript
// vite.config.ts — build.rolldownOptions
build: {
  rolldownOptions: {
    output: {
      // Object form: group specific packages
      manualChunks: {
        'react-vendor': ['react', 'react-dom'],
        'ui-vendor': ['@radix-ui/react-dialog', '@radix-ui/react-popover'],
      },
    },
  },
}
```

```typescript
// Function form: split by heuristic
manualChunks(id) {
  if (id.includes('node_modules/react')) return 'react-vendor'
  if (id.includes('node_modules')) return 'vendor'
}
```

### パフォーマンス

#### バレルファイルの回避

バレルファイル（ディレクトリ内のすべてを再エクスポートする `index.ts`）は、単一のシンボルをインポートしても Vite に再エクスポートされたすべてのファイルを読み込ませてしまう。公式ドキュメントが指摘する開発サーバー低速化の最大原因 #1 がこれだ。

```typescript
// BAD — importing one util forces Vite to load the whole barrel
import { slash } from '@/utils'

// GOOD — direct import, only the one file is loaded
import { slash } from '@/utils/slash'
```

#### インポート拡張子を明示する

暗黙の拡張子ごとに `resolve.extensions` を通じて最大 6 回のファイルシステム確認が走る。大規模なコードベースでは積み重なる。

```typescript
// BAD
import Component from './Component'

// GOOD
import Component from './Component.tsx'
```

`tsconfig.json` の `allowImportingTsExtensions` と `resolve.extensions` を実際に使用する拡張子のみに絞ること。

#### ホットパスルートのウォームアップ

`server.warmup.clientFiles` はブラウザがリクエストする前に既知のホットエントリを事前変換し、大規模アプリでのコールドロードリクエストウォーターフォールをなくす。

```typescript
// vite.config.ts
server: {
  warmup: {
    clientFiles: ['./src/main.tsx', './src/routes/**/*.tsx'],
  },
}
```

#### 遅い開発サーバーのプロファイリング

`vite dev` が遅く感じる場合は、`vite --profile` で起動してアプリを操作し、`p+enter` で `.cpuprofile` を保存する。[Speedscope](https://www.speedscope.app) で読み込み、どのプラグインが時間を食っているかを特定する——通常はコミュニティプラグインの `buildStart`、`config`、または `configResolved` フックが原因だ。

### ライブラリモード

npm パッケージを公開する際は `build.lib` を使用する。設定の詳細よりも重要な落とし穴が 2 つある:

1. **型は生成されない**——`vite-plugin-dts` を追加するか `tsc --emitDeclarationOnly` を別途実行する。
2. **ピア依存関係は必ず外部化しなければならない**——リストされていないピアがライブラリにバンドルされ、消費側で重複ランタイムエラーが発生する。

```typescript
// vite.config.ts
build: {
  lib: {
    entry: 'src/index.ts',
    formats: ['es', 'cjs'],
    fileName: (format) => `my-lib.${format}.js`,
  },
  rolldownOptions: {
    external: ['react', 'react-dom', 'react/jsx-runtime'],  // every peer dep
  },
}
```

### SSR 外部化

素の `createServer({ middlewareMode: true })` セットアップはフレームワーク作者向けの領域だ。多くのアプリは Nuxt、Remix、SvelteKit、Astro、または TanStack Start を使用すべきだ。フレームワークユーザーとして*実際に*調整するのは、依存関係が SSR で壊れたときの外部化設定だ:

```typescript
// vite.config.ts — ssr options
ssr: {
  external: ['node-native-package'],           // keep as require() in SSR bundle
  noExternal: ['esm-only-package'],            // force-bundle into SSR output (fixes most SSR errors)
  target: 'node',                              // 'node' or 'webworker'
}
```

### 依存関係プリバンドル

Vite は CJS/UMD を ESM に変換し、リクエスト数を減らすために依存関係をプリバンドルする。

```typescript
// vite.config.ts — optimizeDeps
optimizeDeps: {
  include: [
    'lodash-es',                              // force pre-bundle known heavy deps
    'cjs-package',                            // CJS deps that cause interop issues
    'deep-lib/components/**',                 // glob for deep imports
  ],
  exclude: ['local-esm-package'],             // must be valid ESM if excluded
  force: true,                                // ignore cache, re-optimize (temporary debugging)
}
```

### よくある落とし穴

#### 開発とビルドで挙動が異なる

開発では変換に esbuild/Rolldown を使用し、ビルドではバンドルに Rolldown を使用する。CJS ライブラリは両者で挙動が異なる場合がある。デプロイ前に必ず `vite build && vite preview` で確認すること。

#### デプロイ後に古いチャンクが残る

新しいビルドは新しいチャンクハッシュを生成する。アクティブなセッションを持つユーザーが存在しなくなった古いファイル名をリクエストしてしまう。Vite にはビルトインの解決策がない。軽減策:

- 旧 `dist/assets/` ファイルをデプロイウィンドウ中は有効なままにしておく
- ルーターで動的インポートエラーをキャッチしてページリロードを強制する

#### Docker とコンテナ

Vite はデフォルトで `localhost` にバインドするため、コンテナ外からアクセスできない:

```typescript
// vite.config.ts — Docker/container setup
server: {
  host: true,                                  // bind 0.0.0.0
  hmr: { clientPort: 3000 },                   // if behind a reverse proxy
}
```

#### モノレポのファイルアクセス

Vite はファイル提供をプロジェクトルートに制限する。ルート外のパッケージはブロックされる:

```typescript
// vite.config.ts — monorepo file access
server: {
  fs: {
    allow: ['..'],                             // allow parent directory (workspace root)
  },
}
```

### アンチパターン

```typescript
// BAD: Setting envPrefix to '' exposes ALL env vars (including secrets) to the client
envPrefix: ''

// BAD: Assuming require() works in application source code — Vite is ESM-first
const lib = require('some-lib')                // use import instead

// BAD: Splitting every node_module into its own chunk — creates hundreds of tiny files
manualChunks(id) {
  if (id.includes('node_modules')) {
    return id.split('node_modules/')[1].split('/')[0]   // one chunk per package
  }
}

// BAD: Not externalizing peer deps in library mode — causes duplicate runtime errors
// build.lib without rolldownOptions.external

// BAD: Using deprecated esbuild minifier
build: { minify: 'esbuild' }                  // use 'oxc' (default) or 'terser'

// BAD: Mutating import.meta.hot.data by reassignment
import.meta.hot.data = { count: 0 }           // WRONG: must mutate properties, not reassign
import.meta.hot.data.count = 0                 // CORRECT
```

**プロセス上のアンチパターン:**

- **`vite preview` は本番サーバーではない**——ビルドバンドルのスモークテストだ。`dist/` を実際の静的ホスト（NGINX、Cloudflare Pages、Vercel static）にデプロイするか、マルチステージ Dockerfile を使用すること。
- **`vite build` が型チェックを行うと期待する**——トランスパイルのみを行う。型エラーはサイレントに本番へ届く。`vite-plugin-checker` を追加するか CI で `tsc --noEmit` を実行すること。
- **`@vitejs/plugin-legacy` をデフォルトで含める**——バンドルを約 40% 膨らませ、ソースマップバンドルアナライザを壊し、モダンブラウザを使う 95% 以上のユーザーには不要だ。仮定ではなく実際のアナリティクスに基づいてゲートすること。
- **`tsconfig.json` のパスを複製する 30 以上の `resolve.alias` エントリを手作業で記述する**——代わりに `vite-tsconfig-paths` を使うこと。Excalidraw や PostHog で見られたパターンだが、新規プロジェクトでは避けること。
- **依存関係変更後に古い `node_modules/.vite` を残す**——プリバンドルキャッシュが幻のエラーを引き起こす。ブランチ切り替え時や依存関係パッチ適用後にクリアすること。

## クイックリファレンス

| パターン | 使用タイミング |
|---------|-------------|
| `defineConfig` | 常に使用——型推論を提供する |
| `loadEnv(mode, root, ['VITE_'])` | 設定内で環境変数にアクセスする（明示的なプレフィックス） |
| `vite-plugin-checker` | 任意の TypeScript アプリ（型チェックのギャップを埋める） |
| `vite-tsconfig-paths` | 手作業の `resolve.alias` の代わりに |
| `optimizeDeps.include` | 相互運用の問題を起こす CJS の依存関係 |
| `server.proxy` | 開発時に API リクエストをバックエンドへルーティング |
| `server.host: true` | Docker、コンテナ、リモートアクセス |
| `server.warmup.clientFiles` | ホットパスルートの事前変換 |
| `build.lib` + `external` | npm パッケージの公開 |
| `manualChunks`（オブジェクト形式） | ベンダーバンドルの分割 |
| `vite --profile` | 遅い開発サーバーのデバッグ |
| `vite build && vite preview` | ビルドバンドルをローカルでスモークテスト（本番サーバーではない） |

## 関連スキル

- `frontend-patterns` — React コンポーネントパターン
- `docker-patterns` — Vite を使ったコンテナ化開発
- `nextjs-turbopack` — Next.js 向けの代替バンドラー
