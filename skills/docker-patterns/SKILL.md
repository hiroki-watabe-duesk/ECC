---
name: docker-patterns
description: ローカル開発、コンテナセキュリティ、ネットワーキング、ボリューム戦略、マルチサービスオーケストレーションのための Docker および Docker Compose パターン。
origin: ECC
---

# Docker パターン

コンテナ化された開発のための Docker および Docker Compose ベストプラクティス。

## 有効にするタイミング

- ローカル開発用の Docker Compose をセットアップするとき
- マルチコンテナアーキテクチャを設計するとき
- コンテナのネットワークやボリュームの問題をトラブルシューティングするとき
- セキュリティとサイズの観点から Dockerfile をレビューするとき
- ローカル開発からコンテナ化ワークフローへ移行するとき

## ローカル開発用の Docker Compose

### 標準的な Web アプリスタック

```yaml
# docker-compose.yml
services:
  app:
    build:
      context: .
      target: dev                     # マルチステージ Dockerfile の dev ステージを使用
    ports:
      - "3000:3000"
    volumes:
      - .:/app                        # ホットリロード用バインドマウント
      - /app/node_modules             # 匿名ボリューム -- コンテナの依存関係を保持
    environment:
      - DATABASE_URL=postgres://postgres:postgres@db:5432/app_dev
      - REDIS_URL=redis://redis:6379/0
      - NODE_ENV=development
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    command: npm run dev

  db:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: app_dev
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redisdata:/data

  mailpit:                            # ローカルメールテスト
    image: axllent/mailpit
    ports:
      - "8025:8025"                   # Web UI
      - "1025:1025"                   # SMTP

volumes:
  pgdata:
  redisdata:
```

### 開発用と本番用の Dockerfile

```dockerfile
# ステージ: 依存関係
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

# ステージ: dev（ホットリロード、デバッグツール）
FROM node:22-alpine AS dev
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
EXPOSE 3000
CMD ["npm", "run", "dev"]

# ステージ: ビルド
FROM node:22-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build && npm prune --production

# ステージ: 本番（最小イメージ）
FROM node:22-alpine AS production
WORKDIR /app
RUN addgroup -g 1001 -S appgroup && adduser -S appuser -u 1001
USER appuser
COPY --from=build --chown=appuser:appgroup /app/dist ./dist
COPY --from=build --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=build --chown=appuser:appgroup /app/package.json ./
ENV NODE_ENV=production
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://localhost:3000/health || exit 1
CMD ["node", "dist/server.js"]
```

### オーバーライドファイル

```yaml
# docker-compose.override.yml（自動読み込み、開発専用設定）
services:
  app:
    environment:
      - DEBUG=app:*
      - LOG_LEVEL=debug
    ports:
      - "9229:9229"                   # Node.js デバッガ

# docker-compose.prod.yml（本番用に明示的に使用）
services:
  app:
    build:
      target: production
    restart: always
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
```

```bash
# 開発（オーバーライドを自動読み込み）
docker compose up

# 本番
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

## ネットワーキング

### サービスディスカバリー

同じ Compose ネットワーク内のサービスはサービス名で解決される:
```
# "app" コンテナからアクセス:
postgres://postgres:postgres@db:5432/app_dev    # "db" は db コンテナに解決される
redis://redis:6379/0                             # "redis" は redis コンテナに解決される
```

### カスタムネットワーク

```yaml
services:
  frontend:
    networks:
      - frontend-net

  api:
    networks:
      - frontend-net
      - backend-net

  db:
    networks:
      - backend-net              # api からのみ到達可能、frontend からは不可

networks:
  frontend-net:
  backend-net:
```

### 必要なものだけ公開する

```yaml
services:
  db:
    ports:
      - "127.0.0.1:5432:5432"   # ホストからのみアクセス可能、ネットワーク経由は不可
    # 本番ではポートを省略 -- Docker ネットワーク内のみからアクセス可能
```

## ボリューム戦略

```yaml
volumes:
  # 名前付きボリューム: コンテナ再起動後も永続化、Docker が管理
  pgdata:

  # バインドマウント: ホストディレクトリをコンテナにマップ（開発用）
  # - ./src:/app/src

  # 匿名ボリューム: バインドマウントオーバーライドからコンテナ生成コンテンツを保持
  # - /app/node_modules
```

### 一般的なパターン

```yaml
services:
  app:
    volumes:
      - .:/app                   # ソースコード（ホットリロード用バインドマウント）
      - /app/node_modules        # ホストからコンテナの node_modules を保護
      - /app/.next               # ビルドキャッシュを保護

  db:
    volumes:
      - pgdata:/var/lib/postgresql/data          # 永続データ
      - ./scripts/init.sql:/docker-entrypoint-initdb.d/init.sql  # 初期化スクリプト
```

## コンテナセキュリティ

### Dockerfile のハードニング

```dockerfile
# 1. 特定タグを使用（:latest は絶対に使わない）
FROM node:22.12-alpine3.20

# 2. 非ルートで実行
RUN addgroup -g 1001 -S app && adduser -S app -u 1001
USER app

# 3. ケーパビリティを削除（compose で）
# 4. 可能な場合は読み取り専用ルートファイルシステム
# 5. イメージレイヤーにシークレットを含めない
```

### Compose セキュリティ

```yaml
services:
  app:
    security_opt:
      - no-new-privileges:true
    read_only: true
    tmpfs:
      - /tmp
      - /app/.cache
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE          # 1024 未満のポートにバインドする場合のみ
```

### シークレット管理

```yaml
# 良い例: 環境変数を使用（実行時に注入）
services:
  app:
    env_file:
      - .env                     # .env を git にコミットしない
    environment:
      - API_KEY                  # ホスト環境から継承

# 良い例: Docker シークレット（Swarm モード）
secrets:
  db_password:
    file: ./secrets/db_password.txt

services:
  db:
    secrets:
      - db_password

# 悪い例: イメージにハードコード
# ENV API_KEY=sk-proj-xxxxx      # 絶対にやってはいけない
```

## .dockerignore

```
node_modules
.git
.env
.env.*
dist
coverage
*.log
.next
.cache
docker-compose*.yml
Dockerfile*
README.md
tests/
```

## デバッグ

### よく使うコマンド

```bash
# ログを見る
docker compose logs -f app           # app のログをフォロー
docker compose logs --tail=50 db     # db の最新 50 行

# 実行中のコンテナでコマンドを実行
docker compose exec app sh           # app のシェルに入る
docker compose exec db psql -U postgres  # postgres に接続

# 状態確認
docker compose ps                     # 実行中のサービス
docker compose top                    # 各コンテナのプロセス
docker stats                          # リソース使用状況

# 再ビルド
docker compose up --build             # イメージを再ビルド
docker compose build --no-cache app   # 強制的にフルビルド

# クリーンアップ
docker compose down                   # コンテナを停止・削除
docker compose down -v                # ボリュームも削除（破壊的操作）
docker system prune                   # 未使用イメージ/コンテナを削除
```

### ネットワーク問題のデバッグ

```bash
# コンテナ内で DNS 解決を確認
docker compose exec app nslookup db

# 接続性を確認
docker compose exec app wget -qO- http://api:3000/health

# ネットワークを調査
docker network ls
docker network inspect <project>_default
```

## アンチパターン

```
# 悪い例: オーケストレーションなしで本番に docker compose を使う
# 本番のマルチコンテナワークロードには Kubernetes、ECS、Docker Swarm を使う

# 悪い例: ボリュームなしでコンテナにデータを保存する
# コンテナはエフェメラル -- ボリュームなしでは再起動時にすべてのデータが失われる

# 悪い例: root として実行する
# 常に非 root ユーザーを作成して使用する

# 悪い例: :latest タグを使う
# 再現性のあるビルドのために特定バージョンにピン留めする

# 悪い例: すべてのサービスを 1 つの巨大なコンテナに入れる
# 関心事を分離する: コンテナごとに 1 プロセス

# 悪い例: docker-compose.yml にシークレットを書く
# .env ファイル（gitignore 済み）または Docker シークレットを使う
```
