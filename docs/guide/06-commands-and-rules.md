# 第06章 コマンドとルール

## この章で学ぶこと

- コマンドのフロントマター仕様（`description`・`argument-hint`・`allowed-tools`）
- ECC コマンドカタログの分類と各コマンドの役割
- `commands/` がレガシー互換シムである位置づけと、スキルへの移行指針
- `legacy-command-shims/` の役割
- `validate-commands.js` による自動検証（フロントマター・クロスリファレンス）
- `rules/` の構成（`common/` + 言語別）と「常時ガードレール」の仕組み
- `paths:` glob によるファイルマッチングと言語別ルールの適用
- `validate-rules.js` による検証

---

## 第一部: コマンド

## Claude Code におけるコマンドとは（初心者向け）

Claude Code の **スラッシュコマンド** は、チャット欄に `/コマンド名` と入力するだけでワークフローを起動できる、人間工学的な操作インターフェースです。Claude Code が `/` を検出するとコマンドファイルを探し、その内容をプロンプトとして解釈・実行します。

### コマンドはどこに置くのか

Claude Code はプロジェクトの `.claude/commands/` ディレクトリ内の Markdown ファイルをスラッシュコマンドとして認識します。ファイル名（拡張子なし）がそのままコマンド名になります。

```
.claude/
└── commands/
    ├── plan.md        # → /plan
    ├── tdd.md         # → /tdd
    └── code-review.md # → /code-review
```

ECC（Everything Claude Code）はプラグインとして多数のコマンドを提供しており、インストール後は `/plan` や `/tdd` などをすぐに使えます。

### 引数を渡す（argument-hint）

コマンドは引数を受け取れます。フロントマターの `argument-hint` フィールドに書式例を記述することで、Claude Code の UI にヒントが表示されます。

```yaml
---
argument-hint: "[feature description | path/to/*.prd.md]"
---
```

たとえば `/plan ユーザー認証機能を追加したい` と入力すると、後ろの文字列がコマンド本文のプロンプトに渡されます。

### コマンドはエージェント・スキルへの入口になれる

コマンド自体は薄いエントリポイントです。本体ロジックは **スキル**（ドメイン知識）や **エージェント**（専門サブエージェント）に委譲できます。たとえば `/code-review` はコマンドとして起動されますが、内部で `code-reviewer` エージェントに作業を渡します。この分離により、スキルは別のコマンドや会話からも再利用できます。

> コンポーネント全体の関係については [03-components-overview.md](03-components-overview.md) を参照してください。

---

## 6.1 コマンドとは何か

ECC における **コマンド（command）** は、ユーザーが `/コマンド名` の形式で呼び出すスラッシュエントリです。`commands/` ディレクトリ内の Markdown ファイルが 1 つのコマンドに対応します。

**重要な設計方針**: `AGENTS.md` に明記されているとおり、`commands/` は現在 **レガシー互換シム（スラッシュエントリ互換面）** として位置づけられています。

> `commands/` is a legacy slash-entry compatibility surface and should only be added or updated when a shim is still required for migration or cross-harness parity.

新しいワークフローは `skills/` に実装するのが基本です（詳細は [04-skills.md](04-skills.md) を参照）。コマンドはスキルへの「入口」として機能するか、まだスキルへ移行できていないレガシーワークフローを維持するために使用されます。

---

## 6.2 フロントマター仕様

コマンドファイルの先頭に YAML フロントマターを置きます。フロントマターは任意ですが、`description` を書くことが強く推奨されます。

```yaml
---
description: Restate requirements, assess risks, and create step-by-step implementation plan. WAIT for user CONFIRM before touching any code.
argument-hint: "[feature description | path/to/*.prd.md]"
allowed-tools: ["Read", "Bash"]
---
```

| フィールド | 必須 | 説明 |
|------------|------|------|
| `description` | 推奨 | コマンドの目的と動作の説明。Claude Code の UI や自動選択で表示される |
| `argument-hint` | 任意 | コマンドが受け取る引数の書式ヒント（例: `[pr-number \| pr-url \| blank for local review]`） |
| `allowed-tools` | 任意 | このコマンドが使用できるツールのリスト。明示的に制限したい場合に指定 |

`validate-commands.js` は次の内容を検証します（フロントマターが存在する場合）。

1. `---` 区切りブロックが正しく閉じられていること
2. フロントマターの各行が `key: value` 形式であること
3. `[` で始まる値が `]` で閉じられていること（未閉じ YAML 配列の検出）
4. 本文内のコマンド参照（`` `/build-fix` `` 等）が実在するコマンドを指していること
5. `agents/xxx.md` のパス参照が実在するエージェントを指していること

---

## 6.3 コマンドカタログの分類

ECCには 79 のコマンドが含まれます（`agent.yaml` のコマンドリストより）。機能別に分類します。

### コアワークフロー

| コマンド | 役割 | 委譲先 |
|----------|------|--------|
| `/plan` | 実装計画の作成（コード変更前に確認を待つ） | `planner` エージェント（任意。インラインでも動作） |
| `/tdd` | TDD ワークフロー（テスト先行→実装→リファクタ） | `tdd-guide` エージェント |
| `/code-review` | ローカル変更または GitHub PR のコードレビュー | `code-reviewer` エージェント |
| `/build-fix` | ビルドエラー検出・修正（言語別ビルドリゾルバへ自動委譲） | 言語別 `*-build-resolver` |
| `/quality-gate` | 品質ゲートパイプライン（フォーマット・lint・型チェック） | — |
| `/feature-dev` | 機能開発ワークフロー全体の実行 | 複数エージェント |

### テスト

| コマンド | 役割 |
|----------|------|
| `/tdd` | 汎用 TDD（全言語対応） |
| `/react-test` | React テスト（TDD ワークフロー） |
| `/go-test` | Go テスト（テーブル駆動、`go test -cover`） |
| `/kotlin-test` | Kotlin テスト（Kotest + Kover） |
| `/rust-test` | Rust テスト（`cargo test`、統合テスト） |
| `/cpp-test` | C++ テスト（GoogleTest + gcov/lcov） |
| `/flutter-test` | Flutter テスト |
| `/test-coverage` | カバレッジレポートとギャップ特定 |

### コードレビュー

| コマンド | 対象言語 |
|----------|----------|
| `/code-review` | 汎用（全言語） |
| `/python-review` | Python（PEP 8・型ヒント・セキュリティ） |
| `/go-review` | Go（イディオム・並行安全・エラー処理） |
| `/kotlin-review` | Kotlin（null 安全・コルーチン・クリーンアーキテクチャ） |
| `/rust-review` | Rust（所有権・ライフタイム・unsafe） |
| `/cpp-review` | C++（メモリ安全・モダンイディオム・並行） |
| `/react-review` | React |
| `/fastapi-review` | FastAPI |
| `/flutter-review` | Flutter |

### ビルド修正

| コマンド | 対象 |
|----------|------|
| `/build-fix` | 言語自動検出（汎用） |
| `/go-build` | Go ビルドエラー・`go vet` 警告 |
| `/kotlin-build` | Kotlin/Gradle コンパイルエラー |
| `/rust-build` | Rust ビルド・借用チェッカー |
| `/cpp-build` | C++ CMake・リンカ問題 |
| `/gradle-build` | Gradle（Android/KMP） |
| `/react-build` | React ビルドエラー |
| `/flutter-build` | Flutter ビルドエラー |

### 計画・アーキテクチャ

| コマンド | 役割 |
|----------|------|
| `/plan` | 実装計画（リスク評価・フェーズ分解） |
| `/plan-prd` | PRD（Product Requirements Document）の生成 |
| `/multi-plan` | マルチモデル協調計画 |
| `/multi-workflow` | マルチモデル協調開発 |
| `/multi-backend` | バックエンド重点のマルチモデル開発 |
| `/multi-frontend` | フロントエンド重点のマルチモデル開発 |
| `/multi-execute` | マルチモデル協調実行 |

### セッション管理

| コマンド | 役割 |
|----------|------|
| `/save-session` | セッション状態を `~/.claude/session-data/` に保存 |
| `/resume-session` | 最新のセッションを復元 |
| `/sessions` | セッション履歴の閲覧・検索・管理 |
| `/checkpoint` | セッションにチェックポイントを設定 |
| `/aside` | コンテキストを失わずにサイドクエスチョンに回答 |

### 学習・進化

| コマンド | 役割 |
|----------|------|
| `/learn` | セッションから再利用可能なパターンを抽出 |
| `/learn-eval` | パターン抽出＋品質自己評価 |
| `/evolve` | 習得したインスティンクトの分析と改善提案 |
| `/promote` | プロジェクトスコープのインスティンクトをグローバルへ昇格 |
| `/instinct-status` | インスティンクト一覧と信頼スコアの表示 |
| `/skill-create` | git 履歴からスキルを自動生成 |
| `/skill-health` | スキルポートフォリオのヘルスダッシュボード |

### リファクタリング・クリーンアップ

| コマンド | 役割 |
|----------|------|
| `/refactor-clean` | デッドコード削除・重複統合・構造クリーンアップ |

### ドキュメント

| コマンド | 役割 |
|----------|------|
| `/update-docs` | プロジェクトドキュメントの更新 |
| `/update-codemaps` | コードマップの再生成 |

### ループ・自動化

| コマンド | 役割 |
|----------|------|
| `/loop-start` | 定期実行ループの起動 |
| `/loop-status` | 実行中ループの状態確認 |

### プロジェクト・インフラ

| コマンド | 役割 |
|----------|------|
| `/harness-audit` | ハーネス設定の信頼性・コスト監査 |
| `/model-route` | タスクを最適モデル（Haiku/Sonnet/Opus）へルーティング |
| `/setup-pm` | パッケージマネージャ設定（npm/pnpm/yarn/bun） |
| `/security-scan` | セキュリティスキャンの実行 |
| `/review-pr` | PR レビュー |
| `/pr` | プルリクエスト作成 |

---

## 6.4 レガシー互換シムと移行指針

`legacy-command-shims/` ディレクトリには、デフォルトのプラグインコマンド面からは除外されたスラッシュコマンドが保管されています。

```
legacy-command-shims/
└── commands/
    ├── agent-sort.md    # 旧 /agent-sort
    ├── claw.md          # 旧 /claw
    ├── context-budget.md # 旧 /context-budget
    └── devfleet.md      # 旧 /devfleet
```

> These slash-entry shims are no longer loaded by the default plugin command surface. They remain here for users who still need short-term migration compatibility with old muscle-memory commands.

これらのシムを使い続ける必要がある場合は、個別の Markdown ファイルをプロジェクトレベルまたはユーザーレベルの Claude コマンドディレクトリにコピーして使用します。アーカイブ全体を有効化しないでください。

**スキルへの移行**: 新しいワークフローは `skills/` に配置し、コマンドはスキルへの薄いラッパーまたは入口として設計します。`/plan` コマンドはその好例で、本体ロジックをインラインで実行しつつ、`planner` エージェント（エージェントファイルがある場合）へのオプション委譲を記述しています。

---

## 第二部: ルール

## Claude Code におけるルール／メモリとは（初心者向け）

Claude Code には、セッションを通じて Claude が「常に参照すべき規約・ガードレール」を与える仕組みが複数あります。

### CLAUDE.md — プロジェクトの常時インストラクション

プロジェクトルートに置かれた `CLAUDE.md` は、セッション開始時に Claude Code が**自動的に読み込む**設定ファイルです。開発者はここにコーディング規約、テスト方針、禁止事項、ディレクトリ構成の説明などを書くことで、Claude がすべての会話でそれらを前提として動作するようになります。

```
myproject/
├── CLAUDE.md          ← Claude が起動時に必ず読む
├── src/
└── ...
```

`CLAUDE.md` はユーザーへの説明ではなく、**Claude への常時指示**です。「ここに書けばすべての会話で有効になる」と理解してください。

### ルールファイル — glob でスコープを絞る

`CLAUDE.md` が「全体に効く常時指示」であるのに対し、**ルールファイル**は特定のファイルパターンにマッチしたときだけ追加で適用される指示です。Claude Code はルールファイルのフロントマターに書かれた `paths:` glob を見て、編集中のファイルがパターンに合致すれば自動的にそのルールを読み込みます。

```yaml
---
paths:
  - "**/*.ts"
  - "**/*.tsx"
---
# TypeScript/JavaScript Coding Style
...（TypeScript 固有の規約）...
```

これにより、「Python ファイルを編集しているときは Python ルールを、TypeScript ファイルには TypeScript ルールを」と自動で切り替わります。ユーザーが手動でルールを呼び出す必要はありません。

### ECC のルール構成

ECC はこの仕組みを活用して、`rules/common/`（言語非依存の普遍原則）と `rules/<言語名>/`（言語別の固有規約）に分けてルールを管理しています。詳細は本章の後半（6.6〜6.9）で解説します。

> コンポーネント全体の関係については [03-components-overview.md](03-components-overview.md) を参照してください。

---

## 6.5 ルールとは何か

**ルール（rule）** は、Claude Code セッション中に**常時適用**される規範ファイルです。スキル（呼び出されたときに動作）やコマンド（ユーザーが起動したときに動作）と異なり、ルールは自動的に読み込まれ、Claude の判断基準として機能します。

ルールは「**何をすべきか**」を定め、スキルは「**どうやるか**」を提供します（`rules/README.md` より）。

---

## 6.6 ルールの構成

`rules/` は 2 層構造を採用しています。

```
rules/
├── common/          # 言語非依存の普遍原則（常にインストール）
│   ├── agents.md           # エージェントオーケストレーション規則
│   ├── code-review.md      # コードレビューの基準
│   ├── coding-style.md     # コーディングスタイル（不変性・DRY・KISS）
│   ├── development-workflow.md  # 開発ワークフロー（Research→Plan→TDD→Review→Commit）
│   ├── git-workflow.md     # コミットフォーマット・PR プロセス
│   ├── hooks.md            # フックシステムの使い方
│   ├── patterns.md         # 共通デザインパターン（リポジトリパターン等）
│   ├── performance.md      # モデル選択・コンテキスト管理・パフォーマンス
│   └── security.md         # セキュリティ必須チェックと対応プロトコル
│   └── testing.md          # テスト要件（TDD・80%+カバレッジ）
├── typescript/      # TypeScript/JavaScript 固有
├── python/          # Python 固有
├── golang/          # Go 固有
├── angular/         # Angular 固有
├── react/           # React 固有
├── cpp/             # C/C++ 固有
├── csharp/          # C# 固有
├── dart/            # Dart 固有
├── fsharp/          # F# 固有
├── java/            # Java 固有
├── kotlin/          # Kotlin 固有
├── perl/            # Perl 固有
├── php/             # PHP 固有
├── ruby/            # Ruby/Rails 固有
├── rust/            # Rust 固有
├── swift/           # Swift 固有
├── web/             # Web/フロントエンド 固有
├── arkts/           # HarmonyOS/ArkTS 固有
└── zh/              # 中国語版 common ルール
```

### common/ ルールの主なトピック

| ファイル | 主なガイドライン |
|----------|-----------------|
| `coding-style.md` | 不変性（CRITICAL）、KISS、DRY、YAGNI、ファイル 200-400 行、命名規則 |
| `security.md` | コミット前チェックリスト（ハードコード秘密・SQL インジェクション・XSS 等）、秘密情報管理、セキュリティ対応プロトコル |
| `testing.md` | 80% カバレッジ必須、TDD ワークフロー（RED→GREEN→REFACTOR）、テスト種別（Unit/Integration/E2E） |
| `development-workflow.md` | Research & Reuse（GitHub 検索→ライブラリ doc→Exa）、Plan→TDD→Review→Commit の順序 |
| `git-workflow.md` | conventional commits（`feat`/`fix`/`refactor`/`docs`/`test`/`chore`/`perf`/`ci`）、PR プロセス |
| `patterns.md` | リポジトリパターン、API レスポンスエンベロープ、スケルトンプロジェクト戦略 |
| `performance.md` | モデル選択ポリシー（Haiku/Sonnet/Opus）、コンテキストウィンドウ管理、拡張思考制御 |
| `hooks.md` | フックタイプ（PreToolUse/PostToolUse/Stop）、TodoWrite のベストプラクティス |
| `agents.md` | エージェント一覧と起動タイミング、並列実行ガイドライン |

---

## 6.7 `paths:` glob による自動適用

言語別ルールファイルの先頭には `paths:` フロントマターがあります。これにより、特定のファイルパターンにマッチしたときだけそのルールが適用されます。

```yaml
---
paths:
  - "**/*.ts"
  - "**/*.tsx"
  - "**/*.js"
  - "**/*.jsx"
---
# TypeScript/JavaScript Coding Style
...
```

Python ルールの例：

```yaml
---
paths:
  - "**/*.py"
  - "**/*.pyi"
---
# Python Coding Style
...
```

`paths:` に列挙したパターンに合致するファイルを編集・参照するコンテキストでは、その言語ルールが**自動的に適用**されます。ユーザーが明示的に呼び出す必要はありません。

---

## 6.8 言語別ルールと common の継承関係

各言語別ルールは `common/` を **拡張**します。ファイル冒頭に継承元を明示するのが規約です。

```markdown
> This file extends [common/coding-style.md](../common/coding-style.md) with TypeScript/JavaScript specific content.
```

継承と優先順位のルール：

1. `common/` は普遍的なデフォルトを定義する
2. 言語別ルールは言語イディオムで `common/` を上書きできる
3. **言語固有ルールが common ルールより優先する**（CSS の詳細度と同じ考え方）

**例**: `common/coding-style.md` は不変性をデフォルト原則として定義しています。しかし Go では構造体の ポインタレシーバによるミューテーションが慣用的なため、`golang/coding-style.md` はその点を上書きします。

各言語ディレクトリが持つファイルの種類：

| ファイル | 内容 |
|----------|------|
| `coding-style.md` | フォーマットツール・イディオム・エラー処理パターン |
| `testing.md` | テストフレームワーク・カバレッジツール・テスト構成 |
| `patterns.md` | 言語固有のデザインパターン |
| `hooks.md` | フォーマッタ・リンタ・型チェッカーの PostToolUse フック |
| `security.md` | 秘密情報管理・セキュリティスキャンツール |

---

## 6.9 ルールを使った「常時ガードレール」の設計

ルールは **セッション全体を通じて有効なガードレール** として機能します。具体的な設計パターンを示します。

### セキュリティチェックリストの常時化

`rules/common/security.md` が定義するチェックリストは、コードを書くたびに Claude が参照すべき基準になります。

```markdown
## 必須セキュリティチェック（任意のコミット前）
- [ ] ハードコードされた秘密情報がないこと（API キー・パスワード・トークン）
- [ ] ユーザー入力がすべてバリデートされていること
- [ ] SQL インジェクション防止（パラメータ化クエリ）
- [ ] XSS 防止（HTML サニタイズ）
```

セキュリティルールが常時ロードされているため、Claude は「後でチェックしよう」ではなく「書くたびに確認する」行動を取ります。

### コーディングスタイルの自動適用

`typescript/coding-style.md` が `.ts`/`.tsx` ファイルに自動適用されるため、TypeScript ファイルを編集するセッションでは常に「`any` を使わない」「不変性パターンを使う」「`Zod` でバリデートする」などのガイドラインが有効です。

### 開発ワークフローの強制

`common/development-workflow.md` は開発手順を定義します。実装を始める前に Research → Plan → TDD の順序を守ることが、ルールとして常時適用されます。

---

## 6.10 `validate-commands.js` による検証

`scripts/ci/validate-commands.js` は以下を検証します。

1. **空ファイルでないこと**: コマンドファイルの内容が空でないこと
2. **フロントマターの構文**: 閉じられていない YAML 配列・マッピングの検出
3. **コマンドクロスリファレンス**: コード本文（コードブロック外）で言及されている `` `/コマンド名` `` が `commands/` に実在すること
4. **エージェントパス参照**: `agents/xxx.md` 形式で参照されるエージェントが実在すること
5. **スキルディレクトリ参照**: `skills/xxx/` 形式の参照について警告（エラーではない）を出力

```bash
node scripts/ci/validate-commands.js
# 出力例: Validated 79 command files (2 warnings)
```

フロントマターが存在しないコマンドファイルも許容されます（validate はフロントマターブロックがある場合のみ検証）。

---

## 6.11 `validate-rules.js` による検証

`scripts/ci/validate-rules.js` は `rules/` 配下を再帰的に走査し、以下を確認します。

1. Markdown ファイルが読み取り可能であること
2. ファイルの内容が空でないこと

現在のバリデーターはシンプルですが、**空のルールファイルはエラー**として扱われます。ルールファイルに何も書かないのではなく、`paths:` フロントマターと内容の両方を必ず記述してください。

```bash
node scripts/ci/validate-rules.js
# 出力例: Validated 97 rule files
```

---

## 6.12 新しい言語ルールの追加手順

`rules/README.md` に定義された手順です。

1. `rules/<言語名>/` ディレクトリを作成する
2. 以下のファイルを作成し、それぞれ先頭に `common/` の継承元を明示する:
   - `coding-style.md` — フォーマットツール・イディオム・エラー処理
   - `testing.md` — テストフレームワーク・カバレッジツール
   - `patterns.md` — 言語固有のデザインパターン
   - `hooks.md` — フォーマッタ・リンタ・型チェッカーの PostToolUse フック
   - `security.md` — 秘密情報管理・セキュリティスキャンツール
3. 各ファイルの冒頭に継承宣言を記述する:
   ```markdown
   > This file extends [common/xxx.md](../common/xxx.md) with <言語名> specific content.
   ```
4. 適用対象のファイルパターンがあれば `paths:` フロントマターを追加する
5. 対応するスキルが `skills/` にあれば参照リンクを追加する

詳細な作成手順は [12-authoring-components.md](12-authoring-components.md) を参照してください。

---

## コマンドとルールの設計思想・ベストプラクティス

### コマンドは「人間工学的な入口（ergonomic entry）」である

Claude Code においてコマンドの本質的な役割は、**ユーザーが覚えやすい短い名前でワークフローを起動できる入口**を提供することです。コマンドファイル自体に複雑なロジックを持たせるべきではなく、永続的・再利用可能なロジックはスキルへ、専門的な実行はエージェントへ委譲するのがベストプラクティスです。

ECC がコマンドを「レガシー互換シム」と位置づけているのはこの思想を反映しています。つまり、コマンドはスラッシュエントリの互換面として存在し、新しいワークフローは `skills/` に実装するのが基本です。コマンドはその薄いラッパーに留まります。

```
/plan → commands/plan.md（薄いエントリ）
              └─ インラインで実行 or planner エージェントへ委譲
              └─ tdd-workflow スキルを参照
```

この設計の利点は明確です。スキルはコマンド経由でも、会話から直接参照でも、別のエージェントからも呼び出せます。コマンドとスキルを分離することでコンポーネントの再利用性が高まります。

**コマンドを書くときのベストプラクティス**:

- `description:` は必ず記述する（Claude Code UI での自動表示とドキュメント生成に使われる）
- `argument-hint:` で引数の書式例を示す（ユーザーの入力ミスを減らす）
- コマンド本文にロジックを詰め込まない。スキルへのリンクやエージェントへの委譲を明示する
- コマンド間の参照は実在するコマンド名を使う（`validate-commands.js` が自動検証する）
- 新しいワークフローはまずスキルとして実装し、コマンドは最後に薄いエントリとして追加する

### ルールは「決定論的ガードレール」として設計する

ルール（`CLAUDE.md` やルールファイル）の強みは、スキルやプロンプトと違い、**Claude がそれを「忘れる」ことがない**点にあります。セッション開始時に読み込まれ、セッション全体を通じて Claude の判断基準として機能します。これが「ガードレール」という表現の意味です。

ただし、ルールが強力であるからこそ、**スコープを絞って最小限に保つ**ことが重要です。

**ルールを書くときのベストプラクティス**:

| 原則 | 理由 |
|------|------|
| `paths:` glob はできるだけ具体的に（`**/*.ts` など） | 無関係なコンテキストにルールが効くとノイズになる |
| `common/` を拡張する形で言語別ルールを書く | 共通原則の重複を避け、差分だけを記述できる |
| ルールには「常に守るべき最小限」だけを書く | 過剰なルールは Claude の判断を縛りすぎ、柔軟性を損なう |
| セキュリティ・テスト要件・コーディングスタイルはルールに向く | これらはすべての作業で一貫して守るべき制約だから |
| 特定のタスク手順・ワークフローはスキルに向く | ワークフローは状況により異なるため、ルールでなくスキルで管理する |

**ECC がこれをどう実現しているか**:

ECC の `rules/common/` には普遍的な原則（セキュリティチェックリスト・TDD 要件・コミット形式）が集約されています。言語別ルールはそれを `paths:` glob で絞り込んだうえで拡張し、言語固有の事情（Go のポインタレシーバ、Rust の所有権モデルなど）を上書きします。このレイヤー構造により、プロジェクトに新しい言語を追加しても `common/` の資産を引き継げます。

```
rules/
├── common/security.md    ← 全言語共通のセキュリティチェックリスト（常時）
├── typescript/           ← paths: ["**/*.ts", "**/*.tsx"] でスコープ限定
│   └── coding-style.md  ← common/ を拡張し TypeScript 固有事項を追加
└── golang/
    └── coding-style.md  ← common/ を拡張し Go 固有事項（ポインタ等）を上書き
```

**Claude Code が `CLAUDE.md` とルールに込めた意図**:

Claude Code がプロジェクトルートの `CLAUDE.md` を起動時に自動で読み込む設計は、「プロジェクトの文脈を Claude に渡す標準的な方法」を一箇所に集約するためです。散在したドキュメントを Claude が探し回る必要がなく、開発者も「ここに書けば必ず伝わる」と確信できます。ECC はこの仕組みをプラグインとして拡張し、言語固有のルールを `paths:` glob で自動適用することで、手動でコンテキストを渡す手間を排除しています。

---

## 関連章

- [03-components-overview.md](03-components-overview.md) — コマンド・ルールとスキル・エージェントの関係
- [04-skills.md](04-skills.md) — スキルの構造と設計（コマンドの移行先）
- [05-agents.md](05-agents.md) — エージェントへの委譲パターン
- [07-hooks-and-runtime.md](07-hooks-and-runtime.md) — ルールと連携するフックの設計
- [12-authoring-components.md](12-authoring-components.md) — コマンド・ルールの作成手順

## 参照ソース

- `commands/plan.md` — `argument-hint`、PRD モード、コマンドからエージェントへのオプション委譲の実装例
- `commands/code-review.md` — ローカルレビュー／PR レビューのモード切り替え、フェーズ分解の実装例
- `commands/quality-gate.md` — シンプルなコマンドフロントマターの例
- `COMMANDS-QUICK-REF.md` — コマンドカタログの公式クイックリファレンス
- `agent.yaml` — コマンドリスト（79 件）の機械可読な定義
- `legacy-command-shims/README.md` — レガシーシムの位置づけと使用方針
- `rules/README.md` — ルール構成・インストール方法・言語別追加手順
- `rules/common/coding-style.md` — 不変性・命名規則・品質チェックリスト
- `rules/common/security.md` — セキュリティ必須チェックリストとプロトコル
- `rules/common/testing.md` — 80% カバレッジ要件、TDD ワークフロー
- `rules/common/development-workflow.md` — Research→Plan→TDD→Review→Commit の開発フロー
- `rules/common/performance.md` — モデル選択ポリシー
- `rules/typescript/coding-style.md` — `paths:` フロントマターと `common/` 継承の実装例
- `rules/python/coding-style.md` — Python 向け `paths:` フロントマターの例
- `scripts/ci/validate-commands.js` — クロスリファレンス検証ロジック
- `scripts/ci/validate-rules.js` — ルールファイル空チェックロジック
