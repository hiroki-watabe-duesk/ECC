---
name: quarkus-security
description: Quarkusセキュリティのベストプラクティス（認証・認可・JWT/OIDC・RBAC・入力バリデーション・CSRF・シークレット管理・依存関係セキュリティ）。
origin: ECC
---

# Quarkusセキュリティレビュー

認証・認可・入力バリデーションによるQuarkusアプリケーションのセキュリティ強化に関するベストプラクティス。

## 有効化タイミング

- 認証の追加（JWT、OIDC、Basic Auth）
- `@RolesAllowed` または `SecurityIdentity` を使った認可の実装
- ユーザー入力のバリデーション（Bean Validation、カスタムバリデーター）
- CORSまたはセキュリティヘッダーの設定
- シークレットの管理（Vault、環境変数、設定ソース）
- レート制限またはブルートフォース対策の追加
- CVEに関する依存関係のスキャン
- MicroProfile JWT または SmallRye JWT の利用

## 認証

### JWT認証

```java
// JWTで保護されたリソース
@Path("/api/protected")
@Authenticated
public class ProtectedResource {
  
  @Inject
  JsonWebToken jwt;

  @Inject
  SecurityIdentity securityIdentity;

  @GET
  public Response getData() {
    String username = jwt.getName();
    Set<String> roles = jwt.getGroups();
    return Response.ok(Map.of(
        "username", username,
        "roles", roles,
        "principal", securityIdentity.getPrincipal().getName()
    )).build();
  }
}
```

設定（application.properties）:
```properties
mp.jwt.verify.publickey.location=publicKey.pem
mp.jwt.verify.issuer=https://auth.example.com

# OIDC
quarkus.oidc.auth-server-url=https://auth.example.com/realms/myrealm
quarkus.oidc.client-id=backend-service
quarkus.oidc.credentials.secret=${OIDC_SECRET}
```

### カスタム認証フィルター

```java
@Provider
@Priority(Priorities.AUTHENTICATION)
public class CustomAuthFilter implements ContainerRequestFilter {
  
  @Inject
  SecurityIdentity identity;

  @Override
  public void filter(ContainerRequestContext requestContext) {
    String authHeader = requestContext.getHeaderString(HttpHeaders.AUTHORIZATION);
    
    // ヘッダーが存在しないか不正な場合は即座に拒否
    if (authHeader == null || !authHeader.startsWith("Bearer ")) {
      requestContext.abortWith(Response.status(Response.Status.UNAUTHORIZED).build());
      return;
    }
    
    String token = authHeader.substring(7);
    if (!validateToken(token)) {
      requestContext.abortWith(Response.status(Response.Status.UNAUTHORIZED).build());
    }
  }

  private boolean validateToken(String token) {
    // トークン検証ロジック
    return true;
  }
}
```

## 認可

### ロールベースアクセス制御

```java
@Path("/api/admin")
@RolesAllowed("ADMIN")
public class AdminResource {
  
  @GET
  @Path("/users")
  public List<UserDto> listUsers() {
    return userService.findAll();
  }

  @DELETE
  @Path("/users/{id}")
  @RolesAllowed({"ADMIN", "SUPER_ADMIN"})
  public Response deleteUser(@PathParam("id") Long id) {
    userService.delete(id);
    return Response.noContent().build();
  }
}

@Path("/api/users")
public class UserResource {
  
  @Inject
  SecurityIdentity securityIdentity;

  @GET
  @Path("/{id}")
  @RolesAllowed("USER")
  public Response getUser(@PathParam("id") Long id) {
    // 所有者チェック
    if (!securityIdentity.hasRole("ADMIN") && 
        !isOwner(id, securityIdentity.getPrincipal().getName())) {
      return Response.status(Response.Status.FORBIDDEN).build();
    }
    return Response.ok(userService.findById(id)).build();
  }

  private boolean isOwner(Long userId, String username) {
    return userService.isOwner(userId, username);
  }
}
```

### プログラマティックセキュリティ

```java
@ApplicationScoped
public class SecurityService {
  
  @Inject
  SecurityIdentity securityIdentity;

  public boolean canAccessResource(Long resourceId) {
    if (securityIdentity.isAnonymous()) {
      return false;
    }
    
    if (securityIdentity.hasRole("ADMIN")) {
      return true;
    }

    String userId = securityIdentity.getPrincipal().getName();
    return resourceRepository.isOwner(resourceId, userId);
  }
}
```

## 入力バリデーション

### Beanバリデーション

```java
// 悪い例: バリデーションなし
@POST
public Response createUser(UserDto dto) {
  return Response.ok(userService.create(dto)).build();
}

// 良い例: バリデーション済みDTO
public record CreateUserDto(
    @NotBlank @Size(max = 100) String name,
    @NotBlank @Email String email,
    @NotNull @Min(18) @Max(150) Integer age,
    @Pattern(regexp = "^\\+?[1-9]\\d{1,14}$") String phone
) {}

@POST
@Path("/users")
public Response createUser(@Valid CreateUserDto dto) {
  User user = userService.create(dto);
  return Response.status(Response.Status.CREATED).entity(user).build();
}
```

### カスタムバリデーター

```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = UsernameValidator.class)
public @interface ValidUsername {
  String message() default "Invalid username format";
  Class<?>[] groups() default {};
  Class<? extends Payload>[] payload() default {};
}

public class UsernameValidator implements ConstraintValidator<ValidUsername, String> {
  @Override
  public boolean isValid(String value, ConstraintValidatorContext context) {
    if (value == null) return false;
    return value.matches("^[a-zA-Z0-9_-]{3,20}$");
  }
}

// 使用例
public record CreateUserDto(
    @ValidUsername String username,
    @NotBlank @Email String email
) {}
```

## SQLインジェクション対策

### Panacheアクティブレコード（デフォルトで安全）

```java
// 良い例: Panacheを使ったパラメーター化クエリ
List<User> users = User.list("email = ?1 and active = ?2", email, true);

Optional<User> user = User.find("username", username).firstResultOptional();

// 良い例: 名前付きパラメーター
List<User> users = User.list("email = :email and age > :minAge", 
    Parameters.with("email", email).and("minAge", 18));
```

### ネイティブクエリ（パラメーターを使用する）

```java
// 悪い例: 文字列結合
@Query(value = "SELECT * FROM users WHERE name = '" + name + "'", nativeQuery = true)

// 良い例: パラメーター化ネイティブクエリ
@Entity
public class User extends PanacheEntity {
  public static List<User> findByEmailNative(String email) {
    return getEntityManager()
        .createNativeQuery("SELECT * FROM users WHERE email = :email", User.class)
        .setParameter("email", email)
        .getResultList();
  }
}
```

## パスワードハッシュ化

```java
@ApplicationScoped
public class PasswordService {
  
  public String hash(String plainPassword) {
    return BcryptUtil.bcryptHash(plainPassword);
  }

  public boolean verify(String plainPassword, String hashedPassword) {
    return BcryptUtil.matches(plainPassword, hashedPassword);
  }
}

// サービスでの使用
@ApplicationScoped
public class UserService {
  @Inject
  PasswordService passwordService;

  @Transactional
  public User register(CreateUserDto dto) {
    String hashedPassword = passwordService.hash(dto.password());
    User user = new User();
    user.email = dto.email();
    user.password = hashedPassword;
    user.persist();
    return user;
  }

  public boolean authenticate(String email, String password) {
    return User.find("email", email)
        .firstResultOptional()
        .map(u -> passwordService.verify(password, u.password))
        .orElse(false);
  }
}
```

## CORS設定

```properties
# application.properties
quarkus.http.cors=true
quarkus.http.cors.origins=https://app.example.com,https://admin.example.com
quarkus.http.cors.methods=GET,POST,PUT,DELETE
quarkus.http.cors.headers=accept,authorization,content-type,x-requested-with
quarkus.http.cors.exposed-headers=Content-Disposition
quarkus.http.cors.access-control-max-age=24H
quarkus.http.cors.access-control-allow-credentials=true
```

## シークレット管理

```properties
# application.properties - シークレットはここに書かない

# 環境変数を使用する
quarkus.datasource.username=${DB_USER}
quarkus.datasource.password=${DB_PASSWORD}
quarkus.oidc.credentials.secret=${OIDC_CLIENT_SECRET}

# またはVaultを使用する
quarkus.vault.url=https://vault.example.com
quarkus.vault.authentication.kubernetes.role=my-role
```

### HashiCorp Vault連携

```java
@ApplicationScoped
public class SecretService {
  
  @ConfigProperty(name = "api-key")
  String apiKey; // Vaultから取得

  public String getSecret(String key) {
    return ConfigProvider.getConfig().getValue(key, String.class);
  }
}
```

## レート制限

**セキュリティに関する注意**: `X-Forwarded-For` を直接使用しないでください。クライアントによる偽装が可能です。
サーブレットリクエストから実際のリモートアドレスを使用するか、利用可能な場合は認証済みのID（APIキー、JWTサブジェクト）を使用してください。

```java
@ApplicationScoped
public class RateLimitFilter implements ContainerRequestFilter {
  private final Map<String, RateLimiter> limiters = new ConcurrentHashMap<>();

  @Inject
  HttpServletRequest servletRequest;

  @Override
  public void filter(ContainerRequestContext requestContext) {
    String clientId = getClientIdentifier();
    RateLimiter limiter = limiters.computeIfAbsent(clientId, 
        k -> RateLimiter.create(100.0)); // 毎秒100リクエスト

    if (!limiter.tryAcquire()) {
      requestContext.abortWith(
          Response.status(429)
              .entity(Map.of("error", "Too many requests"))
              .build()
      );
    }
  }

  private String getClientIdentifier() {
    // コンテナが提供するリモートアドレスを使用する（X-Forwarded-Forは不可）。
    // 信頼されたプロキシ配下の場合は quarkus.http.proxy.proxy-address-forwarding=true を設定すると
    // getRemoteAddr() が実際のクライアントIPを返す。
    return servletRequest.getRemoteAddr();
  }
}
```

## セキュリティヘッダー

```java
@Provider
public class SecurityHeadersFilter implements ContainerResponseFilter {
  
  @Override
  public void filter(ContainerRequestContext request, ContainerResponseContext response) {
    MultivaluedMap<String, Object> headers = response.getHeaders();
    
    // クリックジャッキング対策
    headers.putSingle("X-Frame-Options", "DENY");
    
    // XSS対策
    headers.putSingle("X-Content-Type-Options", "nosniff");
    headers.putSingle("X-XSS-Protection", "1; mode=block");
    
    // HSTS
    headers.putSingle("Strict-Transport-Security", "max-age=31536000; includeSubDomains");
    
    // CSP — script-srcに 'unsafe-inline' を使うとXSS対策が無効化されるため避ける。
    // 代わりにnonceまたはハッシュを使用する。CSSフレームワークが必要とする場合、
    // style-srcの 'unsafe-inline' は許容されるが、可能な限りnonceを使用すること。
    headers.putSingle("Content-Security-Policy", 
        "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'");
  }
}
```

## 監査ログ

```java
@ApplicationScoped
public class AuditService {
  private static final Logger LOG = Logger.getLogger(AuditService.class);

  @Inject
  SecurityIdentity securityIdentity;

  public void logAccess(String resource, String action) {
    String user = securityIdentity.isAnonymous() 
        ? "anonymous" 
        : securityIdentity.getPrincipal().getName();
    
    LOG.infof("AUDIT: user=%s action=%s resource=%s timestamp=%s", 
        user, action, resource, Instant.now());
  }
}

// リソースでの使用例
@Path("/api/sensitive")
public class SensitiveResource {
  @Inject
  AuditService auditService;

  @GET
  @RolesAllowed("ADMIN")
  public Response getData() {
    auditService.logAccess("sensitive-data", "READ");
    return Response.ok(data).build();
  }
}
```

## 依存関係セキュリティスキャン

```bash
# Maven
mvn org.owasp:dependency-check-maven:check

# Gradle
./gradlew dependencyCheckAnalyze

# Quarkus拡張機能の確認
quarkus extension list --installable
```

## ベストプラクティス

- 本番環境では必ずHTTPSを使用する
- ステートレスな認証のためにJWTまたはOIDCを有効にする
- 宣言的な認可には `@RolesAllowed` を使用する
- Bean Validationですべての入力を検証する
- パスワードはBCryptでハッシュ化する（平文は絶対に禁止）
- シークレットはVaultまたは環境変数に保存する
- SQLインジェクション防止のためにパラメーター化クエリを使用する
- すべてのレスポンスにセキュリティヘッダーを追加する
- 公開エンドポイントにはレート制限を実装する
- 機密性の高い操作は監査ログに記録する
- 依存関係を常に最新に保ち、CVEをスキャンする
- プログラマティックなチェックには `SecurityIdentity` を使用する
- 適切なCORSポリシーを設定する
- 認証と認可のパスをテストする
