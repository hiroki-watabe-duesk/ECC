---
name: springboot-security
description: Java Spring Boot サービスにおける認証/認可、バリデーション、CSRF、シークレット、ヘッダー、レート制限、依存関係セキュリティのための Spring Security ベストプラクティス。
origin: ECC
---

# Spring Boot セキュリティレビュー

認証の追加、入力の処理、エンドポイントの作成、またはシークレットの扱いの際に使用する。

## 有効にするタイミング

- 認証を追加するとき（JWT、OAuth2、セッションベース）
- 認可を実装するとき（@PreAuthorize、ロールベースアクセス）
- ユーザー入力をバリデートするとき（Bean バリデーション、カスタムバリデーター）
- CORS、CSRF、またはセキュリティヘッダーを設定するとき
- シークレットを管理するとき（Vault、環境変数）
- レート制限またはブルートフォース保護を追加するとき
- CVE の依存関係をスキャンするとき

## 認証

- ステートレス JWT または失効リスト付き不透明トークンを優先する
- セッションには `httpOnly`、`Secure`、`SameSite=Strict` Cookie を使用する
- `OncePerRequestFilter` またはリソースサーバーでトークンをバリデートする

```java
@Component
public class JwtAuthFilter extends OncePerRequestFilter {
  private final JwtService jwtService;

  public JwtAuthFilter(JwtService jwtService) {
    this.jwtService = jwtService;
  }

  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
      FilterChain chain) throws ServletException, IOException {
    String header = request.getHeader(HttpHeaders.AUTHORIZATION);
    if (header != null && header.startsWith("Bearer ")) {
      String token = header.substring(7);
      Authentication auth = jwtService.authenticate(token);
      SecurityContextHolder.getContext().setAuthentication(auth);
    }
    chain.doFilter(request, response);
  }
}
```

## 認可

- メソッドセキュリティを有効化する: `@EnableMethodSecurity`
- `@PreAuthorize("hasRole('ADMIN')")` または `@PreAuthorize("@authz.canEdit(#id)")` を使用する
- デフォルトで拒否し、必要なスコープのみを公開する

```java
@RestController
@RequestMapping("/api/admin")
public class AdminController {

  @PreAuthorize("hasRole('ADMIN')")
  @GetMapping("/users")
  public List<UserDto> listUsers() {
    return userService.findAll();
  }

  @PreAuthorize("@authz.isOwner(#id, authentication)")
  @DeleteMapping("/users/{id}")
  public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
    userService.delete(id);
    return ResponseEntity.noContent().build();
  }
}
```

## 入力バリデーション

- コントローラーで `@Valid` を使って Bean バリデーションを使用する
- DTO にコンストレイントを適用する: `@NotBlank`、`@Email`、`@Size`、カスタムバリデーター
- レンダリング前にホワイトリストを使ってすべての HTML をサニタイズする

```java
// 悪い例: バリデーションなし
@PostMapping("/users")
public User createUser(@RequestBody UserDto dto) {
  return userService.create(dto);
}

// 良い例: バリデートされた DTO
public record CreateUserDto(
    @NotBlank @Size(max = 100) String name,
    @NotBlank @Email String email,
    @NotNull @Min(0) @Max(150) Integer age
) {}

@PostMapping("/users")
public ResponseEntity<UserDto> createUser(@Valid @RequestBody CreateUserDto dto) {
  return ResponseEntity.status(HttpStatus.CREATED)
      .body(userService.create(dto));
}
```

## SQL インジェクション防止

- Spring Data リポジトリまたはパラメータ化クエリを使用する
- ネイティブクエリには `:param` バインディングを使用し、文字列を連結しない

```java
// 悪い例: ネイティブクエリでの文字列連結
@Query(value = "SELECT * FROM users WHERE name = '" + name + "'", nativeQuery = true)

// 良い例: パラメータ化されたネイティブクエリ
@Query(value = "SELECT * FROM users WHERE name = :name", nativeQuery = true)
List<User> findByName(@Param("name") String name);

// 良い例: Spring Data 派生クエリ（自動パラメータ化）
List<User> findByEmailAndActiveTrue(String email);
```

## パスワードエンコーディング

- 常に BCrypt または Argon2 でパスワードをハッシュする — 平文で保存しない
- 手動ハッシュではなく `PasswordEncoder` Bean を使用する

```java
@Bean
public PasswordEncoder passwordEncoder() {
  return new BCryptPasswordEncoder(12); // コストファクター 12
}

// サービス内
public User register(CreateUserDto dto) {
  String hashedPassword = passwordEncoder.encode(dto.password());
  return userRepository.save(new User(dto.email(), hashedPassword));
}
```

## CSRF 保護

- ブラウザセッションアプリでは CSRF を有効にしておく; フォーム/ヘッダーにトークンを含める
- Bearer トークンを使った純粋な API では CSRF を無効にし、ステートレス認証に依存する

```java
http
  .csrf(csrf -> csrf.disable())
  .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
```

## シークレット管理

- ソースにシークレットを含めない; 環境変数または Vault から読み込む
- `application.yml` を認証情報がない状態に保つ; プレースホルダーを使用する
- トークンと DB 認証情報を定期的にローテートする

```yaml
# 悪い例: application.yml にハードコード
spring:
  datasource:
    password: mySecretPassword123

# 良い例: 環境変数プレースホルダー
spring:
  datasource:
    password: ${DB_PASSWORD}

# 良い例: Spring Cloud Vault 統合
spring:
  cloud:
    vault:
      uri: https://vault.example.com
      token: ${VAULT_TOKEN}
```

## セキュリティヘッダー

```java
http
  .headers(headers -> headers
    .contentSecurityPolicy(csp -> csp
      .policyDirectives("default-src 'self'"))
    .frameOptions(HeadersConfigurer.FrameOptionsConfig::sameOrigin)
    .xssProtection(Customizer.withDefaults())
    .referrerPolicy(rp -> rp.policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.NO_REFERRER)));
```

## CORS 設定

- コントローラーごとではなく、セキュリティフィルターレベルで CORS を設定する
- 許可するオリジンを制限する — 本番環境では `*` を絶対に使わない

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
  CorsConfiguration config = new CorsConfiguration();
  config.setAllowedOrigins(List.of("https://app.example.com"));
  config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
  config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
  config.setAllowCredentials(true);
  config.setMaxAge(3600L);

  UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
  source.registerCorsConfiguration("/api/**", config);
  return source;
}

// SecurityFilterChain 内:
http.cors(cors -> cors.configurationSource(corsConfigurationSource()));
```

## レート制限

- 高コストなエンドポイントに Bucket4j またはゲートウェイレベルの制限を適用する
- バーストをログに記録してアラートする; リトライヒントと共に 429 を返す

```java
// エンドポイントごとのレート制限に Bucket4j を使用
@Component
public class RateLimitFilter extends OncePerRequestFilter {
  private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();

  private Bucket createBucket() {
    return Bucket.builder()
        .addLimit(Bandwidth.classic(100, Refill.intervally(100, Duration.ofMinutes(1))))
        .build();
  }

  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
      FilterChain chain) throws ServletException, IOException {
    String clientIp = request.getRemoteAddr();
    Bucket bucket = buckets.computeIfAbsent(clientIp, k -> createBucket());

    if (bucket.tryConsume(1)) {
      chain.doFilter(request, response);
    } else {
      response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
      response.getWriter().write("{\"error\": \"Rate limit exceeded\"}");
    }
  }
}
```

## 依存関係セキュリティ

- CI で OWASP Dependency Check / Snyk を実行する
- Spring Boot と Spring Security をサポートされているバージョンに保つ
- 既知の CVE でビルドを失敗させる

## ロギングと PII

- シークレット、トークン、パスワード、または完全な PAN データをログに記録しない
- 機密フィールドを編集する; 構造化 JSON ロギングを使用する

## ファイルアップロード

- サイズ、コンテンツタイプ、拡張子をバリデートする
- Web ルート外に保存する; 必要に応じてスキャンする

## リリース前チェックリスト

- [ ] 認証トークンが正しくバリデートされ期限切れになっている
- [ ] すべての機密パスに認可ガードがある
- [ ] すべての入力がバリデートおよびサニタイズされている
- [ ] 文字列連結された SQL がない
- [ ] アプリタイプに対して CSRF の状態が正しい
- [ ] シークレットが外部化されており、コミットされていない
- [ ] セキュリティヘッダーが設定されている
- [ ] API にレート制限がある
- [ ] 依存関係がスキャンされて最新である
- [ ] ログに機密データが含まれていない

**覚えておくこと**: デフォルトで拒否、入力をバリデート、最小権限、そして設定によるセキュアファーストを徹底する。
