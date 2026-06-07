---
name: agentic-os
description: Claude Code 上で永続的なマルチエージェントオペレーティングシステムを構築します。カーネルアーキテクチャ・スペシャリストエージェント・スラッシュコマンド・ファイルベースのメモリ・スケジュール自動化・外部データベースなしの状態管理を網羅します。
origin: ECC
---

# Agentic OS

Claude Code をチャットセッションではなく、永続的なランタイム/オペレーティングシステムとして扱います。このスキルは、本番環境のエージェント設定で使用されているアーキテクチャを体系化したものです。タスクをスペシャリストエージェントにルーティングするカーネル設定・永続的なファイルベースのメモリ・スケジュール自動化・JSON/Markdown データレイヤーを含みます。

## 有効化のタイミング

- Claude Code 内でマルチエージェントワークフローを構築する場合
- セッション再起動後も継続する Claude Code 自動化を設定する場合
- 繰り返しタスクのための「パーソナル OS」または「エージェント OS」を作成する場合
- ユーザーが「agentic OS」「personal OS」「multi-agent」「agent coordinator」「persistent agent」と言った場合
- コンテキストがセッションをまたいで保持される必要がある長期プロジェクトを構造化する場合

## アーキテクチャ概要

Agentic OS は 4 つのレイヤーで構成されます。各レイヤーはプロジェクトルートのディレクトリです。

```
project-root/
├── CLAUDE.md          # Kernel: identity, routing rules, agent registry
├── agents/            # Specialist agent definitions (markdown prompts)
├── .claude/commands/  # Slash commands: user-facing CLI
├── scripts/           # Daemon scripts: scheduled or event-driven tasks
└── data/              # State: JSON/markdown filesystem, no external DB
```

### レイヤーの責任

| レイヤー | 目的 | 永続性 |
|---|---|---|
| カーネル（`CLAUDE.md`） | アイデンティティ・ルーティング・モデルポリシー・エージェントレジストリ | Git 管理 |
| エージェント（`agents/`） | スコープ付きツールとメモリを持つスペシャリストアイデンティティ | Git 管理 |
| コマンド（`.claude/commands/`） | ユーザー向けスラッシュコマンド（`/daily-sync`・`/outreach`） | Git 管理 |
| スクリプト（`scripts/`） | Cron またはウェブフックでトリガーされる Python/JS デーモン | Git 管理 |
| 状態（`data/`） | 追記専用ログ・プロジェクト状態・意思決定記録 | Git 除外または管理 |

## カーネル

`CLAUDE.md` がカーネルです。COO/オーケストレーターとして機能します。Claude はセッション開始時にこれを読み込み、作業をルーティングするために使用します。

### カーネル構造

```markdown
# CLAUDE.md - Agentic OS Kernel

## Identity
You are the COO of [project-name]. You route tasks to specialist agents.
You never write code directly. You delegate to the right agent and synthesize results.

## Agent Registry

| Agent | Role | Trigger |
|---|---|---|
| @dev | Code, architecture, debugging | User says "build", "fix", "refactor" |
| @writer | Documentation, content, emails | User says "write", "draft", "blog" |
| @researcher | Research, analysis, fact-checking | User says "research", "analyze", "compare" |
| @ops | DevOps, deployment, infrastructure | User says "deploy", "CI", "server" |

## Routing Rules
1. Parse the user request for intent keywords
2. Match to the Agent Registry trigger column
3. Load the corresponding agent file from `agents/<name>.md`
4. Hand off execution with full context
5. Synthesize and present the result back to the user

## Model Policies
- Default model: use the repository or harness default.
- @dev tasks: prefer a higher-reasoning model for complex architecture.
- @researcher tasks: use the configured research-capable model and approved search tools.
- Cost ceiling: warn before exceeding the project's configured spend threshold.
```

### 主要原則

カーネルは**小さく宣言的**であるべきです。ルーティングロジックはコードではなく、プレーンな Markdown テーブルに記述します。これにより、システムはデバッグなしで検査・編集できます。

## スペシャリストエージェント

各エージェントは `agents/` 内のスタンドアロン Markdown ファイルです。Claude はタスクをルーティングする際に対応するエージェントファイルを読み込みます。

### エージェント定義フォーマット

```markdown
# @dev - Software Engineer

## Identity
You are a senior software engineer. You write clean, tested, production-grade code.
You prefer simple solutions. You ask clarifying questions when requirements are ambiguous.

## Memory Scope
- Read `data/projects/<current-project>.md` for context
- Read `data/decisions/` for architectural decisions
- Append execution logs to `data/logs/<date>-@dev.md`

## Tool Access
- Full filesystem access within project root
- Git operations (status, diff, commit, branch)
- Test runner access
- MCP servers as configured in `.claude/mcp.json`

## Constraints
- Always write tests for new features
- Never commit directly to `main`; use feature branches
- Prefer editing existing files over creating new ones
- Keep functions under 50 lines when possible
```

### マルチエージェント協調パターン

タスクが複数のエージェントにまたがる場合、カーネルはそれらを順次または並行して実行します:

```
User: "Build a landing page and write the launch blog post"

Kernel routing:
1. @dev - "Build a landing page with [requirements]"
2. @writer - "Write a launch blog post for [product] using the landing page copy"
3. Kernel synthesizes both outputs into a unified response
```

並行実行には、Claude Code のバックグラウンドタスク機能か、特定のエージェントコンテキストで Claude Code を呼び出すシェルスクリプトを使用します。

## コマンドと日常ワークフロー

スラッシュコマンドは `.claude/commands/` 内の Markdown ファイルです。再利用可能なワークフローを定義します。

### コマンド構造

```markdown
# /daily-sync

Run the morning briefing:

1. Read `data/logs/last-sync.md` for context
2. Check project status: `git status`, pending PRs, CI health
3. Review `data/inbox/` for new tasks or decisions needed
4. Generate a summary of blockers, priorities, and next actions
5. Append the briefing to `data/logs/daily/<date>.md`
```

### 標準コマンドセット

| コマンド | 目的 |
|---|---|
| `/daily-sync` | 朝のブリーフィング: 状態・ブロッカー・優先度 |
| `/outreach` | アウトリーチワークフローの実行（メール・LinkedIn など） |
| `/research <topic>` | 引用追跡付きの詳細調査 |
| `/apply-jobs` | 対象職種向けの履歴書・カバーレターの調整 |
| `/analytics` | Stripe・GitHub・カスタムソースからのメトリクス取得 |
| `/interview-prep` | フラッシュカードまたはモック面接問題の生成 |
| `/decision <topic>` | 賛否両論と選択した方針を含む意思決定の記録 |

### コマンドの有効化

コマンドファイルを `.claude/commands/<command-name>.md` に配置します。Claude Code が自動検出します。ユーザーは `/<command-name>` で呼び出します。

## 永続メモリ

メモリはファイルベースです。ベクター DB・Redis・PostgreSQL は不要です。`data/` 内の JSON と Markdown ファイルがデータベースです。

### メモリディレクトリ構造

```
data/
├── daily-logs/         # Append-only daily activity logs
├── projects/           # Per-project context files
├── decisions/          # Architectural and business decisions (ADR format)
├── inbox/              # New tasks or ideas awaiting triage
├── contacts/           # People, companies, relationship notes
└── templates/          # Reusable prompts and formats
```

### 日次ログフォーマット

```markdown
# 2026-04-22 - Daily Log

## Sessions
- 09:00 - Session 1: Refactored auth module (@dev)
- 11:30 - Session 2: Drafted investor update (@writer)

## Decisions
- Switched from JWT to session cookies (see `data/decisions/2026-04-22-auth.md`)

## Blockers
- Waiting on API key from vendor (follow up 2026-04-24)

## Next Actions
- [ ] Merge auth refactor PR
- [ ] Send investor update for review
```

### 自動リフレクションパターン

各セッションの終わりに、カーネルはリフレクションを追記します:

```markdown
## Reflection - Session 3
- What worked: Parallel agent execution saved 20 minutes
- What didn't: @researcher hit a paywalled source, need better source ranking
- What to change: Add `source-tier` field to research notes (A/B/C credibility)
```

これにより、コードを変更せずにシステムを時間をかけて改善するフィードバックループが形成されます。

## スケジュール自動化

Agentic OS のタスクは、セッション終了時に停止する Claude Code の組み込み Cron ではなく、外部 Cron を使用してスケジュール実行されます。

### macOS: LaunchAgent

```xml
<!-- ~/Library/LaunchAgents/com.agentic.daily-sync.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" ...>
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.agentic.daily-sync</string>
    <key>ProgramArguments</key>
    <array>
        <string>/claude</string>
        <string>--cwd</string>
        <string>/path/to/project</string>
        <string>--command</string>
        <string>/daily-sync</string>
    </array>
    <key>StartCalendarInterval</key>
    <dict>
        <key>Hour</key>
        <integer>8</integer>
        <key>Minute</key>
        <integer>0</integer>
    </dict>
    <key>StandardOutPath</key>
    <string>/tmp/agentic-daily-sync.log</string>
</dict>
</plist>
```

### Linux: systemd タイマー

```ini
# ~/.config/systemd/user/agentic-daily-sync.service
[Unit]
Description=Agentic OS Daily Sync

[Service]
Type=oneshot
ExecStart=/usr/local/bin/claude --cwd /path/to/project --command /daily-sync
```

```ini
# ~/.config/systemd/user/agentic-daily-sync.timer
[Unit]
Description=Run daily sync every morning

[Timer]
OnCalendar=*-*-* 8:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

### クロスプラットフォーム: pm2

```bash
# ecosystem.config.js
module.exports = {
  apps: [{
    name: 'agentic-daily-sync',
    script: 'claude',
    args: '--cwd /path/to/project --command /daily-sync',
    cron_restart: '0 8 * * *',
    autorestart: false
  }]
};
```

## データレイヤー

データレイヤーはファイルシステムです。構造化データには JSON、ナラティブコンテンツには Markdown を使用します。

### 構造化状態向け JSON

```json
// data/projects/website-v2.json
{
  "name": "Website v2",
  "status": "in-progress",
  "milestone": "beta-launch",
  "agents_involved": ["@dev", "@writer"],
  "files": {
    "spec": "docs/website-v2-spec.md",
    "design": "designs/website-v2.fig"
  },
  "metrics": {
    "commits": 47,
    "last_session": "2026-04-22T11:30:00Z"
  }
}
```

### ナラティブ向け Markdown

意思決定・ログ・調査メモ・連絡先記録など、人間が読むものには Markdown を使用します。

### スキーマの進化

既存フィールドの名前変更は行わないでください。新しいフィールドを追加し、古いものを非推奨としてマークします:

```json
{
  "name": "Website v2",
  "status": "in-progress",
  "milestone": "beta-launch",
  "_deprecated_priority": "high",
  "priority_v2": { "level": "high", "rationale": "Blocks investor demo" }
}
```

これにより、マイグレーションスクリプトなしに過去データを読み取り可能な状態に保ちます。

## アンチパターン

### モノリシックな単一エージェント

```markdown
# BAD - One agent does everything
You are a full-stack developer, writer, researcher, and DevOps engineer.
```

スペシャリストエージェントに分割してください。ルーティングはカーネルが担います。

### ステートレスセッション

```markdown
# BAD - No memory between sessions
Starting fresh every time Claude Code opens.
```

セッション開始時に常に `data/` を読み込み、セッション終了時に書き戻してください。

### ハードコードされた認証情報

```markdown
# BAD - API keys in agent files or CLAUDE.md
Your OpenAI API key is sk-xxxxxxxx
```

環境変数またはスクリプトが読み込む `.env` ファイルを使用してください。エージェントは `process.env.API_KEY` を参照します。

### シンプルな状態向けの外部データベース

```markdown
# BAD - PostgreSQL for a solo user's agentic OS
```

複数の同時ユーザーまたは GB 規模のデータが必要になるまで、JSON/Markdown ファイルを使用してください。

### 過度に複雑なルーティング

```markdown
# BAD - Routing logic in code instead of markdown tables
if (intent.includes('deploy')) { agent = opsAgent; }
```

ルーティングは `CLAUDE.md` の Markdown テーブルで宣言的に記述してください。検査・編集・デバッグが容易になります。

## ベストプラクティス

- [ ] `CLAUDE.md` は 200 行以内でコンテキストウィンドウに収まること
- [ ] 各エージェントファイルは 100 行以内で 1 つのドメインに集中していること
- [ ] `data/` は機密ログには Git 除外、意思決定とスペックには Git 管理すること
- [ ] コマンドは命令形の名前を使用すること: `/daily-sync`（`/run-daily-sync` ではなく）
- [ ] ログは追記専用であること。過去の日次ログを編集しないこと
- [ ] すべてのエージェントに、読み込むファイルを定義する `Memory Scope` セクションがあること
- [ ] すべてのセッションの終わりにリフレクションを書くこと
- [ ] スケジュールタスクは Claude Code のセッション Cron ではなく外部 Cron（LaunchAgent・systemd・pm2）を使用すること
- [ ] コスト追跡: API 支出を `data/logs/<date>-costs.json` にセッションごとに記録すること
- [ ] 1 プロジェクト = 1 Agentic OS。無関係なプロジェクト間で単一の `CLAUDE.md` を共有しないこと
