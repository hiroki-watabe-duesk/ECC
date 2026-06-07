---
name: security-review
description: 認証の追加・ユーザー入力の処理・シークレットの取り扱い・APIエンドポイントの作成・決済/機密機能の実装時に使用します。包括的なセキュリティチェックリストとパターンを提供します。
origin: ECC
---

# セキュリティレビュースキル

このスキルは、すべてのコードがセキュリティのベストプラクティスに従い、潜在的な脆弱性を特定することを保証します。

## 有効化のタイミング

- 認証または認可の実装時
- ユーザー入力やファイルアップロードの処理時
- 新しい API エンドポイントの作成時
- シークレットや認証情報の取り扱い時
- 決済機能の実装時
- 機密データの保存または送信時
- サードパーティ API の統合時

## セキュリティチェックリスト

### 1. シークレット管理

#### 失敗: 絶対にしてはいけないこと
```typescript
const apiKey = "sk-proj-xxxxx"  // Hardcoded secret
const dbPassword = "password123" // In source code
```

#### 合格: 常にすべきこと
```typescript
const apiKey = process.env.OPENAI_API_KEY
const dbUrl = process.env.DATABASE_URL

// Verify secrets exist
if (!apiKey) {
  throw new Error('OPENAI_API_KEY not configured')
}
```

#### 確認手順
- [ ] API キー・トークン・パスワードのハードコードがないこと
- [ ] すべてのシークレットが環境変数に格納されていること
- [ ] `.env.local` が .gitignore に含まれていること
- [ ] git 履歴にシークレットが含まれていないこと
- [ ] 本番シークレットがホスティングプラットフォーム（Vercel・Railway）に設定されていること

### 2. 入力バリデーション

#### ユーザー入力を常に検証する
```typescript
import { z } from 'zod'

// Define validation schema
const CreateUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
  age: z.number().int().min(0).max(150)
})

// Validate before processing
export async function createUser(input: unknown) {
  try {
    const validated = CreateUserSchema.parse(input)
    return await db.users.create(validated)
  } catch (error) {
    if (error instanceof z.ZodError) {
      return { success: false, errors: error.errors }
    }
    throw error
  }
}
```

#### ファイルアップロードの検証
```typescript
function validateFileUpload(file: File) {
  // Size check (5MB max)
  const maxSize = 5 * 1024 * 1024
  if (file.size > maxSize) {
    throw new Error('File too large (max 5MB)')
  }

  // Type check
  const allowedTypes = ['image/jpeg', 'image/png', 'image/gif']
  if (!allowedTypes.includes(file.type)) {
    throw new Error('Invalid file type')
  }

  // Extension check
  const allowedExtensions = ['.jpg', '.jpeg', '.png', '.gif']
  const extension = file.name.toLowerCase().match(/\.[^.]+$/)?.[0]
  if (!extension || !allowedExtensions.includes(extension)) {
    throw new Error('Invalid file extension')
  }

  return true
}
```

#### 確認手順
- [ ] すべてのユーザー入力がスキーマで検証されていること
- [ ] ファイルアップロードが制限されていること（サイズ・タイプ・拡張子）
- [ ] クエリにユーザー入力を直接使用していないこと
- [ ] ホワイトリスト検証（ブラックリストではなく）を使用していること
- [ ] エラーメッセージが機密情報を漏洩しないこと

### 3. SQL インジェクション対策

#### 失敗: SQL を絶対に連結しないこと
```typescript
// DANGEROUS - SQL Injection vulnerability
const query = `SELECT * FROM users WHERE email = '${userEmail}'`
await db.query(query)
```

#### 合格: パラメータ化クエリを常に使用する
```typescript
// Safe - parameterized query
const { data } = await supabase
  .from('users')
  .select('*')
  .eq('email', userEmail)

// Or with raw SQL
await db.query(
  'SELECT * FROM users WHERE email = $1',
  [userEmail]
)
```

#### 確認手順
- [ ] すべてのデータベースクエリがパラメータ化クエリを使用していること
- [ ] SQL に文字列連結がないこと
- [ ] ORM/クエリビルダーが正しく使用されていること
- [ ] Supabase クエリが適切にサニタイズされていること

### 4. 認証と認可

#### JWT トークン処理
```typescript
// FAIL: WRONG: localStorage (vulnerable to XSS)
localStorage.setItem('token', token)

// PASS: CORRECT: httpOnly cookies
res.setHeader('Set-Cookie',
  `token=${token}; HttpOnly; Secure; SameSite=Strict; Max-Age=3600`)
```

#### 認可チェック
```typescript
export async function deleteUser(userId: string, requesterId: string) {
  // ALWAYS verify authorization first
  const requester = await db.users.findUnique({
    where: { id: requesterId }
  })

  if (requester.role !== 'admin') {
    return NextResponse.json(
      { error: 'Unauthorized' },
      { status: 403 }
    )
  }

  // Proceed with deletion
  await db.users.delete({ where: { id: userId } })
}
```

#### 行レベルセキュリティ（Supabase）
```sql
-- Enable RLS on all tables
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

-- Users can only view their own data
CREATE POLICY "Users view own data"
  ON users FOR SELECT
  USING (auth.uid() = id);

-- Users can only update their own data
CREATE POLICY "Users update own data"
  ON users FOR UPDATE
  USING (auth.uid() = id);
```

#### 確認手順
- [ ] トークンが httpOnly Cookie に保存されていること（localStorage ではなく）
- [ ] 機密操作の前に認可チェックが行われていること
- [ ] Supabase で行レベルセキュリティが有効になっていること
- [ ] ロールベースのアクセス制御が実装されていること
- [ ] セッション管理が安全であること

### 5. XSS 対策

#### HTML のサニタイズ
```typescript
import DOMPurify from 'isomorphic-dompurify'

// ALWAYS sanitize user-provided HTML
function renderUserContent(html: string) {
  const clean = DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'p'],
    ALLOWED_ATTR: []
  })
  return <div dangerouslySetInnerHTML={{ __html: clean }} />
}
```

#### コンテンツセキュリティポリシー

厳格な設定から始め、文書化された削除計画のもとでのみ緩和します。`'unsafe-inline'` や `'unsafe-eval'` をデフォルトにしないでください。これらは CSP の保護の多くを無効化し、一時的な互換性負債として扱うべきです。

```typescript
// next.config.js
const securityHeaders = [
  {
    key: 'Content-Security-Policy',
    value: `
      default-src 'self';
      base-uri 'self';
      object-src 'none';
      frame-ancestors 'none';
      script-src 'self';
      style-src 'self';
      img-src 'self' data: https:;
      font-src 'self';
      connect-src 'self' https://api.example.com;
    `.replace(/\s{2,}/g, ' ').trim()
  }
]
```

#### 確認手順
- [ ] ユーザー提供の HTML がサニタイズされていること
- [ ] CSP ヘッダーが設定されていること
- [ ] 未検証の動的コンテンツレンダリングがないこと
- [ ] React の組み込み XSS 保護が使用されていること

### 6. CSRF 対策

#### CSRF トークン
```typescript
import { csrf } from '@/lib/csrf'

export async function POST(request: Request) {
  const token = request.headers.get('X-CSRF-Token')

  if (!csrf.verify(token)) {
    return NextResponse.json(
      { error: 'Invalid CSRF token' },
      { status: 403 }
    )
  }

  // Process request
}
```

#### SameSite Cookie
```typescript
res.setHeader('Set-Cookie',
  `session=${sessionId}; HttpOnly; Secure; SameSite=Strict`)
```

#### 確認手順
- [ ] 状態変更操作に CSRF トークンが設定されていること
- [ ] すべての Cookie に SameSite=Strict が設定されていること
- [ ] ダブルサブミット Cookie パターンが実装されていること

### 7. レート制限

#### API レート制限
```typescript
import rateLimit from 'express-rate-limit'

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // 100 requests per window
  message: 'Too many requests'
})

// Apply to routes
app.use('/api/', limiter)
```

#### コストのかかる操作
```typescript
// Aggressive rate limiting for searches
const searchLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 10, // 10 requests per minute
  message: 'Too many search requests'
})

app.use('/api/search', searchLimiter)
```

#### 確認手順
- [ ] すべての API エンドポイントにレート制限が設定されていること
- [ ] コストのかかる操作により厳格な制限が設定されていること
- [ ] IP ベースのレート制限があること
- [ ] ユーザーベースのレート制限があること（認証済み）

### 8. 機密データの露出

#### ロギング
```typescript
// FAIL: WRONG: Logging sensitive data
console.log('User login:', { email, password })
console.log('Payment:', { cardNumber, cvv })

// PASS: CORRECT: Redact sensitive data
console.log('User login:', { email, userId })
console.log('Payment:', { last4: card.last4, userId })
```

#### エラーメッセージ
```typescript
// FAIL: WRONG: Exposing internal details
catch (error) {
  return NextResponse.json(
    { error: error.message, stack: error.stack },
    { status: 500 }
  )
}

// PASS: CORRECT: Generic error messages
catch (error) {
  console.error('Internal error:', error)
  return NextResponse.json(
    { error: 'An error occurred. Please try again.' },
    { status: 500 }
  )
}
```

#### 確認手順
- [ ] ログにパスワード・トークン・シークレットが含まれていないこと
- [ ] ユーザー向けエラーメッセージが汎用的であること
- [ ] 詳細なエラーがサーバーログのみに記録されていること
- [ ] ユーザーにスタックトレースが露出していないこと

### 9. ブロックチェーンセキュリティ（Solana）

#### ウォレット検証
```typescript
import { verify } from '@solana/web3.js'

async function verifyWalletOwnership(
  publicKey: string,
  signature: string,
  message: string
) {
  try {
    const isValid = verify(
      Buffer.from(message),
      Buffer.from(signature, 'base64'),
      Buffer.from(publicKey, 'base64')
    )
    return isValid
  } catch (error) {
    return false
  }
}
```

#### トランザクション検証
```typescript
async function verifyTransaction(transaction: Transaction) {
  // Verify recipient
  if (transaction.to !== expectedRecipient) {
    throw new Error('Invalid recipient')
  }

  // Verify amount
  if (transaction.amount > maxAmount) {
    throw new Error('Amount exceeds limit')
  }

  // Verify user has sufficient balance
  const balance = await getBalance(transaction.from)
  if (balance < transaction.amount) {
    throw new Error('Insufficient balance')
  }

  return true
}
```

#### 確認手順
- [ ] ウォレット署名が検証されていること
- [ ] トランザクションの詳細が検証されていること
- [ ] トランザクション前に残高チェックが行われていること
- [ ] 盲目的なトランザクション署名がないこと

### 10. 依存関係のセキュリティ

#### 定期的な更新
```bash
# Check for vulnerabilities
npm audit

# Fix automatically fixable issues
npm audit fix

# Update dependencies
npm update

# Check for outdated packages
npm outdated
```

#### ロックファイル
```bash
# ALWAYS commit lock files
git add package-lock.json

# Use in CI/CD for reproducible builds
npm ci  # Instead of npm install
```

#### 確認手順
- [ ] 依存関係が最新であること
- [ ] 既知の脆弱性がないこと（npm audit クリーン）
- [ ] ロックファイルがコミットされていること
- [ ] GitHub で Dependabot が有効になっていること
- [ ] 定期的なセキュリティ更新があること

## セキュリティテスト

### 自動セキュリティテスト
```typescript
// Test authentication
test('requires authentication', async () => {
  const response = await fetch('/api/protected')
  expect(response.status).toBe(401)
})

// Test authorization
test('requires admin role', async () => {
  const response = await fetch('/api/admin', {
    headers: { Authorization: `Bearer ${userToken}` }
  })
  expect(response.status).toBe(403)
})

// Test input validation
test('rejects invalid input', async () => {
  const response = await fetch('/api/users', {
    method: 'POST',
    body: JSON.stringify({ email: 'not-an-email' })
  })
  expect(response.status).toBe(400)
})

// Test rate limiting
test('enforces rate limits', async () => {
  const requests = Array(101).fill(null).map(() =>
    fetch('/api/endpoint')
  )

  const responses = await Promise.all(requests)
  const tooManyRequests = responses.filter(r => r.status === 429)

  expect(tooManyRequests.length).toBeGreaterThan(0)
})
```

## 本番デプロイ前のセキュリティチェックリスト

本番デプロイの前に必ず確認:

- [ ] **シークレット**: ハードコードされたシークレットがなく、すべて環境変数に格納されていること
- [ ] **入力バリデーション**: すべてのユーザー入力が検証されていること
- [ ] **SQL インジェクション**: すべてのクエリがパラメータ化されていること
- [ ] **XSS**: ユーザーコンテンツがサニタイズされていること
- [ ] **CSRF**: 保護が有効になっていること
- [ ] **認証**: 適切なトークン処理が行われていること
- [ ] **認可**: ロールチェックが実装されていること
- [ ] **レート制限**: すべてのエンドポイントで有効になっていること
- [ ] **HTTPS**: 本番環境で強制されていること
- [ ] **セキュリティヘッダー**: CSP・X-Frame-Options が設定されていること
- [ ] **エラー処理**: エラーに機密データが含まれていないこと
- [ ] **ロギング**: 機密データがログに記録されていないこと
- [ ] **依存関係**: 最新で脆弱性がないこと
- [ ] **行レベルセキュリティ**: Supabase で有効になっていること
- [ ] **CORS**: 適切に設定されていること
- [ ] **ファイルアップロード**: 検証済みであること（サイズ・タイプ）
- [ ] **ウォレット署名**: 検証済みであること（ブロックチェーンの場合）

## リソース

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Next.js セキュリティ](https://nextjs.org/docs/security)
- [Supabase セキュリティ](https://supabase.com/docs/guides/auth)
- [Web Security Academy](https://portswigger.net/web-security)

---

**覚えておいてください**: セキュリティはオプションではありません。1 つの脆弱性がプラットフォーム全体を危険にさらす可能性があります。疑わしい場合は、慎重な方を選んでください。
