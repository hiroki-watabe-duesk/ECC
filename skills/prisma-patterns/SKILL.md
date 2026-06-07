---
name: prisma-patterns
description: TypeScript バックエンド向け Prisma ORM パターン — スキーマ設計、クエリ最適化、トランザクション、ページネーション、および updateMany がレコードではなく count を返す・$transaction タイムアウト・migrate dev が DB をリセットする・@updatedAt が一括書き込みをスキップする・サーバーレスでの接続枯渇などの重要な落とし穴。
origin: ECC
---

# Prisma パターン

TypeScript バックエンドにおける Prisma ORM のプロダクションパターンと、知っておくべき落とし穴。
Prisma 5.x および 6.x に対してテスト済み。一部の挙動は Prisma 4 と異なります。

バージョン固有のパターンを適用する前に、Prisma のバージョンを確認してください:

```bash
npx prisma --version
```

Prisma 5 では `relationJoins` が導入され、クエリ戦略と設定によっては、別々のクエリではなく JOIN でリレーションを読み込めるようになりました。`omit` フィールド修飾子と `prisma.$extends` Client Extensions API も追加されました。注意: `relationJoins` は大きな 1:N リレーションや深いネストの `include` でロウが爆発的に増える場合があります — リレーションが親ごとに多くのロウを返す可能性がある場合は、両方のアプローチをベンチマークしてください。

## 有効化する場面

- Prisma スキーマのモデルとリレーションを設計または変更する場合
- クエリ、トランザクション、またはページネーションロジックを書く場合
- `updateMany`、`deleteMany`、その他の一括操作を使用する場合
- データベースマイグレーションを実行または計画する場合
- サーバーレス環境（Vercel、Lambda、Cloudflare Workers）にデプロイする場合
- ソフトデリートまたはマルチテナントのロウフィルタリングを実装する場合

## コアコンセプト

### ID 戦略

| 戦略 | 使用する場面 | 避ける場面 |
|---|---|---|
| `@default(cuid())` | デフォルトの選択 — URL セーフ、ソート可能、衝突なし | 外部システムで連番 ID が必要な場合 |
| `@default(uuid())` | Prisma 以外のシステムとの相互運用が必要な場合 | 書き込みが多いテーブル（ランダム UUID は B ツリーインデックスを断片化する） |
| `@default(autoincrement())` | 内部結合テーブル、監査ログ | 公開向け ID（レコード数が露出する） |

### スキーマのデフォルト

```prisma
model User {
  id        String    @id @default(cuid())
  email     String    @unique  // @unique already creates an index — no @@index needed
  name      String
  role      Role      @default(USER)
  posts     Post[]
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
  deletedAt DateTime?

  @@index([createdAt])
  @@index([deletedAt, createdAt]) // composite for soft-delete + sort queries
}
```

- `WHERE` や `ORDER BY` で使用するすべての外部キーとカラムに `@@index` を追加してください。
- ソフトデリートが将来的な要件になりそうな場合は、最初から `deletedAt DateTime?` を宣言してください — 後から追加するにはライブテーブルへのマイグレーションが必要になります。
- `updatedAt @updatedAt` は Prisma が `update` および `upsert` 時にのみ自動的に設定します（一括更新の落とし穴はアンチパターンを参照）。

### `include` vs `select`

| | `include` | `select` |
|---|---|---|
| 返却内容 | すべてのスカラーフィールド + 指定したリレーション | 指定したフィールドのみ |
| 使用する場面 | ほとんどのフィールドとリレーションが必要な場合 | ホットパス、大きなテーブル、オーバーフェッチを避けたい場合 |
| パフォーマンス | 幅の広いテーブルでオーバーフェッチの可能性 | ペイロードが最小、大きなデータセットで高速 |
| Prisma 5 の注意 | デフォルトで JOIN を使用（`relationJoins`） | 同上 |

```ts
// include — all columns + relation
const user = await prisma.user.findUnique({
  where: { id },
  include: { posts: { select: { id: true, title: true } } },
});

// select — explicit allowlist
const user = await prisma.user.findUnique({
  where: { id },
  select: { id: true, email: true, name: true },
});
```

API レスポンスから生の Prisma エンティティを返さないでください — 公開フィールドを制御するためにレスポンス DTO にマッピングしてください:

```ts
// BAD: leaks passwordHash, deletedAt, internal fields
return await prisma.user.findUniqueOrThrow({ where: { id } });

// GOOD: explicit DTO mapping
const user = await prisma.user.findUniqueOrThrow({ where: { id } });
return { id: user.id, name: user.name, email: user.email };
```

### トランザクション形式の選択

| 状況 | 使用する形式 |
|---|---|
| 独立した操作で相互依存なし | 配列形式 |
| 後のステップが前の結果に依存する | インタラクティブ形式 |
| 外部呼び出し（メール、HTTP）が含まれる | トランザクションの外側で実行 |

```ts
// Array form — batched in one round trip
const [user, post] = await prisma.$transaction([
  prisma.user.update({ where: { id }, data: { name } }),
  prisma.post.create({ data: { title, authorId: id } }),
]);

// Interactive form — use tx client only, never the outer prisma client
const post = await prisma.$transaction(async (tx) => {
  const user = await tx.user.findUniqueOrThrow({ where: { id } });
  if (user.role !== 'ADMIN') throw new Error('Forbidden');
  return tx.post.create({ data: { title, authorId: user.id } });
});
```

### PrismaClient シングルトン

`PrismaClient` インスタンスはそれぞれ独自のコネクションプールを開きます。一度だけインスタンス化してください。

```ts
// lib/prisma.ts
import { PrismaClient } from '@prisma/client';

const globalForPrisma = globalThis as unknown as { prisma?: PrismaClient };

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: process.env.NODE_ENV === 'development' ? ['query', 'error'] : ['error'],
  });

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma;
```

`globalThis` パターンにより、ホットリロード（Next.js、nodemon、ts-node-dev）時の重複インスタンス生成を防ぎます。

### N+1 問題

ループ内でリレーションを読み込むと、ロウごとに 1 つのクエリが発行されます。

```ts
// BAD: N+1 — one extra query per user
const users = await prisma.user.findMany();
for (const user of users) {
  const posts = await prisma.post.findMany({ where: { authorId: user.id } });
}

// GOOD: single query
const users = await prisma.user.findMany({ include: { posts: true } });
```

Prisma 5 以降の `relationJoins` では、`include` 形式が単一の JOIN を使用します。大きな 1:N セットでは結果セットのサイズが増加する可能性があります — リレーションが親ごとに多くのロウを返す場合は、両方のアプローチをベンチマークしてください。

## コードサンプル

### カーソルページネーション（フィードや大きなデータセットに推奨）

```ts
async function getPosts(cursor?: string, limit = 20) {
  const items = await prisma.post.findMany({
    where: { published: true },
    orderBy: [
      { createdAt: 'desc' },
      { id: 'desc' }, // secondary sort prevents unstable pagination on duplicate timestamps
    ],
    take: limit + 1,
    ...(cursor && { cursor: { id: cursor }, skip: 1 }),
  });

  const hasNextPage = items.length > limit;
  if (hasNextPage) items.pop();

  return { items, nextCursor: hasNextPage ? items[items.length - 1].id : null };
}
```

`limit + 1` を取得してポップする方法が、追加のカウントクエリなしに `hasNextPage` を検出する標準的なやり方です。複数のロウが同じタイムスタンプを共有する場合の不安定なページネーションを防ぐため、一意なフィールド（例: `id`）を二次的な `orderBy` として常に含めてください。ユーザーが任意のページにジャンプする必要がある場合（管理テーブルなど）にのみオフセットページネーションを使用してください。

### ソフトデリート

```ts
// Always filter explicitly — do not rely on middleware (hides behavior, hard to debug)
const activeUsers = await prisma.user.findMany({ where: { deletedAt: null } });

await prisma.user.update({ where: { id }, data: { deletedAt: new Date() } });
await prisma.user.update({ where: { id }, data: { deletedAt: null } }); // restore
```

### エラーハンドリング

```ts
import { Prisma } from '@prisma/client';

try {
  await prisma.user.create({ data: { email } });
} catch (e) {
  if (e instanceof Prisma.PrismaClientKnownRequestError) {
    if (e.code === 'P2002') throw new ConflictError('Email already exists');
    if (e.code === 'P2025') throw new NotFoundError('Record not found');
    if (e.code === 'P2003') throw new BadRequestError('Referenced record does not exist');
  }
  throw e;
}
```

よく使うコード: `P2002` ユニーク制約違反 · `P2025` 見つからない · `P2003` 外部キー制約違反。

サービス境界でキャッチしてドメインエラーに変換してください。生の Prisma メッセージを API 消費者に公開しないでください。

### コネクションプール — サーバーレス

接続パラメータを `DATABASE_URL` に直接埋め込んでください — URL にすでにクエリパラメータ（例: `?schema=public`）がある場合、文字列連結では壊れます:

```bash
# .env — preferred: embed params in the URL
DATABASE_URL="postgresql://user:pass@host/db?connection_limit=1&pool_timeout=20"

# With an external pooler (PgBouncer, Supabase pooler)
DATABASE_URL="postgresql://user:pass@host/db?pgbouncer=true&connection_limit=1"
```

```ts
// Vercel, AWS Lambda, and similar serverless runtimes: cap pool to 1 per instance
// connection_limit and pool_timeout are controlled via DATABASE_URL
const prisma = new PrismaClient();
```

## アンチパターン

### `updateMany` はレコードではなく count を返す

```ts
// BAD: result is { count: 2 } — users[0] is undefined
const users = await prisma.user.updateMany({ where: { role: 'GUEST' }, data: { role: 'USER' } });

// GOOD: capture IDs first, then update, then fetch only the affected rows
const targets = await prisma.user.findMany({
  where: { role: 'GUEST' },
  select: { id: true },
});
const ids = targets.map((u) => u.id);
await prisma.user.updateMany({ where: { id: { in: ids } }, data: { role: 'USER' } });
const updated = await prisma.user.findMany({ where: { id: { in: ids } } });
```

`deleteMany` も同様 — `{ count: n }` を返し、削除されたロウは返しません。

### `$transaction` インタラクティブ形式は 5 秒後にタイムアウトする

```ts
// BAD: external call inside transaction exceeds 5s default → "Transaction already closed"
await prisma.$transaction(async (tx) => {
  const user = await tx.user.findUniqueOrThrow({ where: { id } });
  await sendWelcomeEmail(user.email); // external call
  await tx.user.update({ where: { id }, data: { emailSent: true } });
});

// GOOD: external calls outside the transaction
const user = await prisma.user.findUniqueOrThrow({ where: { id } });
await sendWelcomeEmail(user.email);
await prisma.user.update({ where: { id }, data: { emailSent: true } });

// Only raise timeout when bulk processing genuinely needs it
await prisma.$transaction(async (tx) => { ... }, { timeout: 30_000 });
```

### `migrate dev` はデータベースをリセットする可能性がある

`migrate dev` はスキーマのドリフトを検出し、DB のリセットを促す場合があり、すべてのデータが失われます。

```bash
# NEVER on shared dev, staging, or production
npx prisma migrate dev --name add_column

# Safe everywhere except local solo dev
npx prisma migrate deploy

# Check drift without applying
npx prisma migrate diff \
  --from-migrations ./prisma/migrations \
  --to-schema-datamodel ./prisma/schema.prisma \
  --shadow-database-url "$SHADOW_DATABASE_URL"
```

### マイグレーションファイルを手動で編集すると将来のデプロイが壊れる

Prisma はすべてのマイグレーションファイルをチェックサムで管理します。適用後に編集すると、すでに元のマイグレーションを実行済みのすべての環境で `P3006 checksum mismatch` が発生します。代わりに新しいマイグレーションを作成してください。

### 破壊的なスキーマ変更にはマルチステップマイグレーションが必要

既存のカラムに `NOT NULL` を追加したり、1 つのマイグレーションでカラムをリネームしたりすると、テーブルがロックされたりデータが失われる可能性があります。展開と縮小（expand-and-contract）を使用してください:

```bash
# Step 1: create migration locally, then deploy
npx prisma migrate dev --name add_new_column   # local only
npx prisma migrate deploy                       # staging / production
```

```ts
// Step 2: backfill data (run in a script or migration job, not in the shell)
await prisma.user.updateMany({ data: { newColumn: derivedValue } });
```

```bash
# Step 3: create the NOT NULL constraint migration locally, then deploy
npx prisma migrate dev --name make_new_column_required  # local only
npx prisma migrate deploy                               # staging / production
```

### `@updatedAt` は `updateMany` では発火しない

`@updatedAt` は `update` および `upsert` 時にのみ自動的に設定されます。一括書き込みでは古い値のままになります。

```ts
// BAD: updatedAt stays at its old value
await prisma.post.updateMany({ where: { authorId }, data: { published: true } });

// GOOD
await prisma.post.updateMany({
  where: { authorId },
  data: { published: true, updatedAt: new Date() },
});
```

### ソフトデリート + `findUniqueOrThrow` は削除済みレコードを漏洩する

`findUniqueOrThrow` は DB にロウが存在しない場合にのみ `P2025` をスローします。ソフトデリートされたロウはまだ存在しており、エラーなしに返されます。

`findUniqueOrThrow` は `where` に一意制約フィールドが必要です — `{ id, deletedAt }` は複合ユニーク制約ではないため、`id` と共に `deletedAt: null` を追加するとタイプエラーになります。代わりに `findFirstOrThrow` を使用してください。

```ts
// BAD: returns soft-deleted user
const user = await prisma.user.findUniqueOrThrow({ where: { id } });

// BAD: Prisma type error — { id, deletedAt } is not a unique constraint
const user = await prisma.user.findUniqueOrThrow({ where: { id, deletedAt: null } });

// GOOD: findFirstOrThrow supports arbitrary where conditions
const user = await prisma.user.findFirstOrThrow({ where: { id, deletedAt: null } });
```

### `where` なしの `deleteMany` はすべてのロウを削除する

```ts
// BAD: silently wipes the table
await prisma.post.deleteMany();

// GOOD
await prisma.post.deleteMany({ where: { authorId: userId } });
```

## ベストプラクティス

| ルール | 理由 |
|---|---|
| CI/CD では `migrate deploy`、ローカルのみ `migrate dev` | `migrate dev` はドリフト時に DB をリセットする可能性がある |
| エンティティをレスポンス DTO にマッピングする | 内部フィールドの漏洩を防ぐ |
| サービス境界で `PrismaClientKnownRequestError` をキャッチする | ドメインエラーに変換する |
| 手動の null チェックより `*OrThrow` メソッドを優先する | P2025 を自動でスロー; 非ユニークフィールドのフィルタリング時は `findFirstOrThrow` を使用する |
| サーバーレスでは `connection_limit=1` + 外部プーラー | コネクション枯渇を防ぐ |
| `deleteMany` には常に `where` を指定する | 誤ってテーブルを全削除することを防ぐ |
| `updateMany` では手動で `updatedAt: new Date()` を設定する | `@updatedAt` は一括書き込みをスキップする |

## 関連スキル

- `nestjs-patterns` — Prisma を統合する NestJS サービスレイヤー
- `postgres-patterns` — PostgreSQL レベルのインデックスとコネクションチューニング
- `database-migrations` — プロダクション向けマルチステップマイグレーション計画
- `backend-patterns` — 一般的な API とサービスレイヤーの設計
