---
name: prompt-optimizer
description: 未加工のプロンプトを分析し、意図とギャップを特定し、ECCコンポーネント（スキル／コマンド／エージェント／フック）をマッチングして、そのまま貼り付けられる最適化済みプロンプトを出力する。アドバイザリー役のみ — タスク自体は実行しない。トリガー条件: ユーザーが「optimize prompt」「improve my prompt」「how to write a prompt for」「help me prompt」「rewrite this prompt」または明示的にプロンプト品質の向上を求めた場合。中国語同等表現にも対応: 「优化prompt」「改进prompt」「怎么写prompt」「帮我优化这个指令」。非トリガー条件: ユーザーがタスクを直接実行させたい場合、または「just do it」/「直接做」と言った場合。「优化代码」「优化性能」「optimize performance」「optimize this code」ではトリガーしない — これらはリファクタリング／パフォーマンスタスクであり、プロンプト最適化ではない。
origin: community
metadata:
  author: YannJY02
  version: "1.0.0"
---

# プロンプト最適化ツール

下書きプロンプトを分析・批評し、ECCエコシステムのコンポーネントにマッチングして、ユーザーがそのまま貼り付けて実行できる完全な最適化プロンプトを出力する。

## 使用するタイミング

- ユーザーが「optimize this prompt」「improve my prompt」「rewrite this prompt」と言った場合
- ユーザーが「help me write a better prompt for...」と言った場合
- ユーザーが「what's the best way to ask Claude Code to...」と言った場合
- ユーザーが「优化prompt」「改进prompt」「怎么写prompt」「帮我优化这个指令」と言った場合
- ユーザーが下書きプロンプトを貼り付けてフィードバックや改善を求めた場合
- ユーザーが「I don't know how to prompt for this」と言った場合
- ユーザーが「how should I use ECC for...」と言った場合
- ユーザーが明示的に `/prompt-optimize` を呼び出した場合

### 使用しないタイミング

- ユーザーがタスクを直接実行させたい場合（そのまま実行する）
- ユーザーが「优化代码」「优化性能」「optimize this code」「optimize performance」と言った場合 — これらはリファクタリングタスクであり、プロンプト最適化ではない
- ユーザーがECC設定について質問している場合（代わりに `configure-ecc` を使用）
- ユーザーがスキル一覧を求めている場合（代わりに `skill-stocktake` を使用）
- ユーザーが「just do it」または「直接做」と言った場合

## 動作の仕組み

**アドバイザリーのみ — ユーザーのタスクは実行しない。**

コードの作成、ファイルの作成、コマンドの実行、その他いかなる実装アクションも行わない。出力するのは分析結果と最適化プロンプトのみ。

ユーザーが「just do it」「直接做」「don't optimize, just execute」と言った場合、このスキル内で実装モードに切り替えない。このスキルは最適化済みプロンプトのみを生成することをユーザーに伝え、代わりに通常のタスクリクエストを行うよう指示する。

以下の6フェーズのパイプラインを順番に実行する。結果は以下の出力フォーマットを使用して提示する。

### 分析パイプライン

### フェーズ0: プロジェクト検出

プロンプトを分析する前に、現在のプロジェクトコンテキストを検出する:

1. 作業ディレクトリに `CLAUDE.md` が存在するか確認 — プロジェクト規約を読む
2. プロジェクトファイルから技術スタックを検出する:
   - `package.json` → Node.js / TypeScript / React / Next.js
   - `go.mod` → Go
   - `pyproject.toml` / `requirements.txt` → Python
   - `Cargo.toml` → Rust
   - `build.gradle` / `pom.xml` → Java / Kotlin（ビルドファイルに `quarkus` があれば→Quarkus、`spring-boot` があれば→Spring Boot）
   - `Package.swift` → Swift
   - `Gemfile` → Ruby
   - `composer.json` → PHP
   - `*.csproj` / `*.sln` → .NET
   - `Makefile` / `CMakeLists.txt` → C / C++
   - `cpanfile` / `Makefile.PL` → Perl
3. 検出した技術スタックをフェーズ3とフェーズ4で使用するためにメモする

プロジェクトファイルが見つからない場合（例: プロンプトが抽象的または新規プロジェクト向けの場合）、検出をスキップしてフェーズ4で「技術スタック不明」とフラグを立てる。

### フェーズ1: 意図検出

ユーザーのタスクを1つ以上のカテゴリに分類する:

| カテゴリ | シグナルワード | 例 |
|----------|-------------|---------|
| 新機能 | build, create, add, implement, 创建, 实现, 添加 | 「Build a login page」 |
| バグ修正 | fix, broken, not working, error, 修复, 报错 | 「Fix the auth flow」 |
| リファクタリング | refactor, clean up, restructure, 重构, 整理 | 「Refactor the API layer」 |
| 調査 | how to, what is, explore, investigate, 怎么, 如何 | 「How to add SSO」 |
| テスト | test, coverage, verify, 测试, 覆盖率 | 「Add tests for the cart」 |
| レビュー | review, audit, check, 审查, 检查 | 「Review my PR」 |
| ドキュメント | document, update docs, 文档 | 「Update the API docs」 |
| インフラ | deploy, CI, docker, database, 部署, 数据库 | 「Set up CI/CD pipeline」 |
| 設計 | design, architecture, plan, 设计, 架构 | 「Design the data model」 |

### フェーズ2: スコープ評価

フェーズ0でプロジェクトが検出された場合はコードベースのサイズをシグナルとして使用する。検出されなかった場合はプロンプトの記述のみから推定し、不確かな推定であることを記す。

| スコープ | ヒューリスティック | オーケストレーション |
|-------|-----------|---------------|
| 極小 | 単一ファイル、50行未満 | 直接実行 |
| 小 | 単一コンポーネントまたはモジュール | 単一コマンドまたはスキル |
| 中 | 複数コンポーネント、同一ドメイン | コマンドチェーン + /verify |
| 大 | クロスドメイン、5ファイル以上 | /plan を先に実行、その後フェーズ実行 |
| 超大 | マルチセッション、マルチPR、アーキテクチャ変更 | blueprintスキルでマルチセッション計画を作成 |

### フェーズ3: ECCコンポーネントのマッチング

意図 + スコープ + 技術スタック（フェーズ0）を特定のECCコンポーネントにマッピングする。

#### 意図タイプ別

| 意図 | コマンド | スキル | エージェント |
|--------|----------|--------|--------|
| 新機能 | /plan, /tdd, /code-review, /verify | tdd-workflow, verification-loop | planner, tdd-guide, code-reviewer |
| バグ修正 | /tdd, /build-fix, /verify | tdd-workflow | tdd-guide, build-error-resolver |
| リファクタリング | /refactor-clean, /code-review, /verify | verification-loop | refactor-cleaner, code-reviewer |
| 調査 | /plan | search-first, iterative-retrieval | — |
| テスト | /tdd, /e2e, /test-coverage | tdd-workflow, e2e-testing | tdd-guide, e2e-runner |
| レビュー | /code-review | security-review | code-reviewer, security-reviewer |
| ドキュメント | /update-docs, /update-codemaps | — | doc-updater |
| インフラ | /plan, /verify | docker-patterns, deployment-patterns, database-migrations | architect |
| 設計（中〜大） | /plan | — | planner, architect |
| 設計（超大） | — | blueprint（スキルとして呼び出す） | planner, architect |

#### 技術スタック別

| 技術スタック | 追加するスキル | エージェント |
|------------|--------------|-------|
| Python / Django | django-patterns, django-tdd, django-security, django-verification, python-patterns, python-testing | python-reviewer |
| Go | golang-patterns, golang-testing | go-reviewer, go-build-resolver |
| Spring Boot / Java | springboot-patterns, springboot-tdd, springboot-security, springboot-verification, java-coding-standards, jpa-patterns | java-reviewer |
| Quarkus / Java | quarkus-patterns, quarkus-tdd, quarkus-security, quarkus-verification, java-coding-standards, jpa-patterns | java-reviewer |
| Kotlin / Android | kotlin-coroutines-flows, compose-multiplatform-patterns, android-clean-architecture | kotlin-reviewer |
| TypeScript / React | frontend-patterns, backend-patterns, coding-standards | code-reviewer |
| Swift / iOS | swiftui-patterns, swift-concurrency-6-2, swift-actor-persistence, swift-protocol-di-testing | code-reviewer |
| PostgreSQL | postgres-patterns, database-migrations | database-reviewer |
| Perl | perl-patterns, perl-testing, perl-security | code-reviewer |
| C++ | cpp-coding-standards, cpp-testing | code-reviewer |
| その他／未リスト | coding-standards（汎用） | code-reviewer |

### フェーズ4: 欠落コンテキストの検出

プロンプトに欠けている重要情報をスキャンする。各項目についてフェーズ0で自動検出されたか、ユーザーが供給する必要があるかを確認する:

- [ ] **技術スタック** — フェーズ0で検出されたか、ユーザーが指定する必要があるか？
- [ ] **対象スコープ** — ファイル、ディレクトリ、またはモジュールが明記されているか？
- [ ] **受け入れ基準** — タスクの完了をどのように判断するか？
- [ ] **エラーハンドリング** — エッジケースと障害モードが対処されているか？
- [ ] **セキュリティ要件** — 認証、入力バリデーション、シークレット？
- [ ] **テスト要件** — ユニット、統合、E2E？
- [ ] **パフォーマンス制約** — 負荷、レイテンシ、リソース制限？
- [ ] **UI/UX要件** — デザイン仕様、レスポンシブ、アクセシビリティ？（フロントエンドの場合）
- [ ] **データベース変更** — スキーマ、マイグレーション、インデックス？（データレイヤーの場合）
- [ ] **既存パターン** — 参照すべきファイルや従うべき規約？
- [ ] **スコープの境界** — 行わないこと？

**3つ以上の重要項目が欠落している場合**、最適化プロンプトを生成する前にユーザーへ最大3つの確認質問を行う。その後、回答を最適化プロンプトに組み込む。

### フェーズ5: ワークフローとモデルの推奨

このプロンプトが開発ライフサイクルのどこに位置するかを判断する:

```
調査 → 計画 → 実装（TDD） → レビュー → 検証 → コミット
```

中規模以上のタスクは常に /plan から開始する。超大規模タスクにはblueprintスキルを使用する。

**モデル推奨**（出力に含める）:

| スコープ | 推奨モデル | 理由 |
|-------|------------------|-----------|
| 極小〜小 | Sonnet 4.6 | 単純なタスクに対して高速かつコスト効率が良い |
| 中 | Sonnet 4.6 | 標準的な作業に最適なコーディングモデル |
| 大 | Sonnet 4.6（メイン）+ Opus 4.6（計画） | アーキテクチャにOpus、実装にSonnet |
| 超大 | Opus 4.6（blueprint）+ Sonnet 4.6（実行） | マルチセッション計画に深い推論 |

**マルチプロンプト分割**（大規模／超大規模スコープ向け）:

単一セッションを超えるタスクは、順番に実行するプロンプトに分割する:
- プロンプト1: 調査 + 計画（search-firstスキルを使用、その後 /plan）
- プロンプト2〜N: 1フェーズずつ実装（各フェーズは /verify で終了）
- 最終プロンプト: 統合テスト + 全フェーズにわたる /code-review
- セッション間でコンテキストを保持するために /save-session と /resume-session を使用

---

## 出力フォーマット

以下の正確な構造で分析を提示する。ユーザーの入力と同じ言語で回答する。

### セクション1: プロンプト診断

**強み:** 元のプロンプトが優れている点をリストアップする。

**問題点:**

| 問題 | 影響 | 修正案 |
|-------|--------|---------------|
| （問題） | （結果） | （修正方法） |

**要確認事項:** ユーザーが回答すべき質問の番号付きリスト。
フェーズ0で自動検出された場合は、質問する代わりにその情報を記載する。

### セクション2: 推奨ECCコンポーネント

| タイプ | コンポーネント | 目的 |
|------|-----------|---------|
| コマンド | /plan | コーディング前にアーキテクチャを計画する |
| スキル | tdd-workflow | TDD方法論のガイダンス |
| エージェント | code-reviewer | 実装後のレビュー |
| モデル | Sonnet 4.6 | このスコープに推奨 |

### セクション3: 最適化プロンプト — 完全版

完全な最適化プロンプトを単一のフェンス付きコードブロック内に提示する。
プロンプトは自己完結型で、コピー&ペーストですぐに使えるものでなければならない。以下を含む:
- コンテキスト付きの明確なタスク説明
- 技術スタック（検出済みまたは指定済み）
- 適切なワークフロー段階での /command 呼び出し
- 受け入れ基準
- 検証ステップ
- スコープの境界（行わないこと）

blueprintを参照する項目の場合: 「blueprintスキルを使用して...」と記述する
（`/blueprint` ではなく、blueprintはスキルであってコマンドではない）。

### セクション4: 最適化プロンプト — 簡略版

ECC経験者向けのコンパクトバージョン。意図タイプ別に変化させる:

| 意図 | 簡略パターン |
|--------|--------------|
| 新機能 | `/plan [機能]. /tdd で実装。/code-review。/verify。` |
| バグ修正 | `/tdd — [バグ]の失敗テストを作成。グリーンになるまで修正。/verify。` |
| リファクタリング | `/refactor-clean [スコープ]。/code-review。/verify。` |
| 調査 | `[トピック]にsearch-firstスキルを使用。調査結果に基づき /plan。` |
| テスト | `/tdd [モジュール]。重要フローに /e2e。/test-coverage。` |
| レビュー | `/code-review。次にsecurity-reviewerエージェントを使用。` |
| ドキュメント | `/update-docs。/update-codemaps。` |
| 超大規模 | `"[目標]"に blueprintスキルを使用。/verify ゲートでフェーズを実行。` |

### セクション5: 改善の根拠

| 改善点 | 理由 |
|-------------|--------|
| （追加した内容） | （重要な理由） |

### フッター

> これが必要なものでない場合は、何を調整すべきか教えてください。プロンプト最適化ではなく実行を希望する場合は、通常のタスクリクエストを行ってください。

---

## 例

### トリガー例

- 「Optimize this prompt for ECC」
- 「Rewrite this prompt so Claude Code uses the right commands」
- 「帮我优化这个指令」
- 「How should I prompt ECC for this task?」

### 例1: 曖昧な中国語プロンプト（プロジェクト検出あり）

**ユーザー入力:**
```
帮我写一个用户登录页面
```

**フェーズ0が検出:** `package.json` に Next.js 15、TypeScript、Tailwind CSS

**最適化プロンプト（完全版）:**
```
使用项目现有技术栈（Next.js 15 + TypeScript + Tailwind CSS）实现用户登录页面。

技术要求：
- 沿用项目现有的组件结构和路由约定
- 表单验证使用项目中已有的验证方案（检查是否已用 Zod/Yup/其他）
- 认证方式：沿用项目现有认证方案（如无，默认 JWT）
- 包含：邮箱/密码登录表单、表单验证、错误提示、加载状态、响应式布局

工作流：
1. /plan 先规划组件结构和认证流程，参考现有页面的模式
2. /tdd 测试先行：编写登录表单的单元测试和认证流程的集成测试
3. 实现登录页面和认证逻辑
4. /code-review 审查实现
5. /verify 验证所有测试通过且页面正常渲染

安全要求：
- 密码不明文传输
- 防止暴力破解（rate limiting）
- XSS 防护
- CSRF token

验收标准：
- 所有测试通过，覆盖率 80%+
- 页面在移动端和桌面端正常渲染
- 登录成功跳转到 dashboard，失败显示错误信息

不要做：
- 不要实现注册页面
- 不要实现忘记密码功能
- 不要修改现有的路由结构
```

### 例2: 中程度の英語プロンプト

**ユーザー入力:**
```
Add a REST API endpoint for user profile updates with validation
```

**フェーズ0が検出:** `go.mod` に Go 1.22、Chiルーター

**最適化プロンプト（完全版）:**
```
Add a REST API endpoint for user profile updates (PATCH /api/users/:id).

Tech stack: Go 1.22 + Chi router (detected from project)

Requirements:
- PATCH /api/users/:id — partial update of user profile
- Input validation for fields: name, email, avatar_url, bio
- Auth: require valid token, users can only update own profile
- Return 200 with updated user on success
- Return 400 with validation errors on invalid input
- Return 401/403 for auth failures
- Follow existing API patterns in the codebase

Workflow:
1. /plan the endpoint structure, middleware chain, and validation logic
2. /tdd — write table-driven tests for success, validation failure, auth failure, not-found
3. Implement following existing handler patterns
4. /go-review
5. /verify — run full test suite, confirm no regressions

Do not:
- Modify existing endpoints
- Change the database schema (use existing user table)
- Add new dependencies without checking existing ones first (use search-first skill)
```

### 例3: 超大規模プロジェクト

**ユーザー入力:**
```
Migrate our monolith to microservices
```

**最適化プロンプト（完全版）:**
```
Use the blueprint skill to plan: "Migrate monolith to microservices architecture"

Before executing, answer these questions in the blueprint:
1. Which domain boundaries exist in the current monolith?
2. Which service should be extracted first (lowest coupling)?
3. Communication pattern: REST APIs, gRPC, or event-driven (Kafka/RabbitMQ)?
4. Database strategy: shared DB initially or database-per-service from start?
5. Deployment target: Kubernetes, Docker Compose, or serverless?

The blueprint should produce phases like:
- Phase 1: Identify service boundaries and create domain map
- Phase 2: Set up infrastructure (API gateway, service mesh, CI/CD per service)
- Phase 3: Extract first service (strangler fig pattern)
- Phase 4: Verify with integration tests, then extract next service
- Phase N: Decommission monolith

Each phase = 1 PR, with /verify gates between phases.
Use /save-session between phases. Use /resume-session to continue.
Use git worktrees for parallel service extraction when dependencies allow.

Recommended: Opus 4.6 for blueprint planning, Sonnet 4.6 for phase execution.
```

---

## 関連コンポーネント

| コンポーネント | 参照するタイミング |
|-----------|------------------|
| `configure-ecc` | ユーザーがまだECCをセットアップしていない場合 |
| `skill-stocktake` | インストール済みコンポーネントの監査（ハードコードされたカタログの代わりに使用） |
| `search-first` | 最適化プロンプトの調査フェーズ |
| `blueprint` | 超大規模スコープの最適化プロンプト（コマンドではなくスキルとして呼び出す） |
| `strategic-compact` | 長いセッションのコンテキスト管理 |
| `cost-aware-llm-pipeline` | トークン最適化の推奨 |
