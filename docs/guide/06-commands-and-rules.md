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
