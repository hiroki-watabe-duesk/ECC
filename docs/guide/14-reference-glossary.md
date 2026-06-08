# 第14章 リファレンス・用語集・早見表

## この章で学ぶこと

- リポジトリのディレクトリ構成と「どこに何があるか」
- 環境変数一覧（ECC・Claude Code 関連）
- npm スクリプト早見表
- 主要コマンド早見（カテゴリ別）
- **Claude Code 基本用語・基本操作 早見**（プラットフォーム標準の用語定義と操作手順）
- ECC 固有の用語集

---

## 14.1 ディレクトリ早見表

### リポジトリ直下

| パス | 用途 |
|------|------|
| `CLAUDE.md` | Claude Code 向けプロジェクト指示（ECC の中核ガイダンス） |
| `AGENTS.md` | 全エージェントへの共通指示（Codex 等も参照） |
| `RULES.md` | ECC 開発者向けの核心的規約サマリ |
| `COMMANDS-QUICK-REF.md` | スラッシュコマンドの逆引き一覧 |
| `CONTRIBUTING.md` | コントリビューションガイド |
| `package.json` | npm スクリプト定義・依存関係 |
| `hooks/hooks.json` | フック登録ファイル（Claude Code が読み込む） |
| `.mcp.json` | MCP サーバー設定（github, context7, exa, memory, playwright 等） |
| `agent.yaml` | Claude Code Marketplace 向けプラグイン定義 |
| `VERSION` | 現在のバージョン文字列 |

### コアコンポーネント

| パス | 用途 |
|------|------|
| `agents/` | エージェント定義（`.md` ファイル、YAML フロントマター付き） |
| `skills/` | curated スキルディレクトリ（各 `<name>/SKILL.md`） |
| `commands/` | スラッシュコマンド定義（`.md` ファイル） |
| `rules/` | 常時適用ルール（`common/`, `python/`, `typescript/` 等） |
| `rules/common/` | 言語横断の共通ルール（security, testing, git-workflow 等） |

### スクリプト類

| パス | 用途 |
|------|------|
| `scripts/hooks/` | フック実装スクリプト（Node.js CommonJS） |
| `scripts/lib/` | 共有ユーティリティ（hook-flags, package-manager, utils 等） |
| `scripts/ci/` | CI バリデーション（validate-agents, validate-skills 等） |
| `scripts/install-apply.js` | ECC インストール実行スクリプト（`ecc-install` CLI） |
| `scripts/install-plan.js` | インストール計画の生成 |
| `scripts/harness-audit.js` | ハーネス設定の品質監査 |
| `scripts/ecc.js` | `ecc` CLI エントリポイント |
| `scripts/catalog.js` | スキルカタログ管理 |
| `scripts/session-inspect.js` | セッションデータの検査 |

### 設定・スキーマ

| パス | 用途 |
|------|------|
| `schemas/` | JSON スキーマ（hooks, provenance, install-profiles 等） |
| `schemas/hooks.schema.json` | `hooks/hooks.json` のスキーマ |
| `schemas/provenance.schema.json` | learned/imported スキルのプロベナンス定義 |
| `manifests/` | インストールマニフェスト（profiles, modules, components） |
| `manifests/install-profiles.json` | プロファイル定義（minimal, core, developer, full 等） |
| `mcp-configs/` | MCP サーバー設定テンプレート |
| `contexts/` | コンテキストファイル（dev.md, research.md, review.md） |

### ドキュメント

| パス | 用途 |
|------|------|
| `docs/guide/` | 本ガイド（章別 Markdown） |
| `docs/architecture/` | アーキテクチャ詳細設計（cross-harness, observability 等） |
| `docs/SKILL-DEVELOPMENT-GUIDE.md` | スキル作成の詳細ガイド |
| `docs/SKILL-PLACEMENT-POLICY.md` | スキル配置・プロベナンス規約 |
| `docs/ECC-2.0-REFERENCE-ARCHITECTURE.md` | ECC 2.0 の参照アーキテクチャ設計書 |
| `docs/COMMAND-REGISTRY.json` | コマンドレジストリ（自動生成） |
| `docs/ja-JP/` | 日本語翻訳ドキュメント |

### テスト

| パス | 用途 |
|------|------|
| `tests/` | テストルート |
| `tests/run-all.js` | 全テスト実行エントリポイント |
| `tests/lib/` | `scripts/lib/` の単体テスト |
| `tests/hooks/` | フックの統合テスト |

### クロスハーネスアダプタ

| パス | 用途 |
|------|------|
| `.claude-plugin/` | Claude Code Marketplace プラグインメタデータ |
| `.codex/` | Codex 向け設定（AGENTS.md, config.toml, エージェント定義） |
| `.cursor/` | Cursor 向けフック・スキル |
| `.gemini/` | Gemini 向け設定（GEMINI.md） |
| `.opencode/` | OpenCode 向けプラグイン |
| `.codebuddy/` | CodeBuddy 向けインストールスクリプト |

---

## 14.2 環境変数一覧

### フック制御

| 変数名 | デフォルト値 | 説明 |
|--------|------------|------|
| `ECC_HOOK_PROFILE` | `standard` | フックプロファイル。`minimal`, `standard`, `strict` |
| `ECC_DISABLED_HOOKS` | （空） | 無効化するフック ID のカンマ区切りリスト |
| `ECC_HOOK_INPUT_MAX_BYTES` | — | フックへの stdin 最大バイト数 |

### セッション管理

| 変数名 | デフォルト値 | 説明 |
|--------|------------|------|
| `ECC_SESSION_START_CONTEXT` | `enabled` | `0`, `false`, `off`, `none`, `disabled` で無効化 |
| `ECC_SESSION_START_MAX_CHARS` | `8000` | セッション開始時に注入するコンテキストの最大文字数。`0` で無効 |
| `ECC_SESSION_RETENTION_DAYS` | `30` | セッションファイルの保持日数 |
| `ECC_SESSION_ID` | — | 現在のセッション ID（内部使用） |
| `ECC_SESSION_RECORDING_DIR` | — | セッション記録ディレクトリの上書き |
| `CLAUDE_SESSION_ID` | — | Claude Code が提供するセッション識別子 |

### ランタイム・パス

| 変数名 | デフォルト値 | 説明 |
|--------|------------|------|
| `CLAUDE_PLUGIN_ROOT` | 自動検出 | ECC プラグインのルートパス（`scripts/lib/utils.js` が参照） |
| `CLAUDE_PACKAGE_MANAGER` | 自動検出 | パッケージマネージャー強制指定（`npm`, `pnpm`, `yarn`, `bun`） |
| `CLAUDE_CONFIG_DIR` | `~/.claude` | Claude Code 設定ディレクトリ |
| `CLAUDE_PROJECT_DIR` | — | 現在のプロジェクトディレクトリ |
| `CLAUDE_RULES_DIR` | — | ルールディレクトリの上書き |
| `CLAUDE_TRANSCRIPT_PATH` | — | 現在のトランスクリプトファイルパス |
| `ECC_PLUGIN_ROOT` | — | `CLAUDE_PLUGIN_ROOT` の別名（旧形式） |
| `ECC_WORKFLOWS_DIR` | — | ワークフロー定義ディレクトリの上書き |

### セキュリティ・ガバナンス

| 変数名 | デフォルト値 | 説明 |
|--------|------------|------|
| `ECC_GOVERNANCE_CAPTURE` | `0` | `1` で governance-capture フックを有効化 |
| `ECC_GATEGUARD` | — | GateGuard ポリシーの設定 |
| `ECC_ENABLE_INSAITS` | — | INSAITS セキュリティモニタリングを有効化 |
| `ECC_UNICODE_SCAN_ROOT` | — | Unicode 安全性スキャンのルートパス |
| `ECC_DISABLED_MCPS` | — | 無効化する MCP サーバー名のカンマ区切りリスト |

### MCP・品質

| 変数名 | デフォルト値 | 説明 |
|--------|------------|------|
| `ECC_MCP_CONFIG_PATH` | — | MCP 設定ファイルパスの上書き |
| `ECC_MCP_HEALTH_FAIL_OPEN` | — | MCP ヘルスチェック失敗時の動作（フェイルオープン） |
| `ECC_MCP_HEALTH_STATE_PATH` | — | MCP ヘルス状態ファイルのパス |
| `ECC_MCP_RECONNECT_COMMAND` | — | MCP 再接続コマンド |
| `ECC_QUALITY_GATE_STRICT` | — | 品質ゲートを厳格モードで実行 |
| `ECC_QUALITY_GATE_FIX` | — | 品質ゲートで自動修正を有効化 |
| `ECC_OBSERVE_RUNNER_TIMEOUT_MS` | — | observe-runner タイムアウト（ミリ秒） |

---

## 14.3 npm スクリプト早見表

```bash
npm run <script>
```

### テスト・検証

| スクリプト | 説明 |
|-----------|------|
| `test` | 全検証を一括実行（validate-* + catalog:check + command-registry:check + tests/run-all.js） |
| `coverage` | c8 でカバレッジ計測（80% ライン閾値） |
| `lint` | ESLint + markdownlint-cli を実行 |

### カタログ・レジストリ

| スクリプト | 説明 |
|-----------|------|
| `catalog:check` | カタログの整合性チェック（書き込みなし） |
| `catalog:sync` | カタログを最新状態に同期（`--write` モード） |
| `command-registry:generate` | コマンドレジストリを標準出力に出力 |
| `command-registry:write` | `docs/COMMAND-REGISTRY.json` に書き込み |
| `command-registry:check` | レジストリが最新かチェック |

### ハーネス監査

| スクリプト | 説明 |
|-----------|------|
| `harness:adapters` | クロスハーネスアダプタのコンプライアンスチェック |
| `harness:audit` | ハーネス設定の品質監査 |
| `observability:ready` | オブザーバビリティ準備状況チェック |
| `operator:dashboard` | オペレーター準備ダッシュボードを表示 |
| `platform:audit` | プラットフォーム互換性監査 |

### セキュリティ

| スクリプト | 説明 |
|-----------|------|
| `security:ioc-scan` | サプライチェーン IOC スキャン |
| `security:advisory-sources` | セキュリティアドバイザリソースの確認 |

### リリース・ビルド

| スクリプト | 説明 |
|-----------|------|
| `preview-pack:smoke` | パッケージプレビューのスモークテスト |
| `release:approval-gate` | リリース承認ゲートチェック |
| `release:video-suite` | リリースビデオスイートの実行 |
| `build:opencode` | OpenCode プラグインのビルド（`prepack` で自動実行） |

### オーケストレーション

| スクリプト | 説明 |
|-----------|------|
| `claw` | NanoClaw v2 REPL を起動 |
| `orchestrate:status` | オーケストレーション状態を表示 |
| `orchestrate:worker` | Codex ワーカーシェルを起動 |
| `orchestrate:tmux` | tmux/worktree オーケストレーターを起動 |
| `discussion:audit` | GitHub Discussions の監査 |

---

## 14.4 主要コマンド早見

コマンドの完全一覧は [COMMANDS-QUICK-REF.md](../../COMMANDS-QUICK-REF.md) を参照してください。

### コアワークフロー

| コマンド | 用途 |
|---------|------|
| `/plan` | 実装計画立案（確認待ち→コード変更） |
| `/tdd` | TDD ワークフロー（インターフェース設計→テスト→実装→検証） |
| `/code-review` | コード品質・セキュリティ・保守性レビュー |
| `/build-fix` | ビルドエラーの検出と修正（言語検出→専門エージェント委譲） |
| `/verify` | ビルド→lint→テスト→型チェックの連続実行 |

### セッション管理

| コマンド | 用途 |
|---------|------|
| `/save-session` | セッション状態を `~/.claude/session-data/` に保存 |
| `/resume-session` | 直近のセッションを読み込んで再開 |
| `/sessions` | セッション履歴の閲覧・検索・エイリアス管理 |
| `/checkpoint` | セッション内にチェックポイントをマーク |
| `/aside` | 現在のタスクを中断せずに副質問に答える |
| `/context-budget` | コンテキストウィンドウ使用量の分析 |

### 学習・改善

| コマンド | 用途 |
|---------|------|
| `/learn` | セッションから再利用可能なパターンを抽出 |
| `/learn-eval` | パターン抽出＋品質評価を実施してから保存 |
| `/evolve` | 蓄積したインスタイトからスキル構造を提案・昇格 |
| `/promote` | プロジェクトスコープのインスタイトをグローバルへ昇格 |
| `/instinct-status` | 全インスタイト（プロジェクト＋グローバル）を信頼度付きで表示 |
| `/skill-create` | ローカル git 履歴を分析して再利用可能スキルを自動生成 |
| `/rules-distill` | スキルから横断的な原則を抽出してルールに蒸留 |

### ドキュメント・調査

| コマンド | 用途 |
|---------|------|
| `/docs` | Context7 経由でライブラリ・API ドキュメントを参照 |
| `/update-docs` | プロジェクトドキュメントを更新 |

### ループ・自動化

| コマンド | 用途 |
|---------|------|
| `/loop-start` | 定期実行エージェントループを開始 |
| `/loop-status` | 実行中ループの状態確認 |
| `/claw` | NanoClaw v2 永続 REPL を起動（モデルルーティング・スキルホットロード付き） |

---

## 14.5 Claude Code 基本用語・基本操作 早見

> Claude Code プラットフォームとしての標準的な意味を簡潔に示します。
> ECC 固有の意味や詳細は後続の [14.6 用語集](#146-用語集) を参照してください。

### 基本用語

| 用語 | 定義（Claude Code プラットフォームとしての意味） |
|------|----------------------------------------------|
| **ハーネス** | Claude Code が動作する実行環境全体の構成。エージェントループ・コンポーネント・設定の組み合わせを指す。ECC はハーネスを構成するプラグイン。 |
| **エージェントループ** | Claude Code が「思考 → ツール実行 → 結果確認 → 次の思考」を繰り返す実行サイクル。ユーザーのリクエストから最終回答までの一連の処理。 |
| **サブエージェント (Task)** | `Task` ツールで起動される独立した Claude インスタンス。独自のコンテキストウィンドウを持ち、専門的なサブタスクを並行処理できる。`agents/<name>.md` で定義する。 |
| **スラッシュコマンド** | ユーザーが `/command-name` と入力することで呼び出すワークフロー定義。`commands/<name>.md`（または `.claude/commands/<name>.md`）に配置する。 |
| **フック** | ライフサイクルイベント（PreToolUse / PostToolUse / SessionStart / Stop / PreCompact）で自動実行されるスクリプト。`settings.json` または `hooks/hooks.json` で登録する。ツール実行とは独立した決定論的な実行が保証される。 |
| **スキル** | タスクに関連すると判断されたとき自動的にロードされる知識モジュール。`skills/<name>/SKILL.md`（または `.claude/skills/<name>/SKILL.md`）に配置する。progressive disclosure（段階的開示）でコンテキストを節約する仕組み。 |
| **ルール・メモリ (CLAUDE.md)** | セッション中、常に有効なガイドライン。`CLAUDE.md`（プロジェクト固有）と `rules/<lang>/<topic>.md`（汎用）の 2 層構造。Claude が**必ず読む**最重要ファイル。 |
| **MCP (Model Context Protocol)** | エージェントが外部ツール・サービスと通信するためのプロトコル。`.mcp.json` で設定する。GitHub・Exa・Memory・Playwright などを統合できる。 |
| **permission mode** | Claude Code がツールを実行する際の許可モード。`settings.json` の `permissions.allow` / `permissions.deny` で制御する。`allow` に列挙されたツールは確認プロンプトなしで実行される。 |
| **compaction** | コンテキストウィンドウが一定量を超えたとき、古い会話履歴を要約・圧縮して継続する仕組み。`PreCompact` フックで圧縮前にスナップショットを取得できる。 |
| **コンテキストウィンドウ** | Claude が一度に処理できるトークン数の上限。会話履歴・ロードされたファイル・スキル・ルールがすべて含まれる。`/context-budget` コマンドで使用量を確認できる。 |
| **プラグイン** | `agent.yaml` で定義される Claude Code Marketplace 向けの配布単位。ECC 自体が Claude Code プラグインとして設計されている。 |
| **`~/.claude/`** | ユーザーグローバルの Claude Code 設定ディレクトリ。すべてのプロジェクトに適用されるスキル・設定・セッションデータを格納する。 |
| **`.claude/`** | プロジェクトローカルの Claude Code 設定ディレクトリ。プロジェクト固有のエージェント・コマンド・スキルを格納する。`~/.claude/` より高い優先度で適用される。 |

### 基本操作 早見

| 操作 | 方法 | 説明 |
|------|------|------|
| 起動 | `claude` または IDE の Claude Code 拡張を開く | プロジェクトディレクトリで実行するとプロジェクト設定が自動ロードされる |
| CLAUDE.md 生成 | `/init` | リポジトリをスキャンして `CLAUDE.md` を自動生成する。初回導入時に必ず実行する |
| ヘルプ表示 | `/help` | 利用可能なスラッシュコマンドの一覧と説明を表示する |
| コマンド実行 | `/command-name [引数]` | 例: `/plan 新機能を追加する`、`/code-review` |
| コンテキスト確認 | `/context-budget` | コンテキストウィンドウの使用量（トークン数）を確認する |
| セッション保存 | `/save-session` | 現在のセッション状態を `~/.claude/session-data/` に保存する |
| セッション再開 | `/resume-session` | 直近のセッションを読み込んで作業を継続する |
| 権限の考え方 | `settings.json` の `permissions.allow` / `permissions.deny` を編集する | 安全のため、最初は必要なコマンドだけ `allow` に追加し、リスクの高い操作は `deny` でブロックする |

---

## 14.6 用語集

### ハーネス (Harness)

AI コーディングアシスタントが動作する実行環境の全体構成。Claude Code, Codex, OpenCode, Cursor, Gemini などが「ハーネス」に相当する。ECC はハーネスに依存しない共有レイヤーとして設計されている。

### スキル (Skill)

`skills/<name>/SKILL.md` に配置されるコンテキストベースの知識モジュール。ユーザーのタスクに関連すると判断された場合に自動的にロードされる受動的な知識。エージェントへの明示的な委譲は不要。

### エージェント (Agent)

`agents/<name>.md` に配置される専門サブエージェント。YAML フロントマターで `name`, `description`, `tools`, `model` を定義。Claude が `Task` ツール経由で明示的に委譲する。

### コマンド (Command)

`commands/<name>.md` に配置されるスラッシュコマンド。ユーザーが `/command-name` と入力することで呼び出す。`description:` フロントマターが必須。

### ルール (Rule)

`rules/<lang>/<topic>.md` に配置される常時適用ガイドライン。アクティブなコンテキストに関係なく常に有効。`rules/common/` は言語横断で適用される。

### フック (Hook)

ツール実行前後・セッション開始終了などのイベントで自動実行されるスクリプト。`hooks/hooks.json` に matcher/type/timeout で登録し、実装は `scripts/hooks/` に置く。

### MCP (Model Context Protocol)

エージェントが外部ツール・サービスと通信するためのプロトコル。`.mcp.json` で設定する。ECC 標準では github, context7, exa, memory, playwright, sequential-thinking が含まれる。

### プロファイル (Profile)

ECC インストール時の構成セット。`minimal`（フックなし）, `core`（基本フック含む）, `developer`（標準）, `security`, `full`（全モジュール）などが定義される（`manifests/install-profiles.json`）。

### プロベナンス (Provenance)

learned/imported スキルの出所・由来情報。`source`, `created_at`, `confidence`, `author` を含む `.provenance.json` ファイルで管理（`schemas/provenance.schema.json` 準拠）。

### インスタイト (Instinct)

`/learn-eval` や observe-runner によって抽出されたパターン・知見。YAML フロントマター（`id`, `confidence`）＋ `## Action` セクションで構成される。`~/.claude/homunculus/instincts/` に保存される。

### ホムンクルス (Homunculus)

`~/.claude/homunculus/` ディレクトリに配置される Claude Code の自己モデル領域。instincts（パーソナル・インヘリテッド）と evolved skills が格納される。

### ワークツリー (Worktree)

git worktree による複数ブランチの並行作業領域。ECC 2.0 ではワークツリーのライフサイクル（作成・一時停止・マージ・クローズ）を一級のセッション概念として扱う。

### pass@k / pass^k

エージェントの能力評価指標。`pass@k` は k 回試行のうち少なくとも 1 回成功する確率、`pass^k` は k 回すべて成功する確率。ハーネス最適化や自己改善ループの評価に使用する。

### Hermes

Anthropic が開発する ECC のオペレーターシェル。ECC のスキル・MCP 規約を消費し、チャット・CLI・cron・ハンドオフワークフローをルーティングする。プライベートな Hermes 状態（OAuth トークン・個人設定）は ECC パブリックリポジトリに含めない。

### AgentShield

ECC のエンタープライズセキュリティプラットフォーム。プロンプトインジェクション検出・SARIF 出力・ポリシースキーマ・サプライチェーン intelligence を提供する（`docs/architecture/agentshield-enterprise-research-roadmap.md`）。

### ECC Tools

GitHub ネイティブの PR チェック・課金・深度分析・Linear 同期レイヤー。`scripts/ci/` の validate スクリプトはローカル CI 相当のチェックを提供する。

### curated スキル

`skills/` リポジトリに収録され、インストールマニフェストで参照される公式スキル。`validate-skills.js` でバリデーションされ、パッケージに同梱される。

### learned スキル

`~/.claude/skills/learned/` に保存されるスキル。`/learn-eval` や `evaluate-session.js` フックが自動生成する。リポジトリには含まれず、プロベナンスファイルが必須。

### imported スキル

`~/.claude/skills/imported/` に保存されるスキル。外部ソース（URL・ファイルコピー）から手動でインストールしたもの。プロベナンスファイルが必須。

### evolved スキル

`~/.claude/homunculus/evolved/skills/` に保存されるスキル。`/evolve` コマンドがインスタイトをクラスタリングして生成する。プロベナンスはインスタイトソースから継承。

### アダプタ (adapter-backed)

ハーネス固有の実装を持つコンポーネント。例: Claude Code はフックを native 実行するが、Codex は instruction-driven（指示ベース）で同等の動作を再現する。

### instruction-backed

ネイティブフック実行をサポートしないハーネス（例: Codex）で、同等の動作を指示（テキスト）で再現するアダプタ方式。

### native

ハーネスが機能を直接サポートしている状態。例: Claude Code は `PreToolUse` フックを native サポートする。

### reference-only

ドキュメント・スキルとして存在するが、実行可能なフックやスクリプトは持たないコンポーネント。指示のみで動作する。

### Two-Instance Kickoff

複雑なタスクを「足場づくり」と「深掘りリサーチ」の 2 つの Claude セッションに分割して並行作業する起動パターン。

### Groundwork（地ならし）

セッション開始時に Claude に現状把握・未完了タスク確認・次アクション提案をさせる起動パターン。

---

## 関連章

- [03 コンポーネント概要](./03-components-overview.md)
- [07 フックとランタイム](./07-hooks-and-runtime.md)
- [12 コンポーネントの作成](./12-authoring-components.md)

## 参照ソース

- `CLAUDE.md`
- `AGENTS.md`
- `COMMANDS-QUICK-REF.md`
- `package.json`（scripts セクション）
- `.mcp.json`
- `scripts/lib/hook-flags.js`（環境変数: ECC_HOOK_PROFILE, ECC_DISABLED_HOOKS）
- `scripts/hooks/session-start.js`（環境変数: ECC_SESSION_START_*, CLAUDE_SESSION_ID）
- `scripts/lib/package-manager.js`（CLAUDE_PACKAGE_MANAGER）
- `docs/SKILL-PLACEMENT-POLICY.md`（curated/learned/imported/evolved 定義）
- `docs/ECC-2.0-REFERENCE-ARCHITECTURE.md`（Hermes, AgentShield, ECC Tools）
- `docs/architecture/cross-harness.md`（adapter-backed/instruction-backed/native）
- `manifests/install-profiles.json`（プロファイル定義）
- `schemas/provenance.schema.json`（プロベナンス定義）
