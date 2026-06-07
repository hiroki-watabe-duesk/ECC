---
name: fsharp-testing
description: xUnit、FsUnit、Unquote、FsCheckプロパティベーステスト、インテグレーションテスト、テスト組織のベストプラクティスを用いたF#テストパターン。
origin: ECC
---

# F# テストパターン

xUnit、FsUnit、Unquote、FsCheck、および最新の.NETテストプラクティスを使用したF#アプリケーションの包括的なテストパターン。

## 有効化する時

- F#コードの新しいテストを書くとき
- テスト品質とカバレッジをレビューするとき
- F#プロジェクトのテストインフラをセットアップするとき
- 不安定なテストや遅いテストをデバッグするとき

## テストフレームワークスタック

| ツール | 目的 |
|---|---|
| **xUnit** | テストフレームワーク（標準的な.NETエコシステムの選択） |
| **FsUnit.xUnit** | xUnit用のF#フレンドリーなアサーション構文 |
| **Unquote** | 明確な失敗メッセージのためにF#クォーテーションを使用するアサーションライブラリ |
| **FsCheck.xUnit** | xUnitと統合されたプロパティベーステスト |
| **NSubstitute** | .NET依存関係のモッキング |
| **Testcontainers** | インテグレーションテストでの実際のインフラ |
| **WebApplicationFactory** | ASP.NET Coreインテグレーションテスト |

## xUnit + FsUnit を使ったユニットテスト

### 基本的なテスト構造

```fsharp
module OrderServiceTests

open Xunit
open FsUnit.Xunit

[<Fact>]
let ``create sets status to Pending`` () =
    let order = Order.create "cust-1" [ validItem ]
    order.Status |> should equal Pending

[<Fact>]
let ``confirm changes status to Confirmed`` () =
    let order = Order.create "cust-1" [ validItem ]
    let confirmed = Order.confirm order
    confirmed.Status |> should be (ofCase <@ Confirmed @>)
```

### Unquote を使ったアサーション

UnquoteはF#クォーテーションを使用するため、失敗メッセージは「expected X got Y」だけでなく、失敗した完全な式を表示します。

```fsharp
module OrderValidationTests

open Xunit
open Swensen.Unquote

[<Fact>]
let ``PlaceOrder returns success when request is valid`` () =
    let request = { CustomerId = "cust-123"; Items = [ validItem ] }
    let result = OrderService.placeOrder request
    test <@ Result.isOk result @>

[<Fact>]
let ``order total sums item prices`` () =
    let items = [ { Sku = "A"; Quantity = 2; Price = 10m }
                  { Sku = "B"; Quantity = 1; Price = 5m } ]
    let total = Order.calculateTotal items
    test <@ total = 25m @>

[<Fact>]
let ``validated email rejects empty input`` () =
    let result = ValidatedEmail.create ""
    test <@ Result.isError result @>
```

### 非同期テスト

```fsharp
[<Fact>]
let ``PlaceOrder returns success when request is valid`` () = task {
    let deps = createTestDeps ()
    let request = { CustomerId = "cust-123"; Items = [ validItem ] }

    let! result = OrderService.placeOrder deps request

    test <@ Result.isOk result @>
}

[<Fact>]
let ``PlaceOrder returns error when items are empty`` () = task {
    let deps = createTestDeps ()
    let request = { CustomerId = "cust-123"; Items = [] }

    let! result = OrderService.placeOrder deps request

    test <@ Result.isError result @>
}
```

### Theory を使ったパラメーター化テスト

```fsharp
[<Theory>]
[<InlineData("")>]
[<InlineData("   ")>]
let ``PlaceOrder rejects empty customer ID`` (customerId: string) =
    let request = { CustomerId = customerId; Items = [ validItem ] }
    let result = OrderService.placeOrder request
    result |> should be (ofCase <@ Error @>)

[<Theory>]
[<InlineData("", false)>]
[<InlineData("a", false)>]
[<InlineData("user@example.com", true)>]
[<InlineData("user+tag@example.co.uk", true)>]
let ``IsValidEmail returns expected result`` (email: string, expected: bool) =
    test <@ EmailValidator.isValid email = expected @>
```

## FsCheck を使ったプロパティベーステスト

### FsCheck.xUnit を使用する

```fsharp
open FsCheck
open FsCheck.Xunit

[<Property>]
let ``order total is always non-negative`` (items: NonEmptyList<PositiveInt * decimal>) =
    let orderItems =
        items.Get
        |> List.map (fun (qty, price) ->
            { Sku = "SKU"; Quantity = qty.Get; Price = abs price })
    let total = Order.calculateTotal orderItems
    total >= 0m

[<Property>]
let ``serialization roundtrips`` (order: Order) =
    let json = JsonSerializer.Serialize order
    let deserialized = JsonSerializer.Deserialize<Order> json
    deserialized = order
```

### カスタムジェネレーター

```fsharp
type OrderGenerators =
    static member ValidEmail () =
        gen {
            let! user = Gen.elements [ "alice"; "bob"; "carol" ]
            let! domain = Gen.elements [ "example.com"; "test.org" ]
            return $"{user}@{domain}"
        }
        |> Arb.fromGen

[<Property(Arbitrary = [| typeof<OrderGenerators> |])>]
let ``valid emails pass validation`` (email: string) =
    EmailValidator.isValid email
```

## 依存関係のモッキング

### 関数スタブ（推奨）

```fsharp
let createTestDeps () =
    let mutable savedOrders = []
    { FindOrder = fun id -> task { return Map.tryFind id testData }
      SaveOrder = fun order -> task { savedOrders <- order :: savedOrders }
      SendNotification = fun _ -> Task.CompletedTask }

[<Fact>]
let ``PlaceOrder saves the confirmed order`` () = task {
    let mutable saved = []
    let deps =
        { createTestDeps () with
            SaveOrder = fun order -> task { saved <- order :: saved } }

    let! _ = OrderService.placeOrder deps validRequest

    test <@ saved.Length = 1 @>
}
```

### .NETインターフェース用 NSubstitute

```fsharp
open NSubstitute

[<Fact>]
let ``calls repository with correct ID`` () = task {
    let repo = Substitute.For<IOrderRepository>()
    repo.FindByIdAsync(Arg.Any<Guid>(), Arg.Any<CancellationToken>())
        .Returns(Task.FromResult(Some testOrder))

    let service = OrderService(repo)
    let! _ = service.GetOrder(testOrder.Id, CancellationToken.None)

    do! repo.Received(1).FindByIdAsync(testOrder.Id, Arg.Any<CancellationToken>())
}
```

## ASP.NET Core インテグレーションテスト

```fsharp
type OrderApiTests (factory: WebApplicationFactory<Program>) =
    interface IClassFixture<WebApplicationFactory<Program>>

    let client =
        factory.WithWebHostBuilder(fun builder ->
            builder.ConfigureServices(fun services ->
                services.RemoveAll<DbContextOptions<AppDbContext>>() |> ignore
                services.AddDbContext<AppDbContext>(fun options ->
                    options.UseInMemoryDatabase("TestDb") |> ignore) |> ignore))
            .CreateClient()

    [<Fact>]
    member _.``GET order returns 404 when not found`` () = task {
        let! response = client.GetAsync($"/api/orders/{Guid.NewGuid()}")
        test <@ response.StatusCode = HttpStatusCode.NotFound @>
    }
```

## テスト組織

```
tests/
  MyApp.Tests/
    Unit/
      OrderServiceTests.fs
      PaymentServiceTests.fs
    Integration/
      OrderApiTests.fs
      OrderRepositoryTests.fs
    Properties/
      OrderPropertyTests.fs
    Helpers/
      TestData.fs
      TestDeps.fs
```

## よくあるアンチパターン

| アンチパターン | 修正 |
|---|---|
| 実装の詳細をテストする | 動作と結果をテストする |
| 可変の共有テスト状態 | テストごとに新鮮な状態を用意する |
| 非同期テストでの `Thread.Sleep` | タイムアウト付きの `Task.Delay` またはポーリングヘルパーを使用する |
| `sprintf` の出力をアサートする | 型付きの値とパターンマッチでアサートする |
| `CancellationToken` を無視する | 常に渡してキャンセルを検証する |
| プロパティベーステストをスキップする | 明確な不変条件を持つ関数にはFsCheckを使用する |

## 関連スキル

- `dotnet-patterns` - イディオマティックな.NETパターン、依存性注入、アーキテクチャ
- `csharp-testing` - C#テストパターン（WebApplicationFactoryやTestcontainersなどの共有インフラはF#にも適用できる）

## テストの実行

```bash
# すべてのテストを実行
dotnet test

# カバレッジ付きで実行
dotnet test --collect:"XPlat Code Coverage"

# 特定のプロジェクトを実行
dotnet test tests/MyApp.Tests/

# テスト名でフィルタリング
dotnet test --filter "FullyQualifiedName~OrderService"

# 開発中のウォッチモード
dotnet watch test --project tests/MyApp.Tests/
```
