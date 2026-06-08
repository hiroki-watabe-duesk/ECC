# 第02章 リポジトリ全体構造と層アーキテクチャ

ECC のディレクトリ地図を把握し、ECC 2.0 の層モデルを理解する章です。「どのファイルがどの役割を持ち、新しいワークフローをどの層に置くべきか」を判断できるようになります。

---

## トップレベルのディレクトリ地図

以下は実際の `ls` で確認したリポジトリのトップレベル構造です。括弧内の数値はファイル・ディレクトリ数（2026 年 6 月時点）です。

### 共有ソース層（ハーネスに依存しない）

| ディレクトリ | 件数 | 役割 |
|------------|------|------|
| `skills/` | 249 スキル | 再利用可能なワークフロー定義（最も重要な共有単位） |
| `agents/` | 63 エージェント | 専門化されたサブエージェント定義 |
| `commands/` | 79 コマンド | スラッシュコマンド互換シム（移行期） |
| `rules/` | 24 ディレクトリ | 常時適用のガイドライン（common + 言語別） |
| `hooks/` | `hooks.json` + `memory-persistence/` | ライフサイクルイベント定義 |
| `scripts/` | 多数 | Node.js ユーティリティ・フックスクリプト・CI ツール |
| `mcp-configs/` | `mcp-servers.json` | MCP サーバー設定テンプレート |
| `contexts/` | `dev.md`、`research.md`、`review.md` | 動的インジェクション用コンテキストテンプレート |
| `schemas/` | 9 スキーマ | JSON Schema 定義（インストール・フック・プラグイン等） |
| `manifests/` | 3 マニフェスト | インストールプロファイル・モジュール・コンポーネント定義 |

### ハーネスアダプタ層（各ハーネス固有）

| ディレクトリ | 対応ハーネス | 内容 |
|------------|-----------|------|
| `.claude/` | Claude Code | ルール・設定・フックスクリプト |
| `.claude-plugin/` | Claude Code プラグイン | プラグインアセット |
| `.codex/` | OpenAI Codex | `AGENTS.md`・エージェント・`config.toml` |
| `.codex-plugin/` | Codex プラグイン | プラグインメタデータ |
| `.cursor/` | Cursor | フック・ルール・スキル翻訳面 |
| `.opencode/` | OpenCode | コマンド・インストラクション・`index.ts` |
| `.gemini/` | Google Gemini | `GEMINI.md` |
| `.kiro/` | Kiro | エージェント・フック・インストール |
| `.agents/` | 汎用エージェント参照 | ハーネス横断エージェントミラー |
| `.zed/` | Zed | Zed 固有設定 |
| `.qwen/` | Qwen | Qwen 固有設定 |
| `.trae/` | Trae | Trae 固有設定 |
| `.codebuddy/` | CodeBuddy | CodeBuddy 固有設定 |

### インフラ・プラットフォーム層

| ディレクトリ/ファイル | 役割 |
|---------------------|------|
| `ecc2/` | Rust 製コントロールプレーン（alpha。`dashboard`・`sessions`・`daemon` 等のコマンドを提供） |
| `src/` | TypeScript/JavaScript ソース |
| `tests/` | テストスイート（`scripts/` 構造をミラー） |
| `integrations/` | 外部インテグレーション |
| `plugins/` | プラグイン定義 |
| `docs/` | アーキテクチャ・ガイド・ロケール別ドキュメント |
| `assets/` | 画像・静的ファイル |
| `examples/` | サンプルコード・テンプレート |
| `research/` | 調査・実験ドキュメント |
| `legacy-command-shims/` | 旧コマンドの互換シム |

### ルールの構造

`rules/` は次の 2 層で構成されています。

```
rules/
  common/          # 言語非依存の共通ルール（coding-style, security, testing 等）
  typescript/      # TypeScript 固有ルール
  python/          # Python 固有ルール
  golang/          # Go 固有ルール
  java/            # Java 固有ルール
  rust/            # Rust 固有ルール
  kotlin/          # Kotlin/Android 固有ルール
  react/           # React 固有ルール
  web/             # Web フロントエンド固有ルール
  cpp/             # C/C++ 固有ルール
  csharp/          # C# 固有ルール
  dart/            # Dart/Flutter 固有ルール
  swift/           # Swift 固有ルール
  php/             # PHP 固有ルール
  perl/            # Perl 固有ルール
  ruby/            # Ruby 固有ルール
  angular/         # Angular 固有ルール
  fsharp/          # F# 固有ルール
  arkts/           # ArkTS 固有ルール
  zh/              # 中国語向けルール
```

---

## 「1 ソース → 多ハーネス」の概念図

ECC の設計を一言で表すと「共有ソース層とアダプタ層の分離」です。

```
┌─────────────────────────────────────────────────────────┐
│                   共有ソース層                           │
│                                                         │
│  skills/      rules/      hooks/      mcp-configs/      │
│  agents/      schemas/    manifests/  scripts/lib/      │
│                                                         │
│  ← ここにワークフローの「何を・どう・なぜ」を書く →     │
└──────────────────────┬──────────────────────────────────┘
                       │ インストール・パッケージング
          ┌────────────┼────────────┐
          ▼            ▼            ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  Claude Code │ │    Codex     │ │  OpenCode    │
│  アダプタ    │ │  アダプタ    │ │  アダプタ    │
│  .claude/    │ │  .codex/     │ │  .opencode/  │
│  .claude-    │ │  .codex-     │ │  （plugin/   │
│   plugin/    │ │   plugin/    │ │   events）   │
└──────────────┘ └──────────────┘ └──────────────┘
          ▼            ▼            ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│   Cursor     │ │   Gemini     │ │  Zed 等      │
│  アダプタ    │ │  アダプタ    │ │  アダプタ    │
│  .cursor/    │ │  .gemini/    │ │  .zed/ 等    │
└──────────────┘ └──────────────┘ └──────────────┘
```

各アダプタが担うのは「ロード方法・イベント形状・コマンド名のマッピング・プラットフォーム制限への対応」だけです。ワークフローのロジックはアダプタ側に置きません。

`docs/architecture/cross-harness.md` が定義するポータビリティモデルでは、`SKILL.md` が「最も可搬性の高い単位」とされています。スキルは YAML フロントマター（`name`、`description`、`origin`）を持ち、ハーネス固有の前提を含まない形式で書かれているため、複数のハーネスがほぼそのまま読み込めます。

---

## ECC 2.0 リファレンスアーキテクチャの層モデル

`docs/ECC-2.0-REFERENCE-ARCHITECTURE.md` は ECC 2.0 を「ハーネスオペレーティングシステム」と定義し、次の 5 層で構成します。

```
┌──────────────────────────────────────────────────────────────┐
│ Operator Surface（オペレータ面）                             │
│ CLI、プラグイン、TUI、HUD/ステータスライン、                  │
│ リリースゲート、PR チェック                                  │
├──────────────────────────────────────────────────────────────┤
│ Harness Adapter Layer（ハーネスアダプタ層）                  │
│ Claude Code、Codex、OpenCode、Cursor、Gemini、Zed、           │
│ dmux、Orca、Superset、Ghast、ターミナル専用                  │
├──────────────────────────────────────────────────────────────┤
│ Worktree, Session, and Queue Runtime（ワークツリー・セッション・キュー） │
│ ワークツリー、ペイン、セッション、todo、チェック、           │
│ マージ/コンフリクトキュー、通知状態、オーナーシップ、         │
│ ハンドオフエクスポート                                       │
├──────────────────────────────────────────────────────────────┤
│ Observability and Evaluation Loop（可観測性と評価ループ）    │
│ JSONL トレース、状態スナップショット、リスク台帳、           │
│ ハーネス監査、シナリオスペック、ベリファイア、               │
│ 昇格済みプレイブック、RAG セット                             │
├──────────────────────────────────────────────────────────────┤
│ Security and Commercial Platform（セキュリティと商用プラットフォーム） │
│ AgentShield ポリシー/SARIF、ECC Tools チェック、             │
│ 課金、Linear/GitHub 同期、エンタープライズレポート           │
└──────────────────────────────────────────────────────────────┘
```

### 各層の役割

**Operator Surface**: エンジニアとシステムが直接触れる面です。`ecc status`・`ecc sessions` などの CLI コマンド、TUI（`ecc2/` の Rust コントロールプレーン）、HUD/ステータスライン、GitHub Actions チェックがここに属します。

**Harness Adapter Layer**: 各 AI ハーネスへの接続を薄いアダプタとして管理します。Claude Code はプラグインアセットとネイティブフック実行、Codex は `AGENTS.md` とプラグインメタデータ、OpenCode はプラグイン/イベントシステム、Cursor は翻訳されたルール・フック・スキルを使います。アダプタは薄く保ち、共有動作は共有ソース層に置くことが設計原則です。

**Worktree, Session, and Queue Runtime**: 複数のエージェントインスタンスが並列動作する際の状態管理を担います。ワークツリーのライフサイクル（作成・一時停止・再開・マージ）、セッションのグループ化（リポジトリ・ブランチ・タスク・オーナー別）、コンフリクトキュー、通知状態を扱います。`ecc2/` の alpha 実装はこの層に対応します。

**Observability and Evaluation Loop**: ハーネスの動作を観測し、自己改善するループです。JSONL トレース・リスク台帳・シナリオスペック・ベリファイアによる昇格済みプレイブックが含まれます。`docs/architecture/evaluator-rag-prototype.md` はこの層の設計を定義しています。

**Security and Commercial Platform**: AgentShield による静的解析・SARIF 出力、ECC Tools GitHub App による PR チェック・課金・Linear 同期がここに属します。

---

## ポータビリティマトリクス（概要）

以下は各コンポーネントが各ハーネスでどう実現されるかの概要です。詳細は [10-cross-harness-and-install.md](10-cross-harness-and-install.md) を参照してください。

| コンポーネント | 共有ソース | Claude Code | Codex | OpenCode | Cursor |
|--------------|-----------|------------|-------|---------|--------|
| スキル | `skills/*/SKILL.md` | プラグイン | プラグイン + `.agents/skills` | プラグイン/config | スキルコピー |
| ルール | `rules/`、`AGENTS.md` | ルールインストール | `AGENTS.md` | インストラクション | Cursor ルール |
| フック | `hooks/hooks.json`、`scripts/hooks/` | ネイティブフック | インストラクション駆動 | プラグインイベント | フックアダプタ |
| MCP | `.mcp.json`、`mcp-configs/` | ネイティブ MCP 設定 | MCP 参照設定 | ネイティブ | サポート範囲に依存 |
| コマンド | `commands/` | スラッシュコマンド | 互換シム | CLI エントリポイント | コマンドセマンティクスが異なる |
| セッション | `ecc2/`、セッションアダプタ | TUI/デーモン | tmux/ワークツリー | ハーネス固有ランナー | alpha |

---

## 新しいワークフローをどの層に置くか

新しいワークフローを追加するときの判断フローです。

```
新しいワークフローを追加したい
         │
         ▼
 複数のハーネスで使いたいか？
         │
     Yes │                No
         │                │
         ▼                ▼
  skills/ に書く     1 ハーネス固有なら
  （共有ソース層）   ハーネスアダプタに書く
         │          ただしその境界を
         │          ドキュメントに明記する
         ▼
  次に確認: フックが必要か？
         │
     Yes │                No
         │                │
         ▼                ▼
  hooks/hooks.json に  スキルだけで完結
  マッチャーを追加      （フック不要）
  scripts/hooks/ に
  実装を書く
         │
         ▼
  各ハーネスアダプタで
  ロード・イベント形状のみ設定
  （ロジックをコピーしない）
```

**判断の基準**:
- ワークフローが「何をするか」「どんな順序でするか」「どんな基準で判断するか」 → `skills/` に置く
- ワークフローを「どのツールが読み込むか」「どのイベントで発火するか」 → ハーネスアダプタに置く
- 常時適用のガイドライン（命名規則・禁止事項・必須チェック） → `rules/` に置く
- ライフサイクルイベント駆動の自動化（フォーマット・検証・リマインダー） → `hooks/` に置く
- 特定の外部サービスへの接続設定 → `mcp-configs/` に置く

---

## 各コンポーネントの位置づけ（概観）

ここでは概観のみ示します。詳細は [03-components-overview.md](03-components-overview.md) を参照してください。

- **`skills/`**: 再利用可能なワークフロー定義。`SKILL.md` 形式。最も重要な共有単位。249 スキルが存在し、言語・フレームワーク・ドメイン別に整理されています。
- **`agents/`**: 専門化されたサブエージェント。YAML フロントマターを持つ Markdown ファイル。63 エージェントが存在し、レビュー・ビルド解決・計画・TDD ガイドなどの役割を担います。
- **`commands/`**: スラッシュコマンド互換シム。79 コマンドが存在しますが、ロジックは `skills/` 側に移行中です。
- **`rules/`**: 常時適用ガイドライン。`common/` + 言語別の 2 層構造。プロジェクトに必要な言語パックだけをインストールできます。
- **`hooks/`**: ライフサイクルイベントの定義と実装。`hooks.json` がマッチャーを定義し、`scripts/hooks/` が実装を持ちます。
- **`scripts/`**: Node.js ユーティリティ。フックスクリプト・インストーラ・CI ツール・ライブラリ関数が含まれます。
- **`mcp-configs/`**: MCP サーバー設定テンプレート。`mcp-servers.json` に主要サービスの設定例が含まれます。
- **`manifests/`**: インストールプロファイル・モジュール・コンポーネント定義。選択的インストールの基盤です。
- **`schemas/`**: JSON Schema 定義。インストール設定・フック・プラグイン・状態ストアのスキーマを管理します。
- **`ecc2/`**: Rust 製コントロールプレーン（alpha）。`dashboard`・`start`・`sessions`・`status`・`stop`・`resume`・`daemon` コマンドを提供します。

---

## 関連章

- [01-philosophy.md](01-philosophy.md) — この構造の背後にある設計思想
- [03-components-overview.md](03-components-overview.md) — 各コンポーネントの詳細な役割と相互関係
- [10-cross-harness-and-install.md](10-cross-harness-and-install.md) — ポータビリティマトリクスの詳細とインストール手順

## 参照ソース

- `CLAUDE.md` — プロジェクト概要とディレクトリ構造
- `README.md` — What's Inside・Background・Key Concepts 節
- `AGENTS.md` — エージェント一覧と Version 情報（v2.0.0-rc.1 時点で 63 エージェント・249 スキル・79 コマンド）
- `docs/ECC-2.0-REFERENCE-ARCHITECTURE.md` — ECC 2.0 層モデルの一次ソース
- `docs/architecture/cross-harness.md` — ポータビリティモデルとアダプタ設計原則
