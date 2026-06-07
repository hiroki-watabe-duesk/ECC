---
name: compose-multiplatform-patterns
description: KMP プロジェクト向け Compose Multiplatform および Jetpack Compose パターン — 状態管理、ナビゲーション、テーマ、パフォーマンス、プラットフォーム固有 UI。
origin: ECC
---

# Compose Multiplatform パターン

Compose Multiplatform と Jetpack Compose を使って Android、iOS、Desktop、Web で共有 UI を構築するためのパターン。状態管理、ナビゲーション、テーマ、パフォーマンスを網羅する。

## 有効にするタイミング

- Compose UI（Jetpack Compose または Compose Multiplatform）を構築するとき
- ViewModel と Compose 状態で UI 状態を管理するとき
- KMP または Android プロジェクトでナビゲーションを実装するとき
- 再利用可能な Composable とデザインシステムを設計するとき
- リコンポジションとレンダリングパフォーマンスを最適化するとき

## 状態管理

### ViewModel + 単一状態オブジェクト

画面状態に単一のデータクラスを使用する。`StateFlow` として公開し、Compose で収集する:

```kotlin
data class ItemListState(
    val items: List<Item> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null,
    val searchQuery: String = ""
)

class ItemListViewModel(
    private val getItems: GetItemsUseCase
) : ViewModel() {
    private val _state = MutableStateFlow(ItemListState())
    val state: StateFlow<ItemListState> = _state.asStateFlow()

    fun onSearch(query: String) {
        _state.update { it.copy(searchQuery = query) }
        loadItems(query)
    }

    private fun loadItems(query: String) {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }
            getItems(query).fold(
                onSuccess = { items -> _state.update { it.copy(items = items, isLoading = false) } },
                onFailure = { e -> _state.update { it.copy(error = e.message, isLoading = false) } }
            )
        }
    }
}
```

### Compose での状態収集

```kotlin
@Composable
fun ItemListScreen(viewModel: ItemListViewModel = koinViewModel()) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    ItemListContent(
        state = state,
        onSearch = viewModel::onSearch
    )
}

@Composable
private fun ItemListContent(
    state: ItemListState,
    onSearch: (String) -> Unit
) {
    // ステートレス Composable — プレビューとテストが容易
}
```

### イベントシンクパターン

複雑な画面では、複数のコールバックラムダの代わりにイベント用のシールドインターフェースを使用する:

```kotlin
sealed interface ItemListEvent {
    data class Search(val query: String) : ItemListEvent
    data class Delete(val itemId: String) : ItemListEvent
    data object Refresh : ItemListEvent
}

// ViewModel 内で
fun onEvent(event: ItemListEvent) {
    when (event) {
        is ItemListEvent.Search -> onSearch(event.query)
        is ItemListEvent.Delete -> deleteItem(event.itemId)
        is ItemListEvent.Refresh -> loadItems(_state.value.searchQuery)
    }
}

// Composable 内 — 多くのラムダの代わりに単一のラムダ
ItemListContent(
    state = state,
    onEvent = viewModel::onEvent
)
```

## ナビゲーション

### 型安全なナビゲーション（Compose Navigation 2.8 以降）

ルートを `@Serializable` オブジェクトとして定義する:

```kotlin
@Serializable data object HomeRoute
@Serializable data class DetailRoute(val id: String)
@Serializable data object SettingsRoute

@Composable
fun AppNavHost(navController: NavHostController = rememberNavController()) {
    NavHost(navController, startDestination = HomeRoute) {
        composable<HomeRoute> {
            HomeScreen(onNavigateToDetail = { id -> navController.navigate(DetailRoute(id)) })
        }
        composable<DetailRoute> { backStackEntry ->
            val route = backStackEntry.toRoute<DetailRoute>()
            DetailScreen(id = route.id)
        }
        composable<SettingsRoute> { SettingsScreen() }
    }
}
```

### ダイアログとボトムシートのナビゲーション

命令的な show/hide の代わりに `dialog()` とオーバーレイパターンを使用する:

```kotlin
NavHost(navController, startDestination = HomeRoute) {
    composable<HomeRoute> { /* ... */ }
    dialog<ConfirmDeleteRoute> { backStackEntry ->
        val route = backStackEntry.toRoute<ConfirmDeleteRoute>()
        ConfirmDeleteDialog(
            itemId = route.itemId,
            onConfirm = { navController.popBackStack() },
            onDismiss = { navController.popBackStack() }
        )
    }
}
```

## Composable の設計

### スロットベース API

柔軟性のためにスロットパラメーターで Composable を設計する:

```kotlin
@Composable
fun AppCard(
    modifier: Modifier = Modifier,
    header: @Composable () -> Unit = {},
    content: @Composable ColumnScope.() -> Unit,
    actions: @Composable RowScope.() -> Unit = {}
) {
    Card(modifier = modifier) {
        Column {
            header()
            Column(content = content)
            Row(horizontalArrangement = Arrangement.End, content = actions)
        }
    }
}
```

### Modifier の順序

Modifier の順序は重要 — この順序で適用する:

```kotlin
Text(
    text = "Hello",
    modifier = Modifier
        .padding(16.dp)          // 1. レイアウト（パディング、サイズ）
        .clip(RoundedCornerShape(8.dp))  // 2. 形状
        .background(Color.White) // 3. 描画（背景、ボーダー）
        .clickable { }           // 4. インタラクション
)
```

## KMP プラットフォーム固有 UI

### プラットフォーム Composable の expect/actual

```kotlin
// commonMain
@Composable
expect fun PlatformStatusBar(darkIcons: Boolean)

// androidMain
@Composable
actual fun PlatformStatusBar(darkIcons: Boolean) {
    val systemUiController = rememberSystemUiController()
    SideEffect { systemUiController.setStatusBarColor(Color.Transparent, darkIcons) }
}

// iosMain
@Composable
actual fun PlatformStatusBar(darkIcons: Boolean) {
    // iOS は UIKit interop または Info.plist でこれを処理する
}
```

## パフォーマンス

### スキップ可能なリコンポジションのための安定型

すべてのプロパティが安定している場合、クラスを `@Stable` または `@Immutable` としてマークする:

```kotlin
@Immutable
data class ItemUiModel(
    val id: String,
    val title: String,
    val description: String,
    val progress: Float
)
```

### `key()` と遅延リストの正しい使用

```kotlin
LazyColumn {
    items(
        items = items,
        key = { it.id }  // 安定したキーでアイテムの再利用とアニメーションが可能
    ) { item ->
        ItemRow(item = item)
    }
}
```

### `derivedStateOf` で読み取りを遅延させる

```kotlin
val listState = rememberLazyListState()
val showScrollToTop by remember {
    derivedStateOf { listState.firstVisibleItemIndex > 5 }
}
```

### リコンポジションでのアロケーションを避ける

```kotlin
// 悪い例 — リコンポジションのたびに新しいラムダとリスト
items.filter { it.isActive }.forEach { ActiveItem(it, onClick = { handle(it) }) }

// 良い例 — 各アイテムにキーを付けてコールバックが正しい行に保持される
val activeItems = remember(items) { items.filter { it.isActive } }
activeItems.forEach { item ->
    key(item.id) {
        ActiveItem(item, onClick = { handle(item) })
    }
}
```

## テーマ

### Material 3 ダイナミックテーマ

```kotlin
@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,
    content: @Composable () -> Unit
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            if (darkTheme) dynamicDarkColorScheme(LocalContext.current)
            else dynamicLightColorScheme(LocalContext.current)
        }
        darkTheme -> darkColorScheme()
        else -> lightColorScheme()
    }

    MaterialTheme(colorScheme = colorScheme, content = content)
}
```

## 避けるべきアンチパターン

- ライフサイクルにより安全な `collectAsStateWithLifecycle` と `MutableStateFlow` があるのに ViewModel で `mutableStateOf` を使う
- `NavController` を Composable の深くに渡す — 代わりにラムダコールバックを渡す
- `@Composable` 関数内で重い計算を行う — ViewModel または `remember {}` に移す
- `LaunchedEffect(Unit)` を ViewModel の init の代替として使う — 一部のセットアップでは設定変更時に再実行される
- Composable パラメーターで新しいオブジェクトインスタンスを作成する — 不必要なリコンポジションを引き起こす

## 参照

スキル `android-clean-architecture` — モジュール構成とレイヤリングについて参照。
スキル `kotlin-coroutines-flows` — コルーチンと Flow パターンについて参照。
