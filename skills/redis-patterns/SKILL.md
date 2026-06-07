---
name: redis-patterns
description: Redisのデータ構造パターン、キャッシュ戦略、分散ロック、レート制限、Pub/Sub、および本番アプリケーション向けコネクション管理。
origin: ECC
---

# Redisパターン

一般的なバックエンドユースケースにおけるRedisベストプラクティスのクイックリファレンス。

## 仕組み

Redisはインメモリデータ構造ストアで、文字列・ハッシュ・リスト・セット・ソート済みセット・ストリームなどをサポートします。個々のRedisコマンドはシングルインスタンスでアトミックですが、複数ステップのワークフローをアトミックに保つにはLuaスクリプト・MULTI/EXECトランザクション・明示的な同期が必要です。データはRDBスナップショットまたはAOFログによってオプションで永続化されます。クライアントはRESPプロトコルを使用してTCP通信を行います。リクエストごとのハンドシェイクオーバーヘッドを避けるためにコネクションプールが必須です。

## 有効化タイミング

- アプリケーションへのキャッシュの追加
- レート制限またはスロットリングの実装
- 分散ロックまたはコーディネーションの構築
- セッションまたはトークンストレージのセットアップ
- メッセージングのためのPub/SubまたはRedisストリームの使用
- 本番環境でのRedisの設定（プーリング・エビクション・クラスタリング）

## データ構造チートシート

| ユースケース | 構造 | キー例 |
|----------|-----------|-------------|
| シンプルなキャッシュ | 文字列 | `product:123` |
| ユーザーセッション | ハッシュ | `session:abc` |
| リーダーボード | ソート済みセット | `scores:weekly` |
| ユニーク訪問者 | セット | `visitors:2024-01-01` |
| アクティビティフィード | リスト | `feed:user:456` |
| イベントストリーム | ストリーム | `events:orders` |
| カウンター / レート制限 | 文字列（INCR） | `ratelimit:user:123` |
| ブルームフィルター / HLL | HyperLogLog | `hll:pageviews` |

## コアパターン

### キャッシュアサイド（遅延ロード）

```python
import redis
import json

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def get_product(product_id: int):
    cache_key = f"product:{product_id}"
    cached = r.get(cache_key)

    if cached:
        return json.loads(cached)

    product = db.query("SELECT * FROM products WHERE id = %s", product_id)
    r.setex(cache_key, 3600, json.dumps(product))  # TTL: 1時間
    return product
```

### ライトスルーキャッシュ

```python
def update_product(product_id: int, data: dict):
    # まずDBに書き込む
    db.execute("UPDATE products SET ... WHERE id = %s", product_id)

    # 即座にキャッシュを更新する
    cache_key = f"product:{product_id}"
    r.setex(cache_key, 3600, json.dumps(data))
```

### キャッシュの無効化

```python
# タグベースの無効化 — 関連するキーをセットにまとめる
def cache_product(product_id: int, category_id: int, data: dict):
    key = f"product:{product_id}"
    tag = f"tag:category:{category_id}"
    pipe = r.pipeline(transaction=True)
    pipe.setex(key, 3600, json.dumps(data))
    pipe.sadd(tag, key)
    pipe.expire(tag, 3600)
    pipe.execute()

def invalidate_category(category_id: int):
    tag = f"tag:category:{category_id}"
    keys = r.smembers(tag)
    if keys:
        r.delete(*keys)
    r.delete(tag)
```

### セッションストレージ

```python
import time
import uuid

def create_session(user_id: int, ttl: int = 86400) -> str:
    session_id = str(uuid.uuid4())
    key = f"session:{session_id}"
    pipe = r.pipeline(transaction=True)
    pipe.hset(key, mapping={
        "user_id": user_id,
        "created_at": int(time.time()),
    })
    pipe.expire(key, ttl)
    pipe.execute()
    return session_id

def get_session(session_id: str) -> dict | None:
    data = r.hgetall(f"session:{session_id}")
    return data if data else None

def delete_session(session_id: str):
    r.delete(f"session:{session_id}")
```

## レート制限

### 固定ウィンドウ（シンプル）

```python
def is_rate_limited(user_id: int, limit: int = 100, window: int = 60) -> bool:
    key = f"ratelimit:{user_id}:{int(time.time()) // window}"
    pipe = r.pipeline(transaction=True)
    pipe.incr(key)
    pipe.expire(key, window)
    count, _ = pipe.execute()
    return count > limit
```

### スライディングウィンドウ（Lua — アトミック）

```lua
-- sliding_window.lua
local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])

redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
local count = redis.call('ZCARD', key)

if count < limit then
    -- 同一ミリ秒内の衝突を避けるためにユニークなメンバー（now + シーケンス）を使用する
    local seq_key = key .. ':seq'
    local seq = redis.call('INCR', seq_key)
    redis.call('EXPIRE', seq_key, math.ceil(window / 1000))
    redis.call('ZADD', key, now, now .. '-' .. seq)
    redis.call('EXPIRE', key, math.ceil(window / 1000))
    return 1
end
return 0
```

```python
sliding_window = r.register_script(open('sliding_window.lua').read())

def allow_request(user_id: int) -> bool:
    key = f"ratelimit:sliding:{user_id}"
    now = int(time.time() * 1000)
    return bool(sliding_window(keys=[key], args=[now, 60000, 100]))
```

## 分散ロック

### 分散ロック（シングルノード — SET NX PX）

```python
import uuid

def acquire_lock(resource: str, ttl_ms: int = 5000) -> str | None:
    lock_key = f"lock:{resource}"
    token = str(uuid.uuid4())
    acquired = r.set(lock_key, token, px=ttl_ms, nx=True)
    return token if acquired else None

def release_lock(resource: str, token: str) -> bool:
    release_script = """
    if redis.call('get', KEYS[1]) == ARGV[1] then
        return redis.call('del', KEYS[1])
    else
        return 0
    end
    """
    result = r.eval(release_script, 1, f"lock:{resource}", token)
    return bool(result)

# 使用例
token = acquire_lock("order:payment:123")
if token:
    try:
        process_payment()
    finally:
        release_lock("order:payment:123", token)
```

> マルチノード構成の場合は、完全なRedlockアルゴリズムを実装する `redlock-py` ライブラリを使用してください。

## Pub/SubとストリームPayloads

### Pub/Sub（ファイア・アンド・フォーゲット）

```python
# パブリッシャー
def publish_event(channel: str, payload: dict):
    r.publish(channel, json.dumps(payload))

# サブスクライバー（ブロッキング — 別スレッド/プロセスで実行）
def subscribe_events(channel: str):
    pubsub = r.pubsub()
    pubsub.subscribe(channel)
    for message in pubsub.listen():
        if message['type'] == 'message':
            handle(json.loads(message['data']))
```

### Redisストリーム（耐久性のあるキュー）

```python
# プロデューサー
def emit(stream: str, event: dict):
    r.xadd(stream, event, maxlen=10000)  # ストリーム長の上限設定

# コンシューマーグループ — 少なくとも1回の配信を保証
try:
    r.xgroup_create('events:orders', 'processor', id='0', mkstream=True)
except Exception:
    pass  # グループはすでに存在する

def consume(stream: str, group: str, consumer: str):
    while True:
        messages = r.xreadgroup(group, consumer, {stream: '>'}, count=10, block=2000)
        for _, entries in (messages or []):
            for msg_id, data in entries:
                process(data)
                r.xack(stream, group, msg_id)
```

> 配信保証・コンシューマーグループ・リプレイが必要な場合はPub/Subより**ストリーム**を優先してください。

## キー設計

### 命名規則

```
# パターン: resource:id:field
user:123:profile
order:456:status
cache:product:789

# パターン: namespace:resource:id
myapp:session:abc123
myapp:ratelimit:user:123

# パターン: resource:date（時間制限付きキー）
stats:pageviews:2024-01-01
```

### TTL戦略

| データ型 | 推奨TTL |
|-----------|--------------|
| ユーザーセッション | 24時間（`86400`） |
| APIレスポンスキャッシュ | 5〜15分 |
| レート制限ウィンドウ | ウィンドウサイズに合わせる |
| 短命なトークン | 5〜10分 |
| リーダーボード | 1時間〜24時間 |
| 静的/参照データ | 1時間〜1週間 |

必ずTTLを設定してください。TTLのないキーは際限なく蓄積され、メモリ負荷を引き起こします。

## コネクション管理

### コネクションプーリング

```python
from redis import ConnectionPool, Redis

pool = ConnectionPool(
    host='localhost',
    port=6379,
    db=0,
    max_connections=20,
    decode_responses=True,
    socket_connect_timeout=2,
    socket_timeout=2,
)

r = Redis(connection_pool=pool)
```

### クラスターモード

```python
from redis.cluster import RedisCluster

r = RedisCluster(
    startup_nodes=[{"host": "redis-1", "port": 6379}],
    decode_responses=True,
    skip_full_coverage_check=True,
)
```

### Sentinel（高可用性）

```python
from redis.sentinel import Sentinel

sentinel = Sentinel(
    [('sentinel-1', 26379), ('sentinel-2', 26379)],
    socket_timeout=0.5,
)
master = sentinel.master_for('mymaster', decode_responses=True)
replica = sentinel.slave_for('mymaster', decode_responses=True)
```

## エビクションポリシー

| ポリシー | 動作 | 最適用途 |
|--------|----------|----------|
| `noeviction` | 満杯時に書き込みエラー | キュー / 重要データ |
| `allkeys-lru` | 最近最も使われていないキーをエビクト | 汎用キャッシュ |
| `volatile-lru` | TTL付きキーのみLRU | 混在データストア |
| `allkeys-lfu` | 最も使用頻度が低いキーをエビクト | アクセス偏りパターン |
| `volatile-ttl` | 最も早く期限切れになるキーをエビクト | 長命なデータを優先 |

`redis.conf` で設定: `maxmemory-policy allkeys-lru`

## アンチパターン

| アンチパターン | 問題 | 対策 |
|---|---|---|
| TTLなしのキー | メモリが無制限に増大 | 常にTTLを設定する |
| 本番環境での `KEYS *` | サーバーをブロック（O(N)） | `SCAN` カーソルを使用する |
| 大きなブロブの保存（>100KB） | シリアライズが遅く、メモリ負荷大 | 参照を保存してオブジェクトストアから取得 |
| 全てに単一のRedisを使用 | キャッシュとキューの分離なし | 別々のDBまたはインスタンスを使用する |
| コネクションプール制限を無視 | 高負荷時のコネクション枯渇 | ワークロードに応じてプールをサイジングする |
| キャッシュミススタンピードの未対策 | コールドスタート時のサンダリングハード | ロックまたは確率的早期期限切れを使用する |
| 考慮なしの `FLUSHALL` | インスタンス全体を消去 | キーパターンでスコープを絞って削除する |

### キャッシュミススタンピード対策

```python
import threading

_locks: dict[str, threading.Lock] = {}
_locks_mutex = threading.Lock()

def get_with_lock(key: str, fetch_fn, ttl: int = 300):
    cached = r.get(key)
    if cached:
        return json.loads(cached)

    with _locks_mutex:
        if key not in _locks:
            _locks[key] = threading.Lock()
        lock = _locks[key]
    with lock:
        cached = r.get(key)  # ロック取得後に再チェック
        if cached:
            return json.loads(cached)
        value = fetch_fn()
        r.setex(key, ttl, json.dumps(value))
        return value
```

> 注意: マルチプロセスのデプロイメントでは、プロセス内ロックを上記の「分散ロック」セクションの `acquire_lock`/`release_lock` に置き換えてください。

## 使用例

**Django/Flask APIエンドポイントへのキャッシュ追加:**
`setex` を使ったキャッシュアサイドと、レスポンスに対する5分間のTTLを使用します。リクエストパラメーターをキーとします。

**ユーザーごとのAPIレート制限:**
低トラフィックエンドポイントには `pipeline(transaction=True)` を使った固定ウィンドウを使用し、正確なユーザーごとのスロットリングにはスライディングウィンドウLuaを使用します。

**ワーカー間でのバックグラウンドジョブの調整:**
予想されるジョブ時間を超えるTTLで `acquire_lock` を使用します。`finally` ブロックで必ず解放してください。

**複数のサブスクライバーへのファンアウト通知:**
ファイア・アンド・フォーゲットにはPub/Subを使用します。遅延コンシューマーへの配信保証やリプレイが必要な場合はストリームに切り替えてください。

## クイックリファレンス

| パターン | 使用タイミング |
|---------|-------------|
| キャッシュアサイド | 読み取り多め、軽微な鮮度低下を許容できる |
| ライトスルー | 強い一貫性が必要 |
| 分散ロック | リソースへの並行アクセスを防ぐ |
| スライディングウィンドウレート制限 | 正確なユーザーごとのスロットリング |
| Redisストリーム | コンシューマーグループ付きの耐久イベントキュー |
| Pub/Sub | 配信保証不要のブロードキャスト |
| ソート済みセットリーダーボード | ランク付きスコアリング、ページネーション |
| HyperLogLog | 低メモリでの近似ユニーク数カウント |

## 関連

- スキル: `postgres-patterns` — リレーショナルデータパターン
- スキル: `backend-patterns` — APIとサービスレイヤーパターン
- スキル: `database-migrations` — スキーマバージョン管理
- スキル: `django-patterns` — DjangoキャッシュフレームワークI連携
- エージェント: `database-reviewer` — データベースレビューワークフロー全体
