---
name: django-verification
description: "Djangoプロジェクト向け検証ループ：マイグレーション、リンティング、カバレッジ付きテスト、セキュリティスキャン、リリースやPR前のデプロイ準備チェック。"
origin: ECC
---

# Django 検証ループ

PR作成前、大規模な変更後、デプロイ前に実行して、Djangoアプリケーションの品質とセキュリティを確保します。

## 起動タイミング

- Djangoプロジェクトのプルリクエストを開く前
- モデルの大幅な変更、マイグレーションの更新、または依存関係のアップグレード後
- ステージングや本番環境へのデプロイ前の検証
- 環境チェック → リント → テスト → セキュリティ → デプロイ準備のフルパイプライン実行
- マイグレーションの安全性とテストカバレッジの検証

## フェーズ1: 環境チェック

```bash
# Pythonバージョンの確認
python --version  # プロジェクトの要件と一致していること

# 仮想環境の確認
which python
pip list --outdated

# 環境変数の確認
python -c "import os; import environ; print('DJANGO_SECRET_KEY set' if os.environ.get('DJANGO_SECRET_KEY') else 'MISSING: DJANGO_SECRET_KEY')"
```

環境の設定が正しくない場合は停止して修正します。

## フェーズ2: コード品質とフォーマット

```bash
# 型チェック
mypy . --config-file pyproject.toml

# ruffによるリンティング
ruff check . --fix

# blackによるフォーマット
black . --check
black .  # 自動修正

# インポートの並び替え
isort . --check-only
isort .  # 自動修正

# Django固有のチェック
python manage.py check --deploy
```

よくある問題:
- publicな関数に型ヒントがない
- PEP 8フォーマット違反
- ソートされていないインポート
- 本番設定にデバッグ設定が残っている

## フェーズ3: マイグレーション

```bash
# 未適用のマイグレーションを確認
python manage.py showmigrations

# 不足しているマイグレーションを作成
python manage.py makemigrations --check

# マイグレーション適用のドライラン
python manage.py migrate --plan

# マイグレーションの適用（テスト環境）
python manage.py migrate

# マイグレーションの競合を確認
python manage.py makemigrations --merge  # 競合がある場合のみ
```

報告内容:
- 未適用マイグレーションの数
- マイグレーションの競合
- マイグレーションのないモデル変更

## フェーズ4: テストとカバレッジ

```bash
# pytestで全テストを実行
pytest --cov=apps --cov-report=html --cov-report=term-missing --reuse-db

# 特定アプリのテストを実行
pytest apps/users/tests/

# マーカー付きで実行
pytest -m "not slow"  # 遅いテストをスキップ
pytest -m integration  # インテグレーションテストのみ

# カバレッジレポート
open htmlcov/index.html
```

報告内容:
- テスト総数: X件合格、Y件失敗、Z件スキップ
- 全体カバレッジ: XX%
- アプリ別カバレッジの内訳

カバレッジ目標:

| コンポーネント | 目標 |
|-----------|--------|
| モデル | 90%以上 |
| シリアライザー | 85%以上 |
| ビュー | 80%以上 |
| サービス | 90%以上 |
| 全体 | 80%以上 |

## フェーズ5: セキュリティスキャン

```bash
# 依存関係の脆弱性
pip-audit
safety check --full-report

# Djangoセキュリティチェック
python manage.py check --deploy

# Banditセキュリティリンター
bandit -r . -f json -o bandit-report.json

# シークレットスキャン（gitleaksがインストールされている場合）
gitleaks detect --source . --verbose

# 環境変数のチェック
python -c "from django.core.exceptions import ImproperlyConfigured; from django.conf import settings; settings.DEBUG"
```

報告内容:
- 脆弱な依存関係が見つかった場合
- セキュリティ設定の問題
- ハードコードされたシークレットが検出された場合
- DEBUGモードの状態（本番環境ではFalseであること）

## フェーズ6: Django管理コマンド

```bash
# モデルの問題を確認
python manage.py check

# 静的ファイルの収集
python manage.py collectstatic --noinput --clear

# スーパーユーザーの作成（テスト用に必要な場合）
echo "from apps.users.models import User; User.objects.create_superuser('admin@example.com', 'admin')" | python manage.py shell

# データベースの整合性
python manage.py check --database default

# キャッシュの確認（Redisを使用している場合）
python -c "from django.core.cache import cache; cache.set('test', 'value', 10); print(cache.get('test'))"
```

## フェーズ7: パフォーマンスチェック

```bash
# Django Debug Toolbarの出力（N+1クエリの確認）
# DEBUG=Trueの開発モードで実行してページにアクセス
# SQLパネルで重複クエリを探す

# クエリ数の分析
django-admin debugsqlshell  # django-debug-sqlshellがインストールされている場合

# インデックスの欠落を確認
python manage.py shell << EOF
from django.db import connection
with connection.cursor() as cursor:
    cursor.execute("SELECT table_name, index_name FROM information_schema.statistics WHERE table_schema = 'public'")
    print(cursor.fetchall())
EOF
```

報告内容:
- ページあたりのクエリ数（一般的なページでは50件未満が目安）
- 不足しているデータベースインデックス
- 検出された重複クエリ

## フェーズ8: 静的ファイル

```bash
# npmの依存関係を確認（npmを使用している場合）
npm audit
npm audit fix

# 静的ファイルのビルド（webpack/viteを使用している場合）
npm run build

# 静的ファイルの確認
ls -la staticfiles/
python manage.py findstatic css/style.css
```

## フェーズ9: 設定レビュー

```python
# 設定を確認するためにPythonシェルで実行
python manage.py shell << EOF
from django.conf import settings
import os

# 重要なチェック
checks = {
    'DEBUG is False': not settings.DEBUG,
    'SECRET_KEY set': bool(settings.SECRET_KEY and len(settings.SECRET_KEY) > 30),
    'ALLOWED_HOSTS set': len(settings.ALLOWED_HOSTS) > 0,
    'HTTPS enabled': getattr(settings, 'SECURE_SSL_REDIRECT', False),
    'HSTS enabled': getattr(settings, 'SECURE_HSTS_SECONDS', 0) > 0,
    'Database configured': settings.DATABASES['default']['ENGINE'] != 'django.db.backends.sqlite3',
}

for check, result in checks.items():
    status = '✓' if result else '✗'
    print(f"{status} {check}")
EOF
```

## フェーズ10: ロギング設定

```bash
# ロギング出力のテスト
python manage.py shell << EOF
import logging
logger = logging.getLogger('django')
logger.warning('Test warning message')
logger.error('Test error message')
EOF

# ログファイルの確認（設定されている場合）
tail -f /var/log/django/django.log
```

## フェーズ11: APIドキュメント（DRFを使用している場合）

```bash
# スキーマの生成
python manage.py generateschema --format openapi-json > schema.json

# スキーマの検証
# schema.jsonが有効なJSONであることを確認
python -c "import json; json.load(open('schema.json'))"

# Swagger UIへのアクセス（drf-yasgを使用している場合）
# ブラウザで http://localhost:8000/swagger/ を開く
```

## フェーズ12: 差分レビュー

```bash
# 差分の統計を表示
git diff --stat

# 実際の変更内容を表示
git diff

# 変更されたファイルを表示
git diff --name-only

# よくある問題を確認
git diff | grep -i "todo\|fixme\|hack\|xxx"
git diff | grep "print("  # デバッグ文
git diff | grep "DEBUG = True"  # デバッグモード
git diff | grep "import pdb"  # デバッガー
```

チェックリスト:
- デバッグ文なし（print、pdb、breakpoint()）
- 重要なコードにTODO/FIXMEコメントなし
- ハードコードされたシークレットや認証情報なし
- モデル変更にデータベースマイグレーションが含まれている
- 設定変更が文書化されている
- 外部呼び出しのエラー処理が存在する
- 必要な箇所にトランザクション管理が適用されている

## 出力テンプレート

```
DJANGO VERIFICATION REPORT
==========================

Phase 1: Environment Check
  ✓ Python 3.11.5
  ✓ Virtual environment active
  ✓ All environment variables set

Phase 2: Code Quality
  ✓ mypy: No type errors
  ✗ ruff: 3 issues found (auto-fixed)
  ✓ black: No formatting issues
  ✓ isort: Imports properly sorted
  ✓ manage.py check: No issues

Phase 3: Migrations
  ✓ No unapplied migrations
  ✓ No migration conflicts
  ✓ All models have migrations

Phase 4: Tests + Coverage
  Tests: 247 passed, 0 failed, 5 skipped
  Coverage:
    Overall: 87%
    users: 92%
    products: 89%
    orders: 85%
    payments: 91%

Phase 5: Security Scan
  ✗ pip-audit: 2 vulnerabilities found (fix required)
  ✓ safety check: No issues
  ✓ bandit: No security issues
  ✓ No secrets detected
  ✓ DEBUG = False

Phase 6: Django Commands
  ✓ collectstatic completed
  ✓ Database integrity OK
  ✓ Cache backend reachable

Phase 7: Performance
  ✓ No N+1 queries detected
  ✓ Database indexes configured
  ✓ Query count acceptable

Phase 8: Static Assets
  ✓ npm audit: No vulnerabilities
  ✓ Assets built successfully
  ✓ Static files collected

Phase 9: Configuration
  ✓ DEBUG = False
  ✓ SECRET_KEY configured
  ✓ ALLOWED_HOSTS set
  ✓ HTTPS enabled
  ✓ HSTS enabled
  ✓ Database configured

Phase 10: Logging
  ✓ Logging configured
  ✓ Log files writable

Phase 11: API Documentation
  ✓ Schema generated
  ✓ Swagger UI accessible

Phase 12: Diff Review
  Files changed: 12
  +450, -120 lines
  ✓ No debug statements
  ✓ No hardcoded secrets
  ✓ Migrations included

RECOMMENDATION: WARNING: Fix pip-audit vulnerabilities before deploying

NEXT STEPS:
1. Update vulnerable dependencies
2. Re-run security scan
3. Deploy to staging for final testing
```

## デプロイ前チェックリスト

- [ ] 全テスト合格
- [ ] カバレッジ80%以上
- [ ] セキュリティ脆弱性なし
- [ ] 未適用のマイグレーションなし
- [ ] 本番設定でDEBUG = False
- [ ] SECRET_KEYが適切に設定されている
- [ ] ALLOWED_HOSTSが正しく設定されている
- [ ] データベースバックアップが有効
- [ ] 静的ファイルが収集・配信されている
- [ ] ロギングが設定・動作している
- [ ] エラー監視（Sentryなど）が設定されている
- [ ] CDNが設定されている（該当する場合）
- [ ] Redis/キャッシュバックエンドが設定されている
- [ ] Celeryワーカーが動作している（該当する場合）
- [ ] HTTPS/SSLが設定されている
- [ ] 環境変数が文書化されている

## 継続的インテグレーション

### GitHub Actionsの例

```yaml
# .github/workflows/django-verification.yml
name: Django Verification

on: [push, pull_request]

jobs:
  verify:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Cache pip
        uses: actions/cache@v3
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install ruff black mypy pytest pytest-django pytest-cov bandit safety pip-audit

      - name: Code quality checks
        run: |
          ruff check .
          black . --check
          isort . --check-only
          mypy .

      - name: Security scan
        run: |
          bandit -r . -f json -o bandit-report.json
          safety check --full-report
          pip-audit

      - name: Run tests
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost:5432/test
          DJANGO_SECRET_KEY: test-secret-key
        run: |
          pytest --cov=apps --cov-report=xml --cov-report=term-missing

      - name: Upload coverage
        uses: codecov/codecov-action@v3
```

## クイックリファレンス

| チェック | コマンド |
|-------|---------|
| 環境 | `python --version` |
| 型チェック | `mypy .` |
| リンティング | `ruff check .` |
| フォーマット | `black . --check` |
| マイグレーション | `python manage.py makemigrations --check` |
| テスト | `pytest --cov=apps` |
| セキュリティ | `pip-audit && bandit -r .` |
| Djangoチェック | `python manage.py check --deploy` |
| Collectstatic | `python manage.py collectstatic --noinput` |
| 差分統計 | `git diff --stat` |

注意: 自動検証は一般的な問題を検出しますが、手動コードレビューとステージング環境でのテストの代替にはなりません。
