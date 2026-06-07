---
name: coding-standards
description: 命名、可読性、イミュータビリティ、コード品質レビューのためのベースラインとなるクロスプロジェクトコーディング規約。フレームワーク固有のパターンには詳細なフロントエンドまたはバックエンドスキルを使用する。
origin: ECC
---

# コーディング標準とベストプラクティス

プロジェクト全体に適用できるベースラインのコーディング規約。

このスキルは共通の土台であり、詳細なフレームワークのプレイブックではない。

- Reactの状態、フォーム、レンダリング、UIアーキテクチャには`frontend-patterns`を使用する。
- リポジトリ/サービス層、エンドポイント設計、バリデーション、サーバー固有の懸念事項には`backend-patterns`または`api-design`を使用する。
- フルスキルの説明ではなく最短の再利用可能なルールレイヤーが必要な場合は`rules/common/coding-style.md`を使用する。

## 起動すべき状況

- 新しいプロジェクトやモジュールを開始する場合
- 品質と保守性についてコードをレビューする場合
- 規約に従うために既存のコードをリファクタリングする場合
- 命名、フォーマット、または構造的一貫性を強制する場合
- リンティング、フォーマット、または型チェックルールを設定する場合
- 新しいコントリビューターにコーディング規約をオンボーディングする場合

## スコープの境界

このスキルを以下のために起動する：
- 説明的な命名
- イミュータビリティのデフォルト
- 可読性、KISS、DRY、YAGNIの強制
- エラーハンドリングの期待とコードスメルのレビュー

以下の主要ソースとしてこのスキルを使用しない：
- Reactのコンポジション、フック、またはレンダリングパターン
- バックエンドアーキテクチャ、API設計、またはデータベース層
- より絞り込まれたECCスキルが既に存在する場合のドメイン固有フレームワークガイダンス

## コード品質の原則

### 1. 可読性第一
- コードは書かれるよりも読まれる
- 明確な変数名と関数名
- コメントよりも自己文書化コードを優先
- 一貫したフォーマット

### 2. KISS（シンプルに保て）
- 機能する最もシンプルな解決策
- 過度なエンジニアリングを避ける
- 早期最適化なし
- 理解しやすい > 賢いコード

### 3. DRY（繰り返すな）
- 共通ロジックを関数に抽出する
- 再利用可能なコンポーネントを作成する
- モジュール間でユーティリティを共有する
- コピーペーストプログラミングを避ける

### 4. YAGNI（それは必要ない）
- 必要になる前に機能を構築しない
- 投機的な汎用性を避ける
- 必要な場合にのみ複雑さを追加する
- シンプルに始め、必要なときにリファクタリングする

## TypeScript/JavaScriptの標準

### 変数命名

```typescript
// PASS: GOOD: 説明的な名前
const marketSearchQuery = 'election'
const isUserAuthenticated = true
const totalRevenue = 1000

// FAIL: BAD: 不明瞭な名前
const q = 'election'
const flag = true
const x = 1000
```

### 関数命名

```typescript
// PASS: GOOD: 動詞-名詞パターン
async function fetchMarketData(marketId: string) { }
function calculateSimilarity(a: number[], b: number[]) { }
function isValidEmail(email: string): boolean { }

// FAIL: BAD: 不明瞭または名詞のみ
async function market(id: string) { }
function similarity(a, b) { }
function email(e) { }
```

### イミュータビリティパターン（重要）

```typescript
// PASS: 常にスプレッド演算子を使用する
const updatedUser = {
  ...user,
  name: 'New Name'
}

const updatedArray = [...items, newItem]

// FAIL: 直接変更しない
user.name = 'New Name'  // BAD
items.push(newItem)     // BAD
```

### エラーハンドリング

```typescript
// PASS: GOOD: 包括的なエラーハンドリング
async function fetchData(url: string) {
  try {
    const response = await fetch(url)

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`)
    }

    return await response.json()
  } catch (error) {
    console.error('Fetch failed:', error)
    throw new Error('Failed to fetch data')
  }
}

// FAIL: BAD: エラーハンドリングなし
async function fetchData(url) {
  const response = await fetch(url)
  return response.json()
}
```

### Async/Awaitのベストプラクティス

```typescript
// PASS: GOOD: 可能な場合は並行実行
const [users, markets, stats] = await Promise.all([
  fetchUsers(),
  fetchMarkets(),
  fetchStats()
])

// FAIL: BAD: 不必要な逐次実行
const users = await fetchUsers()
const markets = await fetchMarkets()
const stats = await fetchStats()
```

### 型安全性

```typescript
// PASS: GOOD: 適切な型
interface Market {
  id: string
  name: string
  status: 'active' | 'resolved' | 'closed'
  created_at: Date
}

function getMarket(id: string): Promise<Market> {
  // 実装
}

// FAIL: BAD: 'any'の使用
function getMarket(id: any): Promise<any> {
  // 実装
}
```

## Reactのベストプラクティス

### コンポーネント構造

```typescript
// PASS: GOOD: 型付き関数コンポーネント
interface ButtonProps {
  children: React.ReactNode
  onClick: () => void
  disabled?: boolean
  variant?: 'primary' | 'secondary'
}

export function Button({
  children,
  onClick,
  disabled = false,
  variant = 'primary'
}: ButtonProps) {
  return (
    <button
      onClick={onClick}
      disabled={disabled}
      className={`btn btn-${variant}`}
    >
      {children}
    </button>
  )
}

// FAIL: BAD: 型なし、不明瞭な構造
export function Button(props) {
  return <button onClick={props.onClick}>{props.children}</button>
}
```

### カスタムフック

```typescript
// PASS: GOOD: 再利用可能なカスタムフック
export function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value)

  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value)
    }, delay)

    return () => clearTimeout(handler)
  }, [value, delay])

  return debouncedValue
}

// 使用例
const debouncedQuery = useDebounce(searchQuery, 500)
```

### 状態管理

```typescript
// PASS: GOOD: 適切な状態更新
const [count, setCount] = useState(0)

// 前の状態に基づく状態には関数型更新を使用
setCount(prev => prev + 1)

// FAIL: BAD: 直接的な状態参照
setCount(count + 1)  // 非同期シナリオで古くなる可能性がある
```

### 条件付きレンダリング

```typescript
// PASS: GOOD: 明確な条件付きレンダリング
{isLoading && <Spinner />}
{error && <ErrorMessage error={error} />}
{data && <DataDisplay data={data} />}

// FAIL: BAD: 三項地獄
{isLoading ? <Spinner /> : error ? <ErrorMessage error={error} /> : data ? <DataDisplay data={data} /> : null}
```

## API設計の標準

### REST API規約

```
GET    /api/markets              # すべてのマーケットをリスト
GET    /api/markets/:id          # 特定のマーケットを取得
POST   /api/markets              # 新しいマーケットを作成
PUT    /api/markets/:id          # マーケットを更新（完全）
PATCH  /api/markets/:id          # マーケットを更新（部分）
DELETE /api/markets/:id          # マーケットを削除

# フィルタリング用クエリパラメーター
GET /api/markets?status=active&limit=10&offset=0
```

### レスポンスフォーマット

```typescript
// PASS: GOOD: 一貫したレスポンス構造
interface ApiResponse<T> {
  success: boolean
  data?: T
  error?: string
  meta?: {
    total: number
    page: number
    limit: number
  }
}

// 成功レスポンス
return NextResponse.json({
  success: true,
  data: markets,
  meta: { total: 100, page: 1, limit: 10 }
})

// エラーレスポンス
return NextResponse.json({
  success: false,
  error: 'Invalid request'
}, { status: 400 })
```

### 入力バリデーション

```typescript
import { z } from 'zod'

// PASS: GOOD: スキーマバリデーション
const CreateMarketSchema = z.object({
  name: z.string().min(1).max(200),
  description: z.string().min(1).max(2000),
  endDate: z.string().datetime(),
  categories: z.array(z.string()).min(1)
})

export async function POST(request: Request) {
  const body = await request.json()

  try {
    const validated = CreateMarketSchema.parse(body)
    // バリデート済みデータで処理を続ける
  } catch (error) {
    if (error instanceof z.ZodError) {
      return NextResponse.json({
        success: false,
        error: 'Validation failed',
        details: error.errors
      }, { status: 400 })
    }
  }
}
```

## ファイル構成

### プロジェクト構造

```
src/
├── app/                    # Next.js App Router
│   ├── api/               # APIルート
│   ├── markets/           # マーケットページ
│   └── (auth)/           # 認証ページ（ルートグループ）
├── components/            # Reactコンポーネント
│   ├── ui/               # 汎用UIコンポーネント
│   ├── forms/            # フォームコンポーネント
│   └── layouts/          # レイアウトコンポーネント
├── hooks/                # カスタムReactフック
├── lib/                  # ユーティリティと設定
│   ├── api/             # APIクライアント
│   ├── utils/           # ヘルパー関数
│   └── constants/       # 定数
├── types/                # TypeScript型
└── styles/              # グローバルスタイル
```

### ファイル命名

```
components/Button.tsx          # コンポーネントはPascalCase
hooks/useAuth.ts              # 'use'プレフィックスのcamelCase
lib/formatDate.ts             # ユーティリティはcamelCase
types/market.types.ts         # .typesサフィックスのcamelCase
```

## コメントとドキュメント

### コメントすべき場合

```typescript
// PASS: GOOD: WHYを説明し、WHATを説明しない
// 停止中のAPIを圧迫しないよう指数バックオフを使用する
const delay = Math.min(1000 * Math.pow(2, retryCount), 30000)

// 大きな配列のパフォーマンスのため意図的にここで変更を使用
items.push(newItem)

// FAIL: BAD: 明白なことを述べる
// カウンターを1増やす
count++

// 名前をユーザーの名前にする
name = user.name
```

### パブリックAPIのJSDoc

```typescript
/**
 * 意味的類似性を使ってマーケットを検索する。
 *
 * @param query - 自然言語検索クエリ
 * @param limit - 最大結果数（デフォルト：10）
 * @returns 類似度スコア順にソートされたマーケットの配列
 * @throws {Error} OpenAI APIが失敗するかRedisが利用不可の場合
 *
 * @example
 * ```typescript
 * const results = await searchMarkets('election', 5)
 * console.log(results[0].name) // "Trump vs Biden"
 * ```
 */
export async function searchMarkets(
  query: string,
  limit: number = 10
): Promise<Market[]> {
  // 実装
}
```

## パフォーマンスのベストプラクティス

### メモ化

```typescript
import { useMemo, useCallback } from 'react'

// PASS: GOOD: 高コストな計算をメモ化する
const sortedMarkets = useMemo(() => {
  return markets.sort((a, b) => b.volume - a.volume)
}, [markets])

// PASS: GOOD: コールバックをメモ化する
const handleSearch = useCallback((query: string) => {
  setSearchQuery(query)
}, [])
```

### 遅延読み込み

```typescript
import { lazy, Suspense } from 'react'

// PASS: GOOD: 重いコンポーネントを遅延読み込みする
const HeavyChart = lazy(() => import('./HeavyChart'))

export function Dashboard() {
  return (
    <Suspense fallback={<Spinner />}>
      <HeavyChart />
    </Suspense>
  )
}
```

### データベースクエリ

```typescript
// PASS: GOOD: 必要な列のみ選択する
const { data } = await supabase
  .from('markets')
  .select('id, name, status')
  .limit(10)

// FAIL: BAD: すべてを選択する
const { data } = await supabase
  .from('markets')
  .select('*')
```

## テスト標準

### テスト構造（AAAパターン）

```typescript
test('calculates similarity correctly', () => {
  // Arrange（準備）
  const vector1 = [1, 0, 0]
  const vector2 = [0, 1, 0]

  // Act（実行）
  const similarity = calculateCosineSimilarity(vector1, vector2)

  // Assert（検証）
  expect(similarity).toBe(0)
})
```

### テスト命名

```typescript
// PASS: GOOD: 説明的なテスト名
test('returns empty array when no markets match query', () => { })
test('throws error when OpenAI API key is missing', () => { })
test('falls back to substring search when Redis unavailable', () => { })

// FAIL: BAD: 曖昧なテスト名
test('works', () => { })
test('test search', () => { })
```

## コードスメル検出

以下のアンチパターンに注意する：

### 1. 長い関数
```typescript
// FAIL: BAD: 50行を超える関数
function processMarketData() {
  // 100行のコード
}

// PASS: GOOD: より小さな関数に分割する
function processMarketData() {
  const validated = validateData()
  const transformed = transformData(validated)
  return saveData(transformed)
}
```

### 2. 深いネスト
```typescript
// FAIL: BAD: 5段階以上のネスト
if (user) {
  if (user.isAdmin) {
    if (market) {
      if (market.isActive) {
        if (hasPermission) {
          // 何かする
        }
      }
    }
  }
}

// PASS: GOOD: 早期リターン
if (!user) return
if (!user.isAdmin) return
if (!market) return
if (!market.isActive) return
if (!hasPermission) return

// 何かする
```

### 3. マジックナンバー
```typescript
// FAIL: BAD: 説明のない数値
if (retryCount > 3) { }
setTimeout(callback, 500)

// PASS: GOOD: 名前付き定数
const MAX_RETRIES = 3
const DEBOUNCE_DELAY_MS = 500

if (retryCount > MAX_RETRIES) { }
setTimeout(callback, DEBOUNCE_DELAY_MS)
```

**覚えておくこと**：コード品質は妥協できない。明確で保守可能なコードが迅速な開発と自信を持ったリファクタリングを可能にする。
