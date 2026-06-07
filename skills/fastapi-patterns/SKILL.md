---
name: fastapi-patterns
description: FastAPI の非同期 API、依存性注入、Pydantic リクエスト/レスポンスモデル、OpenAPI ドキュメント、テスト、セキュリティ、および本番環境対応のパターン。
origin: community
---

# FastAPI パターン

FastAPI サービスのための本番環境を意識したパターン集。

## 使用するタイミング

- FastAPI アプリの構築またはレビュー。
- ルーター、スキーマ、依存関係、データベースアクセスの分割。
- データベースや外部サービスを呼び出す非同期エンドポイントの作成。
- 認証、認可、OpenAPI ドキュメント、テスト、デプロイ設定の追加。
- FastAPI の PR をコピペできる例とプロダクションリスクの観点で確認するとき。

## 仕組み

FastAPI アプリを明示的な依存関係とサービスコードの上に薄い HTTP レイヤーとして扱う:

- `main.py` がアプリ構築、ミドルウェア、例外ハンドラ、ルーター登録を担当する。
- `schemas/` が Pydantic のリクエストおよびレスポンスモデルを担当する。
- `dependencies.py` がデータベース、認証、ページネーション、リクエストスコープの依存関係を担当する。
- `services/` または `crud/` がビジネスおよび永続化処理を担当する。
- `tests/` が本番リソースを開かずに依存関係をオーバーライドする。

小さなルーターと明示的な `response_model` 宣言を優先する。生の ORM オブジェクト、シークレット、フレームワークのグローバルをレスポンススキーマに含めない。

## プロジェクト構成

```text
app/
|-- main.py
|-- config.py
|-- dependencies.py
|-- exceptions.py
|-- api/
|   `-- routes/
|       |-- users.py
|       `-- health.py
|-- core/
|   |-- security.py
|   `-- middleware.py
|-- db/
|   |-- session.py
|   `-- crud.py
|-- models/
|-- schemas/
`-- tests/
```

## アプリケーションファクトリ

テストやワーカーが制御された設定でアプリを構築できるように、ファクトリを使用する。

```python
from contextlib import asynccontextmanager

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.api.routes import health, users
from app.config import settings
from app.db.session import close_db, init_db
from app.exceptions import register_exception_handlers


@asynccontextmanager
async def lifespan(app: FastAPI):
    await init_db()
    yield
    await close_db()


def create_app() -> FastAPI:
    app = FastAPI(
        title=settings.api_title,
        version=settings.api_version,
        lifespan=lifespan,
    )

    app.add_middleware(
        CORSMiddleware,
        allow_origins=settings.cors_origins,
        allow_credentials=bool(settings.cors_origins),
        allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE"],
        allow_headers=["Authorization", "Content-Type"],
    )

    register_exception_handlers(app)
    app.include_router(health.router, prefix="/health", tags=["health"])
    app.include_router(users.router, prefix="/api/v1/users", tags=["users"])
    return app


app = create_app()
```

`allow_origins=["*"]` と `allow_credentials=True` を組み合わせてはならない。ブラウザはその組み合わせを拒否し、Starlette も認証情報付きリクエストに対してそれを禁止している。

## Pydantic スキーマ

リクエスト、更新、レスポンスのモデルを分離して管理する。

```python
from datetime import datetime
from typing import Annotated
from uuid import UUID

from pydantic import BaseModel, ConfigDict, EmailStr, Field


class UserBase(BaseModel):
    email: EmailStr
    full_name: Annotated[str, Field(min_length=1, max_length=100)]


class UserCreate(UserBase):
    password: Annotated[str, Field(min_length=12, max_length=128)]


class UserUpdate(BaseModel):
    email: EmailStr | None = None
    full_name: Annotated[str | None, Field(min_length=1, max_length=100)] = None


class UserResponse(UserBase):
    model_config = ConfigDict(from_attributes=True)

    id: UUID
    created_at: datetime
    updated_at: datetime
```

レスポンスモデルにはパスワードハッシュ、アクセストークン、リフレッシュトークン、内部認可状態を絶対に含めてはならない。

## 依存関係

リクエストスコープのリソースには依存性注入を使用する。

```python
from collections.abc import AsyncIterator
from uuid import UUID

from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.security import decode_token
from app.db.session import session_factory
from app.models.user import User


oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/login")


async def get_db() -> AsyncIterator[AsyncSession]:
    async with session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise


async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> User:
    payload = decode_token(token)
    user_id = UUID(payload["sub"])
    user = await db.get(User, user_id)
    if user is None:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid token")
    return user
```

ルートハンドラ内でセッション、クライアント、認証情報をインラインで生成してはならない。

## 非同期エンドポイント

I/O を行う場合はルートハンドラを非同期にし、内部でも非同期ライブラリを使用する。

```python
from fastapi import APIRouter, Depends, Query
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.dependencies import get_current_user, get_db
from app.models.user import User
from app.schemas.user import UserResponse


router = APIRouter()


@router.get("/", response_model=list[UserResponse])
async def list_users(
    limit: int = Query(default=50, ge=1, le=100),
    offset: int = Query(default=0, ge=0),
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    result = await db.execute(
        select(User).order_by(User.created_at.desc()).limit(limit).offset(offset)
    )
    return result.scalars().all()
```

非同期ハンドラからの外部 HTTP 呼び出しには `httpx.AsyncClient` を使用する。非同期ルート内で `requests` を呼び出してはならない。

## エラーハンドリング

ドメイン例外を一元管理し、レスポンスの形式を安定させる。

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse


class ApiError(Exception):
    def __init__(self, status_code: int, code: str, message: str):
        self.status_code = status_code
        self.code = code
        self.message = message


def register_exception_handlers(app: FastAPI) -> None:
    @app.exception_handler(ApiError)
    async def api_error_handler(request: Request, exc: ApiError):
        return JSONResponse(
            status_code=exc.status_code,
            content={"error": {"code": exc.code, "message": exc.message}},
        )
```

## OpenAPI カスタマイズ

カスタム OpenAPI の呼び出し可能オブジェクトを `app.openapi` に代入する。関数を一度だけ呼び出すのではない。

```python
from fastapi import FastAPI
from fastapi.openapi.utils import get_openapi


def install_openapi(app: FastAPI) -> None:
    def custom_openapi():
        if app.openapi_schema:
            return app.openapi_schema
        app.openapi_schema = get_openapi(
            title="Service API",
            version="1.0.0",
            routes=app.routes,
        )
        return app.openapi_schema

    app.openapi = custom_openapi
```

## テスト

ルートハンドラが参照しない内部ヘルパーではなく、`Depends` が使用する依存関係をオーバーライドする。

```python
import pytest
from httpx import ASGITransport, AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession

from app.dependencies import get_db
from app.main import create_app


@pytest.fixture
async def client(test_session: AsyncSession):
    app = create_app()

    async def override_get_db():
        yield test_session

    app.dependency_overrides[get_db] = override_get_db
    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test",
    ) as test_client:
        yield test_client
    app.dependency_overrides.clear()
```

## セキュリティチェックリスト

- パスワードは `argon2-cffi`、`bcrypt`、または現行の passlib 互換ハッシャーでハッシュ化する。
- JWT の発行者、対象者、有効期限、署名アルゴリズムを検証する。
- CORS オリジンは環境ごとに設定する。
- 認証エンドポイントと書き込み負荷の高いエンドポイントにレート制限を設ける。
- すべてのリクエストボディに Pydantic モデルを使用する。
- ORM パラメータバインディングまたは SQLAlchemy Core 式を使用し、f 文字列で SQL を組み立てない。
- ログからトークン、認可ヘッダー、Cookie、パスワードをマスクする。
- CI で依存関係の監査ツールを実行する。

## パフォーマンスチェックリスト

- データベース接続プールを明示的に設定する。
- リストエンドポイントにページネーションを追加する。
- N+1 クエリに注意し、意図的に Eager Loading を使用する。
- 非同期パスでは非同期 HTTP/データベースクライアントを使用する。
- ペイロードサイズと CPU のトレードオフを確認してから圧縮を追加する。
- 安定した高コストな読み取りには明示的な無効化を伴うキャッシュを活用する。

## 使用例

これらの例はパターンとして参考にするものであり、プロジェクト全体のテンプレートではない:

- アプリケーションファクトリ: `create_app` でミドルウェアとルーターを一度設定する。
- スキーマ分割: `UserCreate`、`UserUpdate`、`UserResponse` はそれぞれ異なる責務を持つ。
- 依存関係オーバーライド: テストでは `get_db` を直接オーバーライドする。
- OpenAPI カスタマイズ: `app.openapi = custom_openapi` と代入する。

## 関連情報

- Agent: `fastapi-reviewer`
- Command: `/fastapi-review`
- Skill: `python-patterns`
- Skill: `python-testing`
- Skill: `api-design`
