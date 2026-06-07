---
name: liquid-glass-design
description: iOS 26 Liquid Glass デザインシステム — SwiftUI、UIKit、WidgetKit 向けのブラー、反射、インタラクティブなモーフィングを備えたダイナミックなガラスマテリアル。
---

# Liquid Glass デザインシステム（iOS 26）

Apple の Liquid Glass を実装するためのパターン。背後のコンテンツをぼかし、周囲のコンテンツから色と光を反射し、タッチおよびポインターのインタラクションに反応するダイナミックなマテリアルです。SwiftUI、UIKit、WidgetKit との統合をカバーします。

## 有効化のタイミング

- 新しいデザイン言語を使った iOS 26 以降のアプリの構築または更新
- ガラススタイルのボタン、カード、ツールバー、コンテナの実装
- ガラス要素間のモーフィングトランジションの作成
- ウィジェットへの Liquid Glass エフェクトの適用
- 既存のブラー/マテリアルエフェクトの新しい Liquid Glass API への移行

## コアパターン — SwiftUI

### 基本的なガラスエフェクト

任意のビューに Liquid Glass を追加する最もシンプルな方法:

```swift
Text("Hello, World!")
    .font(.title)
    .padding()
    .glassEffect()  // Default: regular variant, capsule shape
```

### シェイプとティントのカスタマイズ

```swift
Text("Hello, World!")
    .font(.title)
    .padding()
    .glassEffect(.regular.tint(.orange).interactive(), in: .rect(cornerRadius: 16.0))
```

主なカスタマイズオプション:
- `.regular` — 標準のガラスエフェクト
- `.tint(Color)` — 存在感を出すためのカラーティントを追加
- `.interactive()` — タッチおよびポインターのインタラクションに反応させる
- シェイプ: `.capsule`（デフォルト）、`.rect(cornerRadius:)`、`.circle`

### ガラスボタンスタイル

```swift
Button("Click Me") { /* action */ }
    .buttonStyle(.glass)

Button("Important") { /* action */ }
    .buttonStyle(.glassProminent)
```

### 複数要素向け GlassEffectContainer

パフォーマンスとモーフィングのために、複数のガラスビューは常にコンテナでラップする:

```swift
GlassEffectContainer(spacing: 40.0) {
    HStack(spacing: 40.0) {
        Image(systemName: "scribble.variable")
            .frame(width: 80.0, height: 80.0)
            .font(.system(size: 36))
            .glassEffect()

        Image(systemName: "eraser.fill")
            .frame(width: 80.0, height: 80.0)
            .font(.system(size: 36))
            .glassEffect()
    }
}
```

`spacing` パラメータはマージ距離を制御する — 要素が近いほどガラスシェイプが融合する。

### ガラスエフェクトの統合

`glassEffectUnion` を使って複数のビューを単一のガラスシェイプに結合する:

```swift
@Namespace private var namespace

GlassEffectContainer(spacing: 20.0) {
    HStack(spacing: 20.0) {
        ForEach(symbolSet.indices, id: \.self) { item in
            Image(systemName: symbolSet[item])
                .frame(width: 80.0, height: 80.0)
                .glassEffect()
                .glassEffectUnion(id: item < 2 ? "group1" : "group2", namespace: namespace)
        }
    }
}
```

### モーフィングトランジション

ガラス要素が表示/非表示になる際のスムーズなモーフィングを作成する:

```swift
@State private var isExpanded = false
@Namespace private var namespace

GlassEffectContainer(spacing: 40.0) {
    HStack(spacing: 40.0) {
        Image(systemName: "scribble.variable")
            .frame(width: 80.0, height: 80.0)
            .glassEffect()
            .glassEffectID("pencil", in: namespace)

        if isExpanded {
            Image(systemName: "eraser.fill")
                .frame(width: 80.0, height: 80.0)
                .glassEffect()
                .glassEffectID("eraser", in: namespace)
        }
    }
}

Button("Toggle") {
    withAnimation { isExpanded.toggle() }
}
.buttonStyle(.glass)
```

### サイドバー下への水平スクロールの延長

水平スクロールコンテンツをサイドバーやインスペクターの下まで延長させるには、`ScrollView` のコンテンツがコンテナの leading/trailing エッジまで達するようにする。システムはレイアウトがエッジまで延長されていれば、サイドバー下のスクロール動作を自動的に処理する — 追加のモディファイアは不要。

## コアパターン — UIKit

### 基本的な UIGlassEffect

```swift
let glassEffect = UIGlassEffect()
glassEffect.tintColor = UIColor.systemBlue.withAlphaComponent(0.3)
glassEffect.isInteractive = true

let visualEffectView = UIVisualEffectView(effect: glassEffect)
visualEffectView.translatesAutoresizingMaskIntoConstraints = false
visualEffectView.layer.cornerRadius = 20
visualEffectView.clipsToBounds = true

view.addSubview(visualEffectView)
NSLayoutConstraint.activate([
    visualEffectView.centerXAnchor.constraint(equalTo: view.centerXAnchor),
    visualEffectView.centerYAnchor.constraint(equalTo: view.centerYAnchor),
    visualEffectView.widthAnchor.constraint(equalToConstant: 200),
    visualEffectView.heightAnchor.constraint(equalToConstant: 120)
])

// Add content to contentView
let label = UILabel()
label.text = "Liquid Glass"
label.translatesAutoresizingMaskIntoConstraints = false
visualEffectView.contentView.addSubview(label)
NSLayoutConstraint.activate([
    label.centerXAnchor.constraint(equalTo: visualEffectView.contentView.centerXAnchor),
    label.centerYAnchor.constraint(equalTo: visualEffectView.contentView.centerYAnchor)
])
```

### 複数要素向け UIGlassContainerEffect

```swift
let containerEffect = UIGlassContainerEffect()
containerEffect.spacing = 40.0

let containerView = UIVisualEffectView(effect: containerEffect)

let firstGlass = UIVisualEffectView(effect: UIGlassEffect())
let secondGlass = UIVisualEffectView(effect: UIGlassEffect())

containerView.contentView.addSubview(firstGlass)
containerView.contentView.addSubview(secondGlass)
```

### スクロールエッジエフェクト

```swift
scrollView.topEdgeEffect.style = .automatic
scrollView.bottomEdgeEffect.style = .hard
scrollView.leftEdgeEffect.isHidden = true
```

### ツールバーガラス統合

```swift
let favoriteButton = UIBarButtonItem(image: UIImage(systemName: "heart"), style: .plain, target: self, action: #selector(favoriteAction))
favoriteButton.hidesSharedBackground = true  // Opt out of shared glass background
```

## コアパターン — WidgetKit

### レンダリングモードの検出

```swift
struct MyWidgetView: View {
    @Environment(\.widgetRenderingMode) var renderingMode

    var body: some View {
        if renderingMode == .accented {
            // Tinted mode: white-tinted, themed glass background
        } else {
            // Full color mode: standard appearance
        }
    }
}
```

### 視覚的階層のためのアクセントグループ

```swift
HStack {
    VStack(alignment: .leading) {
        Text("Title")
            .widgetAccentable()  // Accent group
        Text("Subtitle")
            // Primary group (default)
    }
    Image(systemName: "star.fill")
        .widgetAccentable()  // Accent group
}
```

### アクセントモードでの画像レンダリング

```swift
Image("myImage")
    .widgetAccentedRenderingMode(.monochrome)
```

### コンテナバックグラウンド

```swift
VStack { /* content */ }
    .containerBackground(for: .widget) {
        Color.blue.opacity(0.2)
    }
```

## 主要な設計上の判断

| 判断 | 根拠 |
|----------|-----------|
| GlassEffectContainer でラップする | パフォーマンスの最適化、ガラス要素間のモーフィングを有効化する |
| `spacing` パラメータ | マージ距離を制御 — 要素がブレンドするために必要な近さを微調整する |
| `@Namespace` + `glassEffectID` | ビュー階層変更時のスムーズなモーフィングトランジションを有効化する |
| `interactive()` モディファイア | タッチ/ポインター反応の明示的なオプトイン — すべてのガラスが反応すべきではない |
| UIKit での UIGlassContainerEffect | SwiftUI と一貫性を持たせるための同じコンテナパターン |
| ウィジェットのアクセントレンダリングモード | ユーザーがティントホーム画面を選択した際にシステムがティントガラスを適用する |

## ベストプラクティス

- **常に GlassEffectContainer を使用する** 複数の兄弟ビューにガラスを適用する場合 — モーフィングを有効にし、レンダリングパフォーマンスを向上させる
- **`.glassEffect()` は** 他の外観モディファイア（frame、font、padding）の後に適用する
- **`.interactive()` は** ユーザーインタラクション（ボタン、トグル可能なアイテム）に反応する要素にのみ使用する
- **コンテナの spacing を慎重に選択する** ガラスエフェクトがいつマージするかを制御するため
- **`withAnimation` を使用する** ビュー階層を変更する際にスムーズなモーフィングトランジションを有効にするため
- **複数の外観でテストする** — ライトモード、ダークモード、アクセント/ティントモード
- **アクセシビリティのコントラストを確保する** — ガラス上のテキストは読みやすくなければならない

## 避けるべきアンチパターン

- GlassEffectContainer なしで複数のスタンドアロンな `.glassEffect()` ビューを使用する
- ガラスエフェクトを過度にネストする — パフォーマンスと視覚的な明瞭さを低下させる
- すべてのビューにガラスを適用する — インタラクティブな要素、ツールバー、カードに限定する
- UIKit でコーナー半径を使用する際に `clipsToBounds = true` を忘れる
- ウィジェットでアクセントレンダリングモードを無視する — ティントホーム画面の外観が崩れる
- ガラスの後ろに不透明な背景を使用する — 半透明効果が台無しになる

## 使用するタイミング

- 新しい iOS 26 デザインを使ったナビゲーションバー、ツールバー、タブバー
- フローティングアクションボタンとカードスタイルのコンテナ
- 視覚的な奥行きとタッチフィードバックが必要なインタラクティブコントロール
- システムの Liquid Glass 外観と統合すべきウィジェット
- 関連する UI 状態間のモーフィングトランジション
