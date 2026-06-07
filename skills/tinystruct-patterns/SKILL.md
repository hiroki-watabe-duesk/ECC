---
name: tinystruct-patterns
description: tinystruct Java フレームワークを使った開発のエキスパートガイダンス。tinystruct コードベースまたは tinystruct で構築されたプロジェクト（Application クラスの作成、@Action マッピングルート、単体テスト、ActionRegistry、HTTP/CLI デュアルモード処理、組み込み HTTP サーバー、イベントシステム、Builder/Builders を使った JSON、AbstractData によるデータベース永続化、POJO 生成、Server-Sent Events（SSE）、ファイルアップロード、アウトバウンド HTTP ネットワーキングを含む）の作業時に使用する。
origin: ECC
---

# tinystruct 開発パターン

**tinystruct** Java フレームワーク — CLI と HTTP を対等な市民として扱い、`main()` メソッドが不要で最小限の設定で済む軽量かつ高性能なフレームワーク — でモジュールを構築するためのアーキテクチャと実装パターン。

## 基本原則

**CLI と HTTP は対等な市民です。** `@Action` でアノテーションされたすべてのメソッドは、理想的には変更なしにターミナルとウェブブラウザの両方から実行できるべきです。この「デュアルモード」機能が tinystruct の核となる設計哲学です。

## 有効化のタイミング

### 使用するケース

- `AbstractApplication` を継承して新しい `Application` モジュールを作成する。
- `@Action` を使ってルートとコマンドラインアクションを定義する。
- `Context` を介してリクエストごとの状態を処理する。
- ネイティブの `Builder` と `Builders` コンポーネントを使って JSON シリアライゼーションを行う。
- `AbstractData` POJO を介してデータベース永続化を行う。
- `generate` コマンドを使ってデータベーステーブルから POJO を生成する。
- リアルタイムプッシュのための Server-Sent Events（SSE）を実装する。
- マルチパートデータを介したファイルアップロードを処理する。
- `URLRequest` と `HTTPHandler` でアウトバウンド HTTP リクエストを行う。
- `application.properties` でデータベース接続またはシステム設定を構成する。
- ルーティングの競合（アクション）または CLI 引数の解析をデバッグする。

## 仕組み

tinystruct フレームワークは `@Action` でアノテーションされたすべてのメソッドを、ターミナルと Web 環境の両方でルーティング可能なエンドポイントとして扱います。アプリケーションはコアライフサイクルフック（`init()` など）とリクエスト `Context` へのアクセスを提供する `AbstractApplication` を継承して作成します。

ルーティングは `ActionRegistry` が処理し、パスセグメントをメソッド引数に自動的にマッピングして依存関係を注入します。データのみのサービスでは、ゼロ依存のフットプリントを維持するために JSON シリアライゼーションにネイティブの `Builder` と `Builders` コンポーネントを使用すべきです。データベースレイヤーは、外部 ORM ライブラリなしで CRUD 操作を行うために XML マッピングファイルと対になった `AbstractData` POJO を使用します。

## 使用例

### 基本的なアプリケーション（MyService）
```java
public class MyService extends AbstractApplication {
    @Override
    public void init() {
        this.setTemplateRequired(false); // Disable .view lookup for data/API apps
    }

    @Override public String version() { return "1.0.0"; }

    @Action("greet")
    public String greet() {
        return "Hello from tinystruct!";
    }

    // Path parameter: GET /?q=greet/James  OR  bin/dispatcher greet/James
    @Action("greet")
    public String greet(String name) {
        return "Hello, " + name + "!";
    }
}
```

### HTTP モードの識別（login）
```java
@Action(value = "login", mode = Mode.HTTP_POST)
public String doLogin(Request<?, ?> request) throws ApplicationException {
    request.getSession().setAttribute("userId", "42");
    return "Logged in";
}
```

### ネイティブ JSON データ処理（Builder + Builders）
```java
import org.tinystruct.data.component.Builder;
import org.tinystruct.data.component.Builders;

@Action("api/data")
public String getData() throws ApplicationException {
    Builders dataList = new Builders();
    Builder item = new Builder();
    item.put("id", 1);
    item.put("name", "James");
    dataList.add(item);

    Builder response = new Builder();
    response.put("status", "success");
    response.put("data", dataList);
    return response.toString(); // {"status":"success","data":[{"id":1,"name":"James"}]}
}
```

### SSE（Server-Sent Events）
```java
import org.tinystruct.http.SSEPushManager;

@Action("sse/connect")
public String connect() {
    return "{\"type\":\"connect\",\"message\":\"Connected to SSE\"}";
}

// Push to a specific client
String sessionId = getContext().getId();
Builder msg = new Builder();
msg.put("text", "Hello, user!");
SSEPushManager.getInstance().push(sessionId, msg);

// Broadcast to all
// Broadcast to all
SSEPushManager.getInstance().broadcast(msg);
```

### ファイルアップロード
```java
import org.tinystruct.data.FileEntity;

@Action(value = "upload", mode = Mode.HTTP_POST)
public String upload(Request<?, ?> request) throws ApplicationException {
    List<FileEntity> files = request.getAttachments();
    if (files != null) {
        for (FileEntity file : files) {
            System.out.println("Uploaded: " + file.getFilename());
        }
    }
    return "Upload OK";
}
```

## 設定

設定は `src/main/resources/application.properties` で管理される。

```properties
# Database
driver=org.h2.Driver
database.url=jdbc:h2:~/mydb
database.user=sa
database.password=

# Server
default.home.page=hello
server.port=8080

# Locale
default.language=en_US

# Session (Redis for clustered environments)
# default.session.repository=org.tinystruct.http.RedisSessionRepository
# redis.host=127.0.0.1
# redis.port=6379
```

アプリケーション内で設定値にアクセスする:
```java
String port = this.getConfiguration("server.port");
```

## レッドフラグとアンチパターン

| 症状 | 正しいパターン |
|---|---|
| `com.google.gson` または `com.fasterxml.jackson` のインポート | `org.tinystruct.data.component.Builder` / `Builders` を使用する。 |
| JSON 配列に `List<Builder>` を使用 | 総称型消去の問題を避けるために `Builders` を使用する。 |
| `ApplicationRuntimeException: template not found` | API のみのアプリでは `init()` 内で `setTemplateRequired(false)` を呼び出す。 |
| `private` メソッドに `@Action` をアノテート | アクションはフレームワークによって登録されるために `public` でなければならない。 |
| アプリに `main(String[] args)` をハードコーディング | すべてのモジュールのエントリポイントとして `bin/dispatcher` を使用する。 |
| 手動での `ActionRegistry` 登録 | 自動検出のために `@Action` アノテーションを優先する。 |
| ランタイムでアクションが見つからない | `--import` でクラスがインポートされているか `application.properties` にリストされていることを確認する。 |
| CLI 引数が見えない | `--key value` で渡し、`getContext().getAttribute("--key")` でアクセスする。 |
| 同じパスの 2 つのメソッドで間違った方が呼ばれる | 識別のために明示的な `mode`（例: `HTTP_GET` vs `HTTP_POST`）を設定する。 |

## ベストプラクティス

1. **粒度の細かいアプリケーション**: 1 つのモノリシックなクラスではなく、小さな焦点を絞ったアプリケーションにロジックを分割する。
2. **`init()` でのセットアップ**: コンストラクタではなく `init()` をセットアップ（設定、DB）に活用する。`setAction()` を呼び出さない — `@Action` アノテーションを使用する。
3. **モードの意識**: センシティブな操作を `CLI` のみまたは特定の HTTP メソッドに限定するために `@Action` の `Mode` パラメータを使用する。
4. **パラメータよりコンテキスト**: オプションの CLI フラグには、メソッドシグネチャにパラメータを追加するのではなく `getContext().getAttribute("--flag")` を使用する。
5. **非同期イベント**: イベントハンドラによってトリガーされる重いタスクには、ハンドラ内で `CompletableFuture.runAsync()` を使用する。

## テクニカルリファレンス

詳細なガイドは `references/` ディレクトリで利用可能:

- [アーキテクチャと設定](references/architecture.md) — 抽象化、パッケージマップ、プロパティ
- [ルーティングと @Action](references/routing.md) — アノテーションの詳細、モード、パラメータ
- [データ処理](references/data-handling.md) — Builder、Builders、JSON シリアライゼーションと解析
- [データベース永続化](references/database.md) — AbstractData POJO、CRUD、マッピング XML、POJO 生成
- [システムと使用法](references/system-usage.md) — Context、セッション、SSE、ファイルアップロード、イベント、ネットワーキング
- [テストパターン](references/testing.md) — JUnit 5 の単体テストと HTTP 統合テスト

## リファレンスソースファイル（内部）

- `src/main/java/org/tinystruct/AbstractApplication.java` — ライフサイクルフックを持つコアベースクラス
- `src/main/java/org/tinystruct/system/annotation/Action.java` — アノテーションとモード
- `src/main/java/org/tinystruct/application/ActionRegistry.java` — ルーティングエンジン
- `src/main/java/org/tinystruct/data/component/Builder.java` — JSON オブジェクトシリアライザ
- `src/main/java/org/tinystruct/data/component/Builders.java` — JSON 配列シリアライザ
- `src/main/java/org/tinystruct/data/component/AbstractData.java` — CRUD を持つベース POJO クラス
- `src/main/java/org/tinystruct/data/Mapping.java` — マッピング XML パーサー
- `src/main/java/org/tinystruct/data/tools/MySQLGenerator.java` — POJO ジェネレータリファレンス
- `src/main/java/org/tinystruct/data/component/FieldType.java` — SQL から Java への型マッピング
- `src/main/java/org/tinystruct/data/component/Condition.java` — フルエントな SQL クエリビルダー
- `src/main/java/org/tinystruct/http/SSEPushManager.java` — SSE 接続管理
- `src/test/java/org/tinystruct/application/ActionRegistryTest.java` — レジストリテストの例
- `src/test/java/org/tinystruct/system/HttpServerHttpModeTest.java` — HTTP 統合テストのパターン
