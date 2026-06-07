---
name: flox-environments
description: "Nix上に構築された宣言型環境マネージャーFloxを使用して、再現可能なクロスプラットフォーム開発環境を作成します。以下のいずれかが必要な場合は、必ずこのスキルを使用してください: システムレベルの依存関係（コンパイラ、データベース、openssl・libvips・BLAS・LAPACKなどのネイティブライブラリ）を含むプロジェクトのセットアップ、Python・Node.js・Rust・Go・C/C++・Java・Ruby・Elixir・PHPなどの再現可能なツールチェーンの設定、macOSとLinuxで同一に動作する環境の管理、チーム向けの正確なパッケージバージョンの固定、開発ツールと並行したローカルサービス（PostgreSQL・Redis・Kafka）の実行、単一コマンドによる新規開発者のオンボーディング、または「自分のマシンでは動く」問題の解決。AIアシスト開発やバイブコーディングに特に有効です。Floxを使えばエージェントがsudo不要・システム汚染なし・サンドボックス制限なしでプロジェクトスコープの環境にツールをインストールでき、その環境はリポジトリにコミットされるため誰でも即座に再現できます。ユーザーがFloxに言及していなくても、再現可能・宣言的・クロスプラットフォームなシステムパッケージを含む開発環境が必要と述べている場合はこのスキルを使用してください。.flox/、manifest.toml、flox activate、FloxHubに言及している場合も同様です。"
origin: Flox
---

# Flox 環境

Flox は単一の TOML マニフェストで定義される再現可能な開発環境を作成します。チームのすべての開発者がmacOSとLinuxにわたって同一のパッケージ・ツール・設定を取得できます。コンテナやVMは不要です。Nixをベースに150,000以上のパッケージを利用可能です。

## アクティベートするタイミング

ユーザーが環境管理の問題を抱えている場合にこのスキルを使用してください。Floxに言及していなくても構いません。以下の場合にFloxが適しています:

- プロジェクトに言語固有の依存関係と並行して**システムレベルのパッケージ**（コンパイラ、データベース、CLIツール）が必要な場合
- **再現性が重要な**場合 — チームメンバーのマシン、CI、または新しいノートパソコンでも同一にセットアップが動作する必要がある
- **複数のツールを共存**させる必要がある場合 — 例: 1つの環境でPython 3.11 + PostgreSQL 16 + Redis + Node.js
- **クロスプラットフォームサポート**が必要な場合（同じ設定でmacOSとLinuxの両方をサポート）
- **AIエージェントがツールをインストールする必要がある**場合 — Floxを使うとエージェントがsudo不要・システム汚染なし・サンドボックス制限なしでプロジェクトスコープの環境にパッケージを追加できる

単一の言語ランタイムでシステム依存関係が不要な場合は、標準ツール（nvm、pyenv、rustup単体）で十分かもしれません。完全なOSレベルの分離が必要な場合はコンテナが適切な場合があります。Floxはその中間に位置し、コンテナのオーバーヘッドなしに宣言的で再現可能な環境を提供します。

**前提条件:** Floxを最初にインストールする必要があります。macOS、Linux、Dockerへのインストール方法は [flox.dev/docs](https://flox.dev/docs/install-flox/install/) を参照してください。

## コアコンセプト

Flox環境は `.flox/env/manifest.toml` で定義され、`flox activate` でアクティベートされます。マニフェストはパッケージ、環境変数、セットアップフック、シェル設定を宣言します。つまり、どこでも環境を再現するために必要なすべてが含まれています。

**主要なパス:**
- `.flox/env/manifest.toml` — 環境定義（これをコミットしてください）
- `$FLOX_ENV` — インストール済みパッケージへのランタイムパス（`/usr` のようなもので、`bin/`、`lib/`、`include/` を含む）
- `$FLOX_ENV_CACHE` — キャッシュ、仮想環境、データ用の永続的なローカルストレージ（再ビルド後も保持）
- `$FLOX_ENV_PROJECT` — プロジェクトのルートディレクトリ（`.flox/` が存在する場所）

## 基本コマンド

```bash
flox init                       # 新しい環境を作成
flox search <package> [--all]   # パッケージを検索
flox show <package>             # 利用可能なバージョンを表示
flox install <package>          # パッケージを追加
flox list                       # インストール済みパッケージを一覧表示
flox activate                   # 環境に入る
flox activate -- <cmd>          # サブシェルを起動せずに環境内でコマンドを実行
flox edit                       # マニフェストをインタラクティブに編集
```

## マニフェスト構造

```toml
# .flox/env/manifest.toml

[install]
# インストールするパッケージ — 環境の核心部分
ripgrep.pkg-path = "ripgrep"
jq.pkg-path = "jq"

[vars]
# 静的な環境変数
DATABASE_URL = "postgres://localhost:5432/myapp"

[hook]
# 非インタラクティブなセットアップスクリプト（アクティベートのたびに実行）
on-activate = """
  echo "Environment ready"
"""

[profile]
# シェル関数とエイリアス（インタラクティブシェルで利用可能）
common = """
  alias dev="npm run dev"
"""

[options]
# サポートするプラットフォーム
systems = ["x86_64-linux", "aarch64-linux", "x86_64-darwin", "aarch64-darwin"]
```

## パッケージインストールパターン

### 基本インストール

```toml
[install]
nodejs.pkg-path = "nodejs"
python.pkg-path = "python311"
rustup.pkg-path = "rustup"
```

### バージョン固定

```toml
[install]
nodejs.pkg-path = "nodejs"
nodejs.version = "^20.0"          # セマバー範囲: 最新の20.x

postgres.pkg-path = "postgresql"
postgres.version = "16.2"         # 正確なバージョン
```

### プラットフォーム固有のパッケージ

```toml
[install]
# Linuxのみのツール
valgrind.pkg-path = "valgrind"
valgrind.systems = ["x86_64-linux", "aarch64-linux"]

# macOSフレームワーク
Security.pkg-path = "darwin.apple_sdk.frameworks.Security"
Security.systems = ["x86_64-darwin", "aarch64-darwin"]

# macOS上でのGNUツール（BSDデフォルトと異なる場合）
coreutils.pkg-path = "coreutils"
coreutils.systems = ["x86_64-darwin", "aarch64-darwin"]
```

### パッケージの競合解決

2つのパッケージが同じバイナリをインストールする場合、`priority` を使用してください（数値が小さい方が優先）:

```toml
[install]
gcc.pkg-path = "gcc12"
gcc.priority = 3

clang.pkg-path = "clang_18"
clang.priority = 5               # ファイル競合ではgccが勝つ
```

バージョンを一緒に解決すべきパッケージをグループ化するには `pkg-group` を使用してください:

```toml
[install]
python.pkg-path = "python311"
python.pkg-group = "python-stack"

pip.pkg-path = "python311Packages.pip"
pip.pkg-group = "python-stack"    # pythonと一緒に解決される
```

## 言語別レシピ

### uvを使ったPython

```toml
[install]
python.pkg-path = "python311"
uv.pkg-path = "uv"

[vars]
UV_CACHE_DIR = "$FLOX_ENV_CACHE/uv-cache"
PIP_CACHE_DIR = "$FLOX_ENV_CACHE/pip-cache"

[hook]
on-activate = """
  venv="$FLOX_ENV_CACHE/venv"
  if [ ! -d "$venv" ]; then
    uv venv "$venv" --python python3
  fi
  if [ -f "$venv/bin/activate" ]; then
    source "$venv/bin/activate"
  fi

  if [ -f requirements.txt ] && [ ! -f "$FLOX_ENV_CACHE/.deps_installed" ]; then
    uv pip install --python "$venv/bin/python" -r requirements.txt --quiet
    touch "$FLOX_ENV_CACHE/.deps_installed"
  fi
"""
```

### Node.js

```toml
[install]
nodejs.pkg-path = "nodejs"
nodejs.version = "^20.0"

[hook]
on-activate = """
  if [ -f package.json ] && [ ! -d node_modules ]; then
    npm install --silent
  fi
"""
```

### Rust

```toml
[install]
rustup.pkg-path = "rustup"
pkg-config.pkg-path = "pkg-config"
openssl.pkg-path = "openssl"

[vars]
RUSTUP_HOME = "$FLOX_ENV_CACHE/rustup"
CARGO_HOME = "$FLOX_ENV_CACHE/cargo"

[profile]
common = """
  export PATH="$CARGO_HOME/bin:$PATH"
"""
```

### Go

```toml
[install]
go.pkg-path = "go"
gopls.pkg-path = "gopls"
delve.pkg-path = "delve"

[vars]
GOPATH = "$FLOX_ENV_CACHE/go"
GOBIN = "$FLOX_ENV_CACHE/go/bin"

[profile]
common = """
  export PATH="$GOBIN:$PATH"
"""
```

### C/C++

```toml
[install]
gcc.pkg-path = "gcc13"
gcc.pkg-group = "compilers"

# 重要: gcc単体ではlibstdc++ヘッダーが公開されません — gcc-unwrappedが必要です
gcc-unwrapped.pkg-path = "gcc-unwrapped"
gcc-unwrapped.pkg-group = "libraries"

cmake.pkg-path = "cmake"
cmake.pkg-group = "build"

gnumake.pkg-path = "gnumake"
gnumake.pkg-group = "build"

gdb.pkg-path = "gdb"
gdb.systems = ["x86_64-linux", "aarch64-linux"]
```

## フックとプロファイル

### フック — 非インタラクティブなセットアップ

フックはアクティベートのたびに実行されます。高速かつ冪等に保ってください。原則として: **自動的に実行すべきことは `[hook]` に、ユーザーが入力して実行すべきことは `[profile]` に記述してください。**

```toml
[hook]
on-activate = """
  setup_database() {
    if [ ! -d "$FLOX_ENV_CACHE/pgdata" ]; then
      initdb -D "$FLOX_ENV_CACHE/pgdata" --no-locale --encoding=UTF8
    fi
  }
  setup_database
"""
```

### プロファイル — インタラクティブシェルの設定

プロファイルコードはユーザーのシェルセッションで利用可能です。

```toml
[profile]
common = """
  dev() { npm run dev; }
  test() { npm run test -- "$@"; }
"""
```

## アンチパターン

### 絶対パス

```toml
# 悪い例 — 他のマシンで動作しない
[vars]
PROJECT_DIR = "/home/alice/projects/myapp"

# 良い例 — Flox環境変数を使用
[vars]
PROJECT_DIR = "$FLOX_ENV_PROJECT"
```

### フックでの exit の使用

```toml
# 悪い例 — シェルが終了してしまう
[hook]
on-activate = """
  if [ ! -f config.json ]; then
    echo "Missing config"
    exit 1
  fi
"""

# 良い例 — exitではなくreturnを使用
[hook]
on-activate = """
  if [ ! -f config.json ]; then
    echo "Missing config — run setup first"
    return 1
  fi
"""
```

### マニフェストへのシークレットの保存

```toml
# 悪い例 — マニフェストはgitにコミットされる
[vars]
API_KEY = "<set-at-runtime>"

# 良い例 — 外部設定を参照するかランタイムに渡す
# 使い方: API_KEY="<your-api-key>" flox activate
[vars]
API_KEY = "${API_KEY:-}"
```

### 冪等性ガードのない遅いフック

```toml
# 悪い例 — アクティベートのたびに再インストールされる
[hook]
on-activate = """
  pip install -r requirements.txt
"""

# 良い例 — インストール済みの場合はスキップ
[hook]
on-activate = """
  if [ ! -f "$FLOX_ENV_CACHE/.deps_installed" ]; then
    uv pip install -r requirements.txt --quiet
    touch "$FLOX_ENV_CACHE/.deps_installed"
  fi
"""
```

### フックへのユーザーコマンドの記述

```toml
# 悪い例 — フック関数はインタラクティブシェルで利用できない
[hook]
on-activate = """
  deploy() { kubectl apply -f k8s/; }
"""

# 良い例 — ユーザーが呼び出す関数には [profile] を使用
[profile]
common = """
  deploy() { kubectl apply -f k8s/; }
"""
```

## フルスタックの例

PostgreSQLを使ったPython APIの完全な環境:

```toml
[install]
python.pkg-path = "python311"
uv.pkg-path = "uv"
postgresql.pkg-path = "postgresql_16"
redis.pkg-path = "redis"
jq.pkg-path = "jq"
curl.pkg-path = "curl"

[vars]
UV_CACHE_DIR = "$FLOX_ENV_CACHE/uv-cache"
DATABASE_URL = "postgres://localhost:5432/myapp"
REDIS_URL = "redis://localhost:6379"

[hook]
on-activate = """
  if [ ! -d "$FLOX_ENV_CACHE/pgdata" ]; then
    initdb -D "$FLOX_ENV_CACHE/pgdata" --no-locale --encoding=UTF8
  fi

  venv="$FLOX_ENV_CACHE/venv"
  if [ ! -d "$venv" ]; then
    uv venv "$venv" --python python3
  fi
  if [ -f "$venv/bin/activate" ]; then
    source "$venv/bin/activate"
  fi

  if [ -f requirements.txt ] && [ ! -f "$FLOX_ENV_CACHE/.deps_installed" ]; then
    uv pip install --python "$venv/bin/python" -r requirements.txt --quiet
    touch "$FLOX_ENV_CACHE/.deps_installed"
  fi
"""

[profile]
common = """
  serve() { uvicorn app.main:app --reload --host 0.0.0.0 --port 8000; }
  migrate() { alembic upgrade head; }
"""

[services]
postgres.command = "postgres -D $FLOX_ENV_CACHE/pgdata -k $FLOX_ENV_CACHE"
redis.command = "redis-server --port 6379 --daemonize no"

[options]
systems = ["x86_64-linux", "aarch64-linux", "x86_64-darwin", "aarch64-darwin"]
```

サービスと共にアクティベート: `flox activate --start-services`

## 環境の共有

Flox環境はgitネイティブです。`.flox/` ディレクトリをコミットすれば、すべての共同作業者が同じ環境を取得できます:

```bash
git add .flox/
git commit -m "Add Flox environment"
# チームメンバーはこれだけ実行:
git clone <repo> && cd <repo> && flox activate
```

プロジェクト間で再利用可能なベース環境は、FloxHubにプッシュしてください:

```bash
flox push                         # 環境をFloxHubにプッシュ
flox activate -r owner/env-name   # どこでもリモート環境をアクティベート
```

`[include]` で環境を合成できます:

```toml
[include]
base.floxhub = "myorg/python-base"

[install]
# ベースの上にプロジェクト固有の追加
fastapi.pkg-path = "python311Packages.fastapi"
```

## AIアシスト開発とバイブコーディング

FloxはAIアシスト開発とバイブコーディングのワークフローに最適です。AIエージェントが現在の環境にないツール（コンパイラ、データベース、リンター、CLIユーティリティなど）を必要とする場合、sudo権限不要・システムパッケージの汚染なし・サンドボックス制限なしで、プロジェクトのFloxマニフェストに追加できます。

**エージェントにとっての利点:**
- **sudo不要** — `flox install` は完全にユーザースペースで動作するため、エージェントが昇格権限なしにパッケージを追加できる
- **プロジェクトスコープ** — パッケージはプロジェクト環境にのみインストールされ、グローバルにはインストールされないため、異なるプロジェクトが競合なく異なるバージョンを持てる
- **サンドボックスフレンドリー** — サンドボックスや制限された環境で実行中のエージェントでも、Floxを通じて必要なツールをインストールできる
- **可逆的** — すべての変更は `manifest.toml` に記録されるため、不要なパッケージをシステム残留なしに削除できる
- **再現可能** — エージェントが環境をセットアップすると、その正確なセットアップがgitにコミットされ、誰でも利用できる

**エージェントのワークフローパターン:**

```bash
# エージェントがツールを必要としていることを発見（例: JSON処理のためのjq）
flox search jq                    # パッケージの存在を確認
flox install jq                   # プロジェクト環境にインストール

# より細かく制御したい場合、マニフェストを直接編集
tmp_manifest="$(mktemp)"
flox list -c > "$tmp_manifest"
# [install] セクションにパッケージを追加して適用
flox edit -f "$tmp_manifest"

# ツールが利用可能な状態でコマンドを実行
flox activate -- jq '.results[]' data.json
```

これにより、FloxはClaude Codeや他のAIエージェントがプロジェクトのツール環境をオンデマンドで構築するあらゆるワークフローに自然に適合します。

## デバッグ

```bash
flox list -c                      # 生のマニフェストを表示
flox activate -- which python     # どのバイナリが解決されるか確認
flox activate -- env | grep FLOX  # Flox環境変数を表示
flox search <package> --all       # より広いパッケージ検索（大文字小文字を区別）
```

**よくある問題:**
- **パッケージが見つからない:** 検索は大文字小文字を区別します — `flox search --all` を試してください
- **パッケージ間のファイル競合:** 優先すべきパッケージに `priority` を追加してください
- **フックの失敗:** `exit` ではなく `return` を使用してください; `${FLOX_ENV_CACHE:-}` でガードしてください
- **依存関係が古い:** `$FLOX_ENV_CACHE/.deps_installed` フラグファイルを削除してください

## 関連スキル

以下のスキルはより深い統合のために [Flox Claude Codeプラグイン](https://github.com/flox/flox-agentic) の一部として利用可能です:

- **flox-services** — サービス管理、データベースセットアップ、バックグラウンドプロセス
- **flox-builds** — Floxを使った再現可能なビルドとパッケージング
- **flox-containers** — Flox環境からDocker/OCIコンテナを作成
- **flox-sharing** — 環境の合成、リモート環境、チームパターン
- **flox-cuda** — CUDAとGPU開発環境

詳細とインストール方法: [flox.dev/docs](https://flox.dev/docs/install-flox/install/)
