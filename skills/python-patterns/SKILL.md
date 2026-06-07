---
name: python-patterns
description: Pythonらしいイディオム、PEP 8標準、型ヒント、堅牢・効率的・保守性の高いPythonアプリケーション構築のためのベストプラクティス。
origin: ECC
---

# Python 開発パターン

堅牢・効率的・保守性の高いアプリケーションを構築するための、慣用的なPythonパターンとベストプラクティス。

## 有効化するタイミング

- 新しいPythonコードを書くとき
- Pythonコードをレビューするとき
- 既存のPythonコードをリファクタリングするとき
- Pythonパッケージ・モジュールを設計するとき

## 核となる原則

### 1. 可読性が重要

Pythonは可読性を優先します。コードは明瞭で理解しやすいものであるべきです。

```python
# 良い例: 明確で読みやすい
def get_active_users(users: list[User]) -> list[User]:
    """Return only active users from the provided list."""
    return [user for user in users if user.is_active]


# 悪い例: 巧みだが分かりにくい
def get_active_users(u):
    return [x for x in u if x.a]
```

### 2. 暗黙よりも明示

魔法的な処理を避け、コードが何をするかを明確にしましょう。

```python
# 良い例: 明示的な設定
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)

# 悪い例: 隠れた副作用
import some_module
some_module.setup()  # What does this do?
```

### 3. EAFP - 許可を求めるより許しを求める方が簡単

Pythonは条件チェックより例外処理を好みます。

```python
# 良い例: EAFPスタイル
def get_value(dictionary: dict, key: str) -> Any:
    try:
        return dictionary[key]
    except KeyError:
        return default_value

# 悪い例: LBYL（Look Before You Leap）スタイル
def get_value(dictionary: dict, key: str) -> Any:
    if key in dictionary:
        return dictionary[key]
    else:
        return default_value
```

## 型ヒント

### 基本的な型アノテーション

```python
from typing import Optional, List, Dict, Any

def process_user(
    user_id: str,
    data: Dict[str, Any],
    active: bool = True
) -> Optional[User]:
    """Process a user and return the updated User or None."""
    if not active:
        return None
    return User(user_id, data)
```

### モダンな型ヒント（Python 3.9以降）

```python
# Python 3.9以降 - 組み込み型を使う
def process_items(items: list[str]) -> dict[str, int]:
    return {item: len(item) for item in items}

# Python 3.8以前 - typingモジュールを使う
from typing import List, Dict

def process_items(items: List[str]) -> Dict[str, int]:
    return {item: len(item) for item in items}
```

### 型エイリアスとTypeVar

```python
from typing import TypeVar, Union

# 複雑な型の型エイリアス
JSON = Union[dict[str, Any], list[Any], str, int, float, bool, None]

def parse_json(data: str) -> JSON:
    return json.loads(data)

# ジェネリック型
T = TypeVar('T')

def first(items: list[T]) -> T | None:
    """Return the first item or None if list is empty."""
    return items[0] if items else None
```

### プロトコルベースのダックタイピング

```python
from typing import Protocol

class Renderable(Protocol):
    def render(self) -> str:
        """Render the object to a string."""

def render_all(items: list[Renderable]) -> str:
    """Render all items that implement the Renderable protocol."""
    return "\n".join(item.render() for item in items)
```

## エラーハンドリングパターン

### 特定の例外をキャッチする

```python
# 良い例: 特定の例外をキャッチする
def load_config(path: str) -> Config:
    try:
        with open(path) as f:
            return Config.from_json(f.read())
    except FileNotFoundError as e:
        raise ConfigError(f"Config file not found: {path}") from e
    except json.JSONDecodeError as e:
        raise ConfigError(f"Invalid JSON in config: {path}") from e

# 悪い例: 裸のexcept
def load_config(path: str) -> Config:
    try:
        with open(path) as f:
            return Config.from_json(f.read())
    except:
        return None  # Silent failure!
```

### 例外チェーニング

```python
def process_data(data: str) -> Result:
    try:
        parsed = json.loads(data)
    except json.JSONDecodeError as e:
        # トレースバックを保持するために例外をチェーンする
        raise ValueError(f"Failed to parse data: {data}") from e
```

### カスタム例外階層

```python
class AppError(Exception):
    """Base exception for all application errors."""
    pass

class ValidationError(AppError):
    """Raised when input validation fails."""
    pass

class NotFoundError(AppError):
    """Raised when a requested resource is not found."""
    pass

# 使用例
def get_user(user_id: str) -> User:
    user = db.find_user(user_id)
    if not user:
        raise NotFoundError(f"User not found: {user_id}")
    return user
```

## コンテキストマネージャー

### リソース管理

```python
# 良い例: コンテキストマネージャーを使う
def process_file(path: str) -> str:
    with open(path, 'r') as f:
        return f.read()

# 悪い例: 手動でリソースを管理する
def process_file(path: str) -> str:
    f = open(path, 'r')
    try:
        return f.read()
    finally:
        f.close()
```

### カスタムコンテキストマネージャー

```python
from contextlib import contextmanager

@contextmanager
def timer(name: str):
    """Context manager to time a block of code."""
    start = time.perf_counter()
    yield
    elapsed = time.perf_counter() - start
    print(f"{name} took {elapsed:.4f} seconds")

# 使用例
with timer("data processing"):
    process_large_dataset()
```

### コンテキストマネージャークラス

```python
class DatabaseTransaction:
    def __init__(self, connection):
        self.connection = connection

    def __enter__(self):
        self.connection.begin_transaction()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is None:
            self.connection.commit()
        else:
            self.connection.rollback()
        return False  # Don't suppress exceptions

# 使用例
with DatabaseTransaction(conn):
    user = conn.create_user(user_data)
    conn.create_profile(user.id, profile_data)
```

## 内包表記とジェネレーター

### リスト内包表記

```python
# 良い例: 単純な変換にはリスト内包表記
names = [user.name for user in users if user.is_active]

# 悪い例: 手動のループ
names = []
for user in users:
    if user.is_active:
        names.append(user.name)

# 複雑な内包表記は展開すべき
# 悪い例: 複雑すぎる
result = [x * 2 for x in items if x > 0 if x % 2 == 0]

# 良い例: ジェネレーター関数を使う
def filter_and_transform(items: Iterable[int]) -> list[int]:
    result = []
    for x in items:
        if x > 0 and x % 2 == 0:
            result.append(x * 2)
    return result
```

### ジェネレーター式

```python
# 良い例: 遅延評価のためのジェネレーター
total = sum(x * x for x in range(1_000_000))

# 悪い例: 大きな中間リストを作成する
total = sum([x * x for x in range(1_000_000)])
```

### ジェネレーター関数

```python
def read_large_file(path: str) -> Iterator[str]:
    """Read a large file line by line."""
    with open(path) as f:
        for line in f:
            yield line.strip()

# 使用例
for line in read_large_file("huge.txt"):
    process(line)
```

## データクラスと名前付きタプル

### データクラス

```python
from dataclasses import dataclass, field
from datetime import datetime

@dataclass
class User:
    """User entity with automatic __init__, __repr__, and __eq__."""
    id: str
    name: str
    email: str
    created_at: datetime = field(default_factory=datetime.now)
    is_active: bool = True

# 使用例
user = User(
    id="123",
    name="Alice",
    email="alice@example.com"
)
```

### バリデーション付きデータクラス

```python
@dataclass
class User:
    email: str
    age: int

    def __post_init__(self):
        # メール形式を検証する
        if "@" not in self.email:
            raise ValueError(f"Invalid email: {self.email}")
        # 年齢範囲を検証する
        if self.age < 0 or self.age > 150:
            raise ValueError(f"Invalid age: {self.age}")
```

### 名前付きタプル

```python
from typing import NamedTuple

class Point(NamedTuple):
    """Immutable 2D point."""
    x: float
    y: float

    def distance(self, other: 'Point') -> float:
        return ((self.x - other.x) ** 2 + (self.y - other.y) ** 2) ** 0.5

# 使用例
p1 = Point(0, 0)
p2 = Point(3, 4)
print(p1.distance(p2))  # 5.0
```

## デコレーター

### 関数デコレーター

```python
import functools
import time

def timer(func: Callable) -> Callable:
    """Decorator to time function execution."""
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{func.__name__} took {elapsed:.4f}s")
        return result
    return wrapper

@timer
def slow_function():
    time.sleep(1)

# slow_function() prints: slow_function took 1.0012s
```

### パラメーター付きデコレーター

```python
def repeat(times: int):
    """Decorator to repeat a function multiple times."""
    def decorator(func: Callable) -> Callable:
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            results = []
            for _ in range(times):
                results.append(func(*args, **kwargs))
            return results
        return wrapper
    return decorator

@repeat(times=3)
def greet(name: str) -> str:
    return f"Hello, {name}!"

# greet("Alice") returns ["Hello, Alice!", "Hello, Alice!", "Hello, Alice!"]
```

### クラスベースのデコレーター

```python
class CountCalls:
    """Decorator that counts how many times a function is called."""
    def __init__(self, func: Callable):
        functools.update_wrapper(self, func)
        self.func = func
        self.count = 0

    def __call__(self, *args, **kwargs):
        self.count += 1
        print(f"{self.func.__name__} has been called {self.count} times")
        return self.func(*args, **kwargs)

@CountCalls
def process():
    pass

# process() を呼び出すたびに呼び出し回数が表示される
```

## 並行処理パターン

### I/Oバウンドタスクのスレッド処理

```python
import concurrent.futures
import threading

def fetch_url(url: str) -> str:
    """Fetch a URL (I/O-bound operation)."""
    import urllib.request
    with urllib.request.urlopen(url) as response:
        return response.read().decode()

def fetch_all_urls(urls: list[str]) -> dict[str, str]:
    """Fetch multiple URLs concurrently using threads."""
    with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
        future_to_url = {executor.submit(fetch_url, url): url for url in urls}
        results = {}
        for future in concurrent.futures.as_completed(future_to_url):
            url = future_to_url[future]
            try:
                results[url] = future.result()
            except Exception as e:
                results[url] = f"Error: {e}"
    return results
```

### CPUバウンドタスクのマルチプロセス処理

```python
def process_data(data: list[int]) -> int:
    """CPU-intensive computation."""
    return sum(x ** 2 for x in data)

def process_all(datasets: list[list[int]]) -> list[int]:
    """Process multiple datasets using multiple processes."""
    with concurrent.futures.ProcessPoolExecutor() as executor:
        results = list(executor.map(process_data, datasets))
    return results
```

### 並行I/OのためのAsync/Await

```python
import asyncio

async def fetch_async(url: str) -> str:
    """Fetch a URL asynchronously."""
    import aiohttp
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.text()

async def fetch_all(urls: list[str]) -> dict[str, str]:
    """Fetch multiple URLs concurrently."""
    tasks = [fetch_async(url) for url in urls]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    return dict(zip(urls, results))
```

## パッケージ構成

### 標準的なプロジェクトレイアウト

```
myproject/
├── src/
│   └── mypackage/
│       ├── __init__.py
│       ├── main.py
│       ├── api/
│       │   ├── __init__.py
│       │   └── routes.py
│       ├── models/
│       │   ├── __init__.py
│       │   └── user.py
│       └── utils/
│           ├── __init__.py
│           └── helpers.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── test_api.py
│   └── test_models.py
├── pyproject.toml
├── README.md
└── .gitignore
```

### インポートの規約

```python
# 良い例: インポートの順序 - 標準ライブラリ、サードパーティ、ローカル
import os
import sys
from pathlib import Path

import requests
from fastapi import FastAPI

from mypackage.models import User
from mypackage.utils import format_name

# 良い例: isortで自動的にインポートを整列する
# pip install isort
```

### パッケージエクスポート用の __init__.py

```python
# mypackage/__init__.py
"""mypackage - A sample Python package."""

__version__ = "1.0.0"

# パッケージレベルでメインクラス/関数をエクスポートする
from mypackage.models import User, Post
from mypackage.utils import format_name

__all__ = ["User", "Post", "format_name"]
```

## メモリとパフォーマンス

### メモリ効率化のための __slots__ の使用

```python
# 悪い例: 通常のクラスは __dict__ を使う（メモリ消費が多い）
class Point:
    def __init__(self, x: float, y: float):
        self.x = x
        self.y = y

# 良い例: __slots__ はメモリ使用量を削減する
class Point:
    __slots__ = ['x', 'y']

    def __init__(self, x: float, y: float):
        self.x = x
        self.y = y
```

### 大量データのためのジェネレーター

```python
# 悪い例: メモリ上にリスト全体を返す
def read_lines(path: str) -> list[str]:
    with open(path) as f:
        return [line.strip() for line in f]

# 良い例: 1行ずつyieldする
def read_lines(path: str) -> Iterator[str]:
    with open(path) as f:
        for line in f:
            yield line.strip()
```

### ループ内での文字列連結を避ける

```python
# 悪い例: 文字列の不変性によりO(n²)
result = ""
for item in items:
    result += str(item)

# 良い例: joinを使いO(n)
result = "".join(str(item) for item in items)

# 良い例: StringIOを使って構築する
from io import StringIO

buffer = StringIO()
for item in items:
    buffer.write(str(item))
result = buffer.getvalue()
```

## Pythonツールの統合

### 必須コマンド

```bash
# コードフォーマット
black .
isort .

# リント
ruff check .
pylint mypackage/

# 型チェック
mypy .

# テスト
pytest --cov=mypackage --cov-report=html

# セキュリティスキャン
bandit -r .

# 依存関係管理
pip-audit
safety check
```

### pyproject.toml の設定

```toml
[project]
name = "mypackage"
version = "1.0.0"
requires-python = ">=3.9"
dependencies = [
    "requests>=2.31.0",
    "pydantic>=2.0.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.4.0",
    "pytest-cov>=4.1.0",
    "black>=23.0.0",
    "ruff>=0.1.0",
    "mypy>=1.5.0",
]

[tool.black]
line-length = 88
target-version = ['py39']

[tool.ruff]
line-length = 88
select = ["E", "F", "I", "N", "W"]

[tool.mypy]
python_version = "3.9"
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "--cov=mypackage --cov-report=term-missing"
```

## クイックリファレンス: Pythonのイディオム

| イディオム | 説明 |
|-------|-------------|
| EAFP | 許可を求めるより許しを求める方が簡単 |
| コンテキストマネージャー | リソース管理に `with` を使う |
| リスト内包表記 | 単純な変換に使う |
| ジェネレーター | 遅延評価と大量データセットに使う |
| 型ヒント | 関数シグネチャにアノテーションを付ける |
| データクラス | 自動生成メソッド付きのデータコンテナに使う |
| `__slots__` | メモリ最適化に使う |
| f文字列 | 文字列フォーマットに使う（Python 3.6以降） |
| `pathlib.Path` | パス操作に使う（Python 3.4以降） |
| `enumerate` | ループ内でインデックスと要素のペアを取得する |

## 避けるべきアンチパターン

```python
# 悪い例: ミュータブルなデフォルト引数
def append_to(item, items=[]):
    items.append(item)
    return items

# 良い例: Noneを使い新しいリストを作成する
def append_to(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items

# 悪い例: type()で型チェックする
if type(obj) == list:
    process(obj)

# 良い例: isinstanceを使う
if isinstance(obj, list):
    process(obj)

# 悪い例: Noneを == で比較する
if value == None:
    process()

# 良い例: is を使う
if value is None:
    process()

# 悪い例: from module import *
from os.path import *

# 良い例: 明示的なインポート
from os.path import join, exists

# 悪い例: 裸のexcept
try:
    risky_operation()
except:
    pass

# 良い例: 特定の例外
try:
    risky_operation()
except SpecificError as e:
    logger.error(f"Operation failed: {e}")
```

__覚えておこう__: Pythonのコードは読みやすく、明示的であり、最小限の驚きの原則に従うべきです。迷ったときは、巧みさよりも明確さを優先しましょう。
