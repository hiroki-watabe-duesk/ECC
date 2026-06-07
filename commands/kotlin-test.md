---
description: Kotlin の TDD ワークフローを強制します。まず Kotest テストを書いてから実装し、Kover で 80% 以上のカバレッジを確認します。
---

# Kotlin TDD コマンド

このコマンドは Kotest、MockK、Kover を使用した Kotlin コードのテスト駆動開発手法を強制します。

## このコマンドの動作

1. **型/インターフェースの定義**: まず関数のシグネチャをスキャフォールドする
2. **Kotest テストを書く**: 包括的なテストスペックを作成する（RED）
3. **テストを実行する**: 正しい理由でテストが失敗することを確認する
4. **コードを実装する**: 通過するための最小限のコードを書く（GREEN）
5. **リファクタリング**: テストをグリーンに保ちながら改善する
6. **カバレッジを確認する**: Kover で 80% 以上のカバレッジを確保する

## いつ使うか

`/kotlin-test` を使用するのは次の場合です。
- 新しい Kotlin 関数またはクラスを実装するとき
- 既存の Kotlin コードにテストカバレッジを追加するとき
- バグを修正するとき（最初に失敗するテストを書く）
- 重要なビジネスロジックを構築するとき
- Kotlin で TDD ワークフローを学ぶとき

## TDD サイクル

```
RED     -> 失敗する Kotest テストを書く
GREEN   -> 通過するための最小限のコードを実装する
REFACTOR -> コードを改善し、テストをグリーンに保つ
REPEAT  -> 次のテストケース
```

## セッション例

````text
User: /kotlin-test I need a function to validate user registration requests

Agent:
# TDD Session: Registration Validator

## Step 1: Define Interface

```kotlin
// validator/RegistrationValidator.kt
package com.example.validator

data class RegistrationRequest(
    val name: String,
    val email: String,
    val password: String,
)

sealed class ValidationResult {
    data object Valid : ValidationResult()
    data class Invalid(val errors: List<String>) : ValidationResult()
}

fun validateRegistration(request: RegistrationRequest): ValidationResult {
    TODO("not implemented")
}
```

## Step 2: Write Kotest Tests (RED)

```kotlin
// validator/RegistrationValidatorTest.kt
package com.example.validator

import io.kotest.core.spec.style.FunSpec
import io.kotest.matchers.shouldBe
import io.kotest.matchers.types.shouldBeInstanceOf

class RegistrationValidatorTest : FunSpec({
    test("valid registration returns Valid") {
        val request = RegistrationRequest(
            name = "Alice",
            email = "alice@example.com",
            password = "SecureP@ss1",
        )

        val result = validateRegistration(request)

        result.shouldBeInstanceOf<ValidationResult.Valid>()
    }

    test("blank name returns Invalid") {
        val request = RegistrationRequest(
            name = "",
            email = "alice@example.com",
            password = "SecureP@ss1",
        )

        val result = validateRegistration(request)

        val invalid = result.shouldBeInstanceOf<ValidationResult.Invalid>()
        invalid.errors shouldBe listOf("Name is required")
    }

    test("invalid email returns Invalid") {
        val request = RegistrationRequest(
            name = "Alice",
            email = "not-an-email",
            password = "SecureP@ss1",
        )

        val result = validateRegistration(request)

        val invalid = result.shouldBeInstanceOf<ValidationResult.Invalid>()
        invalid.errors shouldBe listOf("Invalid email format")
    }

    test("short password returns Invalid") {
        val request = RegistrationRequest(
            name = "Alice",
            email = "alice@example.com",
            password = "short",
        )

        val result = validateRegistration(request)

        val invalid = result.shouldBeInstanceOf<ValidationResult.Invalid>()
        invalid.errors shouldBe listOf("Password must be at least 8 characters")
    }

    test("multiple errors returns all errors") {
        val request = RegistrationRequest(
            name = "",
            email = "bad",
            password = "short",
        )

        val result = validateRegistration(request)

        val invalid = result.shouldBeInstanceOf<ValidationResult.Invalid>()
        invalid.errors.size shouldBe 3
    }
})
```

## Step 3: Run Tests - Verify FAIL

```bash
$ ./gradlew test

RegistrationValidatorTest > valid registration returns Valid FAILED
  kotlin.NotImplementedError: An operation is not implemented

FAILED (5 tests, 0 passed, 5 failed)
```

✓ テストは期待通り失敗します（NotImplementedError）。

## Step 4: Implement Minimal Code (GREEN)

```kotlin
// validator/RegistrationValidator.kt
package com.example.validator

private val EMAIL_REGEX = Regex("^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$")
private const val MIN_PASSWORD_LENGTH = 8

fun validateRegistration(request: RegistrationRequest): ValidationResult {
    val errors = buildList {
        if (request.name.isBlank()) add("Name is required")
        if (!EMAIL_REGEX.matches(request.email)) add("Invalid email format")
        if (request.password.length < MIN_PASSWORD_LENGTH) add("Password must be at least $MIN_PASSWORD_LENGTH characters")
    }

    return if (errors.isEmpty()) ValidationResult.Valid
    else ValidationResult.Invalid(errors)
}
```

## Step 5: Run Tests - Verify PASS

```bash
$ ./gradlew test

RegistrationValidatorTest > valid registration returns Valid PASSED
RegistrationValidatorTest > blank name returns Invalid PASSED
RegistrationValidatorTest > invalid email returns Invalid PASSED
RegistrationValidatorTest > short password returns Invalid PASSED
RegistrationValidatorTest > multiple errors returns all errors PASSED

PASSED (5 tests, 5 passed, 0 failed)
```

✓ すべてのテストが通過しています！

## Step 6: Check Coverage

```bash
$ ./gradlew koverHtmlReport

Coverage: 100.0% of statements
```

✓ カバレッジ: 100%

## TDD Complete!
````

## テストパターン

### StringSpec（最もシンプル）

```kotlin
class CalculatorTest : StringSpec({
    "add two positive numbers" {
        Calculator.add(2, 3) shouldBe 5
    }
})
```

### BehaviorSpec（BDD）

```kotlin
class OrderServiceTest : BehaviorSpec({
    Given("a valid order") {
        When("placed") {
            Then("should be confirmed") { /* ... */ }
        }
    }
})
```

### データ駆動テスト

```kotlin
class ParserTest : FunSpec({
    context("valid inputs") {
        withData("2026-01-15", "2026-12-31", "2000-01-01") { input ->
            parseDate(input).shouldNotBeNull()
        }
    }
})
```

### コルーチンテスト

```kotlin
class AsyncServiceTest : FunSpec({
    test("concurrent fetch completes") {
        runTest {
            val result = service.fetchAll()
            result.shouldNotBeEmpty()
        }
    }
})
```

## カバレッジコマンド

```bash
# カバレッジ付きでテストを実行する
./gradlew koverHtmlReport

# カバレッジ閾値を確認する
./gradlew koverVerify

# CI 用の XML レポート
./gradlew koverXmlReport

# HTML レポートを開く
open build/reports/kover/html/index.html

# 特定のテストクラスを実行する
./gradlew test --tests "com.example.UserServiceTest"

# 詳細出力で実行する
./gradlew test --info
```

## カバレッジ目標

| コードの種類 | 目標 |
|-----------|--------|
| 重要なビジネスロジック | 100% |
| パブリック API | 90% 以上 |
| 一般的なコード | 80% 以上 |
| 生成されたコード | 除外 |

## TDD のベストプラクティス

**すべきこと:**
- 実装の前にテストを書く
- 変更のたびにテストを実行する
- 表現力豊かなアサーションに Kotest のマッチャーを使用する
- suspend 関数には MockK の `coEvery`/`coVerify` を使用する
- 実装の詳細ではなく振る舞いをテストする
- エッジケース（空、null、最大値）を含める

**すべきでないこと:**
- テストの前に実装を書く
- RED フェーズをスキップする
- プライベート関数を直接テストする
- コルーチンテストで `Thread.sleep()` を使用する
- 不安定なテストを無視する

## 関連コマンド

- `/kotlin-build` - ビルドエラーを修正する
- `/kotlin-review` - 実装後にコードをレビューする
- `verification-loop` スキル - 完全な検証ループを実行する

## 関連情報

- Skill: `skills/kotlin-testing/`
- Skill: `skills/tdd-workflow/`
