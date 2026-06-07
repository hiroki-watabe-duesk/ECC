---
name: react-testing
description: React Testing Library・Vitest/Jest・ネットワークモック用MSW・axeによるアクセシビリティアサーション・コンポーネントテストとPlaywright/CypressのE2Eとの判断基準を使ったReactコンポーネントテスト。Reactコンポーネント・フック・ページのテストを書く・修正するときに使用する。
origin: ECC
---

# Reactテスト

振る舞いに焦点を当てたコンポーネントテスト、カスタムフックテスト、アクセシビリティアサーション、ネットワークレベルモックのための包括的なReactテストパターン集。

## いつ起動するか

- Reactコンポーネント、カスタムフック、またはページのテストを書くとき
- レガシーな未テストコンポーネントにテストカバレッジを追加するとき
- EnzymeまたはクラスコンポーネントのパターンからReact Testing Libraryへ移行するとき
- 新しいReactプロジェクトにVitestまたはJestをセットアップするとき
- テスト内でHTTPリクエストをモックするとき
- アクセシビリティ違反をアサートするとき
- どのテストをRTL・Playwright Component Testing・フルE2Eに振り分けるかを判断するとき

## コア原則

ユーザーが見て行うことをテストし、実装の詳細はテストしない。

テストが行うべきこと：

- 本番環境と同じプロバイダーでコンポーネントをレンダリングする
- アクセシブルなクエリ（role、label）と `userEvent` でコンポーネントと対話する
- 可視出力と観察可能な副作用（コールバックの呼び出し、リクエストの送信）をアサートする

テストが行うべきでないこと：

- コンポーネントの状態、子に渡されるprops、どのフックが呼ばれたかを検査する
- React自体またはフレームワークのフックをモックする
- ユーザーに影響するDOM構造を超えたレンダリング回数をアサートする

## ライブラリの選択

| ランナー | 使用場面 | 備考 |
|---|---|---|
| **Vitest** | Vite・Remix・モダンなセットアップ | 高速・ネイティブESM・Jest互換API |
| **Jest** | Next.js・CRA・既存リポジトリ | 多くのReactプロジェクトのデフォルト |
| **Playwright Component Testing** | 実際のブラウザエンジンが必要なとき | JSDOMに必要な機能がないときに使用 |
| **Cypress Component Testing** | 実際のブラウザ・Cypressが既に導入済みのとき | Playwright CTの代替 |

ひとつを選ぶ。明確なレーン分離がない限り、同じリポジトリでRTL＋VitestとPlaywright CTを両方実行しない。

## クエリの優先順位

React Testing Libraryは3段階のクエリを提供します。上から順に使用します：

1. **誰にでもアクセス可能**: `getByRole`、`getByLabelText`、`getByPlaceholderText`、`getByText`、`getByDisplayValue`
2. **セマンティック**: `getByAltText`、`getByTitle`
3. **テストID（エスケープハッチ）**: `getByTestId`

```tsx
// 最善
screen.getByRole("button", { name: /save/i });

// 入力フィールドには OK
screen.getByLabelText("Email");

// 最終手段
screen.getByTestId("save-btn");
```

バリアント：

- `getBy*` — 一致がない場合にスロー
- `queryBy*` — `null` を返す（「不在をアサート」するのに使用）
- `findBy*` — 非同期、Promiseを返す（非同期処理後に現れる要素に使用）

## `userEvent` を使ったユーザーインタラクション

```tsx
import userEvent from "@testing-library/user-event";

test("submits the form", async () => {
  const user = userEvent.setup();
  const onSubmit = vi.fn();
  render(<UserForm onSubmit={onSubmit} />);

  await user.type(screen.getByLabelText("Email"), "user@example.com");
  await user.click(screen.getByRole("button", { name: /save/i }));

  expect(onSubmit).toHaveBeenCalledWith({ email: "user@example.com" });
});
```

- userEventの呼び出しは常に `await` する
- テストごとに `userEvent.setup()` を一度だけ呼び出し、返された `user` を再利用する
- `userEvent` は実際のブラウザシーケンスをシミュレートする。`fireEvent` は単一の合成イベントをディスパッチするだけなので、`userEvent` を優先する

## 非同期パターン

```tsx
// 非同期処理後に現れる要素
expect(await screen.findByText("Loaded")).toBeInTheDocument();

// 副作用のアサーション
await waitFor(() => expect(saveSpy).toHaveBeenCalled());

// 消えるべき要素
await waitForElementToBeRemoved(() => screen.queryByText("Loading"));
```

`setTimeout` + アサーションは不安定。上記マッチャーを使用すること。

## MSWを使ったネットワークモック

Mock Service Workerはネットワーク層でモックします。コンポーネント・フック・fetchライブラリはすべて本番環境と全く同じように動作します。

### セットアップ

```ts
// test/setup.ts
import { setupServer } from "msw/node";
import { http, HttpResponse } from "msw";

export const handlers = [
  http.get("/api/users/:id", ({ params }) =>
    HttpResponse.json({ id: params.id, name: "Alice" }),
  ),
  http.post("/api/users", async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json({ id: "new-id", ...body }, { status: 201 });
  }),
];

export const server = setupServer(...handlers);

beforeAll(() => server.listen({ onUnhandledRequest: "error" }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

`onUnhandledRequest: "error"` を設定して、モックされていないリクエストが大声でテストを失敗させるようにする — サイレントなpassはredより悪い。

### テストごとのオーバーライド

```tsx
test("renders error on 500", async () => {
  server.use(
    http.get("/api/users/:id", () => new HttpResponse(null, { status: 500 })),
  );
  render(<UserPage id="1" />);
  expect(await screen.findByText(/something went wrong/i)).toBeInTheDocument();
});
```

## プロバイダーラッピング

プロバイダーを一度 `test-utils.tsx` にまとめます：

```tsx
// test-utils.tsx
import { render, RenderOptions } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

export function renderWithProviders(
  ui: React.ReactElement,
  options?: RenderOptions,
) {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  });

  return render(
    <QueryClientProvider client={queryClient}>
      <ThemeProvider theme={lightTheme}>
        <MemoryRouter>{ui}</MemoryRouter>
      </ThemeProvider>
    </QueryClientProvider>,
    options,
  );
}

export * from "@testing-library/react";
```

そして全テストファイルで `import { renderWithProviders, screen } from "test-utils"` と記述します。

## カスタムフックのテスト

```tsx
import { renderHook, act } from "@testing-library/react";

test("useCounter increments and decrements", () => {
  const { result } = renderHook(() => useCounter(0));

  expect(result.current.count).toBe(0);

  act(() => result.current.increment());
  expect(result.current.count).toBe(1);

  act(() => result.current.decrement());
  expect(result.current.count).toBe(0);
});

test("useCounter accepts initial value", () => {
  const { result } = renderHook(() => useCounter(10));
  expect(result.current.count).toBe(10);
});

test("useUser fetches user data", async () => {
  // QueryClientをwrapperの外でテストごとに一度だけインスタンス化し、再レンダリング後も生存させる。
  // wrapper クロージャの中で生成するとレンダリングごとにキャッシュ状態がリセットされ、不安定なテストになる。
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  });
  const wrapper = ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  );

  const { result } = renderHook(() => useUser("1"), { wrapper });

  await waitFor(() => expect(result.current.isSuccess).toBe(true));
  expect(result.current.data).toEqual({ id: "1", name: "Alice" });
});
```

- 状態を変更する呼び出しは `act` でラップする
- フックのパブリックAPIのみを通じてテストする
- コンテキストを使用するフックには `wrapper` を渡す

## アクセシビリティアサーション

```tsx
import { axe, toHaveNoViolations } from "jest-axe"; // または vitest-axe
expect.extend(toHaveNoViolations);

test("UserCard has no a11y violations", async () => {
  const { container } = render(<UserCard user={mockUser} />);
  expect(await axe(container)).toHaveNoViolations();
});
```

すべてのインタラクティブコンポーネントのコンポーネントテストでaxeを実行します。以下を検出します：

- フォーム入力のラベル欠落
- 不正なARIA使用
- 低コントラスト（限定的 — JSDOMには実際のCSSエンジンがないため、インラインスタイルのみ機能する。視覚的コントラストはPlaywrightで）
- 画像のalt属性欠落
- 見出しの順序違反

クロスリンク：より広いa11yテストのプレイブックは [skills/accessibility/SKILL.md](../accessibility/SKILL.md) を参照。

## スナップショットテストを使ってはいけないとき

レンダリング出力のスナップショットは：

- スタイルの変更のたびに壊れる
- レビュー中にゴム印のように承認される
- 振る舞いでなく実装の詳細（DOM構造）をテストする

スナップショットの許容できる使用：

- 純粋なデータシリアライズ関数（`formatInvoice(invoice)` → 安定した文字列）
- 生成された設定ファイル（webpack設定出力など）

コンポーネントのビジュアルリグレッションには、Playwright/CypressのスクリーンショットやPercy/Chromaticを使用 — 実際の視覚的差分であり、DOM文字列ではない。

## Playwright / Cypress を使うべきとき

JSDOM（Vitest/Jestが使用）ができないこと：

- 実際のレイアウトのレンダリング（flexbox、grid、viewportクエリ）
- ネイティブブラウザのアニメーション、CSSトランジションの実行
- スクロール動作、ドラッグアンドドロップ、クリップボードからの貼り付けのテスト
- iframe、ポップアップ、ダウンロード、クロスオリジンフローの処理
- 完全なDevToolsサポートを持つ制御された環境での実際のネットワーク実行

これらのいずれかが必要な場合は、Playwright Component Testing（実際のブラウザでのコンポーネントテスト）またはフルE2Eを使用します。[e2e-testing skill](../e2e-testing/SKILL.md) を参照。

判断基準：

- フック、プレゼンテーショナルコンポーネント、ロジックを持つフォーム → RTL
- レイアウトが重要または JSDOMにないブラウザAPIを使用するコンポーネント → Playwright CT
- 複数ページにまたがる完全なユーザーフロー → Playwright/Cypress E2E

## カバレッジ目標

| レイヤー | 目標 |
|---|---|
| 純粋なユーティリティ | 90%以上 |
| カスタムフック | 85%以上 |
| プレゼンテーショナルコンポーネント | 80%以上 — 行数ではなく振る舞い |
| コンテナコンポーネント | 70%以上 — ゴールデンパス＋エラー状態 |
| ページ | E2Eで別途カバー。最低限スモークテスト |

`vitest.config.ts` / `jest.config.js` で設定：

```ts
// vitest.config.ts
test: {
  coverage: {
    provider: "v8",
    reporter: ["text", "html", "lcov"],
    thresholds: {
      lines: 80,
      functions: 80,
      branches: 70,
      statements: 80,
    },
  },
}
```

## アンチパターン

- `container.querySelector("...")` — アクセシビリティクエリをバイパスし、実際のユーザーは失敗するのにテストがpassになる
- レンダリング回数のアサーション — 実装の詳細
- `jest.mock("react", ...)` — Reactを絶対にモックしない。代わりにコンポーネントをリファクタリングする
- デフォルトで子コンポーネントをモックする — 統合でなく分離をテストする。重い副作用を持つ子のみモックする
- `act()` 警告を無視する — 実際のバグのシグナル（アンマウント後の状態更新、非同期ラッピングの欠如）
- テスト間で可変状態を共有する — テスト順序が変わるとフレーキーになる
- `it.skip()` を削除してもpassするテスト — テストが実際に期待することをアサートしていない

## TDDワークフロー

```
RED     -> 次の要件のための失敗するテストを書く
GREEN   -> テストをpassさせる最小限のコンポーネントコードを書く
REFACTOR -> コンポーネントを改善し、テストはグリーンのまま
REPEAT  -> 次の要件へ
```

新しいコンポーネントの場合：

1. コンポーネントのpropタイプとシグネチャを定義する
2. 最もシンプルなケースの最初のテストを書く
3. 正しい理由で失敗することを確認する
4. passするのに十分な最小限の実装をする
5. 次のテストケースを追加する
6. 3つ目の類似テストがパターンを明らかにしたらリファクタリングする

## テストコマンド

```bash
# Vitest
vitest                            # ウォッチモード
vitest run                        # ワンショット
vitest run --coverage             # カバレッジ付き
vitest run path/to/file.test.tsx  # 単一ファイル

# Jest
jest --watch
jest --coverage
jest path/to/file.test.tsx

# CIモード
CI=true vitest run --coverage
```

## 関連

- ルール: [rules/react/testing.md](../../rules/react/testing.md)
- スキル: [react-patterns](../react-patterns/SKILL.md)、[accessibility](../accessibility/SKILL.md)、[e2e-testing](../e2e-testing/SKILL.md)、[tdd-workflow](../tdd-workflow/SKILL.md)
- エージェント: `react-reviewer`（コードレビュー中のテスト品質をレビュー）、`tdd-guide`（TDDプロセスを強制）
- コマンド: `/react-test`、`/react-review`

## 例

### MSWとuserEventを使ったフォーム送信

```tsx
test("submits user form and shows success", async () => {
  server.use(
    http.post("/api/users", () =>
      HttpResponse.json({ id: "1", name: "Alice" }, { status: 201 }),
    ),
  );

  const user = userEvent.setup();
  renderWithProviders(<UserForm />);

  await user.type(screen.getByLabelText("Name"), "Alice");
  await user.type(screen.getByLabelText("Email"), "alice@example.com");
  await user.click(screen.getByRole("button", { name: /save/i }));

  expect(await screen.findByText(/saved successfully/i)).toBeInTheDocument();
});
```

### エラーバウンダリのテスト

```tsx
function Broken() {
  throw new Error("boom");
}

test("error boundary renders fallback", () => {
  // 予期されたthrowに対するReactのconsole.errorのノイズを抑制し、その後復元する。
  // spyが他のテストにリークして実際のエラーを隠さないようにする。
  const errorSpy = vi.spyOn(console, "error").mockImplementation(() => {});
  try {
    render(
      <ErrorBoundary fallback={<div>Something went wrong</div>}>
        <Broken />
      </ErrorBoundary>,
    );

    expect(screen.getByText("Something went wrong")).toBeInTheDocument();
  } finally {
    errorSpy.mockRestore();
  }
});
```

### Suspenseバウンダリのテスト

```tsx
test("shows loading then content", async () => {
  renderWithProviders(
    <Suspense fallback={<div>Loading...</div>}>
      <UserDetail id="1" />
    </Suspense>,
  );

  expect(screen.getByText("Loading...")).toBeInTheDocument();
  expect(await screen.findByText("Alice")).toBeInTheDocument();
});
```
