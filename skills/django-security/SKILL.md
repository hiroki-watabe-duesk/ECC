---
name: django-security
description: Djangoのセキュリティベストプラクティス、認証、認可、CSRF保護、SQLインジェクション防止、XSS防止、および安全なデプロイ設定。
origin: ECC
---

# Djangoセキュリティベストプラクティス

一般的な脆弱性からDjangoアプリケーションを保護するための包括的なセキュリティガイドライン。

## 有効化するタイミング

- Django認証と認可の設定
- ユーザー権限とロールの実装
- 本番セキュリティ設定の構成
- セキュリティ問題に対するDjangoアプリケーションのレビュー
- Djangoアプリケーションを本番環境へデプロイ

## コアセキュリティ設定

### 本番設定の構成

```python
# settings/production.py
import os

DEBUG = False  # 重要: 本番では絶対にTrueにしない

ALLOWED_HOSTS = os.environ.get('ALLOWED_HOSTS', '').split(',')

# セキュリティヘッダー
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_HSTS_SECONDS = 31536000  # 1年
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
SECURE_CONTENT_TYPE_NOSNIFF = True
SECURE_BROWSER_XSS_FILTER = True
X_FRAME_OPTIONS = 'DENY'

# HTTPSとCookie
SESSION_COOKIE_HTTPONLY = True
CSRF_COOKIE_HTTPONLY = True
SESSION_COOKIE_SAMESITE = 'Lax'
CSRF_COOKIE_SAMESITE = 'Lax'

# シークレットキー（環境変数で設定する必要がある）
SECRET_KEY = os.environ.get('DJANGO_SECRET_KEY')
if not SECRET_KEY:
    raise ImproperlyConfigured('DJANGO_SECRET_KEY environment variable is required')

# パスワードバリデーション
AUTH_PASSWORD_VALIDATORS = [
    {
        'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
        'OPTIONS': {
            'min_length': 12,
        }
    },
    {
        'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator',
    },
]
```

## 認証

### カスタムユーザーモデル

```python
# apps/users/models.py
from django.contrib.auth.models import AbstractUser
from django.db import models

class User(AbstractUser):
    """セキュリティ強化のためのカスタムユーザーモデル。"""

    email = models.EmailField(unique=True)
    phone = models.CharField(max_length=20, blank=True)

    USERNAME_FIELD = 'email'  # ユーザー名としてメールアドレスを使用
    REQUIRED_FIELDS = ['username']

    class Meta:
        db_table = 'users'
        verbose_name = 'User'
        verbose_name_plural = 'Users'

    def __str__(self):
        return self.email

# settings/base.py
AUTH_USER_MODEL = 'users.User'
```

### パスワードハッシュ化

```python
# DjangoはデフォルトでPBKDF2を使用する。より強力なセキュリティのために:
PASSWORD_HASHERS = [
    'django.contrib.auth.hashers.Argon2PasswordHasher',
    'django.contrib.auth.hashers.PBKDF2PasswordHasher',
    'django.contrib.auth.hashers.PBKDF2SHA1PasswordHasher',
    'django.contrib.auth.hashers.BCryptSHA256PasswordHasher',
]
```

### セッション管理

```python
# セッション設定
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'  # または 'db'
SESSION_CACHE_ALIAS = 'default'
SESSION_COOKIE_AGE = 3600 * 24 * 7  # 1週間
SESSION_SAVE_EVERY_REQUEST = False
SESSION_EXPIRE_AT_BROWSER_CLOSE = False  # UXは良いが、セキュリティは低下
```

## 認可

### パーミッション

```python
# models.py
from django.db import models
from django.contrib.auth.models import Permission

class Post(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    author = models.ForeignKey(User, on_delete=models.CASCADE)

    class Meta:
        permissions = [
            ('can_publish', 'Can publish posts'),
            ('can_edit_others', 'Can edit posts of others'),
        ]

    def user_can_edit(self, user):
        """このユーザーが投稿を編集できるか確認する。"""
        return self.author == user or user.has_perm('app.can_edit_others')

# views.py
from django.contrib.auth.mixins import LoginRequiredMixin, PermissionRequiredMixin
from django.views.generic import UpdateView

class PostUpdateView(LoginRequiredMixin, PermissionRequiredMixin, UpdateView):
    model = Post
    permission_required = 'app.can_edit_others'
    raise_exception = True  # リダイレクトの代わりに403を返す

    def get_queryset(self):
        """ユーザーが自分の投稿のみ編集できるようにする。"""
        return Post.objects.filter(author=self.request.user)
```

### カスタムパーミッション

```python
# permissions.py
from rest_framework import permissions

class IsOwnerOrReadOnly(permissions.BasePermission):
    """オーナーのみがオブジェクトを編集できるようにする。"""

    def has_object_permission(self, request, view, obj):
        # 読み取りパーミッションはすべてのリクエストに許可
        if request.method in permissions.SAFE_METHODS:
            return True

        # 書き込みパーミッションはオーナーのみ
        return obj.author == request.user

class IsAdminOrReadOnly(permissions.BasePermission):
    """管理者はすべてを実行でき、その他は読み取りのみ。"""

    def has_permission(self, request, view):
        if request.method in permissions.SAFE_METHODS:
            return True
        return request.user and request.user.is_staff

class IsVerifiedUser(permissions.BasePermission):
    """認証済みユーザーのみ許可する。"""

    def has_permission(self, request, view):
        return request.user and request.user.is_authenticated and request.user.is_verified
```

### ロールベースアクセス制御（RBAC）

```python
# models.py
from django.contrib.auth.models import AbstractUser, Group

class User(AbstractUser):
    ROLE_CHOICES = [
        ('admin', 'Administrator'),
        ('moderator', 'Moderator'),
        ('user', 'Regular User'),
    ]
    role = models.CharField(max_length=20, choices=ROLE_CHOICES, default='user')

    def is_admin(self):
        return self.role == 'admin' or self.is_superuser

    def is_moderator(self):
        return self.role in ['admin', 'moderator']

# ミックスイン
class AdminRequiredMixin:
    """管理者ロールを要求するミックスイン。"""

    def dispatch(self, request, *args, **kwargs):
        if not request.user.is_authenticated or not request.user.is_admin():
            from django.core.exceptions import PermissionDenied
            raise PermissionDenied
        return super().dispatch(request, *args, **kwargs)
```

## SQLインジェクション防止

### Django ORMによる保護

```python
# GOOD: Django ORMはパラメータを自動的にエスケープする
def get_user(username):
    return User.objects.get(username=username)  # 安全

# GOOD: raw()でパラメータを使用
def search_users(query):
    return User.objects.raw('SELECT * FROM users WHERE username = %s', [query])

# BAD: ユーザー入力を直接補間しない
def get_user_bad(username):
    return User.objects.raw(f'SELECT * FROM users WHERE username = {username}')  # 脆弱！

# GOOD: 適切なエスケープでfilterを使用
def get_users_by_email(email):
    return User.objects.filter(email__iexact=email)  # 安全

# GOOD: 複雑なクエリにQオブジェクトを使用
from django.db.models import Q
def search_users_complex(query):
    return User.objects.filter(
        Q(username__icontains=query) |
        Q(email__icontains=query)
    )  # 安全
```

### raw()を使った追加セキュリティ

```python
# 生のSQLを使わなければならない場合は、常にパラメータを使用する
User.objects.raw(
    'SELECT * FROM users WHERE email = %s AND status = %s',
    [user_input_email, status]
)
```

## XSS防止

### テンプレートエスケープ

```django
{# DjangoはデフォルトでHTMLを自動エスケープする - 安全 #}
{{ user_input }}  {# エスケープされたHTML #}

{# 信頼できるコンテンツのみに明示的にsafeをマークする #}
{{ trusted_html|safe }}  {# エスケープされない #}

{# 安全なHTMLのためにテンプレートフィルターを使用する #}
{{ user_input|escape }}  {# デフォルトと同じ #}
{{ user_input|striptags }}  {# すべてのHTMLタグを除去 #}

{# JavaScriptエスケープ #}
<script>
    var username = {{ username|escapejs }};
</script>
```

### 安全な文字列の処理

```python
from django.utils.safestring import mark_safe
from django.utils.html import escape

# BAD: エスケープなしでユーザー入力をsafeにマークしない
def render_bad(user_input):
    return mark_safe(user_input)  # 脆弱！

# GOOD: 最初にエスケープ、次にsafeをマーク
def render_good(user_input):
    return mark_safe(escape(user_input))

# GOOD: 変数付きHTMLにはformat_htmlを使用
from django.utils.html import format_html

def greet_user(username):
    return format_html('<span class="user">{}</span>', escape(username))
```

### HTTPヘッダー

```python
# settings.py
SECURE_CONTENT_TYPE_NOSNIFF = True  # MIMEスニッフィングを防止
SECURE_BROWSER_XSS_FILTER = True  # XSSフィルターを有効化
X_FRAME_OPTIONS = 'DENY'  # クリックジャッキングを防止

# カスタムミドルウェア
from django.conf import settings

class SecurityHeaderMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        response = self.get_response(request)
        response['X-Content-Type-Options'] = 'nosniff'
        response['X-Frame-Options'] = 'DENY'
        response['X-XSS-Protection'] = '1; mode=block'
        response['Content-Security-Policy'] = "default-src 'self'"
        return response
```

## CSRF保護

### デフォルトのCSRF保護

```python
# settings.py - CSRFはデフォルトで有効
CSRF_COOKIE_SECURE = True  # HTTPSでのみ送信
CSRF_COOKIE_HTTPONLY = True  # JavaScriptのアクセスを防止
CSRF_COOKIE_SAMESITE = 'Lax'  # 一部のケースでCSRFを防止
CSRF_TRUSTED_ORIGINS = ['https://example.com']  # 信頼できるドメイン

# テンプレートの使用
<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">Submit</button>
</form>

# AJAXリクエスト
function getCookie(name) {
    let cookieValue = null;
    if (document.cookie && document.cookie !== '') {
        const cookies = document.cookie.split(';');
        for (let i = 0; i < cookies.length; i++) {
            const cookie = cookies[i].trim();
            if (cookie.substring(0, name.length + 1) === (name + '=')) {
                cookieValue = decodeURIComponent(cookie.substring(name.length + 1));
                break;
            }
        }
    }
    return cookieValue;
}

fetch('/api/endpoint/', {
    method: 'POST',
    headers: {
        'X-CSRFToken': getCookie('csrftoken'),
        'Content-Type': 'application/json',
    },
    body: JSON.stringify(data)
});
```

### ビューの免除（慎重に使用）

```python
from django.views.decorators.csrf import csrf_exempt

@csrf_exempt  # 絶対に必要な場合のみ使用！
def webhook_view(request):
    # 外部サービスからのWebhook
    pass
```

## ファイルアップロードセキュリティ

### ファイルバリデーション

```python
import os
from django.core.exceptions import ValidationError

def validate_file_extension(value):
    """ファイル拡張子を検証する。"""
    ext = os.path.splitext(value.name)[1]
    valid_extensions = ['.jpg', '.jpeg', '.png', '.gif', '.pdf']
    if not ext.lower() in valid_extensions:
        raise ValidationError('Unsupported file extension.')

def validate_file_size(value):
    """ファイルサイズを検証する（最大5MB）。"""
    filesize = value.size
    if filesize > 5 * 1024 * 1024:
        raise ValidationError('File too large. Max size is 5MB.')

# models.py
class Document(models.Model):
    file = models.FileField(
        upload_to='documents/',
        validators=[validate_file_extension, validate_file_size]
    )
```

### 安全なファイルストレージ

```python
# settings.py
MEDIA_ROOT = '/var/www/media/'
MEDIA_URL = '/media/'

# 本番環境ではメディアに別のドメインを使用する
MEDIA_DOMAIN = 'https://media.example.com'

# ユーザーアップロードを直接提供しない
# 静的ファイルにはwhitenoiseまたはCDNを使用する
# メディアファイルには別のサーバーまたはS3を使用する
```

## APIセキュリティ

### レート制限

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle'
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/day',
        'user': '1000/day',
        'upload': '10/hour',
    }
}

# カスタムスロットル
from rest_framework.throttling import UserRateThrottle

class BurstRateThrottle(UserRateThrottle):
    scope = 'burst'
    rate = '60/min'

class SustainedRateThrottle(UserRateThrottle):
    scope = 'sustained'
    rate = '1000/day'
```

### API認証

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}

# views.py
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import IsAuthenticated

@api_view(['GET', 'POST'])
@permission_classes([IsAuthenticated])
def protected_view(request):
    return Response({'message': 'You are authenticated'})
```

## セキュリティヘッダー

### コンテンツセキュリティポリシー

```python
# settings.py
CSP_DEFAULT_SRC = "'self'"
CSP_SCRIPT_SRC = "'self' https://cdn.example.com"
CSP_STYLE_SRC = "'self' 'unsafe-inline'"
CSP_IMG_SRC = "'self' data: https:"
CSP_CONNECT_SRC = "'self' https://api.example.com"

# ミドルウェア
class CSPMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        response = self.get_response(request)
        response['Content-Security-Policy'] = (
            f"default-src {CSP_DEFAULT_SRC}; "
            f"script-src {CSP_SCRIPT_SRC}; "
            f"style-src {CSP_STYLE_SRC}; "
            f"img-src {CSP_IMG_SRC}; "
            f"connect-src {CSP_CONNECT_SRC}"
        )
        return response
```

## 環境変数

### シークレットの管理

```python
# python-decoupleまたはdjango-environを使用する
import environ

env = environ.Env(
    # キャスト、デフォルト値を設定
    DEBUG=(bool, False)
)

# .envファイルを読み込む
environ.Env.read_env()

SECRET_KEY = env('DJANGO_SECRET_KEY')
DATABASE_URL = env('DATABASE_URL')
ALLOWED_HOSTS = env.list('ALLOWED_HOSTS')

# .envファイル（コミットしないこと）
DEBUG=False
SECRET_KEY=your-secret-key-here
DATABASE_URL=postgresql://user:password@localhost:5432/dbname
ALLOWED_HOSTS=example.com,www.example.com
```

## セキュリティイベントのログ記録

```python
# settings.py
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'handlers': {
        'file': {
            'level': 'WARNING',
            'class': 'logging.FileHandler',
            'filename': '/var/log/django/security.log',
        },
        'console': {
            'level': 'INFO',
            'class': 'logging.StreamHandler',
        },
    },
    'loggers': {
        'django.security': {
            'handlers': ['file', 'console'],
            'level': 'WARNING',
            'propagate': True,
        },
        'django.request': {
            'handlers': ['file'],
            'level': 'ERROR',
            'propagate': False,
        },
    },
}
```

## セキュリティクイックチェックリスト

| チェック項目 | 説明 |
|-------|-------------|
| `DEBUG = False` | 本番環境でDEBUGを有効にしない |
| HTTPSのみ | SSLを強制し、Cookieを保護する |
| 強力なシークレット | SECRET_KEYに環境変数を使用する |
| パスワードバリデーション | すべてのパスワードバリデーターを有効にする |
| CSRF保護 | デフォルトで有効、無効にしない |
| XSS防止 | DjangoはHTMLを自動エスケープする、ユーザー入力に `&#124;safe` を使わない |
| SQLインジェクション | ORMを使用し、クエリに文字列を連結しない |
| ファイルアップロード | ファイルタイプとサイズを検証する |
| レート制限 | APIエンドポイントにスロットルを設定する |
| セキュリティヘッダー | CSP、X-Frame-Options、HSTS |
| ログ記録 | セキュリティイベントをログに記録する |
| アップデート | DjangoとDependenciesを最新の状態に保つ |

覚えておいてほしいこと: セキュリティはプロダクトではなくプロセスです。定期的にセキュリティプラクティスをレビューし、更新してください。
