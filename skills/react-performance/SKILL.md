---
name: react-performance
description: VercelエンジニアリングのReactベストプラクティス（https://github.com/vercel-labs/agent-skills）を元にしたReact/Next.jsパフォーマンス最適化パターン。ウォーターフォール、バンドルサイズ、サーバーサイド、クライアントフェッチ、再レンダリング、レンダリング、JS マイクロパフォーマンス、高度パターンの8つの優先カテゴリで70以上のルールを整理。React/Next.jsコードのパフォーマンスを書く・レビューする・リファクタリングする際に使用してください。
origin: ECC
---

# Reactパフォーマンス

React 18/19とNext.jsのパフォーマンス最適化パターン。[Vercel Labs `react-best-practices`](https://github.com/vercel-labs/agent-skills/tree/main/skills/react-best-practices)（MIT、v1.0.0）から参考にしています。このスキルはルールを優先度順に整理し、積極的なコードレビューとリファクタリングのための意思決定ツリーガイダンスを提供します。

## 起動タイミング

- パフォーマンスを意識してReact/Next.jsコードを書く・レビューする場合
- ページ読み込みの遅さ、インタラクションの遅さ、クライアントの高CPU使用率を診断する場合
- バンドルサイズやLighthouse Core Web Vitalsのリグレッションを監査する場合
- Server Components / APIルートのウォーターフォールを排除する場合
- クライアントサイドの再レンダリングを削減する場合
- 長いリスト、アニメーション、またはハイドレーションを最適化する場合
- `app/`、`pages/`、`components/`、またはデータ層に触れるPRの最適化選択を監査する場合

## 優先度インデックス

| 優先度 | カテゴリ | プレフィックス | 適用タイミング |
|---|---|---|---|
| 1 — 重大 | ウォーターフォールの排除 | `async-` | 独立した `await` の後に `await` が続く場合 |
| 2 — 重大 | バンドルサイズの最適化 | `bundle-` | 初回ロードJS、ルートレベルのimport、サードパーティライブラリ |
| 3 — 高 | サーバーサイドパフォーマンス | `server-` | RSC、Server Actions、APIルート、SSR |
| 4 — 中高 | クライアントサイドデータフェッチ | `client-` | SWR / TanStack Query / フックでの生の `fetch` |
| 5 — 中 | 再レンダリング最適化 | `rerender-` | 高頻度の状態更新、親子のファンアウト |
| 6 — 中 | レンダリングパフォーマンス | `rendering-` | 長いリスト、アニメーション、ハイドレーション |
| 7 — 低中 | JavaScriptパフォーマンス | `js-` | ホットループ、頻繁なアロケーション |
| 8 — 低 | 高度なパターン | `advanced-` | エフェクトイベント統合、安定したref |

## 1. ウォーターフォールの排除（重大）

> 「ウォーターフォールはパフォーマンスの最大の敵」 — 逐次的な `await` はフルネットワーク遅延を加算します。

### awaitの前に安価な条件チェック

リモートデータをawaitする前に、同期条件（props、env、ハードコードフラグ）を確認します。

```ts
// 誤り
async function Page({ id }: { id: string }) {
  const flag = await getFlag("show-page");
  if (!flag || !id) return null;
  const data = await getData(id);
  // ...
}

// 正しい — 安価な同期条件を先にショートサーキット
async function Page({ id }: { id: string }) {
  if (!id) return null;
  const flag = await getFlag("show-page");
  if (!flag) return null;
  const data = await getData(id);
}
```

### 使用するまでawaitを遅らせる

データを使用するブランチに `await` を移動します。

```ts
// 誤り — データが必要かどうかを決める前にawaitしている
const user = await getUser(id);
if (mode === "guest") return renderGuest();
return renderUser(user);

// 正しい
if (mode === "guest") return renderGuest();
const user = await getUser(id);
return renderUser(user);
```

### 独立した処理にはPromise.allを使用

```ts
// 誤り — 逐次実行
const user = await getUser(id);
const posts = await getPosts(id);
const followers = await getFollowers(id);

// 正しい — 並列実行
const [user, posts, followers] = await Promise.all([
  getUser(id),
  getPosts(id),
  getFollowers(id),
]);
```

### 部分的依存関係 — 早めに開始し、遅くawaitする

```ts
// 正しい — すべてのPromiseを開始し、各結果が必要な時だけawaitする
const userP = getUser(id);
const postsP = getPosts(id);
const profile = await getProfile(id);
if (profile.private) return null;
const [user, posts] = await Promise.all([userP, postsP]);
```

### ストリーミングのためのSuspense

ページが描画できるものを描画しながら、遅いサブツリーがストリームされるよう、`<Suspense>` 境界をデータの近くに配置します。トレードオフ: コンテンツ到着時のレイアウトシフト — スペースを確保してください（スケルトンまたは `min-height`）。

### Server Components: コンポジションによる並列化

```tsx
// 誤り — 1つのコンポーネント内の兄弟awaitは逐次実行される
export default async function Page() {
  const user = await getUser();
  const cart = await getCart();
  return <View user={user} cart={cart} />;
}

// 正しい — 子コンポーネントに分割し、Reactが並列実行する
export default async function Page() {
  return (
    <View>
      <UserSection />
      <CartSection />
    </View>
  );
}
```

## 2. バンドルサイズの最適化（重大）

### バレルではなく直接インポート

バレル `index.ts` ファイルはツリーシェイキングがほとんどを削除しても、バンドラーがモジュールグラフ全体を走査することを強制します。直接インポートは多くの実際のアプリで初回ロードJSを200〜800ms節約します。

```ts
// 誤り
import { Button, Card, Modal } from "@/components";

// 正しい
import { Button } from "@/components/Button";
import { Card } from "@/components/Card";
import { Modal } from "@/components/Modal";
```

Next.js 13.5以降には[Optimize Package Imports](https://nextjs.org/docs/app/api-reference/next-config-js/optimizePackageImports)があり、リストされたパッケージに対してこれを自動化します — 使用してください。リストされていないライブラリには引き続き手動の直接インポートが必要です。

### 静的に解析可能なパス

```ts
// 誤り — バンドラー/トレース解析を妨げる
const mod = await import(`./pages/${name}`);

// 正しい — ブランチごとに明示的
const mod = name === "home" ? await import("./pages/home") : await import("./pages/about");
```

### 重いコンポーネントの動的インポート

```tsx
import dynamic from "next/dynamic";

const HeavyChart = dynamic(() => import("./HeavyChart"), {
  loading: () => <Skeleton />,
  ssr: false, // クライアントのみの場合
});
```

### サードパーティスクリプトの遅延読み込み

アナリティクス、ロギング、サポートウィジェットはハイドレーション後に読み込みます。`next/script` を `strategy="afterInteractive"`（デフォルト）または `"lazyOnload"` で使用します。

### 条件付きモジュール読み込み

```tsx
if (user.role === "admin") {
  const { AdminPanel } = await import("./admin/AdminPanel");
  // ...
}
```

### ホバー/フォーカス時にプリロード

ユーザーがクリックするまでにバンドルがキャッシュされるよう、ホバー時に `<link rel="preload">` または `import()` をトリガーします。

## 3. サーバーサイドパフォーマンス（高）

### Server ActionsをAPIルートと同様に認証する

すべての `"use server"` 関数は公開エンドポイントです。アクション内で認証と認可の両方を行ってください — 呼び出し元のClient Componentのゲーティングに依存しないでください。

```ts
"use server";
export async function deleteUser(formData: FormData) {
  const session = await getSession();
  if (!session?.user) throw new Error("Unauthorized");
  const targetId = String(formData.get("id"));
  if (session.user.role !== "admin" && session.user.id !== targetId) {
    throw new Error("Forbidden");
  }
  await db.user.delete({ where: { id: targetId } });
}
```

### リクエストごとの重複排除に `React.cache()` を使用

```ts
import { cache } from "react";

export const getUser = cache(async (id: string) => {
  return db.user.findUnique({ where: { id } });
});
```

`React.cache` は単一リクエスト内での重複排除を行います。同じレンダリング内の3つのServer Componentから `getUser("1")` を呼び出す = 1回のDBクエリ。

### クロスリクエストデータにはLRUキャッシュを使用

リクエストごとに変化しないデータ（設定、ルックアップテーブル）は、LRUキャッシュまたは `unstable_cache` でReactの外にキャッシュします。

### RSCプロパティの重複シリアライゼーションを避ける

Server Componentが同じデータを複数のClient Componentにレンダリングする場合、データはコンシューマーごとに1回シリアライズされます。Client Componentを上に持ち上げてchildrenを渡します。

### 静的I/Oをモジュールスコープに巻き上げる

```ts
// 正しい — モジュール読み込み時に1回実行
const fontData = readFileSync(fontPath);

export async function Page() {
  return <Banner font={fontData} />;
}
```

### RSC/SSRでミュータブルなモジュールレベル状態を使用しない

サーバー上のモジュール状態はすべてのリクエスト間で共有されます — ユーザー間の競合状態になります。代わりにリクエストスコープのストレージ（`headers()`、`cookies()`、非同期コンテキスト）を使用してください。

### Client Componentsに渡すデータを最小化する

Clientが必要とするものだけをシリアライズします。DBレイヤーでフィールドを省き、ページネーションし、カラムを射影します。

### Promise.allでネストされたフェッチを並列化する

```ts
const users = await getUsers();
const enriched = await Promise.all(
  users.map(async (u) => ({ ...u, posts: await getPostsFor(u.id) })),
);
```

### ノンブロッキング処理に `after()` を使用する

Next.js 15の `after()` はレスポンス送信後に処理を実行します — ロギング、キャッシュウォーミング、アナリティクス。

```ts
import { after } from "next/server";
export async function GET() {
  const data = await getData();
  after(() => logAnalytics(data));
  return Response.json(data);
}
```

## 4. クライアントサイドデータフェッチ（中高）

### 重複排除のためにSWR / TanStack Queryを使用する

`useUser(id)` を呼び出す複数のコンポーネントは1つのネットワークリクエストと1つのキャッシュエントリを共有すべきです。SWRまたはTanStack Queryを使用してください — 共有データに `useEffect` + `fetch` を自前実装しないでください。

### グローバルイベントリスナーの重複排除

```tsx
// 誤り — すべてのコンポーネントが独自のリスナーを追加する
useEffect(() => {
  window.addEventListener("scroll", handler);
  return () => window.removeEventListener("scroll", handler);
}, []);

// 正しい — フック + グローバルサブジェクトによる単一共有リスナー
const useScroll = createScrollHook(); // 内部でシングルトンサブジェクトを使用
```

### スクロールにはpassiveリスナーを使用する

```ts
window.addEventListener("scroll", handler, { passive: true });
```

スクロールのスムーズさを改善します。リスナーは `preventDefault()` を呼び出せません。

### localStorage: バージョン管理とペイロードの最小化

- 常に `version` フィールドを保存し、スキーマ変更時にバンプして古いデータを移行または破棄する
- ペイロードを小さく保つ — `localStorage` は同期APIでメインスレッドをブロックする

## 5. 再レンダリング最適化（中）

### コールバックでのみ使用する状態をサブスクライブしない

```tsx
// 誤り — countが変わるたびに再レンダリング
const count = useStore((s) => s.count);
const handler = () => doSomething(count);

// 正しい — 呼び出し時に1回読み込む
const handler = () => {
  const count = useStore.getState().count;
  doSomething(count);
};
```

### コストのかかる処理をメモ化されたコンポーネントに抽出する

```tsx
// 正しい — childは`items`が変わった時だけ再レンダリング
const Heavy = memo(function Heavy({ items }: { items: Item[] }) {
  return <Chart data={transform(items)} />;
});
```

### デフォルトの非プリミティブpropsを巻き上げる

```tsx
// 誤り — レンダリングごとに新しい配列でmemoが壊れる
<List items={items ?? []} />

// 正しい
const EMPTY: Item[] = [];
<List items={items ?? EMPTY} />
```

### エフェクトにはプリミティブ依存関係を使用する

```tsx
// 誤り — レンダリングごとに新しいオブジェクトのアイデンティティ
useEffect(() => {}, [{ id, name }]);

// 正しい — プリミティブ
useEffect(() => {}, [id, name]);
```

### 生の値ではなく派生ブール値をサブスクライブする

```tsx
// 誤り — カートの変更があるたびに再レンダリング
const cart = useStore((s) => s.cart);
const hasItems = cart.length > 0;

// 正しい — 空/非空の切り替わり時のみ再レンダリング
const hasItems = useStore((s) => s.cart.length > 0);
```

### `useEffect` ではなくレンダリング中に派生する

```tsx
// 誤り
const [full, setFull] = useState("");
useEffect(() => setFull(`${first} ${last}`), [first, last]);

// 正しい
const full = `${first} ${last}`;
```

### 安定したコールバックには関数形式の `setState` を使用する

```tsx
// 正しい
const increment = useCallback(() => setCount((c) => c + 1), []);
```

### コストのかかる値には遅延state初期化を使用する

```tsx
const [tree] = useState(() => parseTree(largeInput));
```

### 単純なプリミティブにはmemoを使用しない

`useMemo(() => x + 1, [x])` はオーバーヘッドです。memoはオブジェクトのアイデンティティとコストのかかる計算で効果を発揮します。

### 独立した依存関係を持つフックを分割する

```tsx
// 誤り — どちらかのソースが変わると両方のセレクタが再実行される
const { a, b } = useSomething(source1, source2);

// 正しい
const a = useA(source1);
const b = useB(source2);
```

### インタラクションロジックをイベントハンドラーに移動する

イベントハンドラーはユーザーアクション時のみ実行されます — `useEffect` は依存関係が変わるたびに再実行されます。

### 緊急でない更新には `startTransition` を使用する

```tsx
const [pending, startTransition] = useTransition();
startTransition(() => setFilters(newFilters));
```

### コストのかかるレンダリングには `useDeferredValue` を使用する

```tsx
const deferredQuery = useDeferredValue(query);
const results = useMemo(() => expensiveSearch(deferredQuery), [deferredQuery]);
```

### 一時的な高頻度の値には `useRef` を使用する

再レンダリングをトリガーすべきでないが頻繁に変わる値（タイムスタンプ、最後のキー、アキュムレーター）に使用します。

### コンポーネント内にコンポーネントを定義しない

```tsx
// 誤り — Innerは毎回のOuterレンダリングで新しいコンポーネントになる
function Outer() {
  const Inner = () => <span />;
  return <Inner />;
}
```

レンダリングごとに新しい `Inner` 型が生成され、リコンシリエーションが壊れ、childrenがアンマウントされます。

## 6. レンダリングパフォーマンス（中）

### SVGではなくラッパーをアニメーションする

SVGの周りの `<div>` ラッパーを変換するとGPUアクセラレーションになりますが、SVG自体を変換するとペイントが発生します。

### 長いリストには `content-visibility: auto` を使用する

```css
.row { content-visibility: auto; contain-intrinsic-size: auto 80px; }
```

ブラウザはオフスクリーンのレンダリングをスキップします — 数百行のリストに大きな効果があります。

### 静的JSXを巻き上げる

```tsx
const STATIC_HEADER = <h1>Title</h1>;
function Page() {
  return <>{STATIC_HEADER}<Body /></>;
}
```

### SVG: 座標の精度を下げる

`d="M10.123456,20.654321"` → `d="M10.12,20.65"`。各桁はバイトコストになりますが、視覚的な差はサブピクセルです。

### インラインスクリプトによるハイドレーションのちらつき防止

ハイドレーション前に必要な値（テーマ、ロケール）については、Reactがマウントする前に `document.documentElement.dataset.*` を設定するインライン `<script>` を組み込みます。

### 予期されるハイドレーションの不一致を狭く抑制する

```tsx
<time suppressHydrationWarning>{new Date().toLocaleString()}</time>
```

既知の分岐リーフノードにのみ使用してください — 他のchildrenを含むツリーには絶対に使用しないでください。

### マウント/アンマウントの代わりに表示/非表示に `<Activity>` を使用する

React 19の `<Activity mode="visible|hidden">` はツリーの状態とエフェクトをマウントしたまま非表示にします — タブやアコーディオンのアンマウント/再マウントよりコストが低い。

### 条件付きレンダリングには `&&` より三項演算子を使用する

```tsx
// 誤り — `0`がテキストノードとしてレンダリングされる
{count && <Badge>{count}</Badge>}

// 正しい
{count > 0 ? <Badge>{count}</Badge> : null}
```

### ローディング状態に `useTransition` を使用する

`startTransition` をアクションとペアにします。Reactは次の状態が計算される間、`isPending` として前のUIを表示します。

### React DOMリソースヒント

```tsx
import { preload, preconnect } from "react-dom";
preload("/api/critical", { as: "fetch" });
preconnect("https://api.example.com");
```

### `<script>` タグに `defer` / `async` を使用する

`defer` はDOMContentLoaded後の順序付き実行のため、`async` はファイアアンドフォーゲットのために使用します。

## 7. JavaScriptパフォーマンス（低中）

- **DOM/CSS変更をバッチ処理** — プロパティ個別ではなく、クラスの切り替えや `cssText` で適用
- **繰り返しルックアップに `Map` を使用** — `O(1)` vs `O(n)` の線形スキャン
- **ループ内でプロパティアクセスをキャッシュ** — `const len = arr.length`
- **純粋関数をメモ化** — モジュールレベルの `Map<key, result>`
- **`localStorage` 読み取りをキャッシュ** — 同期API；レンダリングごとに1回読み込む
- **`filter().map()` を1パスにまとめる** — `flatMap` または単一の `for`
- **コストのかかる比較の前に配列の長さを確認**
- **関数から早期リターン**
- **RegExpをループの外に巻き上げる** — コンパイルコストは無視できない
- **min/maxには `sort()` の代わりにループを使用** — `O(n)` vs `O(n log n)`
- **メンバーシップには `Set`/`Map` を使用** — `O(1)` vs `Array.includes` の `O(n)`
- **イミュータビリティが重要な場合は `toSorted()` を使用**（ミューテーションではなく）
- **マップとフィルターを1パスで行うには `flatMap` を使用**
- **クリティカルでない処理には `requestIdleCallback` を使用**

## 8. 高度なパターン（低）

### `useEffectEvent` の依存関係

`useEffectEvent` の値は安定しています — エフェクトの依存関係に追加しないでください。

### イベントハンドラーref

メモ化されたchildrenに渡す安定したコールバックのために：

```tsx
const handlerRef = useRef(handler);
useEffect(() => { handlerRef.current = handler; });
const stable = useCallback((arg) => handlerRef.current(arg), []);
```

### アプリロード時に1回だけ初期化する

モジュールレベルのシングルトン（テレメトリ、ロガー）には、モジュールスコープのフラグで保護します — `useEffect` は使いません。

### 安定したコールバックrefに `useLatest` を使用する

```tsx
function useLatest<T>(value: T) {
  const ref = useRef(value);
  ref.current = value;
  return ref;
}
```

## 自動化ツール

これらのルールの多くは現在自動化されています：

- **Next.js 13.5+ Optimize Package Imports** — バレルインポートの最適化
- **React Compiler**（RFC、canaryで提供中） — 自動メモ化
- **Turbopack** — より速いビルド、より良いツリーシェイキング
- **Bundle Analyzer**（`@next/bundle-analyzer`） — 初回ロードJSの可視化

プロジェクトがReact Compilerを採用した場合、`rerender-*` の手動メモ化ルールを「レビューのみ」に降格してください — コンパイラーが処理します。手動の `useMemo`/`useCallback` は不要なノイズになります。

## Lighthouse / Web Vitalsのマッピング

| メトリクス | 最も関連するカテゴリ |
|---|---|
| **LCP**（Largest Contentful Paint） | ウォーターフォール、バンドルサイズ、リソースヒント |
| **INP**（Interaction to Next Paint） | 再レンダリング、レンダリング、JavaScript |
| **CLS**（Cumulative Layout Shift） | レンダリング（Suspense配置、画像サイズ） |
| **TBT**（Total Blocking Time） | バンドルサイズ、JavaScript、サードパーティの遅延読み込み |
| **FID**（レガシー） | バンドルサイズ、ハイドレーション |

## 関連情報

- スキル: [react-patterns](../react-patterns/SKILL.md)、[react-testing](../react-testing/SKILL.md)、[frontend-patterns](../frontend-patterns/SKILL.md)、[accessibility](../accessibility/SKILL.md)、[nextjs-turbopack](../nextjs-turbopack/SKILL.md)
- ルール: [rules/react/](../../rules/react/)
- エージェント: `react-reviewer` がコードレビューでこれらのルールを適用；`react-build-resolver` が関連するビルド失敗を処理
- コマンド: `/react-review`、`/react-build`、`/react-test`

## クレジット

Vercel Labs `react-best-practices` スキル（MITライセンス、著作権 Vercel Engineering、v1.0.0 2026年1月）から参考にしています。ソース: [https://github.com/vercel-labs/agent-skills/tree/main/skills/react-best-practices](https://github.com/vercel-labs/agent-skills/tree/main/skills/react-best-practices)。

このスキルはオリジナルの70ルールカタログを単一のナビゲート可能なリファレンスに再構成・適応しています。拡張例を含む完全なオリジナルルールセットについては、上流リポジトリを参照してください。
