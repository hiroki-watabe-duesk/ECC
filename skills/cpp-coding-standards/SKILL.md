---
name: cpp-coding-standards
description: C++ Core Guidelines（isocpp.github.io）に基づくC++コーディング標準。モダンで安全かつイディオマティックなプラクティスを適用するために、C++コードの作成、レビュー、またはリファクタリング時に使用する。
origin: ECC
---

# C++ コーディング標準（C++ Core Guidelines）

[C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines)から導出されたモダンC++（C++17/20/23）向けの包括的なコーディング標準。型安全性、リソース安全性、不変性、明確さを強制します。

## 使用タイミング

- 新しいC++コードの作成（クラス、関数、テンプレート）
- 既存のC++コードのレビューまたはリファクタリング
- C++プロジェクトでのアーキテクチャ上の決定
- C++コードベース全体での一貫したスタイルの強制
- 言語機能の選択（例：`enum` vs `enum class`、生ポインタ vs スマートポインタ）

### 使用しない場合

- C++以外のプロジェクト
- モダンC++機能を採用できないレガシーCコードベース
- 特定のガイドラインがハードウェア制約と衝突する組み込み/ベアメタルコンテキスト（選択的に適応）

## 横断的原則

これらのテーマはガイドライン全体を通じて繰り返し現れ、基盤を形成します。

1. **どこでもRAII**（P.8、R.1、E.6、CP.20）：リソースのライフタイムをオブジェクトのライフタイムに結びつける
2. **デフォルトで不変**（P.10、Con.1-5、ES.25）：`const`/`constexpr` から始める。可変性は例外
3. **型安全性**（P.4、I.4、ES.46-49、Enum.3）：型システムを使用してコンパイル時にエラーを防ぐ
4. **意図を表現する**（P.3、F.1、NL.1-2、T.10）：名前、型、概念が目的を伝えるべき
5. **複雑さを最小化する**（F.2-3、ES.5、Per.4-5）：シンプルなコードが正しいコード
6. **ポインタセマンティクスより値セマンティクス**（C.10、R.3-5、F.20、CP.31）：値を返すこととスコープ付きオブジェクトを優先する

## 哲学とインターフェース（P.*、I.*）

### 主要ルール

| ルール | 概要 |
|------|---------|
| **P.1** | コードでアイデアを直接表現する |
| **P.3** | 意図を表現する |
| **P.4** | 理想的には、プログラムは静的に型安全であるべき |
| **P.5** | ランタイムチェックよりコンパイル時チェックを優先する |
| **P.8** | リソースをリークしない |
| **P.10** | ミュータブルなデータより不変データを優先する |
| **I.1** | インターフェースを明示的にする |
| **I.2** | non-constのグローバル変数を避ける |
| **I.4** | インターフェースを正確かつ強く型付けする |
| **I.11** | 生ポインタや参照で所有権を転送しない |
| **I.23** | 関数引数の数を少なくする |

### DO

```cpp
// P.10 + I.4: 不変で強く型付けされたインターフェース
struct Temperature {
    double kelvin;
};

Temperature boil(const Temperature& water);
```

### DON'T

```cpp
// 弱いインターフェース: 所有権が不明確、単位が不明確
double boil(double* temp);

// non-constのグローバル変数
int g_counter = 0;  // I.2 違反
```

## 関数（F.*）

### 主要ルール

| ルール | 概要 |
|------|---------|
| **F.1** | 意味のある操作を慎重に命名した関数としてパッケージ化する |
| **F.2** | 関数は単一の論理操作を実行すべき |
| **F.3** | 関数を短くシンプルに保つ |
| **F.4** | 関数がコンパイル時に評価される可能性がある場合、`constexpr` を宣言する |
| **F.6** | 関数がスローしてはならない場合、`noexcept` を宣言する |
| **F.8** | 純粋関数を優先する |
| **F.16** | 「in」パラメーターには、安価にコピーできる型は値で、それ以外は `const&` で渡す |
| **F.20** | 「out」値には、出力パラメーターより戻り値を優先する |
| **F.21** | 複数の「out」値を返すには、structを返すことを優先する |
| **F.43** | ローカルオブジェクトへのポインタや参照を返さない |

### パラメーターの受け渡し

```cpp
// F.16: 安価な型は値で、それ以外はconst&で
void print(int x);                           // 安価: 値で
void analyze(const std::string& data);       // 高コスト: const&で
void transform(std::string s);               // シンク: 値で（移動する）

// F.20 + F.21: 出力パラメーターではなく戻り値
struct ParseResult {
    std::string token;
    int position;
};

ParseResult parse(std::string_view input);   // GOOD: structを返す

// BAD: 出力パラメーター
void parse(std::string_view input,
           std::string& token, int& pos);    // これを避ける
```

### 純粋関数とconstexpr

```cpp
// F.4 + F.8: 可能な場合は純粋でconstexpr
constexpr int factorial(int n) noexcept {
    return (n <= 1) ? 1 : n * factorial(n - 1);
}

static_assert(factorial(5) == 120);
```

### アンチパターン

- 関数から `T&&` を返す（F.45）
- `va_arg` / Cスタイルの可変長引数の使用（F.55）
- 他のスレッドに渡されるラムダで参照キャプチャ（F.53）
- ムーブセマンティクスを抑制する `const T` の返却（F.49）

## クラスとクラス階層（C.*）

### 主要ルール

| ルール | 概要 |
|------|---------|
| **C.2** | 不変条件がある場合は `class` を使用。データメンバーが独立して変化する場合は `struct` |
| **C.9** | メンバーの公開を最小化する |
| **C.20** | デフォルト操作の定義を避けられる場合は避ける（ゼロの法則） |
| **C.21** | コピー/ムーブ/デストラクタのいずれかを定義または `=delete` する場合、すべてを処理する（5の法則） |
| **C.35** | 基底クラスのデストラクタ: パブリック仮想またはプロテクト非仮想 |
| **C.41** | コンストラクタは完全に初期化されたオブジェクトを作成すべき |
| **C.46** | 単一引数コンストラクタに `explicit` を宣言する |
| **C.67** | ポリモーフィッククラスはパブリックのコピー/ムーブを抑制すべき |
| **C.128** | 仮想関数: `virtual`、`override`、`final` のいずれか1つだけを指定する |

### ゼロの法則

```cpp
// C.20: コンパイラに特殊メンバーを生成させる
struct Employee {
    std::string name;
    std::string department;
    int id;
    // デストラクタ、コピー/ムーブコンストラクタ、代入演算子は不要
};
```

### 5の法則

```cpp
// C.21: リソースを管理する必要がある場合、5つすべてを定義する
class Buffer {
public:
    explicit Buffer(std::size_t size)
        : data_(std::make_unique<char[]>(size)), size_(size) {}

    ~Buffer() = default;

    Buffer(const Buffer& other)
        : data_(std::make_unique<char[]>(other.size_)), size_(other.size_) {
        std::copy_n(other.data_.get(), size_, data_.get());
    }

    Buffer& operator=(const Buffer& other) {
        if (this != &other) {
            auto new_data = std::make_unique<char[]>(other.size_);
            std::copy_n(other.data_.get(), other.size_, new_data.get());
            data_ = std::move(new_data);
            size_ = other.size_;
        }
        return *this;
    }

    Buffer(Buffer&&) noexcept = default;
    Buffer& operator=(Buffer&&) noexcept = default;

private:
    std::unique_ptr<char[]> data_;
    std::size_t size_;
};
```

### クラス階層

```cpp
// C.35 + C.128: 仮想デストラクタ、overrideを使用
class Shape {
public:
    virtual ~Shape() = default;
    virtual double area() const = 0;  // C.121: 純粋インターフェース
};

class Circle : public Shape {
public:
    explicit Circle(double r) : radius_(r) {}
    double area() const override { return 3.14159 * radius_ * radius_; }

private:
    double radius_;
};
```

### アンチパターン

- コンストラクタ/デストラクタで仮想関数を呼び出す（C.82）
- 非trivialな型に `memset`/`memcpy` を使用する（C.90）
- 仮想関数とオーバーライドに異なるデフォルト引数を提供する（C.140）
- ムーブ/コピーを抑制する `const` や参照のデータメンバーを作成する（C.12）

## リソース管理（R.*）

### 主要ルール

| ルール | 概要 |
|------|---------|
| **R.1** | RAIIを使用してリソースを自動的に管理する |
| **R.3** | 生ポインタ（`T*`）は非所有 |
| **R.5** | スコープ付きオブジェクトを優先する。不必要にヒープ割り当てしない |
| **R.10** | `malloc()`/`free()` を避ける |
| **R.11** | `new` と `delete` を明示的に呼び出すことを避ける |
| **R.20** | 所有権を表すために `unique_ptr` または `shared_ptr` を使用する |
| **R.21** | 所有権を共有しない限り `shared_ptr` より `unique_ptr` を優先する |
| **R.22** | `shared_ptr` を作成するために `make_shared()` を使用する |

### スマートポインタの使用

```cpp
// R.11 + R.20 + R.21: スマートポインタによるRAII
auto widget = std::make_unique<Widget>("config");  // 唯一の所有権
auto cache  = std::make_shared<Cache>(1024);        // 共有所有権

// R.3: 生ポインタ = 非所有のオブザーバー
void render(const Widget* w) {  // wを所有しない
    if (w) w->draw();
}

render(widget.get());
```

### RAIIパターン

```cpp
// R.1: リソース取得は初期化
class FileHandle {
public:
    explicit FileHandle(const std::string& path)
        : handle_(std::fopen(path.c_str(), "r")) {
        if (!handle_) throw std::runtime_error("Failed to open: " + path);
    }

    ~FileHandle() {
        if (handle_) std::fclose(handle_);
    }

    FileHandle(const FileHandle&) = delete;
    FileHandle& operator=(const FileHandle&) = delete;
    FileHandle(FileHandle&& other) noexcept
        : handle_(std::exchange(other.handle_, nullptr)) {}
    FileHandle& operator=(FileHandle&& other) noexcept {
        if (this != &other) {
            if (handle_) std::fclose(handle_);
            handle_ = std::exchange(other.handle_, nullptr);
        }
        return *this;
    }

private:
    std::FILE* handle_;
};
```

### アンチパターン

- 裸の `new`/`delete`（R.11）
- C++コードでの `malloc()`/`free()`（R.10）
- 単一の式での複数のリソース割り当て（R.13 -- 例外安全性の危険）
- `unique_ptr` で十分な場所での `shared_ptr`（R.21）

## 式と文（ES.*）

### 主要ルール

| ルール | 概要 |
|------|---------|
| **ES.5** | スコープを小さく保つ |
| **ES.20** | 常にオブジェクトを初期化する |
| **ES.23** | `{}` 初期化構文を優先する |
| **ES.25** | 変更が意図されない限り、オブジェクトを `const` または `constexpr` として宣言する |
| **ES.28** | `const` 変数の複雑な初期化にラムダを使用する |
| **ES.45** | マジック定数を避ける。シンボリック定数を使用する |
| **ES.46** | ナローイング/損失のある算術変換を避ける |
| **ES.47** | `0` や `NULL` の代わりに `nullptr` を使用する |
| **ES.48** | キャストを避ける |
| **ES.50** | `const` を外さない |

### 初期化

```cpp
// ES.20 + ES.23 + ES.25: 常に初期化し、{}を優先し、デフォルトでconst
const int max_retries{3};
const std::string name{"widget"};
const std::vector<int> primes{2, 3, 5, 7, 11};

// ES.28: 複雑なconst初期化にラムダ
const auto config = [&] {
    Config c;
    c.timeout = std::chrono::seconds{30};
    c.retries = max_retries;
    c.verbose = debug_mode;
    return c;
}();
```

### アンチパターン

- 初期化されていない変数（ES.20）
- ポインタとして `0` や `NULL` を使用する（ES.47 -- `nullptr` を使用する）
- Cスタイルのキャスト（ES.48 -- `static_cast`、`const_cast` などを使用する）
- `const` を外すキャスト（ES.50）
- 名前付き定数のないマジックナンバー（ES.45）
- 符号付きと符号なし算術の混合（ES.100）
- ネストされたスコープで名前を再利用する（ES.12）

## エラー処理（E.*）

### 主要ルール

| ルール | 概要 |
|------|---------|
| **E.1** | 設計の早い段階でエラー処理戦略を開発する |
| **E.2** | 関数が割り当てられたタスクを実行できないことを示すために例外をスローする |
| **E.6** | リークを防ぐためにRAIIを使用する |
| **E.12** | スローが不可能または許容できない場合に `noexcept` を使用する |
| **E.14** | 例外として目的設計されたユーザー定義型を使用する |
| **E.15** | 値でスロー、参照でキャッチ |
| **E.16** | デストラクタ、解放、swapは絶対に失敗してはならない |
| **E.17** | すべての関数ですべての例外をキャッチしようとしない |

### 例外階層

```cpp
// E.14 + E.15: カスタム例外型、値でスロー、参照でキャッチ
class AppError : public std::runtime_error {
public:
    using std::runtime_error::runtime_error;
};

class NetworkError : public AppError {
public:
    NetworkError(const std::string& msg, int code)
        : AppError(msg), status_code(code) {}
    int status_code;
};

void fetch_data(const std::string& url) {
    // E.2: 失敗を示すためにスロー
    throw NetworkError("connection refused", 503);
}

void run() {
    try {
        fetch_data("https://api.example.com");
    } catch (const NetworkError& e) {
        log_error(e.what(), e.status_code);
    } catch (const AppError& e) {
        log_error(e.what());
    }
    // E.17: ここですべてをキャッチしない -- 予期しないエラーは伝播させる
}
```

### アンチパターン

- `int` や文字列リテラルのようなビルトイン型をスローする（E.14）
- 値でキャッチする（スライシングリスク）（E.15）
- エラーをサイレントに飲み込む空のcatchブロック
- フロー制御に例外を使用する（E.3）
- `errno` のようなグローバル状態に基づくエラー処理（E.28）

## 定数と不変性（Con.*）

### 全ルール

| ルール | 概要 |
|------|---------|
| **Con.1** | デフォルトでオブジェクトを不変にする |
| **Con.2** | デフォルトでメンバー関数を `const` にする |
| **Con.3** | デフォルトで `const` へのポインタと参照を渡す |
| **Con.4** | 構築後に変更されない値に `const` を使用する |
| **Con.5** | コンパイル時に計算可能な値に `constexpr` を使用する |

```cpp
// Con.1からCon.5: デフォルトで不変
class Sensor {
public:
    explicit Sensor(std::string id) : id_(std::move(id)) {}

    // Con.2: デフォルトでconstメンバー関数
    const std::string& id() const { return id_; }
    double last_reading() const { return reading_; }

    // 変更が必要な場合のみnon-const
    void record(double value) { reading_ = value; }

private:
    const std::string id_;  // Con.4: 構築後に変更されない
    double reading_{0.0};
};

// Con.3: const参照で渡す
void display(const Sensor& s) {
    std::cout << s.id() << ": " << s.last_reading() << '\n';
}

// Con.5: コンパイル時定数
constexpr double PI = 3.14159265358979;
constexpr int MAX_SENSORS = 256;
```

## 並行性と並列性（CP.*）

### 主要ルール

| ルール | 概要 |
|------|---------|
| **CP.2** | データ競合を避ける |
| **CP.3** | 書き込み可能なデータの明示的な共有を最小化する |
| **CP.4** | スレッドではなくタスクの観点で考える |
| **CP.8** | 同期に `volatile` を使用しない |
| **CP.20** | RAIIを使用する。プレーンな `lock()`/`unlock()` は使用しない |
| **CP.21** | 複数のmutexを取得するために `std::scoped_lock` を使用する |
| **CP.22** | ロックを保持している間に未知のコードを呼び出さない |
| **CP.42** | 条件なしに待機しない |
| **CP.44** | `lock_guard` と `unique_lock` に名前を付けることを忘れない |
| **CP.100** | 絶対に必要でない限りロックフリープログラミングを使用しない |

### 安全なロック

```cpp
// CP.20 + CP.44: RAIIロック、常に名前付き
class ThreadSafeQueue {
public:
    void push(int value) {
        std::lock_guard<std::mutex> lock(mutex_);  // CP.44: 名前付き!
        queue_.push(value);
        cv_.notify_one();
    }

    int pop() {
        std::unique_lock<std::mutex> lock(mutex_);
        // CP.42: 常に条件付きで待機
        cv_.wait(lock, [this] { return !queue_.empty(); });
        const int value = queue_.front();
        queue_.pop();
        return value;
    }

private:
    std::mutex mutex_;             // CP.50: mutexをデータと共に
    std::condition_variable cv_;
    std::queue<int> queue_;
};
```

### 複数のMutex

```cpp
// CP.21: 複数のmutex向けのstd::scoped_lock（デッドロックフリー）
void transfer(Account& from, Account& to, double amount) {
    std::scoped_lock lock(from.mutex_, to.mutex_);
    from.balance_ -= amount;
    to.balance_ += amount;
}
```

### アンチパターン

- 同期に `volatile`（CP.8 -- ハードウェアI/O専用）
- スレッドのデタッチ（CP.26 -- ライフタイム管理がほぼ不可能になる）
- 無名のロックガード: `std::lock_guard<std::mutex>(m);` は即座に破壊される（CP.44）
- コールバックを呼び出す際にロックを保持する（CP.22 -- デッドロックリスク）
- 深い専門知識なしのロックフリープログラミング（CP.100）

## テンプレートとジェネリックプログラミング（T.*）

### 主要ルール

| ルール | 概要 |
|------|---------|
| **T.1** | テンプレートを使用して抽象化のレベルを上げる |
| **T.2** | テンプレートを使用して多くの引数型のアルゴリズムを表現する |
| **T.10** | すべてのテンプレート引数にコンセプトを指定する |
| **T.11** | 可能な限り標準コンセプトを使用する |
| **T.13** | シンプルなコンセプトには省略記法を優先する |
| **T.43** | `typedef` より `using` を優先する |
| **T.120** | 本当に必要な場合のみテンプレートメタプログラミングを使用する |
| **T.144** | 関数テンプレートを特殊化しない（代わりにオーバーロード） |

### コンセプト（C++20）

```cpp
#include <concepts>

// T.10 + T.11: 標準コンセプトでテンプレートを制約
template<std::integral T>
T gcd(T a, T b) {
    while (b != 0) {
        a = std::exchange(b, a % b);
    }
    return a;
}

// T.13: 省略記法のコンセプト構文
void sort(std::ranges::random_access_range auto& range) {
    std::ranges::sort(range);
}

// ドメイン固有の制約のカスタムコンセプト
template<typename T>
concept Serializable = requires(const T& t) {
    { t.serialize() } -> std::convertible_to<std::string>;
};

template<Serializable T>
void save(const T& obj, const std::string& path);
```

### アンチパターン

- 可視ネームスペースでの非制約テンプレート（T.47）
- オーバーロードの代わりに関数テンプレートを特殊化する（T.144）
- `constexpr` で十分な場所でのテンプレートメタプログラミング（T.120）
- `using` の代わりに `typedef`（T.43）

## 標準ライブラリ（SL.*）

### 主要ルール

| ルール | 概要 |
|------|---------|
| **SL.1** | 可能な限りライブラリを使用する |
| **SL.2** | 他のライブラリより標準ライブラリを優先する |
| **SL.con.1** | C配列より `std::array` または `std::vector` を優先する |
| **SL.con.2** | デフォルトで `std::vector` を優先する |
| **SL.str.1** | 文字シーケンスを所有するために `std::string` を使用する |
| **SL.str.2** | 文字シーケンスを参照するために `std::string_view` を使用する |
| **SL.io.50** | `endl` を避ける（`'\n'` を使用する -- `endl` はフラッシュを強制する） |

```cpp
// SL.con.1 + SL.con.2: C配列よりvector/arrayを優先
const std::array<int, 4> fixed_data{1, 2, 3, 4};
std::vector<std::string> dynamic_data;

// SL.str.1 + SL.str.2: stringが所有し、string_viewが参照する
std::string build_greeting(std::string_view name) {
    return "Hello, " + std::string(name) + "!";
}

// SL.io.50: endlではなく'\n'を使用
std::cout << "result: " << value << '\n';
```

## 列挙型（Enum.*）

### 主要ルール

| ルール | 概要 |
|------|---------|
| **Enum.1** | マクロより列挙型を優先する |
| **Enum.3** | プレーンな `enum` より `enum class` を優先する |
| **Enum.5** | 列挙子にALL_CAPSを使用しない |
| **Enum.6** | 無名の列挙型を避ける |

```cpp
// Enum.3 + Enum.5: スコープ付きenum、ALL_CAPSなし
enum class Color { red, green, blue };
enum class LogLevel { debug, info, warning, error };

// BAD: プレーンなenumは名前をリーク、ALL_CAPSはマクロと衝突
enum { RED, GREEN, BLUE };           // Enum.3 + Enum.5 + Enum.6 違反
#define MAX_SIZE 100                  // Enum.1 違反 -- constexprを使用する
```

## ソースファイルと命名（SF.*、NL.*）

### 主要ルール

| ルール | 概要 |
|------|---------|
| **SF.1** | コードファイルに `.cpp`、インターフェースファイルに `.h` を使用する |
| **SF.7** | グローバルスコープのヘッダーに `using namespace` を書かない |
| **SF.8** | すべての `.h` ファイルに `#include` ガードを使用する |
| **SF.11** | ヘッダーファイルは自己完結すべき |
| **NL.5** | 名前に型情報をエンコードしない（ハンガリアン記法なし） |
| **NL.8** | 一貫した命名スタイルを使用する |
| **NL.9** | マクロ名にのみ ALL_CAPS を使用する |
| **NL.10** | `underscore_style` の名前を優先する |

### ヘッダーガード

```cpp
// SF.8: インクルードガード（または#pragma once）
#ifndef PROJECT_MODULE_WIDGET_H
#define PROJECT_MODULE_WIDGET_H

// SF.11: 自己完結 -- このヘッダーが必要とするものをすべてインクルードする
#include <string>
#include <vector>

namespace project::module {

class Widget {
public:
    explicit Widget(std::string name);
    const std::string& name() const;

private:
    std::string name_;
};

}  // namespace project::module

#endif  // PROJECT_MODULE_WIDGET_H
```

### 命名規則

```cpp
// NL.8 + NL.10: 一貫したunderscore_style
namespace my_project {

constexpr int max_buffer_size = 4096;  // NL.9: ALL_CAPSではない（マクロではない）

class tcp_connection {                 // underscore_styleクラス
public:
    void send_message(std::string_view msg);
    bool is_connected() const;

private:
    std::string host_;                 // メンバーには末尾アンダースコア
    int port_;
};

}  // namespace my_project
```

### アンチパターン

- グローバルスコープのヘッダーで `using namespace std;`（SF.7）
- インクルード順序に依存するヘッダー（SF.10、SF.11）
- `strName`、`iCount` のようなハンガリアン記法（NL.5）
- マクロ以外のものにALL_CAPS（NL.9）

## パフォーマンス（Per.*）

### 主要ルール

| ルール | 概要 |
|------|---------|
| **Per.1** | 理由なく最適化しない |
| **Per.2** | 時期尚早に最適化しない |
| **Per.6** | 計測なしにパフォーマンスについて主張しない |
| **Per.7** | 最適化を可能にする設計をする |
| **Per.10** | 静的型システムに依存する |
| **Per.11** | 計算をランタイムからコンパイル時に移動する |
| **Per.19** | メモリに予測可能にアクセスする |

### ガイドライン

```cpp
// Per.11: 可能な限りコンパイル時計算
constexpr auto lookup_table = [] {
    std::array<int, 256> table{};
    for (int i = 0; i < 256; ++i) {
        table[i] = i * i;
    }
    return table;
}();

// Per.19: キャッシュフレンドリーのために連続したデータを優先
std::vector<Point> points;           // GOOD: 連続している
std::vector<std::unique_ptr<Point>> indirect_points; // BAD: ポインタチェイス
```

### アンチパターン

- プロファイリングデータなしに最適化する（Per.1、Per.6）
- 明確な抽象化より「賢い」低レベルコードを選択する（Per.4、Per.5）
- データレイアウトとキャッシュ動作を無視する（Per.19）

## クイックリファレンスチェックリスト

C++作業を完了とする前に:

- [ ] 生の `new`/`delete` なし -- スマートポインタかRAIIを使用（R.11）
- [ ] 宣言時にオブジェクトを初期化している（ES.20）
- [ ] 変数はデフォルトで `const`/`constexpr`（Con.1、ES.25）
- [ ] メンバー関数は可能な限り `const`（Con.2）
- [ ] プレーンな `enum` の代わりに `enum class`（Enum.3）
- [ ] `0`/`NULL` の代わりに `nullptr`（ES.47）
- [ ] ナローイング変換なし（ES.46）
- [ ] Cスタイルのキャストなし（ES.48）
- [ ] 単一引数コンストラクタが `explicit`（C.46）
- [ ] ゼロの法則または5の法則が適用されている（C.20、C.21）
- [ ] 基底クラスのデストラクタがパブリック仮想またはプロテクト非仮想（C.35）
- [ ] テンプレートがコンセプトで制約されている（T.10）
- [ ] グローバルスコープのヘッダーに `using namespace` なし（SF.7）
- [ ] ヘッダーにインクルードガードがあり自己完結している（SF.8、SF.11）
- [ ] ロックがRAIIを使用している（`scoped_lock`/`lock_guard`）（CP.20）
- [ ] 例外はカスタム型で、値でスローし、参照でキャッチ（E.14、E.15）
- [ ] `std::endl` の代わりに `'\n'`（SL.io.50）
- [ ] マジックナンバーなし（ES.45）
