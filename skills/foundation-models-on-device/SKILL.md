---
name: foundation-models-on-device
description: iOS 26+ 向けオンデバイス LLM 用 Apple FoundationModels フレームワーク — テキスト生成、@Generable を使ったガイド付き生成、ツール呼び出し、スナップショットストリーミング。
---

# FoundationModels: オンデバイス LLM（iOS 26）

FoundationModels フレームワークを使って Apple のオンデバイス言語モデルをアプリに統合するためのパターン。テキスト生成、`@Generable` による構造化出力、カスタムツール呼び出し、スナップショットストリーミングを網羅 — すべてプライバシー保護とオフラインサポートのためにオンデバイスで動作する。

## 有効にするタイミング

- Apple Intelligence オンデバイスを使った AI 機能を構築するとき
- クラウド依存なしにテキストを生成・要約するとき
- 自然言語入力から構造化データを抽出するとき
- ドメイン固有の AI アクション向けにカスタムツール呼び出しを実装するとき
- リアルタイム UI 更新のために構造化レスポンスをストリーミングするとき
- プライバシー保護 AI が必要なとき（デバイス外にデータが出ない）

## 基本パターン — 利用可能性チェック

セッションを作成する前に常にモデルの利用可能性を確認する:

```swift
struct GenerativeView: View {
    private var model = SystemLanguageModel.default

    var body: some View {
        switch model.availability {
        case .available:
            ContentView()
        case .unavailable(.deviceNotEligible):
            Text("Device not eligible for Apple Intelligence")
        case .unavailable(.appleIntelligenceNotEnabled):
            Text("Please enable Apple Intelligence in Settings")
        case .unavailable(.modelNotReady):
            Text("Model is downloading or not ready")
        case .unavailable(let other):
            Text("Model unavailable: \(other)")
        }
    }
}
```

## 基本パターン — 基本セッション

```swift
// シングルターン: 毎回新しいセッションを作成
let session = LanguageModelSession()
let response = try await session.respond(to: "What's a good month to visit Paris?")
print(response.content)

// マルチターン: 会話コンテキストのためにセッションを再利用
let session = LanguageModelSession(instructions: """
    You are a cooking assistant.
    Provide recipe suggestions based on ingredients.
    Keep suggestions brief and practical.
    """)

let first = try await session.respond(to: "I have chicken and rice")
let followUp = try await session.respond(to: "What about a vegetarian option?")
```

instructions のポイント:
- モデルの役割を定義する（「あなたはメンターです」）
- 行うべきことを指定する（「カレンダーイベントを抽出する手助けをしてください」）
- スタイルの好みを設定する（「できるだけ簡潔に答えてください」）
- 安全策を追加する（「危険なリクエストには『対応できません』と答えてください」）

## 基本パターン — @Generable を使ったガイド付き生成

生の文字列の代わりに構造化された Swift 型を生成する:

### 1. Generable 型を定義する

```swift
@Generable(description: "Basic profile information about a cat")
struct CatProfile {
    var name: String

    @Guide(description: "The age of the cat", .range(0...20))
    var age: Int

    @Guide(description: "A one sentence profile about the cat's personality")
    var profile: String
}
```

### 2. 構造化出力をリクエストする

```swift
let response = try await session.respond(
    to: "Generate a cute rescue cat",
    generating: CatProfile.self
)

// 構造化フィールドに直接アクセス
print("Name: \(response.content.name)")
print("Age: \(response.content.age)")
print("Profile: \(response.content.profile)")
```

### サポートされる @Guide 制約

- `.range(0...20)` — 数値の範囲
- `.count(3)` — 配列の要素数
- `description:` — 生成のためのセマンティックガイダンス

## 基本パターン — ツール呼び出し

ドメイン固有のタスクのためにモデルがカスタムコードを呼び出せるようにする:

### 1. ツールを定義する

```swift
struct RecipeSearchTool: Tool {
    let name = "recipe_search"
    let description = "Search for recipes matching a given term and return a list of results."

    @Generable
    struct Arguments {
        var searchTerm: String
        var numberOfResults: Int
    }

    func call(arguments: Arguments) async throws -> ToolOutput {
        let recipes = await searchRecipes(
            term: arguments.searchTerm,
            limit: arguments.numberOfResults
        )
        return .string(recipes.map { "- \($0.name): \($0.description)" }.joined(separator: "\n"))
    }
}
```

### 2. ツールを持つセッションを作成する

```swift
let session = LanguageModelSession(tools: [RecipeSearchTool()])
let response = try await session.respond(to: "Find me some pasta recipes")
```

### 3. ツールエラーを処理する

```swift
do {
    let answer = try await session.respond(to: "Find a recipe for tomato soup.")
} catch let error as LanguageModelSession.ToolCallError {
    print(error.tool.name)
    if case .databaseIsEmpty = error.underlyingError as? RecipeSearchToolError {
        // 特定のツールエラーを処理
    }
}
```

## 基本パターン — スナップショットストリーミング

`PartiallyGenerated` 型を使ってリアルタイム UI のために構造化レスポンスをストリーミングする:

```swift
@Generable
struct TripIdeas {
    @Guide(description: "Ideas for upcoming trips")
    var ideas: [String]
}

let stream = session.streamResponse(
    to: "What are some exciting trip ideas?",
    generating: TripIdeas.self
)

for try await partial in stream {
    // partial: TripIdeas.PartiallyGenerated（すべてのプロパティが Optional）
    print(partial)
}
```

### SwiftUI との統合

```swift
@State private var partialResult: TripIdeas.PartiallyGenerated?
@State private var errorMessage: String?

var body: some View {
    List {
        ForEach(partialResult?.ideas ?? [], id: \.self) { idea in
            Text(idea)
        }
    }
    .overlay {
        if let errorMessage { Text(errorMessage).foregroundStyle(.red) }
    }
    .task {
        do {
            let stream = session.streamResponse(to: prompt, generating: TripIdeas.self)
            for try await partial in stream {
                partialResult = partial
            }
        } catch {
            errorMessage = error.localizedDescription
        }
    }
}
```

## 主要な設計判断

| 判断 | 根拠 |
|----------|-----------|
| オンデバイス実行 | プライバシー — デバイス外にデータが出ない; オフライン動作 |
| 4,096 トークン制限 | オンデバイスモデルの制約; 大きなデータはセッションを跨いでチャンク分割 |
| スナップショットストリーミング（デルタではない） | 構造化出力に適している; 各スナップショットは完全な部分状態 |
| `@Generable` マクロ | 構造化生成のコンパイル時安全性; `PartiallyGenerated` 型を自動生成 |
| セッションあたり 1 リクエスト | `isResponding` が並行リクエストを防ぐ; 必要なら複数セッションを作成 |
| `response.content`（`.output` ではない） | 正しい API — 常に `.content` プロパティを通じて結果にアクセス |

## ベストプラクティス

- **常に `model.availability` を確認する** — セッションを作成する前に; すべての利用不可ケースを処理する
- **`instructions` を使う** — モデルの振る舞いをガイドする — プロンプトよりも優先される
- **`isResponding` を確認する** — 新しいリクエストを送る前に — セッションは一度に 1 リクエストを処理する
- **`response.content` にアクセスする** — 結果のために — `.output` ではない
- **大きな入力をチャンクに分割する** — 4,096 トークン制限は instructions + プロンプト + 出力の合計に適用される
- **`@Generable` を使う** — 構造化出力のために — 生の文字列を解析するより強力な保証
- **`GenerationOptions(temperature:)`** を使って創造性を調整する（高いほど創造的）
- **Instruments で監視する** — Xcode Instruments を使ってリクエストパフォーマンスをプロファイルする

## 避けるべきアンチパターン

- `model.availability` を確認せずにセッションを作成する
- 4,096 トークンのコンテキストウィンドウを超える入力を送る
- 1 つのセッションで並行リクエストを試みる
- レスポンスデータにアクセスするために `.content` の代わりに `.output` を使う
- `@Generable` の構造化出力が使えるのに生の文字列レスポンスを解析する
- 1 つのプロンプトに複雑なマルチステップロジックを構築する — 複数の焦点を絞ったプロンプトに分割する
- モデルが常に利用可能と仮定する — デバイスの対象要件と設定は異なる

## 使用するタイミング

- プライバシー重視のアプリ向けのオンデバイステキスト生成
- ユーザー入力（フォーム、自然言語コマンド）からの構造化データ抽出
- オフラインで動作しなければならない AI アシスト機能
- 生成されたコンテンツを段階的に表示するストリーミング UI
- ツール呼び出しによるドメイン固有の AI アクション（検索、計算、ルックアップ）
