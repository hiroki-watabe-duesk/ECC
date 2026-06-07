---
name: database-migrations
description: データベースマイグレーションのベストプラクティス（スキーマ変更・データ移行・ロールバック・ゼロダウンタイムデプロイ）。PostgreSQL・MySQL・主要ORM（Prisma・Drizzle・Kysely・Django・TypeORM・golang-migrate）に対応。
origin: ECC
---

# データベースマイグレーションパターン

本番システム向けの安全で可逆的なデータベーススキーマ変更。

## 有効化タイミング

- データベーステーブルの作成または変更
- カラムまたはインデックスの追加・削除
- データマイグレーションの実行（バックフィル、変換）
- ゼロダウンタイムスキーマ変更の計画
- 新プロジェクトへのマイグレーションツールのセットアップ

## 基本原則

1. **すべての変更はマイグレーション** — 本番データベースを手動で変更しない
2. **本番ではマイグレーションは一方向** — ロールバックは新しい前方マイグレーションで行う
3. **スキーマとデータのマイグレーションは分離** — DDLとDMLを一つのマイグレーションに混在させない
4. **本番規模のデータでマイグレーションをテスト** — 100行で動くマイグレーションが1000万行でロックする場合がある
5. **デプロイ済みマイグレーションは不変** — 本番で実行済みのマイグレーションは編集しない

## マイグレーション安全性チェックリスト

マイグレーションを適用する前に:

- [ ] マイグレーションにUPとDOWNの両方がある（または明示的に不可逆とマークされている）
- [ ] 大きなテーブルでの完全テーブルロックなし（コンカレント操作を使用）
- [ ] 新しいカラムにデフォルトがあるかNULL可（デフォルトなしでNOT NULLを追加しない）
- [ ] インデックスはコンカレントに作成（既存テーブルのCREATE TABLEにインラインで含めない）
- [ ] データバックフィルはスキーマ変更とは別のマイグレーション
- [ ] 本番データのコピーに対してテスト済み
- [ ] ロールバック計画が文書化されている

## PostgreSQLパターン

### カラムの安全な追加

```sql
-- 良い例: NULL可のカラム、ロックなし
ALTER TABLE users ADD COLUMN avatar_url TEXT;

-- 良い例: デフォルト付きカラム（Postgres 11+では即座で書き換えなし）
ALTER TABLE users ADD COLUMN is_active BOOLEAN NOT NULL DEFAULT true;

-- 悪い例: 既存テーブルへのデフォルトなしNOT NULL（完全な書き換えが必要）
ALTER TABLE users ADD COLUMN role TEXT NOT NULL;
-- これはテーブルをロックしてすべての行を書き換える
```

### ダウンタイムなしのインデックス追加

```sql
-- 悪い例: 大きなテーブルで書き込みをブロック
CREATE INDEX idx_users_email ON users (email);

-- 良い例: ノンブロッキング、コンカレント書き込みを許可
CREATE INDEX CONCURRENTLY idx_users_email ON users (email);

-- 注意: CONCURRENTLYはトランザクションブロック内では実行できない
-- 多くのマイグレーションツールでは特別な処理が必要
```

### カラム名の変更（ゼロダウンタイム）

本番環境では直接名前変更しないでください。展開-縮小パターンを使用します:

```sql
-- ステップ1: 新しいカラムを追加（マイグレーション001）
ALTER TABLE users ADD COLUMN display_name TEXT;

-- ステップ2: データをバックフィル（マイグレーション002、データマイグレーション）
UPDATE users SET display_name = username WHERE display_name IS NULL;

-- ステップ3: 両方のカラムを読み書きするようアプリケーションコードを更新する
-- アプリケーションの変更をデプロイ

-- ステップ4: 旧カラムへの書き込みを停止してドロップ（マイグレーション003）
ALTER TABLE users DROP COLUMN username;
```

### カラムの安全な削除

```sql
-- ステップ1: カラムへのすべてのアプリケーション参照を削除する
-- ステップ2: カラム参照なしでアプリケーションをデプロイする
-- ステップ3: 次のマイグレーションでカラムをドロップする
ALTER TABLE orders DROP COLUMN legacy_status;

-- Djangoの場合: SeparateDatabaseAndStateを使用してモデルから削除する
-- DROP COLUMNを生成せず（次のマイグレーションでドロップ）
```

### 大規模なデータマイグレーション

```sql
-- 悪い例: 一つのトランザクションで全行を更新（テーブルをロック）
UPDATE users SET normalized_email = LOWER(email);

-- 良い例: 進捗付きバッチ更新
DO $$
DECLARE
  batch_size INT := 10000;
  rows_updated INT;
BEGIN
  LOOP
    UPDATE users
    SET normalized_email = LOWER(email)
    WHERE id IN (
      SELECT id FROM users
      WHERE normalized_email IS NULL
      LIMIT batch_size
      FOR UPDATE SKIP LOCKED
    );
    GET DIAGNOSTICS rows_updated = ROW_COUNT;
    RAISE NOTICE 'Updated % rows', rows_updated;
    EXIT WHEN rows_updated = 0;
    COMMIT;
  END LOOP;
END $$;
```

## Prisma（TypeScript/Node.js）

### ワークフロー

```bash
# スキーマ変更からマイグレーションを作成
npx prisma migrate dev --name add_user_avatar

# 本番環境で保留中のマイグレーションを適用
npx prisma migrate deploy

# データベースをリセット（開発環境のみ）
npx prisma migrate reset

# スキーマ変更後にクライアントを生成
npx prisma generate
```

### スキーマ例

```prisma
model User {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String?
  avatarUrl String?  @map("avatar_url")
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")
  orders    Order[]

  @@map("users")
  @@index([email])
}
```

### カスタムSQLマイグレーション

Prismaが表現できない操作（コンカレントインデックス、データバックフィル）の場合:

```bash
# 空のマイグレーションを作成し、手動でSQLを編集する
npx prisma migrate dev --create-only --name add_email_index
```

```sql
-- migrations/20240115_add_email_index/migration.sql
-- PrismaはCONCURRENTLYを生成できないため、手動で記述する
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_users_email ON users (email);
```

## Drizzle（TypeScript/Node.js）

### ワークフロー

```bash
# スキーマ変更からマイグレーションを生成
npx drizzle-kit generate

# マイグレーションを適用
npx drizzle-kit migrate

# スキーマを直接プッシュ（開発環境のみ、マイグレーションファイルなし）
npx drizzle-kit push
```

### スキーマ例

```typescript
import { pgTable, text, timestamp, uuid, boolean } from "drizzle-orm/pg-core";

export const users = pgTable("users", {
  id: uuid("id").primaryKey().defaultRandom(),
  email: text("email").notNull().unique(),
  name: text("name"),
  isActive: boolean("is_active").notNull().default(true),
  createdAt: timestamp("created_at").notNull().defaultNow(),
  updatedAt: timestamp("updated_at").notNull().defaultNow(),
});
```

## Kysely（TypeScript/Node.js）

### ワークフロー（kysely-ctl）

```bash
# 設定ファイルを初期化（kysely.config.ts）
kysely init

# 新しいマイグレーションファイルを作成
kysely migrate make add_user_avatar

# 保留中のすべてのマイグレーションを適用
kysely migrate latest

# 最後のマイグレーションをロールバック
kysely migrate down

# マイグレーションの状態を表示
kysely migrate list
```

### マイグレーションファイル

```typescript
// migrations/2024_01_15_001_create_user_profile.ts
import { type Kysely, sql } from 'kysely'

// 重要: 型付きDBインターフェースではなく、常にKysely<any>を使用する。
// マイグレーションは時間的に固定されており、現在のスキーマ型に依存してはならない。
export async function up(db: Kysely<any>): Promise<void> {
  await db.schema
    .createTable('user_profile')
    .addColumn('id', 'serial', (col) => col.primaryKey())
    .addColumn('email', 'varchar(255)', (col) => col.notNull().unique())
    .addColumn('avatar_url', 'text')
    .addColumn('created_at', 'timestamp', (col) =>
      col.defaultTo(sql`now()`).notNull()
    )
    .execute()

  await db.schema
    .createIndex('idx_user_profile_avatar')
    .on('user_profile')
    .column('avatar_url')
    .execute()
}

export async function down(db: Kysely<any>): Promise<void> {
  await db.schema.dropTable('user_profile').execute()
}
```

### プログラマティックマイグレーター

```typescript
import { Migrator, FileMigrationProvider } from 'kysely'
import { promises as fs } from 'fs'
import * as path from 'path'
// ESMのみ — CJSは__dirnameを直接使用できる
import { fileURLToPath } from 'url'
const migrationFolder = path.join(
  path.dirname(fileURLToPath(import.meta.url)),
  './migrations',
)

// `db` はKysely<any>データベースインスタンス
const migrator = new Migrator({
  db,
  provider: new FileMigrationProvider({
    fs,
    path,
    migrationFolder,
  }),
  // 警告: 開発環境のみで有効にする。タイムスタンプ順序の
  // 検証を無効化し、環境間でスキーマのずれが生じる可能性がある。
  // allowUnorderedMigrations: true,
})

const { error, results } = await migrator.migrateToLatest()

results?.forEach((it) => {
  if (it.status === 'Success') {
    console.log(`migration "${it.migrationName}" executed successfully`)
  } else if (it.status === 'Error') {
    console.error(`failed to execute migration "${it.migrationName}"`)
  }
})

if (error) {
  console.error('migration failed', error)
  process.exit(1)
}
```

## Django（Python）

### ワークフロー

```bash
# モデル変更からマイグレーションを生成
python manage.py makemigrations

# マイグレーションを適用
python manage.py migrate

# マイグレーションの状態を表示
python manage.py showmigrations

# カスタムSQL用の空マイグレーションを生成
python manage.py makemigrations --empty app_name -n description
```

### データマイグレーション

```python
from django.db import migrations

def backfill_display_names(apps, schema_editor):
    User = apps.get_model("accounts", "User")
    batch_size = 5000
    users = User.objects.filter(display_name="")
    while users.exists():
        batch = list(users[:batch_size])
        for user in batch:
            user.display_name = user.username
        User.objects.bulk_update(batch, ["display_name"], batch_size=batch_size)

def reverse_backfill(apps, schema_editor):
    pass  # データマイグレーションのため逆処理は不要

class Migration(migrations.Migration):
    dependencies = [("accounts", "0015_add_display_name")]

    operations = [
        migrations.RunPython(backfill_display_names, reverse_backfill),
    ]
```

### SeparateDatabaseAndState

Djangoモデルからカラムを削除する際に、データベースから即座にドロップしない:

```python
class Migration(migrations.Migration):
    operations = [
        migrations.SeparateDatabaseAndState(
            state_operations=[
                migrations.RemoveField(model_name="user", name="legacy_field"),
            ],
            database_operations=[],  # まだDBには触らない
        ),
    ]
```

## golang-migrate（Go）

### ワークフロー

```bash
# マイグレーションペアを作成
migrate create -ext sql -dir migrations -seq add_user_avatar

# 保留中のすべてのマイグレーションを適用
migrate -path migrations -database "$DATABASE_URL" up

# 最後のマイグレーションをロールバック
migrate -path migrations -database "$DATABASE_URL" down 1

# バージョンを強制（ダーティ状態を修正）
migrate -path migrations -database "$DATABASE_URL" force VERSION
```

### マイグレーションファイル

```sql
-- migrations/000003_add_user_avatar.up.sql
ALTER TABLE users ADD COLUMN avatar_url TEXT;
CREATE INDEX CONCURRENTLY idx_users_avatar ON users (avatar_url) WHERE avatar_url IS NOT NULL;

-- migrations/000003_add_user_avatar.down.sql
DROP INDEX IF EXISTS idx_users_avatar;
ALTER TABLE users DROP COLUMN IF EXISTS avatar_url;
```

## ゼロダウンタイムマイグレーション戦略

重要な本番変更には、展開-縮小パターンに従ってください:

```
フェーズ1: 展開（EXPAND）
  - 新しいカラム/テーブルを追加（NULL可またはデフォルト付き）
  - デプロイ: アプリが旧と新の両方に書き込む
  - 既存データをバックフィル

フェーズ2: 移行（MIGRATE）
  - デプロイ: アプリが新から読み込み、両方に書き込む
  - データの一貫性を検証

フェーズ3: 縮小（CONTRACT）
  - デプロイ: アプリが新のみを使用
  - 別のマイグレーションで旧カラム/テーブルをドロップ
```

### タイムライン例

```
1日目: マイグレーションでnew_statusカラムを追加（NULL可）
1日目: アプリv2をデプロイ — statusとnew_statusの両方に書き込む
2日目: 既存行のバックフィルマイグレーションを実行
3日目: アプリv3をデプロイ — new_statusのみから読み込む
7日目: 旧statusカラムをドロップするマイグレーション
```

## アンチパターン

| アンチパターン | 失敗する理由 | より良いアプローチ |
|-------------|-------------|-----------------|
| 本番環境での手動SQL | 監査証跡なし、再現不可 | 常にマイグレーションファイルを使用する |
| デプロイ済みマイグレーションの編集 | 環境間でずれが生じる | 代わりに新しいマイグレーションを作成する |
| デフォルトなしのNOT NULL | テーブルをロックして全行を書き換える | NULL可で追加し、バックフィル後に制約を追加する |
| 大きなテーブルへのインラインインデックス | 構築中に書き込みをブロック | CREATE INDEX CONCURRENTLY |
| 一つのマイグレーションにスキーマとデータ | ロールバックが困難、長いトランザクション | マイグレーションを分離する |
| コードを削除する前にカラムをドロップ | 欠損カラムでアプリケーションエラー | まずコードを削除し、次のデプロイでカラムをドロップする |
