---
name: quarkus-patterns
description: Apache Camel を使ったメッセージング、RESTful API 設計、CDI サービス、Panache によるデータアクセス、非同期処理を含む Quarkus 3.x LTS アーキテクチャパターン。イベント駆動アーキテクチャを採用した Java Quarkus バックエンド開発に使用。
origin: ECC
---

# Quarkus 開発パターン

Apache Camel を用いたクラウドネイティブかつイベント駆動型サービスのための Quarkus 3.x アーキテクチャおよび API パターン。

## いつ有効にするか

- JAX-RS または RESTEasy Reactive を使った REST API の構築
- リソース → サービス → リポジトリのレイヤー設計
- Apache Camel と RabbitMQ を使ったイベント駆動パターンの実装
- Hibernate Panache、キャッシュ、リアクティブストリームの設定
- バリデーション、例外マッピング、ページネーションの追加
- dev/staging/production 環境向けプロファイルのセットアップ（YAML 設定）
- LogContext と Logback/Logstash エンコーダーを用いたカスタムロギング
- 非同期処理のための CompletableFuture の活用
- 条件付きフロー処理の実装
- GraalVM ネイティブコンパイルとの連携

## 複数の依存関係を持つサービスレイヤー

```java
@Slf4j
@ApplicationScoped
@RequiredArgsConstructor
public class OrderProcessingService {

    private final OrderValidator orderValidator;
    private final EventService eventService;
    private final OrderRepository orderRepository;
    private final FulfillmentPublisher fulfillmentPublisher;
    private final AuditPublisher auditPublisher;

    @Transactional
    public OrderReceipt process(CreateOrderCommand command) {
        ValidationResult validation = orderValidator.validate(command);
        if (!validation.valid()) {
            eventService.createErrorEvent(command, "ORDER_REJECTED", validation.message());
            throw new WebApplicationException(validation.message(), Response.Status.BAD_REQUEST);
        }

        Order order = Order.from(command);
        orderRepository.persist(order);

        OrderReceipt receipt = OrderReceipt.from(order);
        fulfillmentPublisher.publishAsync(receipt);
        auditPublisher.publish("ORDER_ACCEPTED", receipt);
        eventService.createSuccessEvent(receipt, "ORDER_ACCEPTED");

        log.info("Processed order {}", order.id);
        return receipt;
    }
}
```

**主要パターン:**
- Lombok によるコンストラクタインジェクションのための `@RequiredArgsConstructor`
- Logback ロギングのための `@Slf4j`
- Panache またはリポジトリを通じてデータを書き込むサービスメソッドへの `@Transactional`
- 永続化やメッセージ発行の前に入力を検証する
- 成功・エラー時のシナリオに応じたイベント追跡
- Camel メッセージの非同期発行

## カスタムロギングコンテキストパターン（Logback）

```java
@ApplicationScoped
public class ProcessingService {
    
    public void processDocument(Document doc) {
        LogContext logContext = CustomLog.getCurrentContext();
        try (SafeAutoCloseable ignored = CustomLog.startScope(logContext)) {
            // Add context to all log statements
            logContext.put("documentId", doc.getId().toString());
            logContext.put("documentType", doc.getType());
            logContext.put("userId", SecurityContext.getUserId());
            
            log.info("Starting document processing");
            
            // All logs within this scope inherit the context
            processInternal(doc);
            
            log.info("Document processing completed");
        } catch (Exception e) {
            log.error("Document processing failed", e);
            throw e;
        }
    }
}
```

**Logback 設定（logback.xml）:**

```xml
<configuration>
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeContext>true</includeContext>
            <includeMdc>true</includeMdc>
        </encoder>
    </appender>
    
    <logger name="com.example" level="INFO"/>
    <root level="WARN">
        <appender-ref ref="CONSOLE"/>
    </root>
</configuration>
```

## イベントサービスパターン

```java
@Slf4j
@ApplicationScoped
@RequiredArgsConstructor
public class EventService {
    private final EventRepository eventRepository;
    private final ObjectMapper objectMapper;
    
    public void createSuccessEvent(Object payload, String eventType) {
        Objects.requireNonNull(payload, "Payload cannot be null");
        Event event = new Event();
        event.setType(eventType);
        event.setStatus(EventStatus.SUCCESS);
        event.setPayload(serializePayload(payload));
        event.setTimestamp(Instant.now());
        
        eventRepository.persist(event);
        log.info("Success event created: {}", eventType);
    }
    
    public void createErrorEvent(Object payload, String eventType, String errorMessage) {
        Objects.requireNonNull(payload, "Payload cannot be null");
        if (errorMessage == null || errorMessage.isBlank()) {
            throw new IllegalArgumentException("Error message cannot be blank");
        }
        Event event = new Event();
        event.setType(eventType);
        event.setStatus(EventStatus.ERROR);
        event.setErrorMessage(errorMessage);
        event.setPayload(serializePayload(payload));
        event.setTimestamp(Instant.now());
        
        eventRepository.persist(event);
        log.error("Error event created: {} - {}", eventType, errorMessage);
    }
    
    private String serializePayload(Object payload) {
        try {
            return objectMapper.writeValueAsString(payload);
        } catch (JsonProcessingException e) {
            throw new IllegalStateException("Failed to serialize event payload", e);
        }
    }
}
```

## Camel メッセージ発行（RabbitMQ）

```java
@Slf4j
@ApplicationScoped
@RequiredArgsConstructor
public class BusinessRulesPublisher {
    private final ProducerTemplate producerTemplate;
    
    public void publishSync(BusinessRulesPayload payload) {
        producerTemplate.sendBody(
            "direct:business-rules-publisher", 
            payload
        );
    }
}
```

**Camel ルート設定:**

```java
@ApplicationScoped
public class BusinessRulesRoute extends RouteBuilder {
    
    @ConfigProperty(name = "camel.rabbitmq.queue.business-rules")
    String businessRulesQueue;
    
    @ConfigProperty(name = "rabbitmq.host")
    String rabbitHost;
    
    @ConfigProperty(name = "rabbitmq.port")
    Integer rabbitPort;
    
    @Override
    public void configure() {
        from("direct:business-rules-publisher")
            .routeId("business-rules-publisher")
            .log("Publishing message to RabbitMQ: ${body}")
            .marshal().json(JsonLibrary.Jackson)
            .toF("spring-rabbitmq:%s?hostname=%s&portNumber=%d", 
                businessRulesQueue, rabbitHost, rabbitPort);
    }
}
```

## Camel Direct ルート（インメモリ）

```java
@ApplicationScoped
public class DocumentProcessingRoute extends RouteBuilder {
    
    @Override
    public void configure() {
        // Error handling
        onException(ValidationException.class)
            .handled(true)
            .to("direct:validation-error-handler")
            .log("Validation error: ${exception.message}");
        
        // Main processing route
        from("direct:process-document")
            .routeId("document-processing")
            .log("Processing document: ${header.documentId}")
            .bean(DocumentValidator.class, "validate")
            .bean(DocumentTransformer.class, "transform")
            .choice()
                .when(header("documentType").isEqualTo("INVOICE"))
                    .to("direct:process-invoice")
                .when(header("documentType").isEqualTo("CREDIT_NOTE"))
                    .to("direct:process-credit-note")
                .otherwise()
                    .to("direct:process-generic")
            .end();
        
        from("direct:validation-error-handler")
            .bean(EventService.class, "createErrorEvent")
            .log("Validation error handled");
    }
}
```

## Camel ファイル処理

```java
@ApplicationScoped
public class FileMonitoringRoute extends RouteBuilder {
    
    @ConfigProperty(name = "file.input.directory")
    String inputDirectory;
    
    @ConfigProperty(name = "file.processed.directory")
    String processedDirectory;
    
    @ConfigProperty(name = "file.error.directory")
    String errorDirectory;
    
    @Override
    public void configure() {
        from("file:" + inputDirectory + "?move=" + processedDirectory + 
             "&moveFailed=" + errorDirectory + "&delay=5000")
            .routeId("file-monitor")
            .log("Processing file: ${header.CamelFileName}")
            .to("direct:process-file");
        
        from("direct:process-file")
            .bean(OrderProcessingService.class, "processFile")
            .log("File processing completed");
    }
}
```

## Camel Bean 呼び出し

```java
@ApplicationScoped
public class InvoiceRoute extends RouteBuilder {
    
    @Override
    public void configure() {
        from("direct:invoice-validation")
            .bean(InvoiceFlowValidator.class, "validateFlowWithConfig")
            .log("Validation result: ${body}");
        
        from("direct:persist-and-publish")
            .bean(DocumentJobService.class, "createDocumentAndJobEntities")
            .bean(BusinessRulesPublisher.class, "publishAsync")
            .bean(EventService.class, "createSuccessEvent(${body}, 'PUBLISHED')");
    }
}
```

## REST API 構造

```java
@Path("/api/documents")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
@RequiredArgsConstructor
public class DocumentResource {
  private final DocumentService documentService;

  @GET
  public Response list(
      @QueryParam("page") @DefaultValue("0") int page,
      @QueryParam("size") @DefaultValue("20") int size) {
    List<Document> documents = documentService.list(page, size);
    return Response.ok(documents).build();
  }

  @POST
  public Response create(@Valid CreateDocumentRequest request, @Context UriInfo uriInfo) {
    Document document = documentService.create(request);
    URI location = uriInfo.getAbsolutePathBuilder()
        .path(String.valueOf(document.id))
        .build();
    return Response.created(location).entity(DocumentResponse.from(document)).build();
  }

  @GET
  @Path("/{id}")
  public Response getById(@PathParam("id") Long id) {
    return documentService.findById(id)
        .map(DocumentResponse::from)
        .map(Response::ok)
        .orElse(Response.status(Response.Status.NOT_FOUND))
        .build();
  }
}
```

## リポジトリパターン（Panache リポジトリ）

```java
@ApplicationScoped
public class DocumentRepository implements PanacheRepository<Document> {
  
  public List<Document> findByStatus(DocumentStatus status, int page, int size) {
    return find("status = ?1 order by createdAt desc", status)
        .page(page, size)
        .list();
  }

  public Optional<Document> findByReferenceNumber(String referenceNumber) {
    return find("referenceNumber", referenceNumber).firstResultOptional();
  }
  
  public long countByStatusAndDate(DocumentStatus status, LocalDate date) {
    return count("status = ?1 and createdAt >= ?2", status, date.atStartOfDay());
  }
}
```

## トランザクション付きサービスレイヤー

```java
@ApplicationScoped
@RequiredArgsConstructor
public class DocumentService {
  private final DocumentRepository repo;
  private final EventService eventService;

  @Transactional
  public Document create(CreateDocumentRequest request) {
    Document document = new Document();
    document.setReferenceNumber(request.referenceNumber());
    document.setDescription(request.description());
    document.setStatus(DocumentStatus.PENDING);
    document.setCreatedAt(Instant.now());
    
    repo.persist(document);
    
    eventService.createSuccessEvent(document, "DOCUMENT_CREATED");
    
    return document;
  }

  public Optional<Document> findById(Long id) {
    return repo.findByIdOptional(id);
  }

  public List<Document> list(int page, int size) {
    return repo.findAll()
        .page(page, size)
        .list();
  }
}
```

## DTO とバリデーション

```java
public record CreateDocumentRequest(
    @NotBlank @Size(max = 200) String referenceNumber,
    @NotBlank @Size(max = 2000) String description,
    @NotNull @FutureOrPresent Instant validUntil,
    @NotEmpty List<@NotBlank String> categories) {}

public record DocumentResponse(Long id, String referenceNumber, DocumentStatus status) {
  public static DocumentResponse from(Document document) {
    return new DocumentResponse(document.getId(), document.getReferenceNumber(), 
        document.getStatus());
  }
}
```

## 例外マッピング

```java
@Provider
public class ValidationExceptionMapper implements ExceptionMapper<ConstraintViolationException> {
  @Override
  public Response toResponse(ConstraintViolationException exception) {
    String message = exception.getConstraintViolations().stream()
        .map(cv -> cv.getPropertyPath() + ": " + cv.getMessage())
        .collect(Collectors.joining(", "));
    
    return Response.status(Response.Status.BAD_REQUEST)
        .entity(Map.of("error", "validation_error", "message", message))
        .build();
  }
}

@Provider
@Slf4j
public class GenericExceptionMapper implements ExceptionMapper<Exception> {

  @Override
  public Response toResponse(Exception exception) {
    log.error("Unhandled exception", exception);
    return Response.status(Response.Status.INTERNAL_SERVER_ERROR)
        .entity(Map.of("error", "internal_error", "message", "An unexpected error occurred"))
        .build();
  }
}
```

## CompletableFuture 非同期処理

```java
@Slf4j
@ApplicationScoped
@RequiredArgsConstructor
public class FileStorageService {
    private final S3Client s3Client;
    private final ExecutorService executorService;
    
    @ConfigProperty(name = "storage.bucket-name")
    String bucketName;
    
    public CompletableFuture<StoredDocumentInfo> uploadOriginalFile(
            InputStream inputStream, 
            long size, 
            LogContext logContext,
            InvoiceFormat format) {
        
        return CompletableFuture.supplyAsync(() -> {
            try (SafeAutoCloseable ignored = CustomLog.startScope(logContext)) {
                String path = generateStoragePath(format);
                
                PutObjectRequest request = PutObjectRequest.builder()
                    .bucket(bucketName)
                    .key(path)
                    .contentLength(size)
                    .build();
                
                s3Client.putObject(request, RequestBody.fromInputStream(inputStream, size));
                
                log.info("File uploaded to S3: {}", path);
                
                return new StoredDocumentInfo(path, size, Instant.now());
            } catch (Exception e) {
                log.error("Failed to upload file to S3", e);
                throw new StorageException("Upload failed", e);
            }
        }, executorService);
    }
}
```

## キャッシュ

```java
@ApplicationScoped
@RequiredArgsConstructor
public class DocumentCacheService {
  private final DocumentRepository repo;

  @CacheResult(cacheName = "document-cache")
  public Optional<Document> getById(@CacheKey Long id) {
    return repo.findByIdOptional(id);
  }

  @CacheInvalidate(cacheName = "document-cache")
  public void evict(@CacheKey Long id) {}

  @CacheInvalidateAll(cacheName = "document-cache")
  public void evictAll() {}
}
```

## YAML による設定

```yaml
# application.yml
"%dev":
  quarkus:
    datasource:
      jdbc:
        url: jdbc:postgresql://localhost:5432/dev_db
      username: dev_user
      password: ${DB_PASSWORD}
    hibernate-orm:
      database:
        generation: drop-and-create
  
  rabbitmq:
    host: localhost
    port: 5672
    username: ${RABBITMQ_USER}
    password: ${RABBITMQ_PASSWORD}

"%test":
  quarkus:
    datasource:
      jdbc:
        url: jdbc:h2:mem:test
    hibernate-orm:
      database:
        generation: drop-and-create

"%prod":
  quarkus:
    datasource:
      jdbc:
        url: ${DATABASE_URL}
      username: ${DB_USER}
      password: ${DB_PASSWORD}
    hibernate-orm:
      database:
        generation: validate
  
  rabbitmq:
    host: ${RABBITMQ_HOST}
    port: ${RABBITMQ_PORT}
    username: ${RABBITMQ_USER}
    password: ${RABBITMQ_PASSWORD}

# Camel configuration
camel:
  rabbitmq:
    queue:
      business-rules: business-rules-queue
      invoice-processing: invoice-processing-queue
```

## ヘルスチェック

```java
@Readiness
@ApplicationScoped
@RequiredArgsConstructor
public class DatabaseHealthCheck implements HealthCheck {
  private final AgroalDataSource dataSource;

  @Override
  public HealthCheckResponse call() {
    try (Connection conn = dataSource.getConnection()) {
      boolean valid = conn.isValid(2);
      return HealthCheckResponse.named("Database connection")
          .status(valid)
          .build();
    } catch (SQLException e) {
      return HealthCheckResponse.down("Database connection");
    }
  }
}

@Liveness
@ApplicationScoped
public class CamelHealthCheck implements HealthCheck {
  @Inject
  CamelContext camelContext;

  @Override
  public HealthCheckResponse call() {
    boolean isStarted = camelContext.getStatus().isStarted();
    return HealthCheckResponse.named("Camel Context")
        .status(isStarted)
        .build();
  }
}
```

## 依存関係（Maven）

```xml
<properties>
    <quarkus.platform.version>3.27.0</quarkus.platform.version>
    <lombok.version>1.18.42</lombok.version>
    <assertj-core.version>3.24.2</assertj-core.version>
    <jacoco-maven-plugin.version>0.8.13</jacoco-maven-plugin.version>
    <maven.compiler.release>17</maven.compiler.release>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.quarkus.platform</groupId>
            <artifactId>quarkus-bom</artifactId>
            <version>${quarkus.platform.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>io.quarkus.platform</groupId>
            <artifactId>quarkus-camel-bom</artifactId>
            <version>${quarkus.platform.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- Quarkus Core -->
    <dependency>
        <groupId>io.quarkus</groupId>
        <artifactId>quarkus-arc</artifactId>
    </dependency>
    <dependency>
        <groupId>io.quarkus</groupId>
        <artifactId>quarkus-config-yaml</artifactId>
    </dependency>
    
    <!-- Camel Extensions -->
    <dependency>
        <groupId>org.apache.camel.quarkus</groupId>
        <artifactId>camel-quarkus-spring-rabbitmq</artifactId>
    </dependency>
    <dependency>
        <groupId>org.apache.camel.quarkus</groupId>
        <artifactId>camel-quarkus-direct</artifactId>
    </dependency>
    <dependency>
        <groupId>org.apache.camel.quarkus</groupId>
        <artifactId>camel-quarkus-bean</artifactId>
    </dependency>
    
    <!-- Lombok -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>${lombok.version}</version>
        <scope>provided</scope>
    </dependency>
    
    <!-- Logging -->
    <dependency>
        <groupId>io.quarkiverse.logging.logback</groupId>
        <artifactId>quarkus-logging-logback</artifactId>
    </dependency>
    <dependency>
        <groupId>net.logstash.logback</groupId>
        <artifactId>logstash-logback-encoder</artifactId>
    </dependency>
</dependencies>
```

## ベストプラクティス

### アーキテクチャ
- Lombok によるコンストラクタインジェクションのために `@RequiredArgsConstructor` を使用する
- サービスレイヤーを薄く保ち、複雑なロジックは専門クラスに委譲する
- メッセージルーティングと統合パターンには Camel ルートを使用する
- データアクセスには Panache リポジトリパターンを優先する

### イベント駆動
- 常に EventService で操作を追跡する（成功・エラーイベント）
- インメモリルーティングには Camel の `direct:` エンドポイントを使用する
- RabbitMQ 統合には `spring-rabbitmq` コンポーネントを使用する
- `ProducerTemplate.asyncSendBody()` を使った非同期発行を実装する

### ロギング
- 構造化ロギングのために Logback と Logstash エンコーダーを使用する
- `SafeAutoCloseable` を使ってサービス呼び出し全体で LogContext を伝播する
- リクエストトレースのために LogContext にコンテキスト情報を追加する
- 手動ロガー生成の代わりに `@Slf4j` を使用する

### 非同期処理
- ノンブロッキング I/O 処理には CompletableFuture を使用する
- 完了を待つ必要がある場合は `.join()` を呼び出す
- CompletableFuture の例外を適切に処理する
- トレースのために非同期処理へ LogContext を渡す

### 設定
- YAML 設定（`quarkus-config-yaml`）を使用する
- dev/test/prod 環境向けのプロファイル対応設定
- 機密設定は環境変数に外部化する
- 型安全な設定インジェクションのために `@ConfigProperty` を使用する

### バリデーション
- `@Valid` を使ってリソースレイヤーでバリデーションを行う
- DTO には Bean Validation アノテーションを使用する
- `@Provider` を使って例外を適切な HTTP レスポンスにマッピングする

### トランザクション
- データを変更するサービスメソッドには `@Transactional` を使用する
- トランザクションを短く、目的に集中させる
- トランザクション内で非同期処理を呼び出すことを避ける

### テスト
- ルートテストには `camel-quarkus-junit5` を使用する
- アサーションには AssertJ を使用する
- 外部依存関係はすべてモックする
- 条件付きフローロジックを徹底的にテストする

### Quarkus 固有
- 最新の LTS バージョン（3.x）を使い続ける
- ホットリロードには Quarkus dev モードを使用する
- 本番環境への対応のためにヘルスチェックを追加する
- GraalVM ネイティブコンパイルの互換性を定期的にテストする
