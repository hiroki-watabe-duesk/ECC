---
name: python-testing
description: pytestを使ったPythonテスト戦略。TDD手法、フィクスチャ、モック、パラメータ化、カバレッジ要件を含む。
origin: ECC
---

# Pythonテストパターン

pytestとTDD手法を使ったPythonアプリケーションの包括的なテスト戦略とベストプラクティス。

## アクティブにするタイミング

- 新しいPythonコードを書くとき（TDDに従う: レッド、グリーン、リファクタリング）
- Pythonプロジェクトのテストスイートを設計するとき
- Pythonのテストカバレッジをレビューするとき
- テストインフラをセットアップするとき

## テストの基本哲学

### テスト駆動開発（TDD）

常にTDDサイクルに従う:

1. **レッド**: 目的の動作に対する失敗するテストを書く
2. **グリーン**: テストをパスさせる最小限のコードを書く
3. **リファクタリング**: テストをグリーンに保ちながらコードを改善する

```python
# ステップ1: 失敗するテストを書く（レッド）
def test_add_numbers():
    result = add(2, 3)
    assert result == 5

# ステップ2: 最小限の実装を書く（グリーン）
def add(a, b):
    return a + b

# ステップ3: 必要に応じてリファクタリング（リファクタリング）
```

### カバレッジ要件

- **目標**: コードカバレッジ80%以上
- **重要なパス**: 100%のカバレッジが必要
- `pytest --cov`を使ってカバレッジを計測する

```bash
pytest --cov=mypackage --cov-report=term-missing --cov-report=html
```

## pytestの基礎

### 基本的なテスト構造

```python
import pytest

def test_addition():
    """基本的な加算のテスト。"""
    assert 2 + 2 == 4

def test_string_uppercase():
    """文字列の大文字変換のテスト。"""
    text = "hello"
    assert text.upper() == "HELLO"

def test_list_append():
    """リストへの追加のテスト。"""
    items = [1, 2, 3]
    items.append(4)
    assert 4 in items
    assert len(items) == 4
```

### アサーション

```python
# 等価
assert result == expected

# 不等価
assert result != unexpected

# 真偽
assert result  # 真
assert not result  # 偽
assert result is True  # 正確にTrue
assert result is False  # 正確にFalse
assert result is None  # 正確にNone

# 包含
assert item in collection
assert item not in collection

# 比較
assert result > 0
assert 0 <= result <= 100

# 型チェック
assert isinstance(result, str)

# 例外テスト（推奨アプローチ）
with pytest.raises(ValueError):
    raise ValueError("error message")

# 例外メッセージの確認
with pytest.raises(ValueError, match="invalid input"):
    raise ValueError("invalid input provided")

# 例外属性の確認
with pytest.raises(ValueError) as exc_info:
    raise ValueError("error message")
assert str(exc_info.value) == "error message"
```

## フィクスチャ

### 基本的なフィクスチャの使用

```python
import pytest

@pytest.fixture
def sample_data():
    """サンプルデータを提供するフィクスチャ。"""
    return {"name": "Alice", "age": 30}

def test_sample_data(sample_data):
    """フィクスチャを使ったテスト。"""
    assert sample_data["name"] == "Alice"
    assert sample_data["age"] == 30
```

### セットアップ/ティアダウン付きフィクスチャ

```python
@pytest.fixture
def database():
    """セットアップとティアダウン付きのフィクスチャ。"""
    # セットアップ
    db = Database(":memory:")
    db.create_tables()
    db.insert_test_data()

    yield db  # テストに提供

    # ティアダウン
    db.close()

def test_database_query(database):
    """データベース操作のテスト。"""
    result = database.query("SELECT * FROM users")
    assert len(result) > 0
```

### フィクスチャのスコープ

```python
# 関数スコープ（デフォルト）- 各テストごとに実行
@pytest.fixture
def temp_file():
    with open("temp.txt", "w") as f:
        yield f
    os.remove("temp.txt")

# モジュールスコープ - モジュールごとに1回実行
@pytest.fixture(scope="module")
def module_db():
    db = Database(":memory:")
    db.create_tables()
    yield db
    db.close()

# セッションスコープ - テストセッションごとに1回実行
@pytest.fixture(scope="session")
def shared_resource():
    resource = ExpensiveResource()
    yield resource
    resource.cleanup()
```

### パラメーター付きフィクスチャ

```python
@pytest.fixture(params=[1, 2, 3])
def number(request):
    """パラメータ化されたフィクスチャ。"""
    return request.param

def test_numbers(number):
    """各パラメーターで3回テストが実行される。"""
    assert number > 0
```

### 複数フィクスチャの使用

```python
@pytest.fixture
def user():
    return User(id=1, name="Alice")

@pytest.fixture
def admin():
    return User(id=2, name="Admin", role="admin")

def test_user_admin_interaction(user, admin):
    """複数のフィクスチャを使ったテスト。"""
    assert admin.can_manage(user)
```

### 自動使用フィクスチャ

```python
@pytest.fixture(autouse=True)
def reset_config():
    """全てのテストの前に自動的に実行される。"""
    Config.reset()
    yield
    Config.cleanup()

def test_without_fixture_call():
    # reset_configが自動的に実行される
    assert Config.get_setting("debug") is False
```

### 共有フィクスチャのためのConftest.py

```python
# tests/conftest.py
import pytest

@pytest.fixture
def client():
    """全テスト共有のフィクスチャ。"""
    app = create_app(testing=True)
    with app.test_client() as client:
        yield client

@pytest.fixture
def auth_headers(client):
    """APIテスト用の認証ヘッダーを生成する。"""
    response = client.post("/api/login", json={
        "username": "test",
        "password": "test"
    })
    token = response.json["token"]
    return {"Authorization": f"Bearer {token}"}
```

## パラメータ化

### 基本的なパラメータ化

```python
@pytest.mark.parametrize("input,expected", [
    ("hello", "HELLO"),
    ("world", "WORLD"),
    ("PyThOn", "PYTHON"),
])
def test_uppercase(input, expected):
    """異なる入力で3回テストが実行される。"""
    assert input.upper() == expected
```

### 複数パラメーター

```python
@pytest.mark.parametrize("a,b,expected", [
    (2, 3, 5),
    (0, 0, 0),
    (-1, 1, 0),
    (100, 200, 300),
])
def test_add(a, b, expected):
    """複数の入力で加算をテストする。"""
    assert add(a, b) == expected
```

### IDを使ったパラメータ化

```python
@pytest.mark.parametrize("input,expected", [
    ("valid@email.com", True),
    ("invalid", False),
    ("@no-domain.com", False),
], ids=["valid-email", "missing-at", "missing-domain"])
def test_email_validation(input, expected):
    """読みやすいテストIDでメールバリデーションをテストする。"""
    assert is_valid_email(input) is expected
```

### パラメータ化されたフィクスチャ

```python
@pytest.fixture(params=["sqlite", "postgresql", "mysql"])
def db(request):
    """複数のデータベースバックエンドに対してテストする。"""
    if request.param == "sqlite":
        return Database(":memory:")
    elif request.param == "postgresql":
        return Database("postgresql://localhost/test")
    elif request.param == "mysql":
        return Database("mysql://localhost/test")

def test_database_operations(db):
    """各データベースで3回テストが実行される。"""
    result = db.query("SELECT 1")
    assert result is not None
```

## マーカーとテスト選択

### カスタムマーカー

```python
# 遅いテストをマーク
@pytest.mark.slow
def test_slow_operation():
    time.sleep(5)

# 統合テストをマーク
@pytest.mark.integration
def test_api_integration():
    response = requests.get("https://api.example.com")
    assert response.status_code == 200

# ユニットテストをマーク
@pytest.mark.unit
def test_unit_logic():
    assert calculate(2, 3) == 5
```

### 特定のテストを実行する

```bash
# 速いテストのみ実行
pytest -m "not slow"

# 統合テストのみ実行
pytest -m integration

# 統合または遅いテストを実行
pytest -m "integration or slow"

# ユニットだが遅くないテストを実行
pytest -m "unit and not slow"
```

### pytest.iniでマーカーを設定する

```ini
[pytest]
markers =
    slow: marks tests as slow
    integration: marks tests as integration tests
    unit: marks tests as unit tests
    django: marks tests as requiring Django
```

## モックとパッチ

### 関数のモック

```python
from unittest.mock import patch, Mock

@patch("mypackage.external_api_call")
def test_with_mock(api_call_mock):
    """外部APIをモックしたテスト。"""
    api_call_mock.return_value = {"status": "success"}

    result = my_function()

    api_call_mock.assert_called_once()
    assert result["status"] == "success"
```

### 戻り値のモック

```python
@patch("mypackage.Database.connect")
def test_database_connection(connect_mock):
    """モックデータベース接続のテスト。"""
    connect_mock.return_value = MockConnection()

    db = Database()
    db.connect()

    connect_mock.assert_called_once_with("localhost")
```

### 例外のモック

```python
@patch("mypackage.api_call")
def test_api_error_handling(api_call_mock):
    """モック例外を使ったエラーハンドリングのテスト。"""
    api_call_mock.side_effect = ConnectionError("Network error")

    with pytest.raises(ConnectionError):
        api_call()

    api_call_mock.assert_called_once()
```

### コンテキストマネージャーのモック

```python
@patch("builtins.open", new_callable=mock_open)
def test_file_reading(mock_file):
    """モックopenを使ったファイル読み込みのテスト。"""
    mock_file.return_value.read.return_value = "file content"

    result = read_file("test.txt")

    mock_file.assert_called_once_with("test.txt", "r")
    assert result == "file content"
```

### Autospecの使用

```python
@patch("mypackage.DBConnection", autospec=True)
def test_autospec(db_mock):
    """API誤用を検出するためのautospecを使ったテスト。"""
    db = db_mock.return_value
    db.query("SELECT * FROM users")

    # DBConnectionにqueryメソッドがない場合はここで失敗する
    db_mock.assert_called_once()
```

### モッククラスインスタンス

```python
class TestUserService:
    @patch("mypackage.UserRepository")
    def test_create_user(self, repo_mock):
        """モックリポジトリを使ったユーザー作成のテスト。"""
        repo_mock.return_value.save.return_value = User(id=1, name="Alice")

        service = UserService(repo_mock.return_value)
        user = service.create_user(name="Alice")

        assert user.name == "Alice"
        repo_mock.return_value.save.assert_called_once()
```

### モックプロパティ

```python
@pytest.fixture
def mock_config():
    """プロパティを持つモックを作成する。"""
    config = Mock()
    type(config).debug = PropertyMock(return_value=True)
    type(config).api_key = PropertyMock(return_value="test-key")
    return config

def test_with_mock_config(mock_config):
    """モック設定プロパティを使ったテスト。"""
    assert mock_config.debug is True
    assert mock_config.api_key == "test-key"
```

## 非同期コードのテスト

### pytest-asyncioによる非同期テスト

```python
import pytest

@pytest.mark.asyncio
async def test_async_function():
    """非同期関数のテスト。"""
    result = await async_add(2, 3)
    assert result == 5

@pytest.mark.asyncio
async def test_async_with_fixture(async_client):
    """非同期フィクスチャを使った非同期テスト。"""
    response = await async_client.get("/api/users")
    assert response.status_code == 200
```

### 非同期フィクスチャ

```python
@pytest.fixture
async def async_client():
    """非同期テストクライアントを提供する非同期フィクスチャ。"""
    app = create_app()
    async with app.test_client() as client:
        yield client

@pytest.mark.asyncio
async def test_api_endpoint(async_client):
    """非同期フィクスチャを使ったテスト。"""
    response = await async_client.get("/api/data")
    assert response.status_code == 200
```

### 非同期関数のモック

```python
@pytest.mark.asyncio
@patch("mypackage.async_api_call")
async def test_async_mock(api_call_mock):
    """モックを使った非同期関数のテスト。"""
    api_call_mock.return_value = {"status": "ok"}

    result = await my_async_function()

    api_call_mock.assert_awaited_once()
    assert result["status"] == "ok"
```

## 例外のテスト

### 期待される例外のテスト

```python
def test_divide_by_zero():
    """ゼロ除算でZeroDivisionErrorが発生することをテストする。"""
    with pytest.raises(ZeroDivisionError):
        divide(10, 0)

def test_custom_exception():
    """カスタム例外とメッセージのテスト。"""
    with pytest.raises(ValueError, match="invalid input"):
        validate_input("invalid")
```

### 例外属性のテスト

```python
def test_exception_with_details():
    """カスタム属性を持つ例外のテスト。"""
    with pytest.raises(CustomError) as exc_info:
        raise CustomError("error", code=400)

    assert exc_info.value.code == 400
    assert "error" in str(exc_info.value)
```

## 副作用のテスト

### ファイル操作のテスト

```python
import tempfile
import os

def test_file_processing():
    """一時ファイルを使ったファイル処理のテスト。"""
    with tempfile.NamedTemporaryFile(mode='w', delete=False, suffix='.txt') as f:
        f.write("test content")
        temp_path = f.name

    try:
        result = process_file(temp_path)
        assert result == "processed: test content"
    finally:
        os.unlink(temp_path)
```

### pytestのtmp_pathフィクスチャを使ったテスト

```python
def test_with_tmp_path(tmp_path):
    """pytestの組み込み一時パスフィクスチャを使ったテスト。"""
    test_file = tmp_path / "test.txt"
    test_file.write_text("hello world")

    result = process_file(str(test_file))
    assert result == "hello world"
    # tmp_pathは自動的にクリーンアップされる
```

### tmpdirフィクスチャを使ったテスト

```python
def test_with_tmpdir(tmpdir):
    """pytestのtmpdirフィクスチャを使ったテスト。"""
    test_file = tmpdir.join("test.txt")
    test_file.write("data")

    result = process_file(str(test_file))
    assert result == "data"
```

## テストの構成

### ディレクトリ構造

```
tests/
├── conftest.py                 # 共有フィクスチャ
├── __init__.py
├── unit/                       # ユニットテスト
│   ├── __init__.py
│   ├── test_models.py
│   ├── test_utils.py
│   └── test_services.py
├── integration/                # 統合テスト
│   ├── __init__.py
│   ├── test_api.py
│   └── test_database.py
└── e2e/                        # エンドツーエンドテスト
    ├── __init__.py
    └── test_user_flow.py
```

### テストクラス

```python
class TestUserService:
    """関連するテストをクラスにまとめる。"""

    @pytest.fixture(autouse=True)
    def setup(self):
        """このクラスの各テストの前に実行されるセットアップ。"""
        self.service = UserService()

    def test_create_user(self):
        """ユーザー作成のテスト。"""
        user = self.service.create_user("Alice")
        assert user.name == "Alice"

    def test_delete_user(self):
        """ユーザー削除のテスト。"""
        user = User(id=1, name="Bob")
        self.service.delete_user(user)
        assert not self.service.user_exists(1)
```

## ベストプラクティス

### すること

- **TDDに従う**: コードの前にテストを書く（レッド・グリーン・リファクタリング）
- **1つのことをテストする**: 各テストは1つの動作を検証する
- **説明的な名前を使う**: `test_user_login_with_invalid_credentials_fails`
- **フィクスチャを使う**: フィクスチャで重複を排除する
- **外部依存関係をモックする**: 外部サービスに依存しない
- **エッジケースをテストする**: 空の入力、None値、境界条件
- **カバレッジ80%以上を目指す**: 重要なパスに集中する
- **テストを速く保つ**: マークを使って遅いテストを分ける

### しないこと

- **実装をテストしない**: 内部ではなく動作をテストする
- **テスト内で複雑な条件分岐を使わない**: テストはシンプルに保つ
- **テストの失敗を無視しない**: 全てのテストはパスする必要がある
- **サードパーティコードをテストしない**: ライブラリが動作することを信頼する
- **テスト間で状態を共有しない**: テストは独立している必要がある
- **テスト内で例外をキャッチしない**: `pytest.raises`を使う
- **print文を使わない**: アサーションとpytestの出力を使う
- **脆すぎるテストを書かない**: 過度に具体的なモックを避ける

## 一般的なパターン

### APIエンドポイントのテスト（FastAPI/Flask）

```python
@pytest.fixture
def client():
    app = create_app(testing=True)
    return app.test_client()

def test_get_user(client):
    response = client.get("/api/users/1")
    assert response.status_code == 200
    assert response.json["id"] == 1

def test_create_user(client):
    response = client.post("/api/users", json={
        "name": "Alice",
        "email": "alice@example.com"
    })
    assert response.status_code == 201
    assert response.json["name"] == "Alice"
```

### データベース操作のテスト

```python
@pytest.fixture
def db_session():
    """テスト用データベースセッションを作成する。"""
    session = Session(bind=engine)
    session.begin_nested()
    yield session
    session.rollback()
    session.close()

def test_create_user(db_session):
    user = User(name="Alice", email="alice@example.com")
    db_session.add(user)
    db_session.commit()

    retrieved = db_session.query(User).filter_by(name="Alice").first()
    assert retrieved.email == "alice@example.com"
```

### クラスメソッドのテスト

```python
class TestCalculator:
    @pytest.fixture
    def calculator(self):
        return Calculator()

    def test_add(self, calculator):
        assert calculator.add(2, 3) == 5

    def test_divide_by_zero(self, calculator):
        with pytest.raises(ZeroDivisionError):
            calculator.divide(10, 0)
```

## pytest設定

### pytest.ini

```ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts =
    --strict-markers
    --disable-warnings
    --cov=mypackage
    --cov-report=term-missing
    --cov-report=html
markers =
    slow: marks tests as slow
    integration: marks tests as integration tests
    unit: marks tests as unit tests
```

### pyproject.toml

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
addopts = [
    "--strict-markers",
    "--cov=mypackage",
    "--cov-report=term-missing",
    "--cov-report=html",
]
markers = [
    "slow: marks tests as slow",
    "integration: marks tests as integration tests",
    "unit: marks tests as unit tests",
]
```

## テストの実行

```bash
# 全テストを実行
pytest

# 特定のファイルを実行
pytest tests/test_utils.py

# 特定のテストを実行
pytest tests/test_utils.py::test_function

# 詳細出力で実行
pytest -v

# カバレッジ付きで実行
pytest --cov=mypackage --cov-report=html

# 速いテストのみ実行
pytest -m "not slow"

# 最初の失敗で停止
pytest -x

# N回失敗で停止
pytest --maxfail=3

# 最後に失敗したテストを実行
pytest --lf

# パターンでテストを実行
pytest -k "test_user"

# 失敗時にデバッガーを起動
pytest --pdb
```

## クイックリファレンス

| パターン | 使用方法 |
|---------|---------|
| `pytest.raises()` | 期待される例外のテスト |
| `@pytest.fixture()` | 再利用可能なテストフィクスチャの作成 |
| `@pytest.mark.parametrize()` | 複数の入力でテストを実行 |
| `@pytest.mark.slow` | 遅いテストをマーク |
| `pytest -m "not slow"` | 遅いテストをスキップ |
| `@patch()` | 関数とクラスのモック |
| `tmp_path` fixture | 自動一時ディレクトリ |
| `pytest --cov` | カバレッジレポートの生成 |
| `assert` | シンプルで読みやすいアサーション |

**覚えておきたいこと**: テストもコードです。クリーンで読みやすく、メンテナンスしやすく保ちましょう。良いテストはバグを検出し、優れたテストはバグを防ぎます。
