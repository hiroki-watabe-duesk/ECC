---
name: cpp-testing
description: C++ のテストを作成・更新・修正する場合、GoogleTest/CTest を設定する場合、失敗またはフレイキーなテストを診断する場合、あるいはカバレッジ/サニタイザーを追加する場合にのみ使用する。
origin: ECC
---

# C++ テスト（エージェントスキル）

CMake/CTest を使った GoogleTest/GoogleMock によるモダン C++（C++17/20）のエージェント向けテストワークフロー。

## 使用するタイミング

- 新しい C++ テストの作成または既存テストの修正
- C++ コンポーネントの単体/統合テストカバレッジの設計
- テストカバレッジ、CI ゲーティング、リグレッション保護の追加
- 一貫した実行のための CMake/CTest ワークフローの設定
- テストの失敗やフレイキーな動作の調査
- メモリ/競合診断のためのサニタイザーの有効化

### 使用しないタイミング

- テストを変更しない新機能の実装
- テストカバレッジや失敗とは無関係な大規模リファクタリング
- テストのリグレッションを検証しないパフォーマンスチューニング
- C++ 以外のプロジェクトまたはテスト以外のタスク

## 基本概念

- **TDD ループ**: red → green → refactor（テストを先に書き、最小限の修正をしてからクリーンアップ）。
- **分離**: グローバル状態よりも依存性注入とフェイクを優先する。
- **テストレイアウト**: `tests/unit`、`tests/integration`、`tests/testdata`。
- **モック vs フェイク**: インタラクションにはモックを、ステートフルな動作にはフェイクを使用する。
- **CTest ディスカバリ**: 安定したテスト検出のために `gtest_discover_tests()` を使用する。
- **CI シグナル**: まずサブセットを実行し、次に `--output-on-failure` でフルスイートを実行する。

## TDD ワークフロー

RED → GREEN → REFACTOR のループに従う:

1. **RED**: 新しい動作を捉える失敗するテストを書く
2. **GREEN**: テストを通過させる最小限の変更を実装する
3. **REFACTOR**: テストが緑のままクリーンアップする

```cpp
// tests/add_test.cpp
#include <gtest/gtest.h>

int Add(int a, int b); // Provided by production code.

TEST(AddTest, AddsTwoNumbers) { // RED
  EXPECT_EQ(Add(2, 3), 5);
}

// src/add.cpp
int Add(int a, int b) { // GREEN
  return a + b;
}

// REFACTOR: simplify/rename once tests pass
```

## コード例

### 基本的な単体テスト（gtest）

```cpp
// tests/calculator_test.cpp
#include <gtest/gtest.h>

int Add(int a, int b); // Provided by production code.

TEST(CalculatorTest, AddsTwoNumbers) {
    EXPECT_EQ(Add(2, 3), 5);
}
```

### フィクスチャ（gtest）

```cpp
// tests/user_store_test.cpp
// Pseudocode stub: replace UserStore/User with project types.
#include <gtest/gtest.h>
#include <memory>
#include <optional>
#include <string>

struct User { std::string name; };
class UserStore {
public:
    explicit UserStore(std::string /*path*/) {}
    void Seed(std::initializer_list<User> /*users*/) {}
    std::optional<User> Find(const std::string &/*name*/) { return User{"alice"}; }
};

class UserStoreTest : public ::testing::Test {
protected:
    void SetUp() override {
        store = std::make_unique<UserStore>(":memory:");
        store->Seed({{"alice"}, {"bob"}});
    }

    std::unique_ptr<UserStore> store;
};

TEST_F(UserStoreTest, FindsExistingUser) {
    auto user = store->Find("alice");
    ASSERT_TRUE(user.has_value());
    EXPECT_EQ(user->name, "alice");
}
```

### モック（gmock）

```cpp
// tests/notifier_test.cpp
#include <gmock/gmock.h>
#include <gtest/gtest.h>
#include <string>

class Notifier {
public:
    virtual ~Notifier() = default;
    virtual void Send(const std::string &message) = 0;
};

class MockNotifier : public Notifier {
public:
    MOCK_METHOD(void, Send, (const std::string &message), (override));
};

class Service {
public:
    explicit Service(Notifier &notifier) : notifier_(notifier) {}
    void Publish(const std::string &message) { notifier_.Send(message); }

private:
    Notifier &notifier_;
};

TEST(ServiceTest, SendsNotifications) {
    MockNotifier notifier;
    Service service(notifier);

    EXPECT_CALL(notifier, Send("hello")).Times(1);
    service.Publish("hello");
}
```

### CMake/CTest クイックスタート

```cmake
# CMakeLists.txt (excerpt)
cmake_minimum_required(VERSION 3.20)
project(example LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

include(FetchContent)
# Prefer project-locked versions. If using a tag, use a pinned version per project policy.
set(GTEST_VERSION v1.17.0) # Adjust to project policy.
FetchContent_Declare(
  googletest
  # Google Test framework (official repository)
  URL https://github.com/google/googletest/archive/refs/tags/${GTEST_VERSION}.zip
)
FetchContent_MakeAvailable(googletest)

add_executable(example_tests
  tests/calculator_test.cpp
  src/calculator.cpp
)
target_link_libraries(example_tests GTest::gtest GTest::gmock GTest::gtest_main)

enable_testing()
include(GoogleTest)
gtest_discover_tests(example_tests)
```

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build -j
ctest --test-dir build --output-on-failure
```

## テストの実行

```bash
ctest --test-dir build --output-on-failure
ctest --test-dir build -R ClampTest
ctest --test-dir build -R "UserStoreTest.*" --output-on-failure
```

```bash
./build/example_tests --gtest_filter=ClampTest.*
./build/example_tests --gtest_filter=UserStoreTest.FindsExistingUser
```

## 失敗のデバッグ

1. gtest フィルターで失敗した単一テストを再実行する。
2. 失敗したアサーションの周囲にスコープ付きロギングを追加する。
3. サニタイザーを有効にして再実行する。
4. 根本原因が修正されたらフルスイートに拡大する。

## カバレッジ

グローバルフラグの代わりにターゲットレベルの設定を優先する。

```cmake
option(ENABLE_COVERAGE "Enable coverage flags" OFF)

if(ENABLE_COVERAGE)
  if(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
    target_compile_options(example_tests PRIVATE --coverage)
    target_link_options(example_tests PRIVATE --coverage)
  elseif(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    target_compile_options(example_tests PRIVATE -fprofile-instr-generate -fcoverage-mapping)
    target_link_options(example_tests PRIVATE -fprofile-instr-generate)
  endif()
endif()
```

GCC + gcov + lcov:

```bash
cmake -S . -B build-cov -DENABLE_COVERAGE=ON
cmake --build build-cov -j
ctest --test-dir build-cov
lcov --capture --directory build-cov --output-file coverage.info
lcov --remove coverage.info '/usr/*' --output-file coverage.info
genhtml coverage.info --output-directory coverage
```

Clang + llvm-cov:

```bash
cmake -S . -B build-llvm -DENABLE_COVERAGE=ON -DCMAKE_CXX_COMPILER=clang++
cmake --build build-llvm -j
LLVM_PROFILE_FILE="build-llvm/default.profraw" ctest --test-dir build-llvm
llvm-profdata merge -sparse build-llvm/default.profraw -o build-llvm/default.profdata
llvm-cov report build-llvm/example_tests -instr-profile=build-llvm/default.profdata
```

## サニタイザー

```cmake
option(ENABLE_ASAN "Enable AddressSanitizer" OFF)
option(ENABLE_UBSAN "Enable UndefinedBehaviorSanitizer" OFF)
option(ENABLE_TSAN "Enable ThreadSanitizer" OFF)

if(ENABLE_ASAN)
  add_compile_options(-fsanitize=address -fno-omit-frame-pointer)
  add_link_options(-fsanitize=address)
endif()
if(ENABLE_UBSAN)
  add_compile_options(-fsanitize=undefined -fno-omit-frame-pointer)
  add_link_options(-fsanitize=undefined)
endif()
if(ENABLE_TSAN)
  add_compile_options(-fsanitize=thread)
  add_link_options(-fsanitize=thread)
endif()
```

## フレイキーテストのガードレール

- 同期に `sleep` を使わない。条件変数またはラッチを使用する。
- テストごとに一意な一時ディレクトリを作成し、必ずクリーンアップする。
- 単体テストで実際の時刻、ネットワーク、ファイルシステムへの依存を避ける。
- ランダム化された入力には決定論的なシードを使用する。

## ベストプラクティス

### すべきこと

- テストを決定論的かつ分離されたものに保つ
- グローバル状態より依存性注入を優先する
- 前提条件には `ASSERT_*`、複数チェックには `EXPECT_*` を使用する
- CTest のラベルまたはディレクトリで単体テストと統合テストを分離する
- CI でメモリと競合検出のためにサニタイザーを実行する

### すべきでないこと

- 単体テストで実際の時刻やネットワークに依存しない
- 条件変数を使える場面で同期に sleep を使わない
- 単純な値オブジェクトを過剰にモックしない
- 重要でないログに脆弱な文字列マッチングを使わない

### よくある落とし穴

- **固定の一時パスの使用** → テストごとに一意な一時ディレクトリを生成してクリーンアップする。
- **ウォールクロック時刻への依存** → クロックを注入するか偽の時刻ソースを使用する。
- **フレイキーな並行性テスト** → 条件変数/ラッチと境界付き待機を使用する。
- **隠れたグローバル状態** → フィクスチャでグローバル状態をリセットするかグローバルを排除する。
- **過剰なモック** → ステートフルな動作にはフェイクを優先し、インタラクションのみをモックする。
- **サニタイザーの実行漏れ** → CI に ASan/UBSan/TSan ビルドを追加する。
- **デバッグビルドのみでのカバレッジ** → カバレッジターゲットが一貫したフラグを使用していることを確認する。

## 付録（任意）: ファジング / プロパティテスト

プロジェクトがすでに LLVM/libFuzzer またはプロパティテストライブラリをサポートしている場合のみ使用する。

- **libFuzzer**: I/O が最小限の純粋関数に最適。
- **RapidCheck**: 不変条件を検証するプロパティベーステスト。

最小限の libFuzzer ハーネス（擬似コード: ParseConfig を置き換える）:

```cpp
#include <cstddef>
#include <cstdint>
#include <string>

extern "C" int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
    std::string input(reinterpret_cast<const char *>(data), size);
    // ParseConfig(input); // project function
    return 0;
}
```

## GoogleTest の代替

- **Catch2**: ヘッダーオンリー、表現力豊かなマッチャー
- **doctest**: 軽量でコンパイルオーバーヘッドが最小
