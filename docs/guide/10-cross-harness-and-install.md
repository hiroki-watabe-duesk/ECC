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

## ハーネスとは何か／なぜ移植性が要るか（初心者向け）

> この節は「ハーネス」や「クロスハーネス」という言葉に初めて触れる方を対象にした基礎解説です。Claude Code の基礎（エージェントループ・権限モデル・プラグイン配布の仕組み全般）は [第03章 コンポーネント体系と選択基準](./03-components-overview.md) で詳しく扱っています。この節ではクロスハーネス移植性に必要な部分だけを簡潔にまとめます。

### ハーネスとは

**ハーネス（Harness）** とは、LLM（大規模言語モデル）にツール・コンテキスト・制御ループを与えて、単なるテキスト生成以上のことができるようにした **実行環境** の総称です。Claude Code はそのハーネスの一例ですが、同様の概念を実装したツールは他にも多数存在します。

代表的なハーネスの例：

| ハーネス | 提供元 | 主な特徴 |
|---|---|---|
| Claude Code | Anthropic | CLI / IDE拡張 / デスクトップ。スキル・エージェント・フック・MCPを持つ |
| Codex CLI | OpenAI | CLI中心。ネイティブフックなし（指示ベースで代替） |
| Cursor | Anysphere | IDE組み込み。ルールは `.mdc` 形式 |
| OpenCode | OpenCode | CLIエージェント。プラグインイベント名が異なる |
| Gemini CLI | Google | CLI。フックパリティが限定的 |
| Zed | Zed Industries | エディタ組み込み。エージェントモデルが異なる |
| Kiro | AWS | IDE組み込み。ステアリングファイル形式を使用 |

### なぜ各ハーネスで設定が違うのか

ハーネスはそれぞれ独立したエコシステムとして設計されており、設定の **形式・ファイル配置・イベント名** が異なります。

- Claude Code はフックイベントを `PreToolUse` / `PostToolUse` / `Stop` などと呼ぶ
- OpenCode の同等イベントは `tool.execute.before` / `tool.execute.after` / `session.idle`
- Kiro では `preToolUse` / `postToolUse` / `agentStop`
- Cursor のルールは `.md` ではなく `.mdc` 拡張子を使い、ディレクトリ構造をフラット化する必要がある

このためワークフローを「1つのハーネス向けに書いてコピーして回る」アプローチは、変更のたびにすべてのコピーを更新しなければならない保守地獄を生みます。

### Claude Code のプラグイン・マーケットプレース超入門

Claude Code には **プラグイン** の概念があります。スキル・エージェント・コマンド・フック・MCPをひとまとめにして配布する仕組みです。

```
プラグイン = スキル + エージェント + コマンド + フック + MCP を束ねたパッケージ
```

プラグインは Claude Code のマーケットプレースから導入できます。ECC 自体も `.claude-plugin/plugin.json` マニフェストを持つプラグインとして配布されており、マーケットプレースからインストールすると `skills/` と `commands/` が自動ロードされます。

```
.claude-plugin/
└── plugin.json   ← name・version・skills・commands・features を宣言するマニフェスト
```

プラグインを **自作・配布** する場合も、この形式に従えば同じマーケットプレースに登録できます。

### クロスハーネス移植性が要る理由

チームが複数のハーネスを併用するケース（例：あるメンバーは Claude Code、別のメンバーは Cursor、CI は OpenCode）では、同じワークフロー定義を各ハーネス向けに提供しなければなりません。そこで ECC が採用しているのが **「1ソース→多ハーネス」** モデルです。ワークフローの本質的なロジックを共有ソースに置き、各ハーネス向けの差異は薄いアダプタ層だけで吸収します。詳細は以下の節で解説します。

---

## クロスハーネスの使い方（実践クイックスタート）

「クロスハーネスを使う」とは、ECC の同じワークフロー資産（スキル・エージェント・ルール・フック・MCP）を、**自分が使う各ハーネスにインストールして、どのツールでも同じ作法で開発できるようにする**ことです。ここでは最短の導入から、複数ハーネス併用・更新・アンインストールまでを順に示します。仕組みの詳細は後続の節へ、ここは「まず動かす」ための手順に絞ります。

### ステップ0: 自分が使うハーネスを把握する

まず、どのツールに入れたいかを決めます（Claude Code / Cursor / Codex / OpenCode / Gemini / Zed / Qwen / Codebuddy / Antigravity など）。各ツールの対応度合い（Native / Adapter-backed / Instruction-backed）は本章の [ハーネス別コンプライアンス状態](#ハーネス別コンプライアンス状態) で確認できます。フックが効くかどうかなどはここで決まります。

### ステップ1: 導入方法を1つ選ぶ（重要：方法を混ぜない）

導入経路は3つあります。

1. **プラグイン（Claude Code 向け・最も簡単）**: Claude Code の `/plugin` でマーケットプレースから `ecc@ecc` を入れる。
2. **インストーラ `install.sh` / `install.ps1`（クロスハーネスの主役）**: リポジトリ／パッケージから任意のハーネスへ展開する。
3. **npm**: `npx ecc-install ...`（`install.sh` と同じ動作。クローン不要）。

> ⚠️ **最重要の注意：導入方法を重ねない**
> `/plugin install` 済みのところに `install.sh --profile full`（や `npx ecc-install --profile full`）を実行すると、スキル・コマンド・フックが**二重に入り挙動が重複**します。プラグインで入れたら、その上に追加のフルインストールはしないでください。

### ステップ2: まず dry-run で計画を確認する

いきなり書き込まず、`--dry-run` で「何がどこに入るか」だけ見ます。

```bash
./install.sh --profile minimal --target claude --dry-run
```

`--json` を付けると機械可読な計画を出力します。問題なければ `--dry-run` を外して実行します。

### ステップ3: ハーネスごとにインストールする

ターゲットを `--target` で切り替えるだけで、各ハーネスのネイティブな場所に展開されます。

| ハーネス | コマンド例 | 入る場所 |
|---|---|---|
| Claude Code（ユーザ全体・既定） | `./install.sh --profile minimal --target claude` | `~/.claude/` |
| Claude Code（プロジェクト内） | `./install.sh --profile minimal --target claude-project` | `./.claude/` |
| Cursor | `./install.sh --profile minimal --target cursor` | `./.cursor/` |
| Codex | `./install.sh --profile minimal --target codex` | `~/.codex/` |
| OpenCode | `./install.sh --profile minimal --target opencode` | `~/.opencode/` |
| Gemini | `./install.sh --profile minimal --target gemini` | `./.gemini/` |
| Zed | `./install.sh --profile minimal --target zed` | `./.zed/` |
| Qwen | `./install.sh --profile minimal --target qwen` | `~/.qwen/` |
| Codebuddy | `./install.sh --profile minimal --target codebuddy` | `./.codebuddy/` |
| Antigravity | `./install.sh --profile minimal --target antigravity` | `./.agent/` |

- **Windows** は `.\install.ps1 ...`、**クローンしたくない場合**は `npx ecc-install ...` に置き換えます（フラグは同じ）。
- `--target` を省略すると既定の `claude` に入ります。

### ステップ4: 入れる量（プロファイル）を選ぶ

プロファイルは同梱コンポーネントのバンドルです。**小さく始めて、必要に応じて増やす**のが定石です。

| プロファイル | 目安 |
|---|---|
| `minimal` | 中核ルール＋最小限。まず動かす用 |
| `core` | 標準的な日常開発 |
| `developer` | 開発者向け一式 |
| `security` | セキュリティ重視 |
| `research` | リサーチ用途 |
| `full` | 全部入り |

- 微調整は `--with <component>` / `--without <component>` で個別に増減できます。
- 明示選択は `--modules <id,id,...>`（モジュール指定）や `--skills <id>`（スキル単体）も可能です。
- プロファイルの正確な内訳は `manifests/install-profiles.json`、詳細は本章の [インストールプロファイル一覧](#インストールプロファイル一覧) を参照。

### ステップ5: 複数ハーネスへ展開する（クロスハーネスの本番）

使うハーネスの数だけ、同じ手順をターゲットを変えて流すだけです。

```bash
# 例: Claude Code（自分用）と Cursor（このリポジトリ）を併用する
./install.sh --profile core --target claude
./install.sh --profile core --target cursor
```

ここが「1ソース→多ハーネス」の効きどころです。**共有ソース（`skills/` `rules/` `hooks/` など）を1か所編集し、各ターゲットへ再インストールするだけで全ハーネスへ反映**されます。ハーネスごとに別々に書き直す必要はありません（編集してよいのは共有ソースだけ、という原則は後続の[設計思想](#設計思想1ソース多ハーネス)で詳述）。

日本語の翻訳ドキュメントも一緒に入れたい場合（`claude` / `claude-project` ターゲットのみ）は `--locale ja` を付けます。

```bash
./install.sh --profile core --target claude --locale ja
```

### ステップ6: 動作確認する

- インストール後、ハーネスを再起動またはリロードし、スキルやスラッシュコマンドが現れるかを確認します。
- リポジトリ運用者向けには整合性チェックのコマンドがあります。

```bash
npm run harness:adapters   # 各ハーネスアダプタの整合性を確認
npm run harness:audit      # ツールカバレッジ・品質ゲートのスコアを確認
```

### 更新とアンインストール

- **更新**: 新しいソース／版を取得し、同じ `--target` ・ `--profile` で再実行します。非破壊で、`install-state` により差分管理されます。
- **アンインストール**: 以下を実行します。manifest 追跡により、ECC が入れたものだけを安全に除去します。

```bash
node scripts/uninstall.js --target <target> --dry-run   # まず計画を確認
node scripts/uninstall.js --target <target>             # 実行
```

### よくあるつまずき（ミニ・トラブルシュート）

| 症状 | 原因 | 対処 |
|---|---|---|
| スキル／コマンドが二重に出る | プラグイン導入後にフルインストーラを重ねた | どちらか一方に統一し、重複分を除去する |
| フックが効かない | Instruction-backed なハーネス（Codex・Gemini 等）はフック非対応 | そのハーネスは `AGENTS.md` 等の**指示ベース**で担保される（[コンプライアンス状態](#ハーネス別コンプライアンス状態)参照） |
| 別ハーネスで挙動が違う | アダプタによりイベント名・形式が異なる | [ポータビリティ・マトリクス](#ポータビリティマトリクス)・[フックのイベント対応表](#フックのイベント対応表)を確認 |
| どこに入ったか分からない | ターゲットごとに配置先が異なる | ステップ3の表の「入る場所」、または `--dry-run` で確認 |

> ここまでが「使い方」の最短経路です。**なぜこの形なのか**（共有層とアダプタ層の分離、3コピー原則など）は次節以降で解説します。自分のプロジェクトにこの仕組みを持ち込む方法は [13-applying-to-your-project.md](./13-applying-to-your-project.md) を参照してください。

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

## クロスハーネス設計の思想・ベストプラクティス

この節では、ECC の実装から抽出した普遍的なベストプラクティスをまとめます。ECC 固有の文脈を超えて、任意のハーネスや任意の規模のプロジェクトに適用できる設計原則です。

### 原則1：「1ソース→多ハーネス」を徹底する

ワークフローの **永続的・本質的なロジック**（スキルの手順、ルールの方針、フックのビジネスロジック）は共有ソース層に集約し、**ハーネス固有の差異**（イベント名の変換、ファイル形式の変換、コマンド名のマッピング）はエッジのアダプタ層だけに押し込めます。

```
【良い構造】
共有ソース層（skills/, rules/, scripts/hooks/）
        ↓
ハーネスアダプタ（イベント名変換・形式変換のみ）
        ↓
各ハーネス（Claude Code / Cursor / OpenCode / Kiro ...）

【悪い構造】
Claude Code 向けロジック（.claude/ に直接書く）
Cursor 向けコピー（.cursor/ に手で書く）
OpenCode 向けコピー（.opencode/ に手で書く）
  → 変更のたびに3箇所を同期しなければならない
```

### 原則2：「3コピー編集が必要なら設計が間違い」

変更を1箇所加えたときに、3つ以上のハーネス固有ファイルを同時に編集しなければならない状況は **設計の問題を示すシグナル** です。その場合はワークフローのコアロジックを `skills/<name>/SKILL.md` または `scripts/hooks/` に移動し、各アダプタはイベント名や形式の変換だけを担当するよう再構成します。

### 原則3：永続ロジックは共有層に、ハーネス固有はエッジだけに

各コンポーネントの配置ルールは単純です。

| 配置先 | 置くもの |
|---|---|
| `skills/` | ワークフロー手順・ドメイン知識（最もポータブルな単位） |
| `rules/` | コーディング規約・セキュリティ方針（ハーネス非依存） |
| `scripts/hooks/` | フックの実装ロジック（Node.js スクリプト） |
| ハーネスアダプタ（`.claude/`, `.cursor/` 等） | ロード方法・イベント名・形式の変換のみ |

「このロジックは5年後も別のハーネスで再利用できるか？」という問いを持つことで、共有層に置くべきかアダプタに置くべきかを判断できます。

### 原則4：非破壊インストールを守る

インストーラは **既存ファイルを上書きしない** ことが大原則です。ユーザがカスタマイズしたファイルを無断で消してしまうインストーラは、1回の `install` コマンドで作業を無に帰します。ECC は `ecc-install-state.json` で自身が管理するファイルを追跡し、更新時には「ECCが生成したファイルのみ」を置換することでこの原則を守っています。

独自のインストーラを作る場合も、「初回インストールで作成したファイルの一覧を保存し、更新時はその一覧のみ上書き」というパターンを採用すると安全です。

### 原則5：最小プロファイルから始め、必要に応じて拡張する

新しい環境や新しいプロジェクトにハーネス設定を導入するとき、**最初から全モジュールを有効化しない**ことを推奨します。

```bash
# 推奨：最小プロファイルで始める
./install.sh --profile minimal --target claude-project

# 動作確認してから拡張
./install.sh --profile core --target claude-project

# チームの規模・用途が確定してから完全インストール
./install.sh --profile full --target claude
```

「あとで減らすのは増やすより難しい」。最初に全機能を有効化して後からノイズとなるフックを無効にしていくより、必要な機能だけを足していく方が管理しやすくなります。ECCの `--dry-run` オプションを使うとファイル操作を実際に行わずインストール計画を確認できます。

### 原則6：コンプライアンス状態を定期的に把握する

ハーネスは独自のペースで進化します。今日 Native 対応しているハーネスが、6か月後には仕様変更で Adapter-backed に後退することもあります。定期的にスコアカードを確認し、ドリフト（ずれ）を検出・文書化する習慣をつけます。

```bash
# ハーネスアダプタの整合性確認
npm run harness:adapters -- --check

# 観測可能性シグナルの確認
npm run observability:ready
```

「使っていないハーネスのアダプタは削除」「新しいハーネスのアダプタは最小限で始める」という姿勢が長期的な保守性を保ちます。

### まとめ：クロスハーネス設計の判断軸

> - ロジックは共有層に。アダプタにはロジックを書かない。
> - 3つ以上のコピーを同時に直す必要が生じたら設計を疑う。
> - インストーラは非破壊。最小プロファイルから始める。
> - ハーネスのコンプライアンス状態を定期レビューする。

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
