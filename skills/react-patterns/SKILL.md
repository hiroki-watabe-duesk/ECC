---
name: react-patterns
description: React 18/19のパターン：フック規律、サーバー/クライアントコンポーネント境界、Suspense + エラー境界、フォームアクション、データフェッチ、状態管理の決定木、アクセシビリティファーストのコンポジション。Reactコンポーネントの作成またはレビュー時に使用。
origin: ECC
---

# React パターン

堅牢でアクセシブルかつパフォーマントなコンポーネントツリーを構築するためのReact 18/19の慣用的なパターン。

## 起動タイミング

- React関数コンポーネント、カスタムフック、またはコンポーネントツリーを作成または変更するとき
- JSX/TSXファイルをレビューするとき
- 状態の形状やコンポーネントのコンポジションを設計するとき
- クラスコンポーネントや古い `forwardRef`/`useEffect` 多用コードを移行するとき
- ローカル状態、リフトした状態、コンテキスト、外部ストアの選択をするとき
- サーバーコンポーネント / クライアントコンポーネント（Next.js App Router、RSC）を使用するとき
- React 19のアクションまたはコントロールされた入力でフォームを実装するとき
- TanStack Query / SWR / RSCでデータフェッチを連携するとき

## コア原則

### 1. レンダーはPropsとStateの純粋関数

```tsx
// 良い: レンダー中に導出する
function Cart({ items }: { items: CartItem[] }) {
  const total = items.reduce((sum, i) => sum + i.price * i.qty, 0);
  return <span>{formatMoney(total)}</span>;
}

// 悪い: 導出状態を別に格納する
function Cart({ items }: { items: CartItem[] }) {
  const [total, setTotal] = useState(0);
  useEffect(() => {
    setTotal(items.reduce((sum, i) => sum + i.price * i.qty, 0));
  }, [items]);
  return <span>{formatMoney(total)}</span>;
}
```

`useEffect` での導出状態はレンダーサイクルを追加し、非同期になる可能性があり、データフローを不明瞭にします。

### 2. レンダー外のサイドエフェクト

エフェクト、ミューテーション、ネットワーク呼び出し、サブスクリプションはイベントハンドラーまたは `useEffect` に配置します — レンダーボディには絶対に配置しません。

### 3. 継承よりコンポジション

Reactにはコンポーネントの継承モデルがありません。`children`、レンダープロップ、またはコンポーネントプロップでコンポーズします。

## フック規律

完全なルールセットは[rules/react/hooks.md](../../rules/react/hooks.md)を参照してください。ハイライト:

- トップレベルのみ、条件付きは不可
- すべてのサブスクリプション、インターバル、リスナーをクリーンアップする
- 新しい状態が古い状態に依存する場合は関数型アップデーター（`setX(prev => prev + 1)`）を使用する
- デフォルトの立場: メモ化しない — プロファイラーや依存チェーンが必要性を証明した場合のみ `useMemo`/`useCallback` を追加する
- 同じフックシーケンスが2つ以上のコンポーネントに現れる場合のみカスタムフックを抽出する

## 状態の配置の決定木

```
1つのコンポーネントで使用？
  -> その中のuseState

親と少数の子孫で使用？
  -> 最も近い共通祖先にリフトする

離れた分岐で使用 AND 低頻度読み取り（テーマ、認証、ロケール）？
  -> React Context

ツリー全体で共有される高頻度更新？
  -> 外部ストア（Zustand、Jotai、Redux Toolkit）

サーバーから導出？
  -> サーバー状態ライブラリ（TanStack Query、SWR、RSC fetch）
```

ほとんどのページはコンテキストやグローバルストアを必要としません。リフトの重複が苦痛になるまで抽象化に抵抗してください。

## サーバー / クライアントコンポーネント（RSC）

```tsx
// サーバーコンポーネント - デフォルト、非同期、自身のJSを配信しない
export default async function ProductPage({ params }: { params: { id: string } }) {
  const product = await db.product.findUnique({ where: { id: params.id } });
  if (!product) notFound();
  return <ProductView product={product} />;
}

// クライアントコンポーネント - "use client"でオプトイン
"use client";
export function AddToCartButton({ productId }: { productId: string }) {
  const [pending, startTransition] = useTransition();
  return (
    <button
      disabled={pending}
      onClick={() => startTransition(() => addToCart(productId))}
    >
      {pending ? "Adding..." : "Add to cart"}
    </button>
  );
}
```

境界:

- サーバー -> クライアント: シリアライズ可能なpropsまたは `children` を渡す
- クライアント -> サーバー: `<form action={...}>` または イベントハンドラーから命令的にサーバーアクションを呼び出す
- クライアントコンポーネントファイルからサーバーコンポーネントを `import` しない — 代わりに `children` でコンポーズする

## Suspense + エラー境界

```tsx
<ErrorBoundary fallback={<ErrorView />}>
  <Suspense fallback={<UserSkeleton />}>
    <UserDetail id={id} />
  </Suspense>
</ErrorBoundary>
```

- Suspense境界はルートルートではなく、データの近くに配置する — コンテンツを段階的に表示する
- Error BoundaryはクラスベースのAPIのまま。フックに優しいラッパーには `react-error-boundary` を使用する
- 境界はレンダー、ライフサイクル、子のコンストラクター中に投げられたエラーをキャッチする — イベントハンドラーや非同期コードではキャッチしない

## フォーム

### React 19のフォームアクション（新しいコードで推奨）

```tsx
"use client";
import { useActionState } from "react";

const initial = { error: null as string | null };

async function updateUserAction(_prev: typeof initial, formData: FormData) {
  "use server";
  const parsed = UserSchema.safeParse(Object.fromEntries(formData));
  if (!parsed.success) return { error: "Invalid input" };
  await db.user.update({ where: { id: parsed.data.id }, data: parsed.data });
  return { error: null };
}

export function UserForm() {
  const [state, formAction, pending] = useActionState(updateUserAction, initial);
  return (
    <form action={formAction}>
      <input name="name" required />
      <button type="submit" disabled={pending}>Save</button>
      {state.error && <p role="alert">{state.error}</p>}
    </form>
  );
}
```

### コントロールされた入力

値が他のUIを駆動する場合、キーストロークごとにフォーマットする場合、またはリアルタイムバリデーションを実装する場合にコントロールを使用します。

### 複雑なフォーム

マルチステップフォーム、動的フィールド配列、またはクロスフィールドバリデーションには: ライブラリを使用します（React Hook Form、TanStack Form）。些細な複雑さを超えたフォームのためのロールユア自前状態管理は保守の罠です。

## データフェッチの決定マトリクス

| ニーズ | ツール |
|---|---|
| Next.js App Routerのリクエストごとのデータ | RSC `await fetch()` |
| クライアント側キャッシュ + ミューテーション + 無効化 | TanStack Query |
| 軽量クライアントキャッシュ + 再検証 | SWR |
| リアルタイムサブスクリプション | Server-Sent Events、WebSockets、またはライブラリのサブスクリプションAPI |
| 一度きりのファイアアンドフォーゲット | イベントハンドラー内の `fetch()` |

アプリケーションデータには `useEffect` + `fetch` を避ける — 競合状態、キャッシュなし、リトライなし、Suspense統合なし。

## コンポジションレシピ

### `children` によるスロット

```tsx
<Layout>
  <Header />
  <Main>{content}</Main>
</Layout>
```

### 名前付きスロット

```tsx
<Page header={<Nav />} sidebar={<Filters />}>
  <Results />
</Page>
```

### 複合コンポーネント（Contextによる共有状態）

```tsx
<Tabs defaultValue="profile">
  <Tabs.List>
    <Tabs.Trigger value="profile">Profile</Tabs.Trigger>
    <Tabs.Trigger value="settings">Settings</Tabs.Trigger>
  </Tabs.List>
  <Tabs.Panel value="profile"><Profile /></Tabs.Panel>
  <Tabs.Panel value="settings"><Settings /></Tabs.Panel>
</Tabs>
```

### レンダープロップ / 関数as子

親がレンダーされた出力にパラメーターを渡す必要がある場合に有用:

```tsx
<DataLoader id={id}>
  {({ data, isLoading }) => isLoading ? <Spinner /> : <UserCard user={data} />}
</DataLoader>
```

現代の代替: 同じ形状を返すフック（`useData(id)`）— 通常はよりクリーン。

## パフォーマンス

### `React.memo` が実際に効果を発揮するとき

コンポーネントを `React.memo` でラップするのは以下の場合のみ:

1. 頻繁に再レンダーする
2. レンダー間でpropsがほぼ同じ
3. レンダーが計測上コストが高い

`React.memo` はレンダーごとに等価チェックを追加します。ほとんどのレンダーでpropsが変わる場合、チェックは純粋なオーバーヘッドです。

### レンダーカスケードの回避

- 可能な限り上ではなく下に状態をリフトする
- コンテキストを分割する: 関心事ごとに1つのコンテキスト。`themeContext` の変更が認証コンシューマーを再レンダーしないように
- 外部状態ライブラリには `useSyncExternalStore` を使用する — 安全な並行レンダリングに必須

### リスト

- 安定した `key` プロップを提供する（データベースIDを使用し、配列インデックスは不可）
- 非自明な行で表示アイテム数が約50を超えたら `@tanstack/react-virtual` または `react-window` で長いリストを仮想化する

## アクセシビリティファーストのコンポジション

- `role` 属性を使う前に常にセマンティックなHTML（`<button>`、`<a>`、`<nav>`、`<main>`）をレンダーする
- すべてのインタラクティブ要素はキーボードで到達可能でなければならない
- フォーム入力にはラベルが必要 — アイコンで視覚的にラベル付けされる場合は `<label htmlFor>` または `aria-label`
- ルート変更とモーダルの開閉時にフォーカスを管理する
- コンポーネントテストで `axe` を実行する（[skills/react-testing](../react-testing/SKILL.md)を参照）
- クロスリンク: [skills/accessibility/SKILL.md](../accessibility/SKILL.md) でWCAG基準とパターンライブラリをカバー

## ルーティング

このスキルはルーター非依存です。上記のパターンはReact Router、TanStack Router、Next.js App Router、Remix Routerで機能します。ルーター固有のパターン（ローダー、アクション、ネストされたレイアウト）はルーターのドキュメントに従います — それらはReactコアの上に重なるフレームワークの関心事です。

## スコープ外（参照セクション）

- **Next.js固有**: App Routerのデータローディング、ルートハンドラー、ミドルウェア、並行ルート — 別の関心事、Next.jsドキュメントを使用
- **React Native**: プラットフォーム固有のパターンは別の `react-native-patterns` スキルが必要（まだ存在しない）
- **Remix**: ローダー/アクションの規約はRSCと重複しているがRemixドキュメントに従う

## 関連

- ルール: [rules/react/](../../rules/react/) — コーディングスタイル、フック、パターン、セキュリティ、テスト
- スキル: [react-performance](../react-performance/SKILL.md) (Vercel由来のパフォーマンスルールセット)、[frontend-patterns](../frontend-patterns/SKILL.md) (クロスフレームワークUIの関心事)、[accessibility](../accessibility/SKILL.md)、[angular-developer](../angular-developer/SKILL.md) (フレームワーク比較)
- エージェント: コードレビュー用 `react-reviewer`、ビルド/バンドラーエラー用 `react-build-resolver`
- コマンド: `/react-review`、`/react-build`、`/react-test`

## 例

### デバウンス検索のカスタムフック

```tsx
function useDebounce<T>(value: T, delay = 300): T {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);
  }, [value, delay]);
  return debounced;
}

function SearchBox() {
  const [query, setQuery] = useState("");
  const debounced = useDebounce(query, 300);
  const { data } = useQuery({
    queryKey: ["search", debounced],
    queryFn: () => searchApi(debounced),
    enabled: debounced.length > 0,
  });
  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <Results items={data ?? []} />
    </>
  );
}
```

### React 19の `useOptimistic` によるオプティミスティックUI

```tsx
"use client";
import { useOptimistic } from "react";

export function MessageList({ messages }: { messages: Message[] }) {
  const [optimistic, addOptimistic] = useOptimistic(
    messages,
    (state, newMessage: Message) => [...state, newMessage],
  );

  async function send(formData: FormData) {
    const text = String(formData.get("text"));
    addOptimistic({ id: "pending", text, sender: "me" });
    await saveMessage(text);
  }

  return (
    <>
      <ul>{optimistic.map((m) => <li key={m.id}>{m.text}</li>)}</ul>
      <form action={send}>
        <input name="text" />
        <button type="submit">Send</button>
      </form>
    </>
  );
}
```

### レンダーカスケードを避けるためのコンテキスト分割

```tsx
// 2つのコンテキスト: 一方はほぼ変化しない、もう一方は頻繁に変化する
const ThemeContext = createContext<Theme>("light");
const NotificationsContext = createContext<Notification[]>([]);

// ThemeContextのみを消費するコンポーネントは通知が変わっても再レンダーしない
```
