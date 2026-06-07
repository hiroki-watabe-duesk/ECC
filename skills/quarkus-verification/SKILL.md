---
name: quarkus-verification
description: "Quarkusプロジェクト向け検証ループ：PRやリリース前にビルド・静的解析・カバレッジ付きテスト・セキュリティスキャン・ネイティブコンパイル・差分レビューを実行する。"
origin: ECC
---

# Quarkus 検証ループ

PR前、大規模変更後、デプロイ前に実行する。

## アクティベートするタイミング

- QuarkusサービスのPRを開く前
- 大規模なリファクタリングや依存ライブラリのアップグレード後
- ステージングまたは本番へのデプロイ前検証
- ビルド → リント → テスト → セキュリティスキャン → ネイティブコンパイルのパイプライン全体を実行するとき
- テストカバレッジが閾値（80%以上）を満たしているか検証するとき
- ネイティブイメージの互換性をテストするとき

## フェーズ1: ビルド

```bash
# Maven
mvn clean verify -DskipTests

# Gradle
./gradlew clean assemble -x test
```

ビルドが失敗した場合は停止してコンパイルエラーを修正する。

## フェーズ2: 静的解析

### Checkstyle、PMD、SpotBugs（Maven）

```bash
mvn checkstyle:check pmd:check spotbugs:check
```

### SonarQube（設定済みの場合）

```bash
mvn sonar:sonar \
  -Dsonar.projectKey=my-quarkus-project \
  -Dsonar.host.url=http://localhost:9000 \
  -Dsonar.login=${SONAR_TOKEN}
```

### 対処すべき一般的な問題

- 未使用のインポートや変数
- 複雑なメソッド（高い循環的複雑度）
- nullポインタ参照の可能性
- SpotBugsが検出したセキュリティ問題

## フェーズ3: テスト + カバレッジ

```bash
# 全テストを実行
mvn clean test

# カバレッジレポートを生成
mvn jacoco:report

# カバレッジ閾値を適用（80%）
mvn jacoco:check

# またはGradleで
./gradlew test jacocoTestReport jacocoTestCoverageVerification
```

### テストカテゴリ

#### ユニットテスト
モックした依存関係でサービスロジックをテストする:

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
  @Mock UserRepository userRepository;
  @InjectMocks UserService userService;

  @Test
  void createUser_validInput_returnsUser() {
    var dto = new CreateUserDto("Alice", "alice@example.com");

    // Panache persist() は void — doNothing + verify を使用
    doNothing().when(userRepository).persist(any(User.class));

    User result = userService.create(dto);

    assertThat(result.name).isEqualTo("Alice");
    verify(userRepository).persist(any(User.class));
  }
}
```

#### 統合テスト
実際のデータベース（Testcontainers）でテストする:

```java
@QuarkusTest
@QuarkusTestResource(PostgresTestResource.class)
class UserRepositoryIntegrationTest {

  @Inject
  UserRepository userRepository;

  @Test
  @Transactional
  void findByEmail_existingUser_returnsUser() {
    User user = new User();
    user.name = "Alice";
    user.email = "alice@example.com";
    userRepository.persist(user);

    Optional<User> found = userRepository.findByEmail("alice@example.com");

    assertThat(found).isPresent();
    assertThat(found.get().name).isEqualTo("Alice");
  }
}
```

#### APIテスト
REST AssuredでRESTエンドポイントをテストする:

```java
@QuarkusTest
class UserResourceTest {

  @Test
  void createUser_validInput_returns201() {
    given()
        .contentType(ContentType.JSON)
        .body("""
            {"name": "Alice", "email": "alice@example.com"}
            """)
        .when().post("/api/users")
        .then()
        .statusCode(201)
        .body("name", equalTo("Alice"));
  }

  @Test
  void createUser_invalidEmail_returns400() {
    given()
        .contentType(ContentType.JSON)
        .body("""
            {"name": "Alice", "email": "invalid"}
            """)
        .when().post("/api/users")
        .then()
        .statusCode(400);
  }
}
```

### カバレッジレポート

`target/site/jacoco/index.html` で詳細なカバレッジを確認する:
- 全体の行カバレッジ（目標: 80%以上）
- 分岐カバレッジ（目標: 70%以上）
- カバーされていない重要なパスを特定する

## フェーズ4: セキュリティスキャン

### 依存ライブラリの脆弱性チェック（Maven）

```bash
mvn org.owasp:dependency-check-maven:check
```

CVEについては `target/dependency-check-report.html` を確認する。

### Quarkusセキュリティ監査

```bash
# 脆弱なエクステンションを確認
mvn quarkus:audit

# 全エクステンションを一覧表示
mvn quarkus:list-extensions
```

### OWASP ZAP（APIセキュリティテスト）

```bash
docker run -t owasp/zap2docker-stable zap-api-scan.py \
  -t http://localhost:8080/q/openapi \
  -f openapi
```

### 一般的なセキュリティチェック

- [ ] 全シークレットが環境変数に保存されている（コード内ではない）
- [ ] 全エンドポイントで入力バリデーションを実施
- [ ] 認証/認可が設定されている
- [ ] CORSが適切に設定されている
- [ ] セキュリティヘッダーが設定されている
- [ ] パスワードがBCryptでハッシュ化されている
- [ ] SQLインジェクション対策が施されている（パラメータ化クエリ）
- [ ] 公開エンドポイントにレート制限が設定されている

## フェーズ5: ネイティブコンパイル

GraalVMネイティブイメージの互換性をテストする:

```bash
# ネイティブ実行ファイルをビルド
mvn package -Dnative

# またはコンテナを使用
mvn package -Dnative -Dquarkus.native.container-build=true

# ネイティブ実行ファイルをテスト
./target/*-runner

# 基本的なスモークテストを実行
curl http://localhost:8080/q/health/live
curl http://localhost:8080/q/health/ready
```

### ネイティブイメージのトラブルシューティング

一般的な問題:
- **リフレクション**: 動的クラスのリフレクション設定を追加する
- **リソース**: `quarkus.native.resources.includes` でリソースを含める
- **JNI**: ネイティブライブラリを使用する場合はJNIクラスを登録する

リフレクション設定の例:
```java
@RegisterForReflection(targets = {MyDynamicClass.class})
public class ReflectionConfiguration {}
```

## フェーズ6: パフォーマンステスト

### K6による負荷テスト

```javascript
// load-test.js
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  stages: [
    { duration: '30s', target: 50 },
    { duration: '1m', target: 100 },
    { duration: '30s', target: 0 },
  ],
};

export default function () {
  const res = http.get('http://localhost:8080/api/markets');
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 200ms': (r) => r.timings.duration < 200,
  });
}
```

実行:
```bash
k6 run load-test.js
```

### 監視すべき指標

- レスポンスタイム（p50、p95、p99）
- スループット（リクエスト/秒）
- エラーレート
- メモリ使用量
- CPU使用率

## フェーズ7: ヘルスチェック

```bash
# 生存確認
curl http://localhost:8080/q/health/live

# 準備確認
curl http://localhost:8080/q/health/ready

# 全ヘルスチェック
curl http://localhost:8080/q/health

# メトリクス（有効な場合）
curl http://localhost:8080/q/metrics
```

期待されるレスポンス:
```json
{
  "status": "UP",
  "checks": [
    {
      "name": "Database connection",
      "status": "UP"
    }
  ]
}
```

## フェーズ8: コンテナイメージビルド

```bash
# コンテナイメージをビルド
mvn package -Dquarkus.container-image.build=true

# または特定のレジストリを指定
mvn package \
  -Dquarkus.container-image.build=true \
  -Dquarkus.container-image.registry=docker.io \
  -Dquarkus.container-image.group=myorg \
  -Dquarkus.container-image.tag=1.0.0

# コンテナをテスト
docker run -p 8080:8080 myorg/my-quarkus-app:1.0.0
```

### コンテナセキュリティスキャン

```bash
# Trivy
trivy image myorg/my-quarkus-app:1.0.0

# Grype
grype myorg/my-quarkus-app:1.0.0
```

## フェーズ9: 設定の検証

```bash
# 全設定プロパティを確認
mvn quarkus:info

# 全設定ソースを一覧表示
curl http://localhost:8080/q/dev/io.quarkus.quarkus-vertx-http/config
```

### 環境固有のチェック

- [ ] 環境ごとにデータベースURLが設定されている
- [ ] シークレットが外部化されている（Vault、環境変数）
- [ ] ログレベルが適切
- [ ] CORSオリジンが正しく設定されている
- [ ] レート制限が設定されている
- [ ] モニタリング/トレーシングが有効

## フェーズ10: ドキュメントのレビュー

- [ ] OpenAPI/Swaggerドキュメントが最新 (`/q/swagger-ui`)
- [ ] READMEにセットアップ手順がある
- [ ] APIの変更がドキュメント化されている
- [ ] 破壊的変更のマイグレーションガイドがある
- [ ] 設定プロパティがドキュメント化されている

OpenAPIスペックを生成:
```bash
curl http://localhost:8080/q/openapi -o openapi.json
```

## 検証チェックリスト

### コード品質
- [ ] ビルドが警告なしに通る
- [ ] 静的解析がクリーン（高/中程度の問題なし）
- [ ] コードがチームの規約に従っている
- [ ] コメントアウトされたコードやTODOがPRに含まれていない

### テスト
- [ ] 全テストが通る
- [ ] コードカバレッジが80%以上
- [ ] 実際のデータベースを使った統合テスト
- [ ] セキュリティテストが通る
- [ ] パフォーマンスが許容範囲内

### セキュリティ
- [ ] 依存ライブラリに脆弱性がない
- [ ] 認証/認可がテスト済み
- [ ] 入力バリデーションが完全
- [ ] ソースコードにシークレットがない
- [ ] セキュリティヘッダーが設定されている

### デプロイ
- [ ] ネイティブコンパイルが成功
- [ ] コンテナイメージがビルドできる
- [ ] ヘルスチェックが正常に応答する
- [ ] ターゲット環境用の設定が有効

### ネイティブイメージ
- [ ] ネイティブ実行ファイルがビルドできる
- [ ] ネイティブテストが通る
- [ ] 起動時間が100ms未満
- [ ] メモリフットプリントが許容範囲内

## 自動検証スクリプト

```bash
#!/bin/bash
set -e

echo "=== Phase 1: Build ==="
mvn clean verify -DskipTests

echo "=== Phase 2: Static Analysis ==="
mvn checkstyle:check pmd:check spotbugs:check

echo "=== Phase 3: Tests + Coverage ==="
mvn test jacoco:report jacoco:check

echo "=== Phase 4: Security Scan ==="
mvn org.owasp:dependency-check-maven:check

echo "=== Phase 5: Native Compilation ==="
mvn package -Dnative -Dquarkus.native.container-build=true

echo "=== All Phases Complete ==="
echo "Review reports:"
echo "  - Coverage: target/site/jacoco/index.html"
echo "  - Security: target/dependency-check-report.html"
echo "  - Native: target/*-runner"
```

## CI/CD統合

### GitHub Actionsの例

```yaml
name: Verification

on: [push, pull_request]

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up JDK 21
        uses: actions/setup-java@v3
        with:
          java-version: '21'
          distribution: 'temurin'
      
      - name: Cache Maven packages
        uses: actions/cache@v3
        with:
          path: ~/.m2
          key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
      
      - name: Build
        run: mvn clean verify -DskipTests
      
      - name: Test with Coverage
        run: mvn test jacoco:report jacoco:check
      
      - name: Security Scan
        run: mvn org.owasp:dependency-check-maven:check
      
      - name: Upload Coverage
        uses: codecov/codecov-action@v3
        with:
          files: target/site/jacoco/jacoco.xml
```

## ベストプラクティス

- 全PRの前に検証ループを実行する
- CI/CDパイプラインで自動化する
- 問題は即座に修正する。技術的負債を蓄積しない
- カバレッジを80%以上に保つ
- 依存ライブラリを定期的に更新する
- ネイティブコンパイルを定期的にテストする
- パフォーマンスのトレンドを監視する
- 破壊的変更をドキュメント化する
- セキュリティスキャンの結果をレビューする
- 各環境の設定を検証する
