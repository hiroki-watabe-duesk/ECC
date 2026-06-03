# 第10章 クロスハーネス移植性とインストーラ

## この章で学ぶこと

- 「1ソース→多ハーネス」モデルの設計思想と実装構造
- 各ハーネスのコンプライアンス状態（Native/Adapter-backed/Instruction-backed/Reference-only）
- ポータビリティ・マトリクス（Skill/Rule/Hook/MCP/Commandが各ハーネスでどう実現されるか）
- フックのイベント名マッピング（Claude → OpenCode → Cursor → Kiro）
- アダプタ変換の実例（Cursor: `.mdc`への平坦化、MCPマージ）
- インストーラの構造（`install.sh` / `install.ps1` → `scripts/install-apply.js`）
- インストールモード・対応ターゲット・プロファイル
- 「3コピー編集が必要なら設計が間違い」原則

---

## 設計思想：1ソース→多ハーネス

ECCの根本原則は `docs/architecture/cross-harness.md` に明文化されています：

> **「変更に3ハーネスのコピー編集が必要なら、共有ソースが間違った場所にある」**

ワークフローの永続的な部分は `skills/`、`rules/`、`hooks/`、`scripts/`、`mcp-configs/` に置き、ハーネス固有のファイルは以下のみを担います：

- 共有アセットのロード方法
- イベント形状の適応
- コマンド名のマッピング
- プラットフォーム制限への対応

### 共有ソースとハーネスアダプタの分離

```
共有ソース（全ハーネス共通）
├── skills/*/SKILL.md       ← 最もポータブルな単位
├── rules/                  ← コーディング規約・セキュリティ方針
├── hooks/hooks.json        ← フック定義
├── scripts/hooks/          ← フック実装スクリプト
├── .mcp.json               ← MCPサーバ参照設定
├── AGENTS.md               ← クロスツールエージェント仕様
└── agents/                 ← エージェント定義

ハーネス別アダプタ（エッジのみ）
├── .claude-plugin/         ← Claude Code プラグインマニフェスト
├── .codex/                 ← Codex CLI 設定（TOML）
├── .codex-plugin/          ← Codex プラグインメタデータ
├── .cursor/                ← Cursor ルール・MCP設定
├── .gemini/                ← Gemini CLI 設定
├── .opencode/              ← OpenCode 設定・プラグイン
├── .qwen/                  ← Qwen 設定
├── .zed/                   ← Zed エディタ設定
├── .kiro/                  ← Kiro IDE 設定
├── .codebuddy/             ← CodeBuddy 設定
├── .agents/                ← 汎用エージェントディレクトリ（antigravity等）
└── .trae/ .vscode/ 等      ← その他ハーネス
```

---

## ハーネス別コンプライアンス状態

`docs/architecture/harness-adapter-compliance.md` が定義する4段階：

| 状態 | 意味 |
|---|---|
| **Native** | ECCがそのハーネス向けに直接インストール・検証できる |
| **Adapter-backed** | 薄いアダプタ・プラグインがあるが、ハーネスによってパリティが異なる |
| **Instruction-backed** | ガイダンスとファイルは提供できるが、フック/セッションの実行面でハーネスのサポートが不完全 |
| **Reference-only** | 設計参考にはなるが、ECCのインストーラ・アダプタは未提供 |

**ハーネス・スコアカード：**

| ハーネス | 状態 | 主な対応アセット | 未対応・差異 |
|---|---|---|---|
| Claude Code | **Native** | プラグイン; skills; commands; hooks; MCP; rules | Claude固有フックは他ハーネスで同等ではない |
| Codex | **Instruction-backed** | `AGENTS.md`; Codexプラグインメタ; skills; MCP参照設定 | ネイティブフック実行なし; スラッシュコマンド非同等 |
| OpenCode | **Adapter-backed** | パッケージ/プラグインメタ; shared skills; MCP; イベントアダプタ | イベント名・コマンドディスパッチがClaude Codeと異なる |
| Cursor | **Adapter-backed** | Cursorルール; project-local skills; フックアダプタ; shared scripts | フックイベントとルールロードがClaude Codeと異なる |
| Gemini | **Instruction-backed** | project-local指示; shared skills; rules; 互換ドキュメント | フックパリティなし; ドリフト文書化が必要 |
| Zed | **Adapter-backed** | プロジェクト設定; 平坦化ルール; shared skills; commands; agents | ZedエージェントはClaude Codeフックではない |
| Kiro | **Adapter-backed** | agents（JSON+MD）; skills; steering files; IDE/CLIフック | ステアリングファイル形式がrules/*.mdと異なる |
| Terminal-only | **Native** | skills; rules; commands; scripts; 監査スクリプト | UI・セッション自動制御なし |
| dmux | **Adapter-backed** | セッションスナップショット; tmux/worktreeオーケストレーション | インストールターゲットではない（オーケストレーションランタイム） |
| Orca / Superset / Ghast | **Reference-only** | 設計参考のみ | ECCインストーラ・アダプタ未提供 |

---

## ポータビリティ・マトリクス

各コンポーネントが6つのハーネスでどのように実現されるか：

| コンポーネント | Claude Code | Codex | OpenCode | Cursor | Gemini | Kiro |
|---|---|---|---|---|---|---|
| **Skill** | プラグイン経由でロード | `.agents/skills/` | `instructions` 配列 | `.cursor/` にコピー | `instructions` 配列 | `.kiro/skills/*/SKILL.md` |
| **Rule** | `rules/` → Claude設定 | `AGENTS.md` に統合 | `instructions` 配列 | `.cursor/rules/*.mdc` | プロジェクト指示 | `.kiro/steering/*.md` |
| **Hook** | `hooks/hooks.json` ネイティブ実行 | 指示テキストとして扱う | プラグインイベント | フックアダプタ | 非対応 | `.kiro/hooks/*.kiro.hook` |
| **MCP** | `.mcp.json` 自動検出 | `.codex/config.toml` TOML | `opencode.json` | `.cursor/mcp.json` マージ | `.gemini/` 設定 | `.kiro/settings/mcp.json` |
| **Command** | `commands/*.md` | CLIスクリプト/shims | `.opencode/commands/*.md` | Cursor互換shims | プロジェクト設定 | `.kiro/skills/` （スキルとして） |
| **Agent** | `agents/*.md` フロントマター | `.codex/agents/*.toml` | `opencode.json` の `agent:` | `.cursor/agents/` | Gemini指示 | `.kiro/agents/*.json + *.md` |
| **Session** | `ecc2/` セッションアダプタ | マルチエージェント TOML | OpenCodeセッション | — | — | — |

---

## フックのイベント対応表

`.opencode/MIGRATION.md` のフックマッピング表：

| Claude Code フック | OpenCode プラグインイベント | Kiro フックトリガ | 備考 |
|---|---|---|---|
| `PreToolUse` | `tool.execute.before` | `preToolUse` | ツール入力を変更可能 |
| `PostToolUse` | `tool.execute.after` | `postToolUse` | ツール出力を変更可能 |
| `Stop` | `session.idle` / `session.status` | `agentStop` | セッションライフサイクル |
| `SessionStart` | `session.created` | — | セッション開始時 |
| — | `file.edited` | `fileEdited` | OpenCode/Kiro固有：ファイル変更 |
| — | `lsp.client.diagnostics` | — | OpenCode固有：LSP統合 |
| — | `tui.toast.show` | — | OpenCode固有：通知 |

**Codexのフック対応：** Codex CLIにはネイティブフックが実装されていません。`.codex/AGENTS.md` にあるように、フックは「指示テキスト（instruction-backed）」として機能します。エージェントがフック相当の動作を指示として実行しますが、ランタイムレベルの強制はありません。

---

## アダプタ変換の実例：Cursor

`scripts/lib/install-targets/cursor-project.js` が実装するCursorアダプタの変換ロジック：

### ルールの平坦化（`.md` → `.mdc`）

```javascript
function toCursorRuleFileName(fileName, sourceRelativeFile) {
  if (path.basename(sourceRelativeFile).toLowerCase() === 'readme.md') {
    return null;  // README.md はスキップ
  }
  return fileName.endsWith('.md')
    ? `${fileName.slice(0, -3)}.mdc`  // .md → .mdc に変換
    : fileName;
}
```

`rules/` 配下の全Markdownファイルは `.cursor/rules/` に `.mdc` 拡張子で平坦化されます。ネストされたディレクトリ構造はフラットになります。

### エージェントの変換

`agents/` 配下のファイルは `.cursor/agents/` に変換されます。`toCursorAgentFileName` 関数で名前変換を行います。

### MCPのマージ

`.mcp.json` の内容を `.cursor/mcp.json` に `merge-json` 戦略でマージします：

```javascript
const cursorMcpOperation = createJsonMergeOperation({
  moduleId: module.id,
  repoRoot,
  sourceRelativePath: '.mcp.json',
  destinationPath: path.join(targetRoot, 'mcp.json'),
});
```

### AGENTS.md の扱い

```javascript
if (sourceRelativePath === 'AGENTS.md') {
  // Cursor はネストされた AGENTS.md をディレクトリコンテキストとして扱う。
  // ECC のルート AGENTS.md をホストプロジェクトの .cursor/ に入れない。
  return [];
}
```

---

## AGENTS.md と .codex/AGENTS.md

### ルートの `AGENTS.md`（クロスツール仕様）

リポジトリルートの `AGENTS.md` は、OpenAI Codex / Claude Code / その他エージェント対応ツールが共通して読むエージェント仕様ファイルです。63の専門エージェント、オーケストレーション方針、セキュリティガイドライン、コーディング規約を定義しています。

これは**クロスツール契約**です。Claude Code・Codex・その他ハーネスのいずれも、このファイルを参照してエージェントの役割と動作方針を把握します。

### `.codex/AGENTS.md`（Codex補足仕様）

ルートの `AGENTS.md` を Codex 固有の内容で補完します：

- Codex 向けモデル推奨（GPT 5.4）
- スキルの自動ロードパス（`.agents/skills/`）
- MCPサーバのマージ戦略説明
- Codex マルチエージェントワークフロー設定（`[features] multi_agent = true`）
- 外部アクション境界（読み取り専用デフォルト）

Codexはネイティブフックを持たないため、この補足仕様が「instruction-backed」のフック代替として機能します。

---

## インストーラの構造

### エントリポイント

```
install.sh (bash)     ─→  scripts/install-apply.js (Node.js)
install.ps1 (PowerShell)      ↑
                              実際の処理はすべてここ
```

`install.sh` はシンボリックリンク解決と MSYS2/Git Bash パス変換を行い、`scripts/install-apply.js` に委譲します。

### 対応ターゲット

`scripts/lib/install-manifests.js` の `SUPPORTED_INSTALL_TARGETS`：

| ターゲット | インストール先 | 種別 |
|---|---|---|
| `claude` | `~/.claude/` | ユーザグローバル |
| `claude-project` | `./.claude/` | プロジェクトローカル |
| `cursor` | `./.cursor/` | プロジェクトローカル |
| `antigravity` | `./.agent/` | プロジェクトローカル |
| `codex` | `~/.codex/` | ユーザグローバル |
| `gemini` | `./.gemini/` | プロジェクトローカル |
| `opencode` | `~/.opencode/` | ユーザグローバル |
| `codebuddy` | `./.codebuddy/` | プロジェクトローカル |
| `joycode` | `./.joycode/` | プロジェクトローカル |
| `qwen` | `~/.qwen/` | ユーザグローバル |
| `zed` | `./.zed/` | プロジェクトローカル |

### 3つのインストールモード

```
モード1: レガシー言語指定（後方互換）
  install.sh typescript python go

モード2: プロファイル指定
  install.sh --profile full --target claude

モード3: モジュール直接指定
  install.sh --modules rules-core,agents-core,hooks-runtime --target cursor
```

**追加オプション：**

- `--locale ja` — 日本語ドキュメントを `~/.claude/docs/ja/` にインストール
- `--with security` / `--without orchestration` — コンポーネントの追加・除外
- `--skills continuous-learning-v2` — 特定スキルのみインストール
- `--dry-run` — 実際のファイル操作なしで計画を表示
- `--json` — 機械可読なプラン/結果JSONを出力
- `--config path/to/ecc-install.json` — JSONファイルからインストール意図を読み込み

### インストールプロファイル一覧

`manifests/install-profiles.json` に定義されています：

| プロファイル | 説明 |
|---|---|
| `minimal` | rules + agents + commands + platform-configs + workflow-quality（フックランタイムなし） |
| `core` | minimal ＋ hooks-runtime |
| `developer` | core ＋ framework-language + database + orchestration |
| `security` | core ＋ security |
| `research` | core ＋ research-apis + business-content + social-distribution |
| `full` | 全モジュール（23モジュール）完全インストール |

### 非破壊インストールとinstall-state追跡

インストーラは各ターゲットに `ecc-install-state.json` を生成し、ECCが管理するファイルを追跡します。これにより：

- 既存のユーザファイルを上書きしない
- アップデート時に以前のECCファイルのみを置換
- アンインストール（`scripts/uninstall.js`）で正確なクリーンアップが可能

---

## インストールターゲット・レジストリとアダプタ

`scripts/lib/install-targets/registry.js` が全アダプタを管理：

```javascript
const ADAPTERS = Object.freeze([
  claudeHome,      // ~/.claude/
  claudeProject,   // ./.claude/
  cursorProject,   // ./.cursor/
  antigravityProject,
  codexHome,       // ~/.codex/
  geminiProject,   // ./.gemini/
  opencodeHome,    // ~/.opencode/
  codebuddyProject,
  joycodeProject,
  qwenHome,        // ~/.qwen/
  zedProject,      // ./.zed/
]);
```

各アダプタは `supports(target)`、`validate(input)`、`resolveRoot(input)`、`planOperations(input)` を実装します。`planInstallTargetScaffold()` がアダプタを解決し、ファイル操作計画（operations配列）を返します。

---

## 配布方法

### npm パッケージ `ecc-universal`

`package.json` の `bin` フィールド：

```json
{
  "bin": {
    "ecc": "scripts/ecc.js",
    "ecc-install": "scripts/install-apply.js"
  }
}
```

インストール後：
```bash
npx ecc typescript          # レガシー言語指定
npx ecc-install typescript  # 後方互換エイリアス
npx ecc --profile full --target cursor
```

`package.json` の `files` フィールドで配布対象を明示管理しています（`.agents/`、`.claude-plugin/`、`.codex/`、`.cursor/`、`.gemini/`、`.opencode/`、`.qwen/`、`.zed/`、`skills/`（全249スキル）、`scripts/`（必要なもの）など）。

### Claude プラグインマーケットプレース

`.claude-plugin/plugin.json` を通じて Claude プラグインとして配布。`skills/` と `commands/` が自動ロードされ、`mcpServers: {}` でMCP自動有効化を抑止。

### 直接クローン

```bash
git clone https://github.com/affaan-m/ECC.git
cd ECC
./install.sh --profile full
```

---

## 設計指針：複数ハーネスへの展開

### ワークフロー展開の手順

新しいワークフローを複数ハーネスに展開する場合の推奨手順：

1. **共有ソースに配置する** — `skills/<name>/SKILL.md` にワークフローを記述
2. **ルールを `rules/` に書く** — ハーネス非依存で再利用可能に
3. **フックを `hooks/hooks.json` に書く** — Claude Code でネイティブ実行される
4. **ハーネス固有は最小限に** — ロード方法・イベント名・コマンド名の適応のみ
5. **インストールモジュールを定義する** — `manifests/install-modules.json` にエントリ追加

### 「3コピー原則」の検査

変更を加えた際に以下を確認します：

- Claude Code版の `hooks/hooks.json` を変更した
- Codex版の `.codex/AGENTS.md` も変更が必要になった
- OpenCode版の `.opencode/plugins/` も変更が必要になった

→ この時点で**設計を見直すシグナル**。ワークフローのコアロジックを `skills/` か `scripts/hooks/` に移動し、各ハーネスアダプタはイベント名の変換のみを担当するように再構成します。

### スコアカード検証

```bash
# ハーネスアダプタの整合性確認
npm run harness:adapters -- --check

# ツールカバレッジ・品質ゲート・メモリ永続化のスコア
npm run harness:audit -- --format json

# 観測可能性シグナルの確認
npm run observability:ready
```

---

## 関連章

- [02 — アーキテクチャ概観](./02-architecture.md)
- [04 — スキル](./04-skills.md)
- [07 — フックとランタイム](./07-hooks-and-runtime.md)
- [09 — MCPと外部連携](./09-mcp-and-integrations.md)
- [11 — 品質・CI・テスト](./11-quality-ci-testing.md)
- [12 — コンポーネントの書き方](./12-authoring-components.md)

## 参照ソース

- `/docs/architecture/cross-harness.md` — クロスハーネスアーキテクチャ設計
- `/docs/architecture/harness-adapter-compliance.md` — コンプライアンス状態マトリクス
- `/AGENTS.md` — クロスツールエージェント仕様
- `/.codex/AGENTS.md` — Codex補足仕様
- `/.opencode/MIGRATION.md` — Claude Code→OpenCodeフックマッピング表
- `/.kiro/README.md` — Kiroハーネスコンポーネント詳細
- `/scripts/lib/install-targets/registry.js` — インストールターゲットレジストリ
- `/scripts/lib/install-targets/cursor-project.js` — Cursorアダプタ実装
- `/scripts/lib/install-manifests.js` — インストールマニフェスト定義
- `/scripts/install-apply.js` — インストーラエントリポイント
- `/manifests/install-profiles.json` — インストールプロファイル定義
- `/package.json` — `bin` / `files` フィールドによる配布設定
- `/legacy-command-shims/README.md` — レガシーコマンドshimsの説明
