---
name: mysql-patterns
description: 本番バックエンド向けのMySQL・MariaDBスキーマ、クエリ、インデックス、トランザクション、レプリケーション、コネクションプールのパターン集。
origin: ECC
---

# MySQL パターン

MySQL または MariaDB のスキーマ設計、マイグレーション、スロークエリ調査、キュー型トランザクション、コネクションプール、本番データベース設定に関わる作業でこのスキルを使用してください。機能固有のパターンを適用する前に、MySQL と MariaDB では SQL の詳細が複数の点で乖離しているため、正確なバージョンを確認することを推奨します。

## 起動条件

- MySQL または MariaDB のテーブル、インデックス、制約の設計
- 大規模本番テーブルで実行する前のマイグレーションレビュー
- スロークエリ、ロック待ち、デッドロック、コネクション枯渇のデバッグ
- キーセットページネーション、アップサート、全文検索、JSON カラム、キューの追加
- アプリケーションのコネクションプール、読み取りレプリカ、TLS、スローログの設定

## バージョン確認

まずエンジンとバージョンを特定します:

```sql
SELECT VERSION();
SHOW VARIABLES LIKE 'version_comment';
```

構文が異なる場合は MySQL と MariaDB のガイダンスを分けて管理します:

- MySQL は `ON DUPLICATE KEY UPDATE` における `VALUES(col)` の代替としてロウエイリアスを文書化しています。`VALUES(col)` はそこでは非推奨です。
- MariaDB は `ON DUPLICATE KEY UPDATE` で挿入値を参照するサポート済みの方法として `VALUES(col)` を文書化しています。クロスエンジン互換性のためにこちらを使用してください。
- `SKIP LOCKED` はキュー型ワークロードにのみ適しています。ロックされた行をスキップして不整合なビューを返す可能性があるため、一般的な会計処理や整合性を重視した読み取りには使用しないでください。

## スキーマのデフォルト

```sql
CREATE TABLE orders (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    account_id BIGINT UNSIGNED NOT NULL,
    status VARCHAR(32) NOT NULL,
    total DECIMAL(15, 2) NOT NULL,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted_at DATETIME NULL,
    PRIMARY KEY (id),
    KEY idx_orders_account_status_created (account_id, status, created_at),
    KEY idx_orders_active (account_id, deleted_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

デフォルトの選択方針:

| ユースケース | 推奨 | 避けるべき |
| --- | --- | --- |
| サロゲート主キー | `BIGINT UNSIGNED AUTO_INCREMENT` | 20億行を超える可能性があるテーブルへの `INT` |
| UUID ルックアップキー | 変換ヘルパー付きの `BINARY(16)` | ホットテーブルへの `VARCHAR(36)` 主キー |
| 金額・厳密な数量 | `DECIMAL(p, s)` | `FLOAT` や `DOUBLE` |
| ユーザー向けテキスト | `utf8mb4` テーブルとインデックス | MySQL の `utf8` / `utf8mb3` デフォルト |
| アプリケーションのタイムスタンプ | アプリが UTC を管理する `DATETIME` | `DATETIME` がタイムゾーンメタデータを格納するという前提 |
| ソフトデリート | `deleted_at DATETIME NULL` + スコープ付きインデックス | インデックスなしでソフトデリート済み行をフィルタリング |
| 拡張可能なステータス値 | ルックアップテーブルまたは制約付き `VARCHAR` | 値が頻繁に変わる場合の `ENUM` |

## インデックス

複合インデックスの順序は通常、等値述語を先にし、次に範囲またはソートカラムを置きます:

```sql
CREATE INDEX idx_orders_account_status_created
    ON orders (account_id, status, created_at);

SELECT id, total
FROM orders
WHERE account_id = ?
  AND status = 'pending'
  AND created_at >= ?
ORDER BY created_at DESC
LIMIT 50;
```

インデックスを追加・変更する前に `EXPLAIN` を使用します:

```sql
EXPLAIN
SELECT id, total
FROM orders
WHERE account_id = 123 AND status = 'pending'
ORDER BY created_at DESC
LIMIT 50;
```

調査すべきシグナル:

| フィールド | リスクシグナル |
| --- | --- |
| `type` | 大きなテーブルへの `ALL` |
| `key` | 選択性の高い述語があるのに `NULL` |
| `rows` | インタラクティブパスにしては非常に高い行数推定 |
| `Extra` | `Using temporary`、`Using filesort`、広範な `Using where` |

インデックスを盲目的に追加しないでください。各インデックスは書き込みコスト、マイグレーション時間、バックアップサイズ、バッファプールへの圧力を増大させます。

## クエリパターン

### アップサート

クロスエンジン互換の形式:

```sql
INSERT INTO user_settings (user_id, setting_key, setting_value)
VALUES (?, ?, ?)
ON DUPLICATE KEY UPDATE
    setting_value = VALUES(setting_value),
    updated_at = CURRENT_TIMESTAMP;
```

MySQL ロウエイリアス形式:

```sql
INSERT INTO user_settings (user_id, setting_key, setting_value)
VALUES (?, ?, ?) AS new
ON DUPLICATE KEY UPDATE
    setting_value = new.setting_value,
    updated_at = CURRENT_TIMESTAMP;
```

ロウエイリアス形式はターゲットが MySQL であることを確認した後にのみ使用してください。MariaDB または MySQL/MariaDB 混在環境では `VALUES(col)` を使用してください。

### キーセットページネーション

```sql
SELECT id, name, created_at
FROM products
WHERE (created_at, id) < (?, ?)
ORDER BY created_at DESC, id DESC
LIMIT 50;
```

カーソルに合わせたインデックスで裏付けます:

```sql
CREATE INDEX idx_products_created_id ON products (created_at, id);
```

大きなテーブルで深い `OFFSET` ページネーションを使用しないでください。サーバーがページを返す前に行をスキャンして破棄するため、パフォーマンスが悪化します。

### JSON フィールド

JSON カラムは拡張データに使用し、重いリレーショナルフィルタリングや制約が必要なフィールドには使用しないでください。

```sql
CREATE TABLE events (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    payload JSON NOT NULL,
    event_type VARCHAR(64)
        GENERATED ALWAYS AS (JSON_UNQUOTE(JSON_EXTRACT(payload, '$.type'))) STORED,
    KEY idx_events_type (event_type)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

頻繁にクエリされる JSON パスには生成カラムを公開してそのカラムにインデックスを張ります。外部キー、所有関係、テナンシー、ライフサイクルフィールドはリレーショナルのままにしてください。

### 全文検索

```sql
ALTER TABLE articles ADD FULLTEXT KEY ft_articles_title_body (title, body);

SELECT id, title, MATCH(title, body) AGAINST (? IN NATURAL LANGUAGE MODE) AS score
FROM articles
WHERE MATCH(title, body) AGAINST (? IN NATURAL LANGUAGE MODE)
ORDER BY score DESC
LIMIT 20;
```

タイポ許容、複雑なランキング、クロステーブルファセット、または組み込みの全文検索を超える言語固有の解析が必要な場合は外部検索エンジンを使用してください。

## トランザクション

トランザクションを短く保ち、一貫した順序で行をロックします:

```sql
START TRANSACTION;

SELECT id, balance
FROM accounts
WHERE id IN (?, ?)
ORDER BY id
FOR UPDATE;

UPDATE accounts SET balance = balance - ? WHERE id = ?;
UPDATE accounts SET balance = balance + ? WHERE id = ?;

COMMIT;
```

デッドロック・ロック待ちのチェックリスト:

- コードパスをまたいで決定論的な順序で行をロックする。
- 外部 API 呼び出しはトランザクションを開く前に行い、内部では行わない。
- `UPDATE`、`DELETE`、ロック読み取りで使用する述語にインデックスを追加する。
- デッドロック発生時はロールバックし、制限付きリトライ予算でトランザクション全体をリトライする。
- デッドロック直後に `SHOW ENGINE INNODB STATUS\G` を取得する。後続のイベントで上書きされるため速やかに行う。

キュー型ワーカークレーム:

```sql
START TRANSACTION;

SELECT id
FROM jobs
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;

UPDATE jobs
SET status = 'processing', started_at = CURRENT_TIMESTAMP
WHERE id = ?;

COMMIT;
```

`SKIP LOCKED` は、ロックされた行をスキップすることが許容されるキュー型ワークロードにのみ使用してください。通常のトランザクション整合性の代替ではありません。

## コネクションプール

SQLAlchemy の例:

```python
from sqlalchemy import create_engine

engine = create_engine(
    "mysql+mysqlconnector://app:secret@db.internal/app",
    pool_size=10,
    max_overflow=5,
    pool_timeout=30,
    pool_recycle=240,
    pool_pre_ping=True,
    connect_args={"connect_timeout": 5},
)
```

Node.js `mysql2` の例:

```javascript
import mysql from 'mysql2/promise';

const pool = mysql.createPool({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
  waitForConnections: true,
  connectionLimit: 10,
  queueLimit: 0,
  enableKeepAlive: true,
  keepAliveInitialDelay: 30000,
});

const [rows] = await pool.execute(
  'SELECT id, total FROM orders WHERE account_id = ? LIMIT 50',
  [accountId],
);
```

アプリケーションのプールリサイクルはサーバーの `wait_timeout` より短く保ってください。サーバーが `wait_timeout = 300` を使用している場合、`pool_recycle` を約 240 秒にするのが適切です。`pool_pre_ping` はネットワーク障害やフェイルオーバーイベントからの回復にも役立ちます。

## 診断

最初に確認する便利なコマンド:

```sql
SHOW FULL PROCESSLIST;
SHOW ENGINE INNODB STATUS\G;
SHOW VARIABLES LIKE 'slow_query_log';
SHOW VARIABLES LIKE 'long_query_time';
```

制御された環境でスローログを有効化:

```sql
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;
SET GLOBAL log_queries_not_using_indexes = 'ON';
```

`EXPLAIN ANALYZE` はクエリを実行しても安全な場合にのみ使用してください。ステートメントを実際に実行するため、本番サイズのデータでは高コストになる場合があります。

## レプリケーション

読み取りレプリカは遅延する可能性があります。書き込み直後に、自分の書き込みを読み取るパス、チェックアウトフロー、権限チェック、冪等性キーの読み取りをレプリカにルーティングしないでください。

```sql
-- MySQL の旧来の用語（既存のシステムでまだ一般的）
SHOW SLAVE STATUS\G;

-- サポートされている場合の新しい用語
SHOW REPLICA STATUS\G;
```

コマンドを統一する前にエンジン/バージョンを確認してください。TCP 接続が生きているかだけでなく、レプリカの SQL スレッド状態、IO スレッド状態、遅延を監視してください。

## セキュリティ

```sql
CREATE USER 'app'@'%' IDENTIFIED BY 'use-a-secret-manager';
GRANT SELECT, INSERT, UPDATE, DELETE ON appdb.* TO 'app'@'%';

ALTER USER 'app'@'%' REQUIRE SSL;

SELECT user, host
FROM mysql.user
WHERE user = '';

DROP USER IF EXISTS ''@'localhost';
DROP USER IF EXISTS ''@'%';
```

セキュリティレビューのポイント:

- アプリケーションユーザーに `ALL PRIVILEGES` や `*.*` を付与しない。
- トラフィックがホストやネットワークをまたぐ場合、アプリケーションユーザーに TLS を要求する。
- 認証情報はプラットフォームのシークレットマネージャーに保存し、例示、スクリプト、リポジトリファイルには記載しない。
- マイグレーション/管理ユーザーとランタイムアプリケーションユーザーを分離する。
- パフォーマンスチューニングの前に公開ネットワークへの露出とバインドアドレスを監査する。

## 設定

専用データベースホスト向けの設定例:

```ini
[mysqld]
innodb_buffer_pool_size = 4G
innodb_flush_log_at_trx_commit = 1
sync_binlog = 1

max_connections = 300
thread_cache_size = 50

wait_timeout = 300
interactive_timeout = 300
innodb_lock_wait_timeout = 10

slow_query_log = ON
long_query_time = 1
log_queries_not_using_indexes = ON

log_bin = mysql-bin
binlog_format = ROW
binlog_expire_logs_seconds = 604800
```

設定値はレビューのためのたたき台として扱い、普遍的なプリセットとして扱わないでください。メモリ、コネクション数、ログ保持期間、耐久性設定はワークロード、ハードウェア、バックアップポリシー、復旧目標から適切にサイズを決定してください。

## アンチパターン

| アンチパターン | リスク | より良いパターン |
| --- | --- | --- |
| ホットパスでの `SELECT *` | 過剰フェッチと脆弱なクライアント | 明示的なカラムを SELECT する |
| 深い `OFFSET` ページネーション | 線形スキャンと遅いページ | キーセットページネーション |
| 外部キー結合のインデックスなし | 遅い結合とロックの多い削除 | FK カラムに意図的にインデックスを張る |
| 長いトランザクション | ロック待ちと大きなアンドゥ履歴 | 小さな作業単位をコミットする |
| `mysql.user` への直接 DML | グラントテーブルの破損リスク | `CREATE USER`、`ALTER USER`、`DROP USER` を使用する |
| 管理権限を持つアプリケーションユーザー | 爆発半径が大きい | 最小権限のランタイムユーザー |
| `wait_timeout` を超えるプールリサイクル | プールされた接続の劣化 | タイムアウト以下でリサイクルし、pre-ping を有効化する |
| 書き込み後のレプリカ読み取り | ユーザーが見る状態の陳腐化 | 書き込み後の読み取りフローはプライマリに固定する |

## 出力の期待値

このスキルをレビューに使用する場合、以下を返します:

1. エンジン/バージョンの前提。
2. 正確性、ロック、セキュリティ、マイグレーションに関する最も高リスクな問題。
3. 安全なパスへの正確な SQL またはコードの変更。
4. 検証計画: `EXPLAIN`、マイグレーションのドライラン、ロック/デッドロックチェック、ロールバック基準。
5. 推奨事項に影響する MySQL/MariaDB の構文の違い。

## 関連

- スキル: `postgres-patterns` - PostgreSQL 固有のスキーマとクエリパターン
- スキル: `database-migrations` - マイグレーション計画とロールアウトの安全性
- スキル: `backend-patterns` - API とサービス層のパターン
- スキル: `security-review` - シークレット処理、認証、最小権限
- エージェント: `database-reviewer` - より広範なデータベースレビューワークフロー
