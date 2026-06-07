---
name: dart-flutter-patterns
description: Null安全、不変ステート、非同期合成、ウィジェットアーキテクチャ、主要なステート管理フレームワーク（BLoC、Riverpod、Provider）、GoRouterナビゲーション、Dioネットワーク、Freezedコード生成、クリーンアーキテクチャを網羅した本番対応のDartおよびFlutterパターン。
origin: ECC
---

# Dart/Flutterパターン

## 使用するタイミング

このスキルを使用する状況:
- 新しいFlutter機能を開始する際、ステート管理・ナビゲーション・データアクセスの慣用パターンが必要な場合
- Dartコードをレビューまたは作成する際、null安全・シールド型・非同期合成についてのガイダンスが必要な場合
- 新しいFlutterプロジェクトをセットアップする際、BLoC・Riverpod・Providerのいずれかを選択する場合
- セキュアなHTTPクライアント、WebView統合、またはローカルストレージを実装する場合
- Flutterウィジェット・Cubit・RiverpodプロバイダーのテストをWriteする場合
- 認証ガード付きのGoRouterを設定する場合

## 動作の仕組み

このスキルは懸念別に整理されたコピー＆ペースト可能なDart/Flutterコードパターンを提供する:
1. **Null安全** — `!` を避け、`?.`/`??`/パターンマッチングを優先
2. **不変ステート** — シールドクラス、`freezed`、`copyWith`
3. **非同期合成** — 並行 `Future.wait`、`await` 後の安全な `BuildContext`
4. **ウィジェットアーキテクチャ** — メソッドではなくクラスに抽出、`const` 伝播、スコープ付き再ビルド
5. **ステート管理** — BLoC/Cubitイベント、Riverpodノティファイアと派生プロバイダー
6. **ナビゲーション** — `refreshListenable` を介したリアクティブな認証ガード付きGoRouter
7. **ネットワーキング** — インターセプター付きDio、ワンタイムリトライガード付きトークンリフレッシュ
8. **エラーハンドリング** — グローバルキャプチャ、`ErrorWidget.builder`、Crashlyticsの配線
9. **テスト** — ユニット（BLocテスト）、ウィジェット（ProviderScopeオーバーライド）、モックよりフェイク

## 例

```dart
// シールドステート — 不可能な状態を防ぐ
sealed class AsyncState<T> {}
final class Loading<T> extends AsyncState<T> {}
final class Success<T> extends AsyncState<T> { final T data; const Success(this.data); }
final class Failure<T> extends AsyncState<T> { final Object error; const Failure(this.error); }

// リアクティブな認証リダイレクト付きGoRouter
final router = GoRouter(
  refreshListenable: GoRouterRefreshStream(authCubit.stream),
  redirect: (context, state) {
    final authed = context.read<AuthCubit>().state is AuthAuthenticated;
    if (!authed && !state.matchedLocation.startsWith('/login')) return '/login';
    return null;
  },
  routes: [...],
);

// 安全なfirstWhereOrNull付きRiverpod派生プロバイダー
@riverpod
double cartTotal(Ref ref) {
  final cart = ref.watch(cartNotifierProvider);
  final products = ref.watch(productsProvider).valueOrNull ?? [];
  return cart.fold(0.0, (total, item) {
    final product = products.firstWhereOrNull((p) => p.id == item.productId);
    return total + (product?.price ?? 0) * item.quantity;
  });
}
```

---

DartおよびFlutterアプリケーションのための実践的で本番対応のパターン。可能な限りライブラリに依存しない設計で、最も一般的なエコシステムパッケージを明示的にカバーする。

---

## 1. Null安全の基礎

### バン演算子よりパターンマッチングを優先

```dart
// BAD — nullの場合実行時クラッシュ
final name = user!.name;

// GOOD — フォールバックを提供
final name = user?.name ?? 'Unknown';

// GOOD — Dart 3パターンマッチング（複雑なケースに推奨）
final display = switch (user) {
  User(:final name, :final email) => '$name <$email>',
  null => 'Guest',
};

// GOOD — 早期リターンのガード
String getUserName(User? user) {
  if (user == null) return 'Unknown';
  return user.name; // チェック後non-nullに昇格
}
```

### `late` の過剰使用を避ける

```dart
// BAD — nullエラーを実行時まで先送り
late String userId;

// GOOD — 明示的な初期化付きのnullable
String? userId;

// OK — 最初のアクセス前に初期化が保証されている場合のみlateを使用
// （例: 任意のウィジェットインタラクションの前の initState() 内）
late final AnimationController _controller;

@override
void initState() {
  super.initState();
  _controller = AnimationController(vsync: this, duration: const Duration(milliseconds: 300));
}
```

---

## 2. 不変ステート

### ステート階層のシールドクラス

```dart
sealed class UserState {}

final class UserInitial extends UserState {}

final class UserLoading extends UserState {}

final class UserLoaded extends UserState {
  const UserLoaded(this.user);
  final User user;
}

final class UserError extends UserState {
  const UserError(this.message);
  final String message;
}

// 網羅的なswitch — コンパイラがすべての分岐を強制
Widget buildFrom(UserState state) => switch (state) {
  UserInitial() => const SizedBox.shrink(),
  UserLoading() => const CircularProgressIndicator(),
  UserLoaded(:final user) => UserCard(user: user),
  UserError(:final message) => ErrorText(message),
};
```

### ボイラープレートフリーな不変性のためのFreezed

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'user.freezed.dart';
part 'user.g.dart';

@freezed
class User with _$User {
  const factory User({
    required String id,
    required String name,
    required String email,
    @Default(false) bool isAdmin,
  }) = _User;

  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
}

// 使用例
final user = User(id: '1', name: 'Alice', email: 'alice@example.com');
final updated = user.copyWith(name: 'Alice Smith'); // 不変更新
final json = user.toJson();
final fromJson = User.fromJson(json);
```

---

## 3. 非同期合成

### Future.waitを使った構造化並行処理

```dart
Future<DashboardData> loadDashboard(UserRepository users, OrderRepository orders) async {
  // 並行実行 — 順番にawaitしない
  final (userList, orderList) = await (
    users.getAll(),
    orders.getRecent(),
  ).wait; // Dart 3レコードの分割代入 + Future.wait拡張

  return DashboardData(users: userList, orders: orderList);
}
```

### ストリームパターン

```dart
// リポジトリはライブデータのためのリアクティブストリームを公開する
Stream<List<Item>> watchCartItems() => _db
    .watchTable('cart_items')
    .map((rows) => rows.map(Item.fromRow).toList());

// ウィジェットレイヤー — 宣言的、手動サブスクリプション不要
StreamBuilder<List<Item>>(
  stream: cartRepository.watchCartItems(),
  builder: (context, snapshot) => switch (snapshot) {
    AsyncSnapshot(connectionState: ConnectionState.waiting) =>
        const CircularProgressIndicator(),
    AsyncSnapshot(:final error?) => ErrorWidget(error.toString()),
    AsyncSnapshot(:final data?) => CartList(items: data),
    _ => const SizedBox.shrink(),
  },
)
```

### await後のBuildContext

```dart
// 重要 — StatefulWidgetでは任意のawaitの後にmountedを確認すること
Future<void> _handleSubmit() async {
  setState(() => _isLoading = true);
  try {
    await authService.login(_email, _password);
    if (!mounted) return; // ← contextを使用する前のガード
    context.go('/home');
  } on AuthException catch (e) {
    if (!mounted) return;
    ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text(e.message)));
  } finally {
    if (mounted) setState(() => _isLoading = false);
  }
}
```

---

## 4. ウィジェットアーキテクチャ

### メソッドではなくクラスに抽出

```dart
// BAD — ウィジェットを返すプライベートメソッド、最適化を妨げる
Widget _buildHeader() {
  return Container(
    padding: const EdgeInsets.all(16),
    child: Text(title, style: Theme.of(context).textTheme.headlineMedium),
  );
}

// GOOD — 独立したウィジェットクラス、constとエレメント再利用が可能
class _PageHeader extends StatelessWidget {
  const _PageHeader(this.title);
  final String title;

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.all(16),
      child: Text(title, style: Theme.of(context).textTheme.headlineMedium),
    );
  }
}
```

### const伝播

```dart
// BAD — 再ビルドのたびに新しいインスタンス
child: Padding(
  padding: EdgeInsets.all(16.0),       // constでない
  child: Icon(Icons.home, size: 24.0), // constでない
)

// GOOD — constが再ビルドの伝播を停止
child: const Padding(
  padding: EdgeInsets.all(16.0),
  child: Icon(Icons.home, size: 24.0),
)
```

### スコープ付き再ビルド

```dart
// BAD — カウンターが変わるたびにページ全体が再ビルドされる
class CounterPage extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final count = ref.watch(counterProvider); // すべてを再ビルド
    return Scaffold(
      body: Column(children: [
        const ExpensiveHeader(), // 不必要に再ビルドされる
        Text('$count'),
        const ExpensiveFooter(), // 不必要に再ビルドされる
      ]),
    );
  }
}

// GOOD — 再ビルドする部分を分離
class CounterPage extends StatelessWidget {
  const CounterPage({super.key});

  @override
  Widget build(BuildContext context) {
    return const Scaffold(
      body: Column(children: [
        ExpensiveHeader(),        // 再ビルドされない（const）
        _CounterDisplay(),        // これだけ再ビルドされる
        ExpensiveFooter(),        // 再ビルドされない（const）
      ]),
    );
  }
}

class _CounterDisplay extends ConsumerWidget {
  const _CounterDisplay();

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final count = ref.watch(counterProvider);
    return Text('$count');
  }
}
```

---

## 5. ステート管理: BLoC/Cubit

```dart
// Cubit — 同期または単純な非同期ステート
class AuthCubit extends Cubit<AuthState> {
  AuthCubit(this._authService) : super(const AuthState.initial());
  final AuthService _authService;

  Future<void> login(String email, String password) async {
    emit(const AuthState.loading());
    try {
      final user = await _authService.login(email, password);
      emit(AuthState.authenticated(user));
    } on AuthException catch (e) {
      emit(AuthState.error(e.message));
    }
  }

  void logout() {
    _authService.logout();
    emit(const AuthState.initial());
  }
}

// ウィジェット内
BlocBuilder<AuthCubit, AuthState>(
  builder: (context, state) => switch (state) {
    AuthInitial() => const LoginForm(),
    AuthLoading() => const CircularProgressIndicator(),
    AuthAuthenticated(:final user) => HomePage(user: user),
    AuthError(:final message) => ErrorView(message: message),
  },
)
```

---

## 6. ステート管理: Riverpod

```dart
// 自動破棄非同期プロバイダー
@riverpod
Future<List<Product>> products(Ref ref) async {
  final repo = ref.watch(productRepositoryProvider);
  return repo.getAll();
}

// 複雑なミューテーションを持つノティファイアー
@riverpod
class CartNotifier extends _$CartNotifier {
  @override
  List<CartItem> build() => [];

  void add(Product product) {
    final existing = state.where((i) => i.productId == product.id).firstOrNull;
    if (existing != null) {
      state = [
        for (final item in state)
          if (item.productId == product.id) item.copyWith(quantity: item.quantity + 1)
          else item,
      ];
    } else {
      state = [...state, CartItem(productId: product.id, quantity: 1)];
    }
  }

  void remove(String productId) =>
      state = state.where((i) => i.productId != productId).toList();

  void clear() => state = [];
}

// 派生プロバイダー（セレクターパターン）
@riverpod
int cartCount(Ref ref) => ref.watch(cartNotifierProvider).length;

@riverpod
double cartTotal(Ref ref) {
  final cart = ref.watch(cartNotifierProvider);
  final products = ref.watch(productsProvider).valueOrNull ?? [];
  return cart.fold(0.0, (total, item) {
    // firstWhereOrNull（collectionパッケージ）はproductが見つからない場合のStateErrorを防ぐ
    final product = products.firstWhereOrNull((p) => p.id == item.productId);
    return total + (product?.price ?? 0) * item.quantity;
  });
}
```

---

## 7. GoRouterを使ったナビゲーション

```dart
final router = GoRouter(
  initialLocation: '/',
  // refreshListenableは認証ステートが変わるたびにリダイレクトを再評価する
  refreshListenable: GoRouterRefreshStream(authCubit.stream),
  redirect: (context, state) {
    final isLoggedIn = context.read<AuthCubit>().state is AuthAuthenticated;
    final isGoingToLogin = state.matchedLocation == '/login';
    if (!isLoggedIn && !isGoingToLogin) return '/login';
    if (isLoggedIn && isGoingToLogin) return '/';
    return null;
  },
  routes: [
    GoRoute(path: '/login', builder: (_, __) => const LoginPage()),
    ShellRoute(
      builder: (context, state, child) => AppShell(child: child),
      routes: [
        GoRoute(path: '/', builder: (_, __) => const HomePage()),
        GoRoute(
          path: '/products/:id',
          builder: (context, state) =>
              ProductDetailPage(id: state.pathParameters['id']!),
        ),
      ],
    ),
  ],
);
```

---

## 8. DioによるHTTP

```dart
final dio = Dio(BaseOptions(
  baseUrl: const String.fromEnvironment('API_URL'),
  connectTimeout: const Duration(seconds: 10),
  receiveTimeout: const Duration(seconds: 30),
  headers: {'Content-Type': 'application/json'},
));

// 認証インターセプターを追加
dio.interceptors.add(InterceptorsWrapper(
  onRequest: (options, handler) async {
    final token = await secureStorage.read(key: 'auth_token');
    if (token != null) options.headers['Authorization'] = 'Bearer $token';
    handler.next(options);
  },
  onError: (error, handler) async {
    // 無限リトライループを防ぐ: リクエストごとに1回のみリフレッシュを試みる
    final isRetry = error.requestOptions.extra['_isRetry'] == true;
    if (!isRetry && error.response?.statusCode == 401) {
      final refreshed = await attemptTokenRefresh();
      if (refreshed) {
        error.requestOptions.extra['_isRetry'] = true;
        return handler.resolve(await dio.fetch(error.requestOptions));
      }
    }
    handler.next(error);
  },
));

// Dioを使うリポジトリ
class UserApiDataSource {
  const UserApiDataSource(this._dio);
  final Dio _dio;

  Future<User> getById(String id) async {
    final response = await _dio.get<Map<String, dynamic>>('/users/$id');
    return User.fromJson(response.data!);
  }
}
```

---

## 9. エラーハンドリングアーキテクチャ

```dart
// グローバルエラーキャプチャ — main()で設定する
void main() {
  FlutterError.onError = (details) {
    FlutterError.presentError(details);
    crashlytics.recordFlutterFatalError(details);
  };

  PlatformDispatcher.instance.onError = (error, stack) {
    crashlytics.recordError(error, stack, fatal: true);
    return true;
  };

  runApp(const App());
}

// 本番向けカスタムErrorWidget
class App extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    ErrorWidget.builder = (details) => ProductionErrorWidget(details);
    return MaterialApp.router(routerConfig: router);
  }
}
```

---

## 10. テストクイックリファレンス

```dart
// ユニットテスト — ユースケース
test('GetUserUseCase returns null for missing user', () async {
  final repo = FakeUserRepository();
  final useCase = GetUserUseCase(repo);
  expect(await useCase('missing-id'), isNull);
});

// BLocテスト
blocTest<AuthCubit, AuthState>(
  'emits loading then error on failed login',
  build: () => AuthCubit(FakeAuthService(throwsOn: 'login')),
  act: (cubit) => cubit.login('user@test.com', 'wrong'),
  expect: () => [const AuthState.loading(), isA<AuthError>()],
);

// ウィジェットテスト
testWidgets('CartBadge shows item count', (tester) async {
  await tester.pumpWidget(
    ProviderScope(
      overrides: [cartNotifierProvider.overrideWith(() => FakeCartNotifier(count: 3))],
      child: const MaterialApp(home: CartBadge()),
    ),
  );
  expect(find.text('3'), findsOneWidget);
});
```

---

## 参考資料

- [Effective Dart: Design](https://dart.dev/effective-dart/design)
- [Flutter Performance Best Practices](https://docs.flutter.dev/perf/best-practices)
- [Riverpod Documentation](https://riverpod.dev/)
- [BLoC Library](https://bloclibrary.dev/)
- [GoRouter](https://pub.dev/packages/go_router)
- [Freezed](https://pub.dev/packages/freezed)
- スキル: `flutter-dart-code-review` — 包括的なレビューチェックリスト
- ルール: `rules/dart/` — コーディングスタイル、パターン、セキュリティ、テスト、フック
