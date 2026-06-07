---
name: windows-desktop-e2e
description: pywinauto と Windows UI Automation を使用した Windows ネイティブデスクトップアプリ（WPF、WinForms、Win32/MFC、Qt）向け E2E テスト。
origin: ECC
---

# Windows デスクトップ E2E テスト

Windows UI Automation（UIA）を基盤とした **pywinauto** を使用した、Windows ネイティブデスクトップアプリケーションのエンドツーエンドテストです。WPF、WinForms、Win32/MFC、Qt（5.x / 6.x）に対応し、Qt 固有のガイダンスを専用セクションとして提供しています。

## 有効化するタイミング

- Windows ネイティブデスクトップアプリケーションの E2E テストを記述・実行する場合
- デスクトップ GUI テストスイートをゼロから構築する場合
- 不安定または失敗しているデスクトップ自動化テストを診断する場合
- 既存アプリにテスト容易性（AutomationId、アクセシブル名）を追加する場合
- GitHub Actions `windows-latest` を使用したデスクトップ E2E の CI/CD パイプライン統合

### 使用しないケース

- Web アプリケーション → `e2e-testing` スキル（Playwright）を使用
- Electron / CEF / WebView2 アプリ → HTML レイヤーにはブラウザ自動化が必要（UIA 不可）
- モバイルアプリ → プラットフォーム固有ツール（UIAutomator、XCUITest）を使用
- 実行中の GUI を必要としない純粋な単体テストまたは統合テスト

## コアコンセプト

すべての Windows デスクトップ自動化は、**UI Automation（UIA）**（Windows 組み込みのアクセシビリティ API）に依存しています。サポートされている各フレームワークは、Claude が読み取り・操作できるプロパティを持つ UIA 要素のツリーを公開します:

```
テスト（Python）
    └── pywinauto（UIA バックエンド）
        └── Windows UI Automation API   ← Windows 組み込み、フレームワーク非依存
            └── アプリの UIA プロバイダー      ← 各フレームワークが独自に実装
                └── 実行中の .exe
```

**フレームワーク別の UIA 品質:**

| フレームワーク | AutomationId | 信頼性 | 備考 |
|-----------|-------------|-------------|-------|
| WPF | ★★★★★ | 優秀 | `x:Name` が AutomationId に直接マップ |
| WinForms | ★★★★☆ | 良好 | `AccessibleName` = AutomationId |
| UWP / WinUI 3 | ★★★★★ | 優秀 | Microsoft が完全サポート |
| Qt 6.x | ★★★★★ | 優秀 | アクセシビリティがデフォルトで有効; クラス名が `Qt6*` に変更 |
| Qt 5.15+ | ★★★★☆ | 良好 | アクセシビリティモジュールが改善 |
| Qt 5.7–5.14 | ★★★☆☆ | 普通 | `QT_ACCESSIBILITY=1` が必要; objectName は手動設定 |
| Win32 / MFC | ★★★☆☆ | 普通 | コントロール ID にアクセス可能; テキストマッチングが一般的 |

## セットアップと前提条件

```bash
# Python 3.8+, Windows のみ
pip install pywinauto pytest pytest-html Pillow pytest-timeout
# オプション: 画面録画
# ffmpeg をインストールして PATH に追加: https://ffmpeg.org/download.html
```

UIA が到達可能か確認:

```python
from pywinauto import Desktop
Desktop(backend="uia").windows()  # すべてのトップレベルウィンドウを一覧表示
```

**Accessibility Insights for Windows**（Microsoft 製、無料）をインストールしてください。テストを記述する前に UIA 要素ツリーを検査するための DevTools 相当ツールです。

## テスト容易性のセットアップ（フレームワーク別）

テスト作成前に最も効果的な施策は、**すべての対話型コントロールに安定した AutomationId を設定する**ことです。

### WPF

```xml
<!-- XAML: x:Name が自動的に AutomationId になる -->
<TextBox x:Name="usernameInput" />
<PasswordBox x:Name="passwordInput" />
<Button x:Name="btnLogin" Content="Login" />
<TextBlock x:Name="lblError" />
```

### WinForms

```csharp
// デザイナーまたはコードで設定
usernameInput.AccessibleName = "usernameInput";
passwordInput.AccessibleName = "passwordInput";
btnLogin.AccessibleName = "btnLogin";
lblError.AccessibleName = "lblError";
```

### Win32 / MFC

```cpp
// .rc ファイルのコントロールリソース ID が AutomationId 文字列として公開される
// IDC_EDIT_USERNAME -> AutomationId "1001"
// Name には SetWindowText を優先; より豊かなサポートには IAccessible を追加
```

### Qt — 以下の専用セクションを参照

---

## ページオブジェクトモデル

```
tests/
├── conftest.py          # アプリ起動フィクスチャ、失敗時スクリーンショット
├── pytest.ini
├── config.py
├── pages/
│   ├── __init__.py      # インポートに必要
│   ├── base_page.py     # ロケーター、待機、スクリーンショットヘルパー
│   ├── login_page.py
│   └── main_page.py
├── tests/
│   ├── __init__.py
│   ├── test_login.py
│   └── test_main_flow.py
└── artifacts/           # スクリーンショット、動画、ログ
```

### base_page.py

```python
import os, time
from pywinauto import Desktop
from config import ACTION_TIMEOUT, ARTIFACT_DIR

class BasePage:
    def __init__(self, window):
        self.window = window

    # --- ロケーター（優先順位順）---

    def by_id(self, auto_id, **kw):
        """AutomationId — 最も安定。最初の選択肢として使用。"""
        return self.window.child_window(auto_id=auto_id, **kw)

    def by_name(self, name, **kw):
        """表示テキスト / アクセシブル名。"""
        return self.window.child_window(title=name, **kw)

    def by_class(self, cls, index=0, **kw):
        """コントロールクラス + インデックス — 脆弱なので可能な限り避ける。"""
        return self.window.child_window(class_name=cls, found_index=index, **kw)

    # --- 待機 ---

    def wait_visible(self, spec, timeout=ACTION_TIMEOUT):
        spec.wait("visible", timeout=timeout)
        return spec

    def wait_gone(self, spec, timeout=ACTION_TIMEOUT):
        spec.wait_not("visible", timeout=timeout)
        return spec

    def wait_window(self, title, timeout=ACTION_TIMEOUT):
        """新しいトップレベルウィンドウ（ダイアログ、子ウィンドウ）を待機。"""
        dlg = Desktop(backend="uia").window(title=title)
        dlg.wait("visible", timeout=timeout)
        return dlg

    def wait_until(self, fn, timeout=ACTION_TIMEOUT, interval=0.3):
        """任意の条件をポーリング — UIA イベントが不安定な場合に使用。"""
        deadline = time.time() + timeout
        while time.time() < deadline:
            try:
                if fn():
                    return True
            except Exception:
                pass
            time.sleep(interval)
        raise TimeoutError(f"Condition not met within {timeout}s")

    # --- アクション ---

    def click(self, spec):
        self.wait_visible(spec)
        spec.click_input()

    def type_text(self, spec, text):
        self.wait_visible(spec)
        ctrl = spec.wrapper_object()
        try:
            ctrl.set_edit_text(text)
        except Exception as e:
            # Qt 5.x フォールバック: UIA Value Pattern が不完全な場合がある
            import sys, pywinauto.keyboard as kb
            print(f"[windows-desktop-e2e] set_edit_text failed ({e}), using keyboard fallback", file=sys.stderr)
            ctrl.click_input()
            kb.send_keys("^a")
            kb.send_keys(text, with_spaces=True)

    def get_text(self, spec):
        ctrl = spec.wrapper_object()
        for attr in ("window_text", "get_value"):
            try:
                v = getattr(ctrl, attr)()
                if v:
                    return v
            except Exception:
                pass
        return ""

    # --- アーティファクト ---

    def screenshot(self, name):
        os.makedirs(ARTIFACT_DIR, exist_ok=True)
        path = os.path.join(ARTIFACT_DIR, f"{name}.png")
        self.window.capture_as_image().save(path)
        return path
```

### login_page.py

```python
from pages.base_page import BasePage

class LoginPage(BasePage):
    @property
    def username(self): return self.by_id("usernameInput")

    @property
    def password(self): return self.by_id("passwordInput")

    @property
    def btn_login(self): return self.by_id("btnLogin")

    @property
    def error_label(self): return self.by_id("lblError")

    def login(self, user, pwd):
        self.type_text(self.username, user)
        self.type_text(self.password, pwd)
        self.click(self.btn_login)

    def login_ok(self, user, pwd, main_title="Main Window"):
        self.login(user, pwd)
        return self.wait_window(main_title)

    def login_fail(self, user, pwd):
        self.login(user, pwd)
        self.wait_visible(self.error_label)
        return self.get_text(self.error_label)
```

### conftest.py

> 新規プロジェクトでは**ティア 1 サンドボックスフィクスチャ**（下記参照）を推奨します — ゼロコストでファイルシステム分離を追加できます。このベーシックフィクスチャは最小構成またはレガシー用途向けです。

```python
import os, pytest
os.environ["QT_ACCESSIBILITY"] = "1"  # Qt 5.x UIA サポートに必要

from pywinauto import Application
from config import APP_PATH, MAIN_WINDOW_TITLE, LAUNCH_TIMEOUT, ARTIFACT_DIR

@pytest.fixture
def app(request):
    if not APP_PATH:
        pytest.exit("APP_PATH environment variable is not set", returncode=1)
    proc = Application(backend="uia").start(APP_PATH, timeout=LAUNCH_TIMEOUT)
    win  = proc.window(title=MAIN_WINDOW_TITLE)
    win.wait("visible", timeout=LAUNCH_TIMEOUT)
    yield win
    # 失敗時にスクリーンショット
    if getattr(getattr(request.node, "rep_call", None), "failed", False):
        os.makedirs(ARTIFACT_DIR, exist_ok=True)
        try:
            win.capture_as_image().save(
                os.path.join(ARTIFACT_DIR, f"FAIL_{request.node.name}.png")
            )
        except Exception:
            pass
    # まずグレースフル終了し、失敗時はフォースキル
    # proc は pywinauto Application — wait_for_process() ではなく wait_for_process_exit() を使用
    try:
        win.close()
        proc.wait_for_process_exit(timeout=5)
    except Exception:
        proc.kill()

@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    outcome = yield
    setattr(item, f"rep_{outcome.get_result().when}", outcome.get_result())
```

### config.py

```python
import os
APP_PATH          = os.environ.get("APP_PATH", "")           # 環境変数で設定 — デフォルトパスなし
MAIN_WINDOW_TITLE = os.environ.get("APP_TITLE", "")
LAUNCH_TIMEOUT    = int(os.environ.get("LAUNCH_TIMEOUT", "15"))
ACTION_TIMEOUT    = int(os.environ.get("ACTION_TIMEOUT", "10"))
ARTIFACT_DIR      = os.path.join(os.path.dirname(__file__), "artifacts")
```

### pytest.ini

```ini
[pytest]
testpaths = tests
markers =
    smoke: fast smoke tests for critical paths
    flaky: known-unstable tests
addopts = -v --tb=short --html=artifacts/report.html --self-contained-html
```

## ロケーター戦略

```
AutomationId  >  Name（テキスト）  >  ClassName + index  >  XPath
  （安定）         （可読性高）         （脆弱）           （最終手段）
```

Accessibility Insights で検査 → **プロパティ** ペイン → まず `AutomationId` を確認。

```python
# 実行時に検査 — ツリーを探索するために REPL に貼り付ける
win.print_control_identifiers()
# またはスコープを絞る:
win.child_window(auto_id="groupBox1").print_control_identifiers()
```

## 待機パターン

```python
# コントロールの出現を待機
page.wait_visible(page.by_id("statusLabel"))

# コントロールの消去を待機（例: ローディングスピナー）
page.wait_gone(page.by_id("spinnerOverlay"))

# ダイアログのポップアップを待機
dlg = page.wait_window("Confirm Delete")

# カスタム条件（例: テキスト変更）
page.wait_until(lambda: page.get_text(page.by_id("lblStatus")) == "Ready")
```

**プライマリ同期として `time.sleep()` を使用しないでください** — `wait()` または `wait_until()` を使用。

## アーティファクト管理

```python
# オンデマンドスクリーンショット
page.screenshot("after_login")

# フルスクリーンキャプチャ（ウィンドウが画面外または最小化時）
import pyautogui
pyautogui.screenshot("artifacts/fullscreen.png")

# ffmpeg による画面録画（テスト前に開始し、テスト後に停止）
import subprocess

def start_recording(name):
    return subprocess.Popen([
        "ffmpeg", "-f", "gdigrab", "-framerate", "10",
        "-i", "desktop", "-y", f"artifacts/videos/{name}.mp4"
    ], stdin=subprocess.PIPE, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)

def stop_recording(proc):
    proc.stdin.write(b"q"); proc.stdin.flush(); proc.wait(timeout=10)
```

## ステップごとのトレース（オプトイン）

デフォルトの失敗スクリーンショットは、不安定なテストの診断に不十分な場合があります。以下のステップレベルトレースは**デフォルトでオフ**です — 不安定なケースを再現する場合にのみ有効化してください。

### 有効化

```bash
E2E_TRACE=1 pytest tests/test_login.py -v
# 入力テキストを JSONL ログに含める（認証情報/PII を入力するテストでは使用禁止）:
E2E_TRACE=1 E2E_TRACE_INCLUDE_TEXT=1 pytest ...
```

### BasePage へのパッチ適用

```python
import os, json, time
TRACE_ENABLED      = os.environ.get("E2E_TRACE") == "1"
TRACE_INCLUDE_TEXT = os.environ.get("E2E_TRACE_INCLUDE_TEXT") == "1"

class BasePage:
    _step = 0

    def _trace(self, action, spec=None, text=None):
        if not TRACE_ENABLED:
            return
        BasePage._step += 1
        idx = f"{BasePage._step:03d}"
        os.makedirs(ARTIFACT_DIR, exist_ok=True)
        try:
            self.window.capture_as_image().save(
                os.path.join(ARTIFACT_DIR, f"step_{idx}_{action}.png"))
        except Exception:
            pass  # キャプチャ失敗でテストを中断しない
        rec = {
            "ts": time.time(), "step": BasePage._step, "action": action,
            "locator": getattr(spec, "criteria", None),
            "text": text if TRACE_INCLUDE_TEXT else ("<redacted>" if text else None),
        }
        with open(os.path.join(ARTIFACT_DIR, "trace.jsonl"), "a") as f:
            f.write(json.dumps(rec) + "\n")

    def click(self, spec):
        self.wait_visible(spec); self._trace("click_before", spec)
        spec.click_input();      self._trace("click_after",  spec)

    def type_text(self, spec, text):
        self.wait_visible(spec); self._trace("type_before", spec, text)
        # ... 既存の set_edit_text / キーボードフォールバック ...
        self._trace("type_after", spec)
```

### 注意事項

- **PII / 認証情報**: `type_text` の内容はデフォルトで `<redacted>` になります。ログインや決済フローで `E2E_TRACE_INCLUDE_TEXT=1` を設定しないでください。
- **オーバーヘッド**: アクションごとに約 50〜200ms + ディスクにステップごと 1 枚の PNG。デフォルトの CI マトリクスでは有効化しないでください — 専用の不安定再現ジョブのみで使用してください。
- **アーティファクト膨張**: 長いフローでは数十 MB が生成されます; `retention-days` を適切に調整してください。
- **並列/再実行時の衛生**: このシンプルな例は `trace.jsonl` に追記し、クラスレベルのカウンターを使用します。再実行前にアーティファクトディレクトリをクリアし、並列テストにはワーカーごとのアーティファクトディレクトリを使用してください。
- **カバレッジギャップ**: `BasePage` 外で実行されるアクション（テストコード内の生の `pywinauto` 呼び出し）はトレースされません。

## 不安定なテストの対処

```python
# 隔離 — Playwright の test.fixme() に相当
@pytest.mark.skip(reason="Flaky: animation race on slow CI. Issue #42")
def test_animated_transition(self, app): ...

# CI のみスキップ
@pytest.mark.skipif(os.environ.get("CI") == "true", reason="Flaky in CI #43")
def test_heavy_load(self, app): ...
```

一般的な原因と修正方法:

| 原因 | 修正 |
|-------|-----|
| コントロールの準備未完了 | `time.sleep` を `wait_visible` に置換 |
| ウィンドウがフォーカスされていない | 操作前に `win.set_focus()` を追加 |
| アニメーション処理中 | `wait_until(lambda: not loading_indicator.exists())` |
| ダイアログのタイミング | `wait_window(title, timeout=15)` |
| CI のディスプレイが未準備 | `DISPLAY` を設定するか CI で仮想デスクトップを使用 |
| `set_edit_text` が NotImplementedError を発生 | UIA ValuePattern が欠落（Qt 5.x で一般的）— `BasePage.type_text` がすでに `keyboard.send_keys` にフォールバック |
| コントロールは存在するが `wait_visible` がタイムアウト | ウィンドウが最小化または画面外 — 待機前に `win.restore()` + `win.set_focus()` を呼び出す |

## テスト分離とサンドボックス

分離の 3 段階 — ニーズを満たす最も軽量な段階を使用してください。

### ティア 1 — ファイルシステム分離（デフォルト、常に使用）

各テストは `subprocess.Popen` と `Application.connect()` を介して独自の `APPDATA` / `LOCALAPPDATA` / `TEMP` を取得します。pytest の `tmp_path` フィクスチャがクリーンアップを自動処理します。

```python
# conftest.py — ベーシックな `app` フィクスチャをこれで置き換え
import os, subprocess, pytest
from pywinauto import Application
from config import APP_PATH, APP_ARGS, APP_TITLE, LAUNCH_TIMEOUT, ACTION_TIMEOUT, ARTIFACT_DIR

@pytest.fixture(scope="function")
def app(request, tmp_path):
    """テストごとに新鮮なプロセス + 分離されたユーザーデータディレクトリ。"""
    if not APP_PATH:
        pytest.exit("APP_PATH not set", returncode=1)

    # すべてのユーザーごとのストレージを分離された tmp ディレクトリにリダイレクト
    sandbox_env = os.environ.copy()
    sandbox_env["QT_ACCESSIBILITY"]  = "1"
    sandbox_env["APPDATA"]           = str(tmp_path / "AppData" / "Roaming")
    sandbox_env["LOCALAPPDATA"]      = str(tmp_path / "AppData" / "Local")
    sandbox_env["TEMP"] = sandbox_env["TMP"] = str(tmp_path / "Temp")
    for p in (sandbox_env["APPDATA"], sandbox_env["LOCALAPPDATA"], sandbox_env["TEMP"]):
        os.makedirs(p, exist_ok=True)

    if not APP_TITLE:
        pytest.exit("APP_TITLE environment variable is not set", returncode=1)

    # shlex.split はスペースを含む引用符付き引数を処理; plain split() では壊れる
    import shlex
    # subprocess 経由で起動し env を渡せるようにする; pywinauto を PID で接続
    proc = subprocess.Popen(
        [APP_PATH] + shlex.split(APP_ARGS),
        env=sandbox_env,
    )
    pw_app = Application(backend="uia").connect(process=proc.pid, timeout=LAUNCH_TIMEOUT)
    win    = pw_app.window(title=APP_TITLE)
    win.wait("visible", timeout=LAUNCH_TIMEOUT)
    yield win

    if getattr(getattr(request.node, "rep_call", None), "failed", False):
        os.makedirs(ARTIFACT_DIR, exist_ok=True)
        try:
            win.capture_as_image().save(
                os.path.join(ARTIFACT_DIR, f"FAIL_{request.node.name}.png")
            )
        except Exception:
            pass
    try:
        win.close()
        proc.wait(timeout=5)
    except Exception:
        proc.kill()
    # tmp_path は pytest が自動的にクリーンアップ

@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    outcome = yield
    setattr(item, f"rep_{outcome.get_result().when}", outcome.get_result())
```

### ティア 2 — Windows ジョブオブジェクト（オプション: プロセスライフタイムの封じ込め）

プロセスをジョブオブジェクトに割り当て、テストフィクスチャのジョブハンドルが GC された際に**自動的に終了**させます。また、フィクスチャのクリーンアップから逃れる子プロセスをアプリが生成するのを防ぎます。

> **分離の範囲:** ジョブオブジェクトはファイルシステムアクセスを仮想化したり、ネットワークトラフィックをブロックしたりしません。ファイル書き込みとネットワーク分離には AppContainer、Windows ファイアウォールルール、またはティア 3（Windows Sandbox）が必要です。ティア 2 はプロセスライフタイムと子プロセスの封じ込めのみに使用してください。

追加の依存関係は不要です。

```python
import ctypes, ctypes.wintypes as wt

def restrict_process(pid: int):
    """
    プロセスをジョブオブジェクトに割り当て、以下を防止する:
    - ジョブ外へのプロセス生成（LIMIT_KILL_ON_JOB_CLOSE）
    ネットワークはブロックしない — それには Windows ファイアウォールルールを使用。
    """
    JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE = 0x00002000
    # 最小権限: SET_QUOTA (0x0100) | TERMINATE (0x0001)
    PROCESS_SET_QUOTA_AND_TERMINATE    = 0x0101

    kernel32 = ctypes.windll.kernel32
    job   = kernel32.CreateJobObjectW(None, None)
    hproc = kernel32.OpenProcess(PROCESS_SET_QUOTA_AND_TERMINATE, False, pid)

    # 正しい構造体レイアウト — LimitFlags はオフセット +16、+44 ではない
    class JOBOBJECT_BASIC_LIMIT_INFORMATION(ctypes.Structure):
        _fields_ = [
            ("PerProcessUserTimeLimit", wt.LARGE_INTEGER),
            ("PerJobUserTimeLimit",     wt.LARGE_INTEGER),
            ("LimitFlags",             wt.DWORD),
            ("MinimumWorkingSetSize",   ctypes.c_size_t),
            ("MaximumWorkingSetSize",   ctypes.c_size_t),
            ("ActiveProcessLimit",      wt.DWORD),
            ("Affinity",               ctypes.c_size_t),
            ("PriorityClass",          wt.DWORD),
            ("SchedulingClass",        wt.DWORD),
        ]

    info = JOBOBJECT_BASIC_LIMIT_INFORMATION()
    info.LimitFlags = JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE
    ok = kernel32.SetInformationJobObject(job, 2, ctypes.byref(info), ctypes.sizeof(info))
    if not ok:
        raise ctypes.WinError()
    kernel32.AssignProcessToJobObject(job, hproc)
    kernel32.CloseHandle(hproc)
    return job  # 生存を維持 — GC 時にジョブが閉じてプロセスを終了
```

### ティア 3 — Windows Sandbox（CI 完全 OS 分離）

実行ごとにクリーンな Windows イメージが必要な場合（残存レジストリキーなし、共有 GPU 状態なし、真の分離）、[Windows Sandbox](https://learn.microsoft.com/windows/security/application-security/application-isolation/windows-sandbox/windows-sandbox-overview) 内で**テストスイート全体**を実行してください。

**要件:** Windows 10/11 Pro または Enterprise、仮想化が有効であること。

プロジェクトルートに `e2e-sandbox.wsb` を作成:

```xml
<Configuration>
  <MappedFolders>
    <!-- アプリバイナリ（読み取り専用）-->
    <MappedFolder>
      <HostFolder>C:\path\to\your\build\Release</HostFolder>
      <SandboxFolder>C:\app</SandboxFolder>
      <ReadOnly>true</ReadOnly>
    </MappedFolder>
    <!-- テストスイート（アーティファクト用に読み書き可能）-->
    <MappedFolder>
      <HostFolder>C:\path\to\your\e2e_test</HostFolder>
      <SandboxFolder>C:\e2e_test</SandboxFolder>
      <ReadOnly>false</ReadOnly>
    </MappedFolder>
  </MappedFolders>
  <LogonCommand>
    <!--
      Windows Sandbox は Python なしで起動する。まずサイレントインストールし、
      その後依存関係をインストールしてテストを実行する。アーティファクトは
      上記のマップドフォルダー経由でホストに書き戻される。
    -->
    <Command>powershell -Command "
      winget install --id Python.Python.3.11 --silent --accept-package-agreements;
      $env:PATH += ';' + $env:LOCALAPPDATA + '\Programs\Python\Python311\Scripts';
      cd C:\e2e_test;
      pip install -r requirements.txt;
      pytest tests\ -v
    "</Command>
  </LogonCommand>
</Configuration>
```

起動: `WindowsSandbox.exe e2e-sandbox.wsb`

> pywinauto とアプリは両方ともサンドボックス**内部**で実行されます（同一セッションが必要）。
> アーティファクトはマップドフォルダー経由でホストに書き戻されます。

### ティアの比較

| ティア | 分離 | セットアップコスト | CI で動作 | 使用タイミング |
|------|-----------|-----------|-------------|----------|
| 1 — `tmp_path` 環境リダイレクト | ファイルシステム | ゼロ | 常時 | 全テストのデフォルト |
| 2 — ジョブオブジェクト | プロセスツリー | 低 | 常時 | 子プロセスの逸脱防止 |
| 3 — Windows Sandbox | 完全 OS | 中 | Pro/Enterprise イメージが必要 | ナイトリーのクリーンルーム実行 |

### ハングテストの防止

任意のテストを上限付きにするために `pytest-timeout` を追加してください。`pytest.ini` で `timeout = 60` および `timeout_method = thread` を設定してください。注意: `thread` メソッドは Windows 上の Qt アプリサブプロセスを終了できません — 孤立プロセスを回収するために `conftest.py` に `atexit.register(lambda: [p.kill() for p in psutil.Process().children(recursive=True)])` を追加してください。

## CI/CD 統合

```yaml
# .github/workflows/e2e-desktop.yml
name: Desktop E2E
on: [push, pull_request]

jobs:
  e2e:
    runs-on: windows-latest   # 本物の GUI 環境、Xvfb 不要
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with: { python-version: "3.11" }

      - name: Install deps
        run: pip install pywinauto pytest pytest-html Pillow

      - name: Build app
        run: cmake --build build --config Release  # ビルドシステムに合わせて調整

      - name: Run E2E
        env:
          APP_PATH: ${{ github.workspace }}\build\Release\MyApp.exe
          APP_TITLE: "My Application"
          CI: "true"
        run: pytest tests/ --html=artifacts/report.html --self-contained-html --junitxml=artifacts/results.xml -v

      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: e2e-artifacts
          path: artifacts/
          retention-days: 14
```

## Qt 固有

### Qt 5.x での UIA の有効化

Qt 5.x のアクセシビリティは一部のビルド（特に 5.7〜5.14）でデフォルト無効になっています。起動前に環境変数を設定してください。Qt 6.x ではアクセシビリティがデフォルトで有効 — Qt 6 ではこの手順をスキップしてください。

```python
# conftest.py — モジュールの先頭に追加
import os
os.environ["QT_ACCESSIBILITY"] = "1"
```

または CI で export:

```yaml
env:
  QT_ACCESSIBILITY: "1"
```

### Qt ウィジェットへの安定した識別子の追加

```cpp
// 推奨: objectName と accessibleName の両方
void setTestId(QWidget* w, const char* id) {
    w->setObjectName(id);
    w->setAccessibleName(id);  // UIA Name プロパティになる
}

// ダイアログコンストラクタ内:
setTestId(ui->usernameEdit, "usernameInput");
setTestId(ui->passwordEdit, "passwordInput");
setTestId(ui->loginButton,  "btnLogin");
setTestId(ui->errorLabel,   "lblError");
```

タイプミスを防ぐためにすべての ID をヘッダーに集中管理:

```cpp
// test_ids.h
#define TID_USERNAME   "usernameInput"
#define TID_PASSWORD   "passwordInput"
#define TID_BTN_LOGIN  "btnLogin"
#define TID_LBL_ERROR  "lblError"
```

### Qt 固有の注意点

**QComboBox** — ドロップダウンは別のトップレベルウィンドウ:

```python
from pywinauto import Desktop

def select_combo_item(page, combo_spec, item_text):
    page.click(combo_spec)
    # ドロップダウンは新しいルートレベルウィンドウとして表示される
    # class_name は Qt バージョンによって異なる — Accessibility Insights で確認
    # Qt 5.x: "Qt5QWindowIcon"  |  Qt 6.x: "Qt6QWindowIcon" — Accessibility Insights で確認
    popup = Desktop(backend="uia").window(class_name_re="Qt[56]QWindowIcon")
    popup.wait("visible", timeout=5)
    popup.child_window(title=item_text).click_input()
```

**QMessageBox / QDialog** — これらも別のトップレベルウィンドウ:

```python
dlg = page.wait_window("Confirm")          # ダイアログタイトルを待機
dlg.child_window(title="OK").click_input() # 内部のボタンをクリック
```

**QTableWidget / QTableView** — 行/セルアクセス:

```python
table = page.by_id("tblUsers").wrapper_object()
cell  = table.cell(row=0, column=1)
print(cell.window_text())
```

**自己描画コントロール**（`paintEvent` のみ、`QGraphicsView`、`QOpenGLWidget`）— UIA はその内部を認識できません。以下のフォールバックセクションを参照してください。

## フォールバック: スクリーンショットモード

UIA 経由でコントロールに到達できない場合（自己描画、サードパーティ、ゲームエンジン）:

```bash
pip install pyautogui Pillow opencv-python
```

```python
import pyautogui, cv2, numpy as np
from PIL import Image

def find_image_on_screen(template_path, confidence=0.85):
    """画面上でテンプレート画像を検索。(x, y) 中心座標または None を返す。"""
    screen   = np.array(pyautogui.screenshot())
    template = np.array(Image.open(template_path))
    result   = cv2.matchTemplate(
        cv2.cvtColor(screen, cv2.COLOR_RGB2BGR),
        cv2.cvtColor(template, cv2.COLOR_RGB2BGR),
        cv2.TM_CCOEFF_NORMED,
    )
    _, max_val, _, max_loc = cv2.minMaxLoc(result)
    if max_val >= confidence:
        h, w = template.shape[:2]
        return max_loc[0] + w // 2, max_loc[1] + h // 2
    return None

def click_image(template_path, confidence=0.85):
    pos = find_image_on_screen(template_path, confidence)
    if pos is None:
        raise RuntimeError(f"Image not found on screen: {template_path}")
    pyautogui.click(*pos)
```

### DPI / スケーリングルール（スクリーンショットモードのみ）

スクリーンショットマッチングは Windows の表示スケーリング（100% / 125% / 150%）に非常に敏感です。3 つの厳格なルール:

1. **対象マシンと同じスケールでテンプレートをキャプチャ。** `PIL.Image.resize` でミスマッチを修正しようとしないでください — `cv2.matchTemplate` はリサンプリングアーティファクトに非常に脆弱です。
2. **CI の表示スケーリングを固定。** `windows-latest` では `Set-DisplayResolution 1920 1080 -Force` のようなステップを追加し、モニターごとの DPI スケーリングを無効にして、スクリーンショットの寸法を再現可能にしてください。
3. **各アーティファクトと一緒にスケールを記録。** キャプチャ時に `GetDpiForWindow(hwnd) / 96` を `artifacts/<test>/metadata.json` に書き込む — 事後分析が推測ではなく明確になります。

> プロセスレベルの DPI 認識（`SetProcessDpiAwarenessContext`）は、テスト対象のアプリが Qt ベースの場合に **Qt 独自の DPI 処理と競合する可能性があります**。フィクスチャでプロセス全体の DPI モードを変更するよりも、「同一スケールのテンプレート + CI 固定」を優先してください。

### マッチ信頼度のデバッグ

`confidence` 閾値を調整する際、唯一の合理的なワークフローはマッチした場所を**視覚的に確認する**ことです。以下のヘルパーは診断専用です — テストコードからは呼び出さないでください。

```python
def debug_match(template_path, out="artifacts/match_debug.png", confidence=0.85):
    """診断専用。現在の画面上にベストマッチの矩形とスコアを描画。

    本番テスト用ではない — 信頼度の調整やフォールスマッチの追跡時に使用。
    """
    import os, cv2, pyautogui, numpy as np
    screen = np.array(pyautogui.screenshot())[:, :, ::-1]
    tpl    = cv2.imread(template_path)
    if tpl is None:
        raise RuntimeError(f"Template unreadable: {template_path}")
    res    = cv2.matchTemplate(screen, tpl, cv2.TM_CCOEFF_NORMED)
    _, mv, _, ml = cv2.minMaxLoc(res)
    h, w   = tpl.shape[:2]
    colour = (0, 255, 0) if mv >= confidence else (0, 0, 255)  # 緑: 合格 / 赤: 失敗
    cv2.rectangle(screen, ml, (ml[0]+w, ml[1]+h), colour, 2)
    cv2.putText(screen, f"score={mv:.3f} thr={confidence}",
                (ml[0], max(20, ml[1]-6)),
                cv2.FONT_HERSHEY_SIMPLEX, 0.7, colour, 2)
    os.makedirs(os.path.dirname(out) or ".", exist_ok=True)
    cv2.imwrite(out, screen)
    return mv
```

**使用は控えめに** — 画像マッチングは DPI 変更、テーマ切り替え、部分的な遮蔽で壊れます。
常に UIA を先に試みてください; 真に到達不能なコントロールのみスクリーンショットにフォールバックしてください。

## アンチパターン

```python
# 悪い例: 固定スリープ
time.sleep(3)
page.click(page.by_id("btnSubmit"))

# 良い例: 条件待機
page.wait_visible(page.by_id("btnSubmit"))
page.click(page.by_id("btnSubmit"))
```

```python
# 悪い例: 脆弱なクラス+インデックスロケーターをプライマリ戦略として使用
page.by_class("Edit", index=2).type_keys("hello")

# 良い例: AutomationId
page.by_id("usernameInput").set_edit_text("hello")
```

```python
# 悪い例: ピクセル座標でアサート
assert btn.rectangle().left == 120

# 良い例: コンテンツ/状態でアサート
assert page.get_text(page.by_id("lblStatus")) == "Logged in"
assert page.by_id("btnLogout").is_enabled()
```

```python
# 悪い例: 全テストでアプリインスタンスを共有（状態リーク）
@pytest.fixture(scope="session")
def app(): ...

# 良い例: テストごとに新鮮なプロセス（またはせいぜいクラス単位）
@pytest.fixture(scope="function")
def app(): ...
```

## テストの実行

```bash
# 全テスト
pytest tests/ -v

# スモークのみ
pytest tests/ -m smoke -v

# 特定ファイル
pytest tests/test_login.py -v

# カスタムアプリパスで実行
APP_PATH="C:\build\Release\MyApp.exe" APP_TITLE="MyApp" pytest tests/ -v

# 不安定なテストを検出（各 5 回繰り返し）
pip install pytest-repeat
pytest tests/test_login.py --count=5 -v
```

## 関連スキル

- `e2e-testing` — Web アプリケーション向け Playwright E2E
- `cpp-testing` — GoogleTest を使用した C++ 単体/統合テスト
- `cpp-coding-standards` — C++ コードスタイルとパターン
