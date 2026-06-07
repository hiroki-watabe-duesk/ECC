---
name: rust-testing
description: Rustのテストパターン（ユニットテスト・統合テスト・非同期テスト・プロパティベーステスト・モッキング・カバレッジ）。TDD手法に従う。
origin: ECC
---

# Rustテストパターン

TDD手法に従った信頼性が高く保守しやすいテストを書くための包括的なRustテストパターン。

## 使用タイミング

- 新しいRust関数・メソッド・トレイトを書く場合
- 既存コードにテストカバレッジを追加する場合
- パフォーマンスクリティカルなコードのベンチマークを作成する場合
- 入力バリデーションのプロパティベーステストを実装する場合
- RustプロジェクトでTDDワークフローに従う場合

## 仕組み

1. **テスト対象を特定** — テストするべき関数・トレイト・モジュールを見つける
2. **テストを書く** — `#[cfg(test)]` モジュール内の `#[test]`、パラメーター化テストにはrstest、プロパティベーステストにはproptestを使用する
3. **依存関係をモック** — mockallを使ってテスト対象のユニットを分離する
4. **テストを実行（RED）** — テストが期待通りのエラーで失敗することを確認する
5. **実装（GREEN）** — 通過させる最小限のコードを書く
6. **リファクタリング** — テストをグリーンに保ちながら改善する
7. **カバレッジを確認** — cargo-llvm-covを使用し、80%以上を目標とする

## RustのTDDワークフロー

### RED-GREEN-REFACTORサイクル

```
RED     → 失敗するテストを先に書く
GREEN   → テストを通す最小限のコードを書く
REFACTOR → テストをグリーンに保ちながらコードを改善する
REPEAT  → 次の要件に進む
```

### RustでのステップバイステップTDD

```rust
// RED: 先にテストを書き、todo!() をプレースホルダーとして使用
pub fn add(a: i32, b: i32) -> i32 { todo!() }

#[cfg(test)]
mod tests {
    use super::*;
    #[test]
    fn test_add() { assert_eq!(add(2, 3), 5); }
}
// cargo test → 'not yet implemented' でパニック
```

```rust
// GREEN: todo!() を最小限の実装で置き換える
pub fn add(a: i32, b: i32) -> i32 { a + b }
// cargo test → PASS、その後テストをグリーンに保ちながらREFACTOR
```

## ユニットテスト

### モジュールレベルのテスト構成

```rust
// src/user.rs
pub struct User {
    pub name: String,
    pub email: String,
}

impl User {
    pub fn new(name: impl Into<String>, email: impl Into<String>) -> Result<Self, String> {
        let email = email.into();
        if !email.contains('@') {
            return Err(format!("invalid email: {email}"));
        }
        Ok(Self { name: name.into(), email })
    }

    pub fn display_name(&self) -> &str {
        &self.name
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn creates_user_with_valid_email() {
        let user = User::new("Alice", "alice@example.com").unwrap();
        assert_eq!(user.display_name(), "Alice");
        assert_eq!(user.email, "alice@example.com");
    }

    #[test]
    fn rejects_invalid_email() {
        let result = User::new("Bob", "not-an-email");
        assert!(result.is_err());
        assert!(result.unwrap_err().contains("invalid email"));
    }
}
```

### アサーションマクロ

```rust
assert_eq!(2 + 2, 4);                                    // 等値
assert_ne!(2 + 2, 5);                                    // 非等値
assert!(vec![1, 2, 3].contains(&2));                     // ブール
assert_eq!(value, 42, "expected 42 but got {value}");    // カスタムメッセージ
assert!((0.1_f64 + 0.2 - 0.3).abs() < f64::EPSILON);   // 浮動小数点比較
```

## エラーとパニックのテスト

### `Result` 戻り値のテスト

```rust
#[test]
fn parse_returns_error_for_invalid_input() {
    let result = parse_config("}{invalid");
    assert!(result.is_err());

    // 特定のエラーバリアントをアサート
    let err = result.unwrap_err();
    assert!(matches!(err, ConfigError::ParseError(_)));
}

#[test]
fn parse_succeeds_for_valid_input() -> Result<(), Box<dyn std::error::Error>> {
    let config = parse_config(r#"{"port": 8080}"#)?;
    assert_eq!(config.port, 8080);
    Ok(()) // ? が Err を返した場合はテスト失敗
}
```

### パニックのテスト

```rust
#[test]
#[should_panic]
fn panics_on_empty_input() {
    process(&[]);
}

#[test]
#[should_panic(expected = "index out of bounds")]
fn panics_with_specific_message() {
    let v: Vec<i32> = vec![];
    let _ = v[0];
}
```

## 統合テスト

### ファイル構造

```text
my_crate/
├── src/
│   └── lib.rs
├── tests/              # 統合テスト
│   ├── api_test.rs     # 各ファイルが別テストバイナリ
│   ├── db_test.rs
│   └── common/         # 共有テストユーティリティ
│       └── mod.rs
```

### 統合テストの書き方

```rust
// tests/api_test.rs
use my_crate::{App, Config};

#[test]
fn full_request_lifecycle() {
    let config = Config::test_default();
    let app = App::new(config);

    let response = app.handle_request("/health");
    assert_eq!(response.status, 200);
    assert_eq!(response.body, "OK");
}
```

## 非同期テスト

### Tokioを使用

```rust
#[tokio::test]
async fn fetches_data_successfully() {
    let client = TestClient::new().await;
    let result = client.get("/data").await;
    assert!(result.is_ok());
    assert_eq!(result.unwrap().items.len(), 3);
}

#[tokio::test]
async fn handles_timeout() {
    use std::time::Duration;
    let result = tokio::time::timeout(
        Duration::from_millis(100),
        slow_operation(),
    ).await;

    assert!(result.is_err(), "should have timed out");
}
```

## テスト構成パターン

### `rstest` によるパラメーター化テスト

```rust
use rstest::{rstest, fixture};

#[rstest]
#[case("hello", 5)]
#[case("", 0)]
#[case("rust", 4)]
fn test_string_length(#[case] input: &str, #[case] expected: usize) {
    assert_eq!(input.len(), expected);
}

// フィクスチャ
#[fixture]
fn test_db() -> TestDb {
    TestDb::new_in_memory()
}

#[rstest]
fn test_insert(test_db: TestDb) {
    test_db.insert("key", "value");
    assert_eq!(test_db.get("key"), Some("value".into()));
}
```

### テストヘルパー

```rust
#[cfg(test)]
mod tests {
    use super::*;

    /// 適切なデフォルト値を持つテストユーザーを作成する。
    fn make_user(name: &str) -> User {
        User::new(name, &format!("{name}@test.com")).unwrap()
    }

    #[test]
    fn user_display() {
        let user = make_user("alice");
        assert_eq!(user.display_name(), "alice");
    }
}
```

## `proptest` によるプロパティベーステスト

### 基本的なプロパティテスト

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn encode_decode_roundtrip(input in ".*") {
        let encoded = encode(&input);
        let decoded = decode(&encoded).unwrap();
        assert_eq!(input, decoded);
    }

    #[test]
    fn sort_preserves_length(mut vec in prop::collection::vec(any::<i32>(), 0..100)) {
        let original_len = vec.len();
        vec.sort();
        assert_eq!(vec.len(), original_len);
    }

    #[test]
    fn sort_produces_ordered_output(mut vec in prop::collection::vec(any::<i32>(), 0..100)) {
        vec.sort();
        for window in vec.windows(2) {
            assert!(window[0] <= window[1]);
        }
    }
}
```

### カスタムストラテジー

```rust
use proptest::prelude::*;

fn valid_email() -> impl Strategy<Value = String> {
    ("[a-z]{1,10}", "[a-z]{1,5}")
        .prop_map(|(user, domain)| format!("{user}@{domain}.com"))
}

proptest! {
    #[test]
    fn accepts_valid_emails(email in valid_email()) {
        assert!(User::new("Test", &email).is_ok());
    }
}
```

## `mockall` によるモッキング

### トレイトベースのモッキング

```rust
use mockall::{automock, predicate::eq};

#[automock]
trait UserRepository {
    fn find_by_id(&self, id: u64) -> Option<User>;
    fn save(&self, user: &User) -> Result<(), StorageError>;
}

#[test]
fn service_returns_user_when_found() {
    let mut mock = MockUserRepository::new();
    mock.expect_find_by_id()
        .with(eq(42))
        .times(1)
        .returning(|_| Some(User { id: 42, name: "Alice".into() }));

    let service = UserService::new(Box::new(mock));
    let user = service.get_user(42).unwrap();
    assert_eq!(user.name, "Alice");
}

#[test]
fn service_returns_none_when_not_found() {
    let mut mock = MockUserRepository::new();
    mock.expect_find_by_id()
        .returning(|_| None);

    let service = UserService::new(Box::new(mock));
    assert!(service.get_user(99).is_none());
}
```

## ドックテスト

### 実行可能なドキュメント

```rust
/// 2つの数値を足し合わせる。
///
/// # 例
///
/// ```
/// use my_crate::add;
///
/// assert_eq!(add(2, 3), 5);
/// assert_eq!(add(-1, 1), 0);
/// ```
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

/// 設定文字列を解析する。
///
/// # エラー
///
/// 入力が有効なTOMLでない場合は `Err` を返す。
///
/// ```no_run
/// use my_crate::parse_config;
///
/// let config = parse_config(r#"port = 8080"#).unwrap();
/// assert_eq!(config.port, 8080);
/// ```
///
/// ```no_run
/// use my_crate::parse_config;
///
/// assert!(parse_config("}{invalid").is_err());
/// ```
pub fn parse_config(input: &str) -> Result<Config, ParseError> {
    todo!()
}
```

## Criterionによるベンチマーク

```toml
# Cargo.toml
[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }

[[bench]]
name = "benchmark"
harness = false
```

```rust
// benches/benchmark.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn fibonacci(n: u64) -> u64 {
    match n {
        0 | 1 => n,
        _ => fibonacci(n - 1) + fibonacci(n - 2),
    }
}

fn bench_fibonacci(c: &mut Criterion) {
    c.bench_function("fib 20", |b| b.iter(|| fibonacci(black_box(20))));
}

criterion_group!(benches, bench_fibonacci);
criterion_main!(benches);
```

## テストカバレッジ

### カバレッジの実行

```bash
# インストール: cargo install cargo-llvm-cov (またはCIでtaiki-e/install-actionを使用)
cargo llvm-cov                    # 概要
cargo llvm-cov --html             # HTMLレポート
cargo llvm-cov --lcov > lcov.info # CI向けLCOVフォーマット
cargo llvm-cov --fail-under-lines 80  # 閾値を下回った場合に失敗
```

### カバレッジ目標

| コードの種類 | 目標 |
|-----------|--------|
| 重要なビジネスロジック | 100% |
| 公開API | 90%以上 |
| 一般コード | 80%以上 |
| 生成済み / FFIバインディング | 除外 |

## テストコマンド

```bash
cargo test                        # すべてのテストを実行
cargo test -- --nocapture         # printlnの出力を表示
cargo test test_name              # パターンに一致するテストを実行
cargo test --lib                  # ユニットテストのみ
cargo test --test api_test        # 統合テストのみ
cargo test --doc                  # ドックテストのみ
cargo test --no-fail-fast         # 最初の失敗で停止しない
cargo test -- --ignored           # 無視されたテストを実行
```

## ベストプラクティス

**すること:**
- 最初にテストを書く（TDD）
- ユニットテストには `#[cfg(test)]` モジュールを使用する
- 実装ではなく振る舞いをテストする
- シナリオを説明する記述的なテスト名を使用する
- より良いエラーメッセージのために `assert!` より `assert_eq!` を優先する
- より明確なエラー出力のために `Result` を返すテストで `?` を使用する
- テストを独立に保つ — 共有可変状態なし

**しないこと:**
- `Result::is_err()` でテストできる場合に `#[should_panic]` を使用しない
- 何でもモックしない — 可能な場合は統合テストを優先する
- 不安定なテストを無視しない — 修正または隔離する
- テストで `sleep()` を使用しない — チャネル・バリア・`tokio::time::pause()` を使用する
- エラーパスのテストをスキップしない

## CI連携

```yaml
# GitHub Actions
test:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - uses: dtolnay/rust-toolchain@stable
      with:
        components: clippy, rustfmt

    - name: Check formatting
      run: cargo fmt --check

    - name: Clippy
      run: cargo clippy -- -D warnings

    - name: Run tests
      run: cargo test

    - uses: taiki-e/install-action@cargo-llvm-cov

    - name: Coverage
      run: cargo llvm-cov --fail-under-lines 80
```

**覚えておいてください**: テストはドキュメントです。コードがどのように使われるべきかを示します。明確に書き、常に最新の状態に保ってください。
