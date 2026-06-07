---
name: django-celery
description: Django + Celery非同期タスクパターン — 設定、タスク設計、Beatスケジューリング、リトライ、キャンバスワークフロー、モニタリング、テスト。Djangoアプリにバックグラウンドジョブやスケジュールされたタスクまたは非同期処理を追加するときに使用する。
origin: ECC
---

# Django + Celery 非同期タスクパターン

RedisまたはRabbitMQを使ったCeleryによるDjangoのバックグラウンドタスク処理のプロダクショングレードのパターン集。

## いつ起動するか

- DjangoアプリにバックグラウンドジョブやAsynic処理を追加するとき
- 定期的/スケジュールされたタスクを実装するとき
- リクエストサイクルから遅い操作（メール・PDF生成・API呼び出し）をオフロードするとき
- cronライクなスケジューリングのためにCelery Beatをセットアップするとき
- タスクの失敗・リトライ・キューのバックログをデバッグするとき
- Celeryタスクのテストを書くとき

## プロジェクトのセットアップ

### インストール

```bash
pip install celery[redis] django-celery-results django-celery-beat
```

### `celery.py` — アプリエントリーポイント

```python
# config/celery.py
import os
from celery import Celery

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings.development')

app = Celery('myproject')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()  # Discovers tasks.py in each INSTALLED_APP

@app.task(bind=True, ignore_result=True)
def debug_task(self):
    print(f'Request: {self.request!r}')
```

```python
# config/__init__.py
from .celery import app as celery_app

__all__ = ('celery_app',)
```

### Django設定

```python
# config/settings/base.py

# ブローカー（本番環境にはRedis推奨）
CELERY_BROKER_URL = env('CELERY_BROKER_URL', default='redis://localhost:6379/0')
CELERY_RESULT_BACKEND = env('CELERY_RESULT_BACKEND', default='django-db')

# シリアライゼーション
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'

# タスク動作
CELERY_TASK_TRACK_STARTED = True
CELERY_TASK_TIME_LIMIT = 30 * 60        # ハードリミット：30分
CELERY_TASK_SOFT_TIME_LIMIT = 25 * 60   # ソフトリミット：SoftTimeLimitExceededを送信
CELERY_WORKER_PREFETCH_MULTIPLIER = 1   # ワーカーが長いタスクを独占しないよう防止
CELERY_TASK_ACKS_LATE = True            # ワーカークラッシュ時に再キュー

# 結果の永続化
CELERY_RESULT_EXPIRES = 60 * 60 * 24   # 結果を24時間保持

# Beatスケジューラ（定期タスク用）
CELERY_BEAT_SCHEDULER = 'django_celery_beat.schedulers:DatabaseScheduler'

# インストール済みアプリ
INSTALLED_APPS += [
    'django_celery_results',
    'django_celery_beat',
]
```

### ワーカーの起動

```bash
# ワーカーの起動（開発環境）
celery -A config worker --loglevel=info

# Beatスケジューラの起動（定期タスク）
celery -A config beat --loglevel=info --scheduler django_celery_beat.schedulers:DatabaseScheduler

# ワーカー＋Beatの同時起動（開発のみ、本番では絶対にしない）
celery -A config worker --beat --loglevel=info

# 本番環境：同時実行数を指定した複数ワーカー
celery -A config worker --loglevel=warning --concurrency=4 -Q default,high_priority
```

## タスク設計パターン

### 基本タスク

```python
# apps/notifications/tasks.py
from celery import shared_task
import logging

logger = logging.getLogger(__name__)

@shared_task(name='notifications.send_welcome_email')
def send_welcome_email(user_id: int) -> None:
    """新規登録ユーザーにウェルカムメールを送信する。"""
    from apps.users.models import User
    from apps.notifications.services import EmailService

    try:
        user = User.objects.get(pk=user_id)
    except User.DoesNotExist:
        logger.warning('send_welcome_email: user %s not found', user_id)
        return  # べき等 — raiseしない、タスクはもう完了不可能

    EmailService.send_welcome(user)
    logger.info('Welcome email sent to user %s', user_id)
```

### リトライ可能なタスク

```python
@shared_task(
    bind=True,
    name='integrations.sync_to_crm',
    max_retries=5,
    default_retry_delay=60,       # 最初のリトライまでの秒数
    autoretry_for=(ConnectionError, TimeoutError),
    retry_backoff=True,           # 指数バックオフ
    retry_backoff_max=600,        # 最大10分にキャップ
    retry_jitter=True,            # サンダーリングハードを防ぐためランダム化
)
def sync_contact_to_crm(self, contact_id: int) -> dict:
    """過渡的障害時にリトライする外部CRMへのコンタクト同期。"""
    from apps.crm.services import CRMClient

    try:
        result = CRMClient().sync(contact_id)
        return result
    except CRMClient.RateLimitError as exc:
        # レスポンスヘッダーからの特定リトライ遅延
        raise self.retry(exc=exc, countdown=int(exc.retry_after))
```

### べき等タスクパターン

同じ入力で複数回安全に実行できるようにタスクを設計します：

```python
@shared_task(name='orders.mark_shipped')
def mark_order_shipped(order_id: int, tracking_number: str) -> None:
    """注文を発送済みとしてマーク — 複数回実行しても安全。"""
    from apps.orders.models import Order

    updated = Order.objects.filter(
        pk=order_id,
        status=Order.Status.PROCESSING,    # ガード：まだ発送済みでない場合のみ更新
    ).update(
        status=Order.Status.SHIPPED,
        tracking_number=tracking_number,
    )

    if not updated:
        logger.info('mark_order_shipped: order %s already shipped or not found', order_id)
```

### ソフトタイムリミット付きタスク

```python
from celery.exceptions import SoftTimeLimitExceeded

@shared_task(
    bind=True,
    name='reports.generate_pdf',
    soft_time_limit=120,
    time_limit=150,
)
def generate_pdf_report(self, report_id: int) -> str:
    """グレースフルなタイムアウト処理を持つPDFレポートを生成する。"""
    from apps.reports.services import PDFGenerator

    try:
        path = PDFGenerator.build(report_id)
        return path
    except SoftTimeLimitExceeded:
        # ハードキル前に部分的なファイルをクリーンアップ
        PDFGenerator.cleanup(report_id)
        raise
```

## タスクの呼び出し

```python
from datetime import timedelta
from django.utils import timezone

# ファイアアンドフォーゲット（非同期）
send_welcome_email.delay(user.pk)

# 将来にスケジュール
send_reminder.apply_async(args=[user.pk], countdown=3600)  # 今から1時間後
send_reminder.apply_async(args=[user.pk], eta=timezone.now() + timedelta(days=1))

# キュールーティングを指定して適用
sync_contact_to_crm.apply_async(args=[contact.pk], queue='high_priority')

# 同期実行（テスト/デバッグのみ）
result = generate_pdf_report.apply(args=[report.pk])
```

## Beatスケジューリング（定期タスク）

### コード定義スケジュール

```python
# config/settings/base.py
from celery.schedules import crontab

CELERY_BEAT_SCHEDULE = {
    'cleanup-expired-sessions': {
        'task': 'users.cleanup_expired_sessions',
        'schedule': crontab(hour=2, minute=0),   # 毎日午前2時
    },
    'sync-inventory': {
        'task': 'products.sync_inventory',
        'schedule': 60.0,                         # 60秒ごと
    },
    'weekly-digest': {
        'task': 'notifications.send_weekly_digest',
        'schedule': crontab(day_of_week='monday', hour=8, minute=0),
    },
}
```

### データベース定義スケジュール（django-celery-beat経由）

```python
# Djangoアドミンまたはコードから定期タスクを管理
from django_celery_beat.models import PeriodicTask, CrontabSchedule
import json

schedule, _ = CrontabSchedule.objects.get_or_create(
    hour='*/6', minute='0',
    timezone='UTC',
)

PeriodicTask.objects.update_or_create(
    name='Sync inventory every 6 hours',
    defaults={
        'crontab': schedule,
        'task': 'products.sync_inventory',
        'args': json.dumps([]),
        'enabled': True,
    }
)
```

## キャンバス：タスクのチェーンとグループ化

```python
from celery import chain, group, chord

# チェーン：タスクを順次実行し、結果を渡す
pipeline = chain(
    fetch_data.s(source_id),
    transform_data.s(),          # fetch_dataの結果を第一引数として受け取る
    load_to_warehouse.s(),
)
pipeline.delay()

# グループ：タスクを並列実行
parallel = group(
    send_welcome_email.s(user_id)
    for user_id in new_user_ids
)
parallel.delay()

# コード：並列タスク＋全完了時のコールバック
result = chord(
    group(process_chunk.s(chunk) for chunk in data_chunks),
    aggregate_results.s(),       # チャンク結果のリストで呼ばれる
)
result.delay()
```

## エラー処理とデッドレターキュー

```python
# apps/core/tasks.py
from celery.signals import task_failure

@task_failure.connect
def on_task_failure(sender, task_id, exception, args, kwargs, traceback, einfo, **kw):
    """全タスク失敗をSentry/アラートシステムへ記録する。"""
    import sentry_sdk
    with sentry_sdk.new_scope() as scope:
        scope.set_context('celery', {
            'task': sender.name,
            'task_id': task_id,
            'args': args,
            'kwargs': kwargs,
        })
        sentry_sdk.capture_exception(exception)
```

```python
# 最大リトライ後に失敗したタスクをデッドレターキューへルーティング
@shared_task(
    bind=True,
    max_retries=3,
    name='payments.charge_card',
)
def charge_card(self, order_id: int) -> None:
    from apps.payments.models import Order, FailedCharge

    try:
        _do_charge(order_id)
    except Exception as exc:
        if self.request.retries >= self.max_retries:
            # 手動レビューのためデッドレターテーブルに永続化
            FailedCharge.objects.create(
                order_id=order_id,
                error=str(exc),
                task_id=self.request.id,
            )
            return  # raiseしない — タスクは永続的に失敗
        raise self.retry(exc=exc)
```

## Celeryタスクのテスト

### ユニットテスト（ブローカーなし）

```python
# tests/test_tasks.py
import pytest
from unittest.mock import patch, MagicMock
from apps.notifications.tasks import send_welcome_email

class TestSendWelcomeEmail:

    @pytest.mark.django_db
    def test_sends_email_to_existing_user(self, user):
        with patch('apps.notifications.services.EmailService') as mock_email:
            send_welcome_email(user.pk)
            mock_email.send_welcome.assert_called_once_with(user)

    @pytest.mark.django_db
    def test_skips_missing_user_gracefully(self):
        """エンキューと実行の間にユーザーが削除されてもraiseしてはいけない。"""
        send_welcome_email(99999)  # 存在しないユーザー — raiseしてはいけない
```

### CELERY_TASK_ALWAYS_EAGERを使った統合テスト

```python
# config/settings/test.py
CELERY_TASK_ALWAYS_EAGER = True      # テストでは同期的にタスクを実行
CELERY_TASK_EAGER_PROPAGATES = True  # タスクからの例外を再raiseする

# tests/test_integration.py
@pytest.mark.django_db
def test_registration_triggers_welcome_email(client):
    with patch('apps.notifications.services.EmailService') as mock_email:
        response = client.post('/api/users/', {
            'email': 'new@example.com',
            'password': 'strongpass123',
        })

    assert response.status_code == 201
    mock_email.send_welcome.assert_called_once()
```

### リトライのテスト

```python
@pytest.mark.django_db
def test_task_retries_on_connection_error():
    with patch('apps.crm.services.CRMClient.sync') as mock_sync:
        mock_sync.side_effect = ConnectionError('timeout')

        with pytest.raises(ConnectionError):
            sync_contact_to_crm.apply(args=[1], throw=True)

        assert mock_sync.call_count == 1  # イーガーモードでは最初の試行のみ
```

## モニタリング

```bash
# アクティブなワーカーとキューを検査
celery -A config inspect active
celery -A config inspect stats
celery -A config inspect reserved

# キュー長の確認（Redis）
redis-cli llen celery

# Flower：Webベースのリアルタイムモニター
pip install flower
celery -A config flower --port=5555
```

## アンチパターン

```python
# 悪い例：モデルインスタンスの渡し方 — 実行時に古くなっている可能性がある
send_welcome_email.delay(user)        # ORMオブジェクトを渡さない
send_welcome_email.delay(user.pk)     # 常にPKを渡す

# 悪い例：本番ビューでタスクを同期呼び出し
result = generate_report.apply()      # リクエストスレッドをブロックする

# 悪い例：ガードなしの非べき等タスク
@shared_task
def charge_and_fulfill(order_id):
    order.charge()     # タスクがリトライされると二重請求になる可能性あり！
    order.fulfill()

# 良い例：ステータスガードを持つべき等
@shared_task
def charge_and_fulfill(order_id):
    order = Order.objects.select_for_update().get(pk=order_id)
    if order.status != Order.Status.PENDING:
        return  # 既に処理済み
    order.charge()
    order.fulfill()
```

## 本番チェックリスト

| チェック項目 | 設定 |
|-------|---------|
| クラッシュ後のワーカー再起動 | `supervisord` または `systemd` ユニット |
| `CELERY_TASK_ACKS_LATE = True` | ワーカークラッシュ時にタスクを再キュー |
| `CELERY_WORKER_PREFETCH_MULTIPLIER = 1` | 長いタスクの公平な分配 |
| 優先度ごとの別キュー | `-Q default,high_priority,low_priority` |
| `CELERY_TASK_SOFT_TIME_LIMIT` の設定 | ハードキル前のグレースフルなタイムアウト |
| Sentry統合 | 全 `task_failure` シグナルのキャプチャ |
| FlowerまたはほかのモニタリングツールFF | キュー深さの可視化 |
| Beatは単一ノードのみで実行 | スケジュールされたタスクの重複実行を防止 |

## 関連スキル

- `django-patterns` — ORM、サービスレイヤー、プロジェクト構造
- `django-tdd` — Djangoのモデル・ビュー・サービスのテスト
- `python-testing` — pytestの設定とフィクスチャ
