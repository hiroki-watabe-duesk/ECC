---
name: java-coding-standards
description: "Spring BootおよびQuarkusサービス向けJavaコーディング標準：命名、不変性、Optionalの使用、ストリーム、例外、ジェネリクス、CDI、リアクティブパターン、プロジェクトレイアウト。フレームワーク固有の規約を自動適用。"
origin: ECC
---

# Java コーディング標準

Spring BootおよびQuarkusサービスにおける読みやすく保守しやすいJava（17以降）コードの標準。

## 使用タイミング

- Spring BootまたはQuarkusプロジェクトでJavaコードを記述またはレビューするとき
- 命名、不変性、例外処理の規約を強制するとき
- レコード、封印クラス、またはパターンマッチング（Java 17以降）を使用するとき
- Optional、ストリーム、またはジェネリクスの使用をレビューするとき
- パッケージとプロジェクトレイアウトを構造化するとき
- **[QUARKUS]**: CDIスコープ、Panacheエンティティ、またはリアクティブパイプラインを使用するとき

## 仕組み

### フレームワークの検出

標準を適用する前に、ビルドファイルからフレームワークを判定します:

- ビルドファイルに `quarkus` が含まれる → **[QUARKUS]** 規約を適用
- ビルドファイルに `spring-boot` が含まれる → **[SPRING]** 規約を適用
- どちらも検出されない → 共通規約のみを適用

## コア原則

- 巧さよりも明確さを優先する
- デフォルトで不変にする。共有された可変状態を最小化する
- 意味のある例外でフェイルファストする
- 一貫した命名とパッケージ構造を維持する
- **[QUARKUS]**: ランタイム処理よりビルド時処理を優先する。可能な限りランタイムリフレクションを避ける

## 例

以下のセクションでは、命名、不変性、依存性注入、リアクティブコード、例外、プロジェクトレイアウト、ロギング、設定、テストについてSpring Boot、Quarkus、および共通Javaの具体的な例を示します。

## 命名

```java
// PASS: クラス/レコード: PascalCase
public class MarketService {}
public record Money(BigDecimal amount, Currency currency) {}

// PASS: メソッド/フィールド: camelCase
private final MarketRepository marketRepository;
public Market findBySlug(String slug) {}

// PASS: 定数: UPPER_SNAKE_CASE
private static final int MAX_PAGE_SIZE = 100;

// PASS: [QUARKUS] JAX-RSリソースは*Resourceと命名（*Controllerではない）
public class MarketResource {}

// PASS: [SPRING] RESTコントローラーは*Controllerと命名
public class MarketController {}
```

## 不変性

```java
// PASS: レコードとfinalフィールドを優先する
public record MarketDto(Long id, String name, MarketStatus status) {}

public class Market {
  private final Long id;
  private final String name;
  // ゲッターのみ、セッターなし
}

// PASS: [QUARKUS] Panacheアクティブレコードエンティティはpublicフィールドを使用（Quarkus規約）
@Entity
public class Market extends PanacheEntity {
  public String name;
  public MarketStatus status;
  // Panacheはビルド時にアクセサーを生成する。publicフィールドはここでは慣用的
}

// PASS: [QUARKUS] Panache MongoDBエンティティ
@MongoEntity(collection = "markets")
public class Market extends PanacheMongoEntity {
  public String name;
  public MarketStatus status;
}
```

## Optionalの使用

```java
// PASS: find*メソッドからOptionalを返す
// [SPRING]
Optional<Market> market = marketRepository.findBySlug(slug);

// [QUARKUS] Panache
Optional<Market> market = Market.find("slug", slug).firstResultOptional();

// PASS: get()の代わりにMap/flatMapを使用
return market
    .map(MarketResponse::from)
    .orElseThrow(() -> new EntityNotFoundException("Market not found"));
```

## ストリームのベストプラクティス

```java
// PASS: 変換にストリームを使用し、パイプラインを短く保つ
List<String> names = markets.stream()
    .map(Market::name)
    .filter(Objects::nonNull)
    .toList();

// FAIL: 複雑にネストしたストリームは避ける。明確さのためにループを優先する
```

## 依存性注入

```java
// PASS: [SPRING] コンストラクター注入（フィールドへの@Autowiredより優先）
@Service
public class MarketService {
  private final MarketRepository marketRepository;

  public MarketService(MarketRepository marketRepository) {
    this.marketRepository = marketRepository;
  }
}

// PASS: [QUARKUS] コンストラクター注入
@ApplicationScoped
public class MarketService {
  private final MarketRepository marketRepository;

  @Inject
  public MarketService(MarketRepository marketRepository) {
    this.marketRepository = marketRepository;
  }
}

// PASS: [QUARKUS] パッケージプライベートフィールド注入（Quarkusでは許容 — プロキシの問題を回避）
@ApplicationScoped
public class MarketService {
  @Inject
  MarketRepository marketRepository;
}

// FAIL: [SPRING] @Autowiredによるフィールド注入
@Autowired
private MarketRepository marketRepository; // コンストラクター注入を使用

// FAIL: [QUARKUS] インターセプションや遅延初期化が必要な場合の@Singleton
@Singleton // プロキシ不可 — 代わりに@ApplicationScopedを使用
public class MarketService {}
```

## リアクティブパターン [QUARKUS]

```java
// PASS: リアクティブエンドポイントからUni/Multiを返す
@GET
@Path("/{slug}")
public Uni<Market> findBySlug(@PathParam("slug") String slug) {
  return Market.find("slug", slug)
      .<Market>firstResult()
      .onItem().ifNull().failWith(() -> new MarketNotFoundException(slug));
}

// PASS: ノンブロッキングなパイプラインの合成
public Uni<OrderConfirmation> placeOrder(OrderRequest req) {
  return validateOrder(req)
      .chain(valid -> persistOrder(valid))
      .chain(order -> notifyFulfillment(order));
}

// FAIL: Uni/Multiパイプライン内でのブロッキング呼び出し
public Uni<Market> find(String slug) {
  Market m = Market.find("slug", slug).firstResult(); // ブロッキング — イベントループを妨害
  return Uni.createFrom().item(m);
}

// FAIL: 共有されたUniへの二重サブスクライブ
Uni<Market> shared = fetchMarket(slug);
shared.subscribe().with(m -> log(m));
shared.subscribe().with(m -> cache(m)); // 二重サブスクライブ — Uni.memoize()を使用
```

## 例外

- ドメインエラーには非検査例外を使用する。技術的な例外はコンテキストとともにラップする
- ドメイン固有の例外を作成する（例: `MarketNotFoundException`）
- 中央でリスロー/ロギングする場合を除き、広い `catch (Exception ex)` は避ける

```java
throw new MarketNotFoundException(slug);
```

### 集中型例外ハンドリング

```java
// [SPRING]
@RestControllerAdvice
public class GlobalExceptionHandler {
  @ExceptionHandler(MarketNotFoundException.class)
  public ResponseEntity<ErrorResponse> handle(MarketNotFoundException ex) {
    return ResponseEntity.status(404).body(ErrorResponse.from(ex));
  }
}

// [QUARKUS] オプションA: ExceptionMapper
@Provider
public class MarketNotFoundMapper implements ExceptionMapper<MarketNotFoundException> {
  @Override
  public Response toResponse(MarketNotFoundException ex) {
    return Response.status(404).entity(ErrorResponse.from(ex)).build();
  }
}

// [QUARKUS] オプションB: @ServerExceptionMapper（RESTEasy Reactive）
@ServerExceptionMapper
public RestResponse<ErrorResponse> handle(MarketNotFoundException ex) {
  return RestResponse.status(Status.NOT_FOUND, ErrorResponse.from(ex));
}
```

## ジェネリクスと型安全性

- 生の型を避ける。ジェネリクスパラメーターを宣言する
- 再利用可能なユーティリティには境界付きジェネリクスを優先する

```java
public <T extends Identifiable> Map<Long, T> indexById(Collection<T> items) { ... }
```

## プロジェクト構造

### [SPRING] Maven/Gradle

```
src/main/java/com/example/app/
  config/
  controller/
  service/
  repository/
  domain/
  dto/
  util/
src/main/resources/
  application.yml
src/test/java/... (mainを反映)
```

### [QUARKUS] Maven/Gradle

```
src/main/java/com/example/app/
  config/              # @ConfigMapping、@ConfigPropertyビーン、Producers
  resource/            # JAX-RSリソース（"controller"ではない）
  service/
  repository/          # PanacheRepositoryの実装（アクティブレコードを使用しない場合）
  domain/              # JPA/Panacheエンティティ、MongoDBエンティティ
  dto/
  util/
  mapper/              # MapStructマッパー（使用する場合）
src/main/resources/
  application.properties   # Quarkus規約（quarkus-config-yamlでYAMLをサポート）
  import.sql               # 開発/テスト用Hibernateオートインポート
src/test/java/... (mainを反映)
```

## フォーマットとスタイル

- 2または4スペースを一貫して使用する（プロジェクト標準に従う）
- ファイルにはpublicトップレベル型を1つ
- メソッドを短く焦点を絞ったものに保つ。ヘルパーを抽出する
- メンバーの順序: 定数、フィールド、コンストラクター、publicメソッド、protected、private

## 避けるべきコードの臭い

- 長いパラメーターリスト → DTO/ビルダーを使用する
- 深いネスト → 早期リターン
- マジックナンバー → 名前付き定数
- 静的な可変状態 → 依存性注入を優先する
- サイレントcatchブロック → ログを記録して対応するか再スローする
- **[QUARKUS]**: `@ApplicationScoped` が意図されている箇所での `@Singleton` — プロキシとインターセプションを破壊する
- **[QUARKUS]**: `quarkus-resteasy-reactive` と `quarkus-resteasy`（クラシック）の混在 — 一方のスタックを選択する
- **[QUARKUS]**: 同じ境界コンテキスト内でのPanacheアクティブレコード + リポジトリパターン — どちらか一方を選択する

## ロギング

```java
// [SPRING] SLF4J
private static final Logger log = LoggerFactory.getLogger(MarketService.class);
log.info("fetch_market slug={}", slug);
log.error("failed_fetch_market slug={}", slug, ex);

// [QUARKUS] JBossロギング（デフォルト、ビルド時にゼロコスト）
private static final Logger log = Logger.getLogger(MarketService.class);
log.infof("fetch_market slug=%s", slug);
log.errorf(ex, "failed_fetch_market slug=%s", slug);

// [QUARKUS] 代替: @InjectによるシンプルなロギングGの
@Inject
Logger log; // CDI注入、宣言クラスにスコープされる
```

## Null処理

- やむを得ない場合のみ `@Nullable` を受け入れる。そうでなければ `@NonNull` を使用する
- 入力にはBean Validation（`@NotNull`、`@NotBlank`）を使用する
- **[QUARKUS]**: `@BeanParam`、`@RestForm`、およびリクエストボディパラメーターに `@Valid` を適用する

## 設定

```java
// [SPRING] @ConfigurationProperties
@ConfigurationProperties(prefix = "market")
public record MarketProperties(int maxPageSize, Duration cacheTtl) {}

// [QUARKUS] @ConfigMapping（型安全、ビルド時に検証）
@ConfigMapping(prefix = "market")
public interface MarketConfig {
  int maxPageSize();
  Duration cacheTtl();
}

// [QUARKUS] @ConfigPropertyによるシンプルな値
@ConfigProperty(name = "market.max-page-size", defaultValue = "100")
int maxPageSize;
```

## テストの期待値

### 共通
- JUnit 5 + AssertJによる流暢なアサーション
- モック用のMockito。可能な限り部分モックを避ける
- 決定論的なテストを優先する。隠れたスリープは使わない

### [SPRING]
- コントローラーのスライステストには `@WebMvcTest`、リポジトリのスライステストには `@DataJpaTest`
- `@SpringBootTest` は完全インテグレーションテストに限定
- Springコンテキストのビーンを置き換えるには `@MockBean`

### [QUARKUS]
- ユニットテストにはプレーンJUnit 5 + Mockito（`@QuarkusTest` なし）
- `@QuarkusTest` はCDIインテグレーションテストに限定
- インテグレーションテストでCDIビーンを置き換えるには `@InjectMock`
- データベース/Kafka/Redis用のDev Services — Dev Servicesで十分な場合は手動Testcontainersの設定を避ける
- カスタム外部サービスライフサイクルには `@QuarkusTestResource`

```java
// [SPRING] コントローラーテスト
@WebMvcTest(MarketController.class)
class MarketControllerTest {
  @Autowired MockMvc mockMvc;
  @MockBean MarketService marketService;
}

// [QUARKUS] インテグレーションテスト
@QuarkusTest
class MarketResourceTest {
  @InjectMock
  MarketService marketService;

  @Test
  void should_return_404_when_market_not_found() {
    given().when().get("/markets/unknown").then().statusCode(404);
  }
}

// [QUARKUS] ユニットテスト（CDIなし、@QuarkusTestなし）
@ExtendWith(MockitoExtension.class)
class MarketServiceTest {
  @Mock MarketRepository marketRepository;
  @InjectMocks MarketService marketService;
}
```

**注意**: コードは意図的で型付けされ、観察可能に保つこと。証明されない限り、マイクロ最適化よりも保守性のために最適化すること。
