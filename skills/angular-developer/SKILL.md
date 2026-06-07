---
name: angular-developer
description: Angular コードを生成し、アーキテクチャのガイダンスを提供します。プロジェクト・コンポーネント・サービスの作成、またはリアクティビティ（シグナル、linkedSignal、resource）・フォーム・依存性注入・ルーティング・SSR・アクセシビリティ（ARIA）・アニメーション・スタイリング（コンポーネントスタイル、Tailwind CSS）・テスト・CLI ツールのベストプラクティスについてトリガーします。
origin: ECC
---

# Angular 開発者ガイドライン

## アクティブにするタイミング

- Angular プロジェクトまたはコードベースで作業するとき
- 新しい Angular プロジェクト、アプリケーション、またはライブラリを作成またはスキャフォールドするとき
- コンポーネント、サービス、ディレクティブ、パイプ、ガード、またはリゾルバを生成するとき
- Angular シグナル、`linkedSignal`、または `resource` でリアクティビティを実装するとき
- Angular フォーム（シグナルフォーム、リアクティブフォーム、またはテンプレート駆動）を使用するとき
- 依存性注入、ルーティング、遅延読み込み、またはルートガードを設定するとき
- アクセシビリティ（ARIA）、アニメーション、またはコンポーネントスタイリングを追加するとき
- Angular 固有のテストを書くまたはデバッグするとき（ユニット、コンポーネントハーネス、E2E）
- Angular CLI ツールまたは Angular MCP サーバーを設定するとき

1. ベストプラクティスと利用可能な機能はバージョンによって大きく異なるため、ガイダンスを提供する前に必ずプロジェクトの Angular バージョンを分析する。Angular CLI で新規プロジェクトを作成する場合、ユーザーに促されない限りバージョンを指定しない。

2. コードを生成する際は、保守性とパフォーマンスのために Angular のスタイルガイドとベストプラクティスに従う。コンポーネント、サービス、ディレクティブ、パイプ、ルートのスキャフォールドには Angular CLI を使用して一貫性を確保する。

3. コードの生成が完了したら、`ng build` を実行してビルドエラーがないことを確認する。エラーがある場合は、エラーメッセージを分析して修正してから次に進む。生成されたコードが正しく機能することを保証するために、このステップをスキップしない。

## 新規プロジェクトの作成

ユーザーからガイドラインが提供されない場合、新しい Angular プロジェクトを作成する際はこれらのデフォルトを使用する:

1. ユーザーが別途指定しない限り、最新の安定版 Angular を使用する。
2. 対象の Angular バージョンがサポートしている場合のみ、新規プロジェクトにはシグナルフォームを優先する。[詳細はこちら](references/signal-forms.md)。

**`ng new` の実行ルール:**
新しい Angular プロジェクトを作成するよう求められた場合、次の厳密なステップに従って正しい実行コマンドを決定する必要がある:

**ステップ 1: ユーザーが明示的なバージョンを指定しているか確認する。**

- **もし**ユーザーが特定のバージョン（例: Angular 15）を要求した場合、ローカルのインストールをバイパスして厳密に `npx` を使用する。
- **コマンド:** `npx @angular/cli@<requested_version> new <project-name>`

**ステップ 2: 既存の Angular インストールを確認する。**

- **もし**特定のバージョンが要求されていない場合、ターミナルで `ng version` を実行して Angular CLI がシステムにインストールされているか確認する。
- **もし**コマンドが成功してインストール済みバージョンが返ってきた場合、ローカル/グローバルインストールを直接使用する。
- **コマンド:** `ng new <project-name>`

**ステップ 3: 最新版へのフォールバック。**

- **もし**特定のバージョンが要求されておらず、かつ `ng version` コマンドが失敗した場合（Angular がインストールされていないことを示す）、`npx` を使用して最新版を取得する必要がある。
- **コマンド:** `npx @angular/cli@latest new <project-name>`

## コンポーネント

Angular コンポーネントを使用する際は、タスクに応じて以下のリファレンスを参照する:

- **基礎**: 構造、メタデータ、コアコンセプト、テンプレートの制御フロー（@if、@for、@switch）。[components.md](references/components.md) を参照
- **インプット**: シグナルベースのインプット、変換、モデルインプット。[inputs.md](references/inputs.md) を参照
- **アウトプット**: シグナルベースのアウトプットとカスタムイベントのベストプラクティス。[outputs.md](references/outputs.md) を参照
- **ホスト要素**: ホストバインディングと属性インジェクション。[host-elements.md](references/host-elements.md) を参照

上記のリファレンスに深いドキュメントがない場合は、`https://angular.dev/guide/components` のドキュメントを参照する。

## リアクティビティとデータ管理

状態とデータのリアクティビティを管理する際は、Angular シグナルを使用して以下のリファレンスを参照する:

- **シグナルの概要**: コアシグナルのコンセプト（`signal`、`computed`）、リアクティブコンテキスト、`untracked`。[signals-overview.md](references/signals-overview.md) を参照
- **依存状態（`linkedSignal`）**: ソースシグナルにリンクした書き込み可能な状態の作成。[linked-signal.md](references/linked-signal.md) を参照
- **非同期リアクティビティ（`resource`）**: 非同期データをシグナル状態に直接取得する。[resource.md](references/resource.md) を参照
- **副作用（`effect`）**: ログ記録、サードパーティ DOM 操作（`afterRenderEffect`）、エフェクトを使用すべきでない場合。[effects.md](references/effects.md) を参照

## フォーム

新しいアプリのほとんどの場合、**シグナルフォームを優先する**。フォームの決定を行う際は、プロジェクトを分析して以下のガイドラインを考慮する:

- アプリケーションバージョンがシグナルフォームをサポートしており、新しいフォームの場合は**シグナルフォームを優先する**。
- 古いアプリケーションや既存のフォームの場合は、アプリケーションの現在のフォーム戦略に合わせる。

- **シグナルフォーム**: フォーム状態管理にシグナルを使用する。[signal-forms.md](references/signal-forms.md) を参照
- **テンプレート駆動フォーム**: シンプルなフォームに使用する。[template-driven-forms.md](references/template-driven-forms.md) を参照
- **リアクティブフォーム**: 複雑なフォームに使用する。[reactive-forms.md](references/reactive-forms.md) を参照

## 依存性注入

Angular での依存性注入を実装する際は、以下のガイドラインに従う:

- **基礎**: 依存性注入、サービス、`inject()` 関数の概要。[di-fundamentals.md](references/di-fundamentals.md) を参照
- **サービスの作成と使用**: サービスの作成、`providedIn: 'root'` オプション、コンポーネントや他のサービスへのインジェクション。[creating-services.md](references/creating-services.md) を参照
- **依存関係プロバイダーの定義**: 自動 vs 手動プロビジョン、`InjectionToken`、`useClass`、`useValue`、`useFactory`、スコープ。[defining-providers.md](references/defining-providers.md) を参照
- **インジェクションコンテキスト**: `inject()` が許可される場所、`runInInjectionContext`、`assertInInjectionContext`。[injection-context.md](references/injection-context.md) を参照
- **階層インジェクター**: `EnvironmentInjector` vs `ElementInjector`、解決ルール、修飾子（`optional`、`skipSelf`）、`providers` vs `viewProviders`。[hierarchical-injectors.md](references/hierarchical-injectors.md) を参照

## Angular Aria

次のいずれかのパターンでアクセシブルなカスタムコンポーネントを構築する場合: Accordion、Listbox、Combobox、Menu、Tabs、Toolbar、Tree、Grid、以下のリファレンスを参照する:

- **Angular Aria コンポーネント**: ヘッドレスでアクセシブルなコンポーネント（Accordion、Listbox、Combobox、Menu、Tabs、Toolbar、Tree、Grid）の構築と ARIA 属性のスタイリング。[angular-aria.md](references/angular-aria.md) を参照

## ルーティング

Angular でのナビゲーションを実装する際は、以下のリファレンスを参照する:

- **ルートの定義**: URL パス、静的 vs 動的セグメント、ワイルドカード、リダイレクト。[define-routes.md](references/define-routes.md) を参照
- **ルート読み込み戦略**: 積極的読み込み vs 遅延読み込み、コンテキスト対応読み込み。[loading-strategies.md](references/loading-strategies.md) を参照
- **アウトレットでルートを表示する**: `<router-outlet>` の使用、ネストされたアウトレット、名前付きアウトレット。[show-routes-with-outlets.md](references/show-routes-with-outlets.md) を参照
- **ルートへのナビゲート**: `RouterLink` による宣言的ナビゲーションと `Router` によるプログラム的ナビゲーション。[navigate-to-routes.md](references/navigate-to-routes.md) を参照
- **ガードによるルートアクセス制御**: セキュリティのための `CanActivate`、`CanMatch`、その他のガードの実装。[route-guards.md](references/route-guards.md) を参照
- **データリゾルバー**: `ResolveFn` によるルートアクティベーション前のデータのプリフェッチ。[data-resolvers.md](references/data-resolvers.md) を参照
- **ルーターライフサイクルとイベント**: ナビゲーションイベントの時系列順とデバッグ。[router-lifecycle.md](references/router-lifecycle.md) を参照
- **レンダリング戦略**: CSR、SSG（プリレンダリング）、ハイドレーションを使用した SSR。[rendering-strategies.md](references/rendering-strategies.md) を参照
- **ルートトランジションアニメーション**: View Transitions API の有効化とカスタマイズ。[route-animations.md](references/route-animations.md) を参照

より深いドキュメントやコンテキストが必要な場合は、[Angular ルーティング公式ガイド](https://angular.dev/guide/routing) を参照する。

## スタイリングとアニメーション

Angular でスタイリングとアニメーションを実装する際は、以下のリファレンスを参照する:

- **Angular で Tailwind CSS を使用する**: Angular プロジェクトへの Tailwind CSS の統合。[tailwind-css.md](references/tailwind-css.md) を参照
- **Angular アニメーション**: 動的エフェクトにはネイティブ CSS（推奨）またはレガシー DSL を使用する。[angular-animations.md](references/angular-animations.md) を参照
- **コンポーネントのスタイリング**: コンポーネントスタイルとカプセル化のベストプラクティス。[component-styling.md](references/component-styling.md) を参照

## テスト

テストを書くまたは更新する際は、タスクに応じて以下のリファレンスを参照する:

- **基礎**: ユニットテスト、非同期パターン、`TestBed` のベストプラクティス。[testing-fundamentals.md](references/testing-fundamentals.md) を参照
- **コンポーネントハーネス**: 堅牢なコンポーネント操作のための標準パターン。[component-harnesses.md](references/component-harnesses.md) を参照
- **ルーターテスト**: 信頼性の高いナビゲーションテストのための `RouterTestingHarness` の使用。[router-testing.md](references/router-testing.md) を参照
- **エンドツーエンド（E2E）テスト**: Cypress または Playwright を使用した E2E テストのベストプラクティス。[e2e-testing.md](references/e2e-testing.md) を参照

## ツール

Angular ツールを使用する際は、以下のリファレンスを参照する:

- **Angular CLI**: アプリケーションの作成、コード生成（コンポーネント、ルート、サービス）、サーブ、ビルド。[cli.md](references/cli.md) を参照
- **Angular MCP サーバー**: 利用可能なツール、設定、実験的な機能。[mcp.md](references/mcp.md) を参照

## アンチパターン

- シグナルフォームフィールドの初期値に `null` または `undefined` を使用する — 代わりに `''`、`0`、`[]` を使用する
- フィールドを最初に呼び出さずにフォームフィールドの状態フラグにアクセスする: `form.field.valid()` — `form.field().valid()` を使用する
- 対象の Angular バージョンがシグナルフォームをサポートしているにも関わらず、古いフォーム API で新しいフォームを始める
- `[formField]` インプットに `min`、`max`、`value`、`disabled`、`readonly` HTML 属性を設定する — これらはスキーマルールとして定義する
- インジェクションコンテキスト外で `inject()` を呼び出す — 必要な場合は `runInInjectionContext` を使用する
- `computed()` を使うべき派生状態に `effect()` を使用する
- ネストされた `@for` ループで `$parent.$index` を参照する — Angular は `$parent` をサポートしない。代わりに `let outerIdx = $index` を使用する

## 関連スキル

- `tdd-workflow` — Angular コンポーネントとサービスに適用可能なテスト駆動開発ワークフロー
- `security-review` — Angular 固有の懸念事項を含む Web アプリケーションのセキュリティチェックリスト
- `frontend-patterns` — React/Next.js アプローチのコンテキストのための一般的なフロントエンドパターン
