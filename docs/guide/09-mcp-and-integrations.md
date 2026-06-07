# 第09章 MCPと外部連携

## この章で学ぶこと

- MCP（Model Context Protocol）の役割とECCにおける位置づけ
- `.mcp.json`（標準バンドル）と `mcp-configs/mcp-servers.json`（拡張カタログ）の構成
- `plugin.json` の `mcpServers: {}` が担う「自動ロード抑止」の意味と理由
- ハーネスごとのMCP設定形式とマージ戦略（Claude JSON / Codex TOML）
- 「MCP vs CLI＋スキル」のトレードオフと判断基準
- `integrations/aura/` — エージェント信頼検証アダプタ
- 自プロジェクトへのMCPの足し方・置き換え方

---

## Claude Code における MCP とは（初心者向け）

> この節は MCP の概念に初めて触れる方向けの基礎解説です。Claude Code 自体の入門（ハーネス・エージェントループ・設定の仕組み）については [03章](./03-components-overview.md) を先にご覧ください。

### MCP はオープン標準プロトコル

**MCP（Model Context Protocol）** は、AI アシスタント（Claude Code など）が外部のツールやデータソースと安全に通信するための **オープンな標準プロトコル** です。Anthropic が主導して策定しましたが、仕様は公開されており他のハーネスや AI ツールも実装できます。

「プロトコル」とは、「どんな形式で情報をやり取りするか」を定めた取り決めです。MCP があることで、異なる AI ハーネス（Claude Code / Codex CLI / Cursor など）と異なるツール（GitHub / Supabase / Playwright など）が、同じルールで接続できます。

### なぜ MCP が存在するのか

Claude（LLM）が外部サービスを操作するには、サービス固有の API を呼び出す方法をモデルに教える必要があります。MCP はその「教え方」を標準化します。

```
MCP がない世界:
  Claude → 独自形式A → GitHub
  Claude → 独自形式B → Supabase   ← 接続方法がバラバラ
  Claude → 独自形式C → Playwright

MCP がある世界:
  Claude → MCP → GitHub サーバ
  Claude → MCP → Supabase サーバ   ← 同じプロトコルで統一
  Claude → MCP → Playwright サーバ
```

標準化によって、MCP サーバを作れば自動的にあらゆる MCP 対応ハーネスで使えるようになります。逆に言えば、Claude Code で使えた MCP サーバは、Cursor や Codex CLI でもほぼそのまま使えます。

### Claude Code はどのように MCP を読み込むか

Claude Code は起動時にいくつかの場所を探索して MCP サーバ設定を読み込みます。

```
読み込み順（優先度が高い順）:
  1. ~/.claude.json            ← ユーザ全体のグローバル設定
  2. プロジェクト直下の .mcp.json  ← プロジェクト固有設定（ECC はここを使用）
  3. .claude-plugin/plugin.json   ← プラグイン経由インストール時
```

設定ファイルには「どのコマンドで MCP サーバを起動するか」または「どの HTTP エンドポイントに繋ぐか」を記述します。Claude Code はセッション開始時にこれらのサーバを起動し、各サーバが公開するツール定義を読み込みます。

### ツール定義がコンテキスト窓を消費する

MCP サーバを有効化すると、そのサーバが提供するすべてのツールの「定義情報（名前・説明・パラメータ仕様）」がコンテキストウィンドウ（[08章](./08-sessions-context-tokens.md) 参照）に読み込まれます。

```
┌─────────────────────────────────────────────────────┐
│ コンテキストウィンドウ                                │
│                                                     │
│  会話履歴 ████████                                  │
│  GitHub MCP ツール定義 ███  ← 接続するだけで消費      │
│  Playwright ツール定義 ████                          │
│  memory ツール定義 ██                               │
│  ...                       ← MCP が増えるほど圧迫   │
└─────────────────────────────────────────────────────┘
```

このため、**使わない MCP を常時有効化しておくことはコストと品質の両方に悪影響を与えます**。ECCが推奨する上限は **プロジェクトあたり 10 個以内** です。

---

## MCPとは何か（ECC の位置づけ）

**MCP（Model Context Protocol）** は、AIアシスタント（Claude Codeなど）と外部サービス・ツールを接続するためのオープンなコネクタプロトコルです。MCP サーバはツール定義をモデルに公開し、モデルはそのツールを関数呼び出しとして利用できます。

ECC がMCPを使う場面は主に2つです：

1. **常時接続が必要なサービス** — GitHub操作、セマンティック検索（Exa）、ドキュメント参照（Context7）、メモリ永続化
2. **リアルタイム連携が必要なサービス** — ブラウザ自動化（Playwright）、逐次思考（Sequential Thinking）

---

## 標準バンドル： `.mcp.json`

リポジトリルートの `.mcp.json` が ECC のデフォルトMCPセットです。Claude Code はこのファイルをプロジェクトのMCP設定として自動検出します。

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github@2025.4.8"]
    },
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp@2.1.4"]
    },
    "exa": {
      "type": "http",
      "url": "https://mcp.exa.ai/mcp"
    },
    "memory": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-memory@2026.1.26"]
    },
    "playwright": {
      "command": "npx",
      "args": ["-y", "@playwright/mcp@0.0.69", "--extension"]
    },
    "sequential-thinking": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-sequential-thinking@2025.12.18"]
    }
  }
}
```

**6サーバの役割まとめ：**

| サーバ | 種別 | 用途 |
|---|---|---|
| `github` | stdio | PRやIssue操作、コード検索 |
| `context7` | stdio | ライブラリドキュメントの最新参照 |
| `exa` | HTTP | Webセマンティック検索・研究 |
| `memory` | stdio | セッション横断のメモリ永続化 |
| `playwright` | stdio | ブラウザ自動化・スクリーンショット |
| `sequential-thinking` | stdio | 複雑問題への逐次推論 |

> **注意：** 同時に有効にするMCPは10個以下に抑えることを推奨します（`mcp-configs/mcp-servers.json` のコメントより）。各MCPのツール定義がコンテキスト窓を消費するためです。

---

## 拡張カタログ： `mcp-configs/mcp-servers.json`

このファイルはプロジェクトに**バンドルされていない**が参照可能な追加MCPのカタログです。Jira、Supabase、Vercel、Firecrawl、Cloudflare、ClickHouse、Playwright（Chrome指定版）などが収録されています。

使い方は「必要なエントリをコピーして `~/.claude.json` の `mcpServers` セクションへ貼り付け、`YOUR_*_HERE` プレースホルダを実際の認証情報に置換する」です。

```jsonc
// mcp-configs/mcp-servers.json より抜粋
{
  "mcpServers": {
    "jira": {
      "command": "uvx",
      "args": ["mcp-atlassian==0.21.0"],
      "env": {
        "JIRA_URL": "YOUR_JIRA_URL_HERE",
        "JIRA_EMAIL": "YOUR_JIRA_EMAIL_HERE",
        "JIRA_API_TOKEN": "YOUR_JIRA_API_TOKEN_HERE"
      },
      "description": "Jira issue tracking"
    },
    "supabase": {
      "command": "npx",
      "args": ["-y", "@supabase/mcp-server-supabase@latest",
               "--project-ref=YOUR_PROJECT_REF"],
      "description": "Supabase database operations"
    }
    // ... 他多数
  }
}
```

**セキュリティ原則：** 認証情報（APIキー、トークン）は設定ファイルに直接埋め込まず、環境変数経由で渡してください。このカタログのプレースホルダ形式がその手本です。

---

## `plugin.json` の `mcpServers: {}` — 自動ロード抑止

`.claude-plugin/plugin.json` は以下の通りです：

```json
{
  "mcpServers": {}
}
```

この**明示的な空オブジェクト**には重要な意味があります。

Claude Code のプラグインシステムはデフォルトで、プラグインルートの `.mcp.json` を自動ロードします。そのままにしておくと、Claude プラグイン経由でインストールした際に ECC のMCPサーバが自動で有効化されます。その結果、ツール名が `mcp__plugin_everything-claude-code_github__create_pull_request_review` のように**64文字を超え**、OpenAI互換ゲートウェイなど一部のプロバイダで拒否されるケースがあります（`.claude-plugin/PLUGIN_SCHEMA_NOTES.md` の「The `mcpServers` Field」節より）。

`"mcpServers": {}` を保持することで：

- Claude プラグインインストール時の自動MCP有効化を抑止
- ツール名の過長問題を回避
- MCP を使いたいユーザは `.mcp.json` または `mcp-configs/mcp-servers.json` から手動で設定

> **誤解注意：** この空オブジェクトを削除すると、Claude プラグインインストール時にルートの `.mcp.json` が再び自動ロードされます。削除しないでください。

---

## ハーネス別のMCP設定形式

### Claude Code — JSON形式

`.mcp.json` をプロジェクトルートに配置するだけで自動検出されます。

### Codex CLI — TOML形式（アドオンマージ）

`.codex/config.toml` の `[mcp_servers.*]` セクションに定義します：

```toml
[mcp_servers.github]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-github"]
startup_timeout_sec = 30

[mcp_servers.context7]
command = "npx"
args = ["-y", "@upstash/context7-mcp@latest"]
startup_timeout_sec = 30

[mcp_servers.exa]
url = "https://mcp.exa.ai/mcp"
```

**マージ戦略の特徴（`.codex/AGENTS.md` より）：**

- `scripts/codex/merge-codex-config.js` が `~/.codex/config.toml` にアドオンマージを実行
- **追記のみ（add-only）** — 既存サーバは変更・削除しない
- ECC が管理する7サーバ（GitHub/Context7/Exa/Memory/Playwright/Sequential Thinking/Supabase）のみ対象
- ユーザ独自の設定は一切保護される
- 既存サーバの設定がECC推奨と乖離している場合は警告をログ出力
- `--update-mcp` フラグで明示的に上書き更新可能
- セクション名は `[mcp_servers.context7]` に正規化（`context7-mcp` などのレガシー名はエイリアス扱い）

### Cursor — `.cursor/mcp.json` へのマージ

`cursor-project.js` インストーラが `.mcp.json` の内容を `.cursor/mcp.json` に `merge-json` 戦略でマージします。ユーザの既存設定を保護しながら ECC サーバを追加します。

---

## MCP vs CLI＋スキル — トレードオフ

`the-longform-guide.md` の「Some MCPs are Replaceable and Will Free Up Your Context Window」節が提示する判断基準：

> MCPはCLIのラッパーに過ぎないことが多い。GitHub、Supabase、VercelはすべてリッチなCLIを持つ。MCPを使うことでツール定義がコンテキスト窓を圧迫する。

| 判断基準 | MCPを残す | CLI＋スキルで置換 |
|---|---|---|
| リアルタイム連携が必要 | yes | — |
| 堅牢なCLIが存在する | — | yes |
| トークン節約が優先 | — | yes |
| 複数ハーネスでの互換性 | — | yes（CLIはハーネス非依存） |
| 操作が状態を持つ | yes | — |

**置換例：**

- GitHub MCP → `gh pr create` をラップした `/gh-pr` コマンド＋`github-ops` スキル
- Supabase MCP → Supabase CLIを使う `database-migrations` スキル

**常時MCPを推奨するケース：**

- `memory` — セッション横断の知識グラフ維持
- `sequential-thinking` — 複雑推論の外部化
- `playwright` — ブラウザ自動化（CLIでの代替困難）

---

## `integrations/aura/` — エージェント信頼検証アダプタ

`integrations/aura/` は、エージェントに委譲や決済を行う前にカウンターパーティの信頼性を検証する**オプトイン・読み取り専用**アダプタです。

**特徴：**

- **ゼロ依存** — Python標準ライブラリのみ。`pip install` 不要
- **読み取り専用** — `GET /check?did=...` のみ実行。署名・送金・ウォレット操作なし
- **明示的呼び出し** — グローバルフックやモンキーパッチなし。信頼境界に置くゲートとして明示的に呼ぶ
- **オフ＝インポート削除** — 使わない場合はインポートしなければ何も起きない

**基本的な使い方：**

```python
from aura import before_settle, AuraUntrusted

def settle(counterparty_did: str, amount: float) -> None:
    try:
        before_settle(counterparty_did)  # high_risk と unknown を拒否
    except AuraUntrusted as e:
        log.warning("blocked: %s", e)
        return
    pay(counterparty_did, amount)        # 既存ロジックはそのまま
```

**評決クラス：**

| 評決 | 意味 | `ok` |
|---|---|---|
| `trusted` | 複合スコア ≥ 0.70 | yes |
| `caution` | 0.40〜0.70 | yes |
| `high_risk` | < 0.40 | no |
| `new` | 登録済みだが実績なし | no |
| `unknown` | 実績なし or AURA到達不能 | no |

**フェイルセーフ：** `aura_verdict()` はネットワーク・パースエラーで例外を投げず `unknown` を返します。デフォルト（`fail_open=False`）では `unknown` は拒否（フェイルクローズド）。`fail_open=True` で通過（フェイルオープン）に切り替えられます。

---

## 自プロジェクトへの適用

### MCPを追加する

1. `mcp-configs/mcp-servers.json` から対象サーバのエントリを探す
2. `~/.claude.json` の `mcpServers` セクションにコピー
3. `YOUR_*_HERE` プレースホルダを実際の認証情報に置換（環境変数経由推奨）
4. プロジェクト固有なら `.mcp.json` をプロジェクトルートに配置

### MCPをCLI＋スキルで置換する

1. MCP が提供するツール群の用途を整理する
2. 対応するCLIコマンドを調べる（例：`gh`, `supabase`, `vercel`）
3. `commands/` にスラッシュコマンドを作成し、CLIを呼び出す
4. `skills/` にワークフロースキルを作成し、手順を記述する
5. `.mcp.json` から当該サーバエントリを削除してコンテキスト窓を節約

### Codex でMCPを追加する

`.codex/config.toml` の `[mcp_servers.<name>]` セクションを追加するか、`scripts/codex/merge-mcp-config.js` を利用してマージします。

---

## MCP/連携の設計思想・ベストプラクティス

MCP はツールを「増やす」ための仕組みですが、増やすほど文脈窓が圧迫されるというトレードオフがあります。以下の原則は ECC 固有のものではなく、**MCP を使うあらゆるプロジェクトで通用する普遍的な設計指針**です。

### 1. 文脈を最小に保つ — 不要な MCP を常時ロードしない

MCP サーバは「接続しているだけ」でツール定義がコンテキスト窓を消費します。使用頻度の低いサーバは常時有効化せず、必要なときだけ追加してください。

```
悪い例: 全 MCP をグローバル設定で常時有効化
  → 毎セッション、使わないツール定義が窓を埋める

良い例: 日常的に使うサーバだけ .mcp.json に定義
       特殊なサーバはプロジェクト固有の .mcp.json またはセッションごとに追加
```

ECC の標準バンドル（`.mcp.json`）が 6 サーバに絞られているのはこの理由です。

### 2. 堅牢な CLI があるなら skill + CLI で置換する

MCP は多くの場合、既存の CLI ツールをラップしているだけです。その CLI が十分に機能するなら、MCP を使わずスラッシュコマンドとスキルで同等の操作を実現できます。これにより：

- コンテキスト窓への影響がゼロになる
- ハーネス非依存（Claude Code 以外でも使える）
- バージョン固定が不要になる

```
GitHub MCP → gh コマンド + /gh-pr スラッシュコマンド
Supabase MCP → Supabase CLI + database-migrations スキル
Vercel MCP → vercel CLI + skill
```

MCP を残すべきケースは「CLI では代替できないリアルタイム連携」（ブラウザ自動化・セッション横断メモリなど）に限られます。

### 3. 秘密情報を設定ファイルに埋め込まない

MCP サーバの認証情報（API キー・トークン）は設定ファイルに直書きしないでください。

```json
// 悪い例
{
  "env": {
    "JIRA_API_TOKEN": "sk-abcdef123456"
  }
}

// 良い例: 環境変数のプレースホルダだけ残す
{
  "env": {
    "JIRA_API_TOKEN": "YOUR_JIRA_API_TOKEN_HERE"
  }
}
```

設定ファイルは git で管理されることが多く、認証情報を直書きすると漏洩リスクが生じます。実際の値はシェルの環境変数・シークレットマネージャ・`.env` ファイル（`.gitignore` 済み）から渡してください。ECC の `mcp-configs/mcp-servers.json` はプレースホルダ形式の手本です。

### 4. ユーザ設定を壊さないマージ戦略

ECC が複数ハーネスで採用しているマージ戦略は **追記のみ（add-only）** です。

- Codex CLI: `merge-codex-config.js` が既存サーバを変更・削除せずに ECC サーバだけ追加
- Cursor: `cursor-project.js` が `.cursor/mcp.json` に `merge-json` 戦略でマージ

自プロジェクトで MCP 設定を配布・インストールする場合も、同じ原則を守ってください。「ユーザが独自に追加した MCP を消さない」ことが信頼性の基盤です。

### 5. 最小権限の原則

MCP サーバに付与する認証情報はそのサーバが必要とする最小限のスコープに絞ってください。

```
悪い例: GitHub トークンに repo:全権限を付与
良い例: PR 作成だけに使うトークンは repo:write のみ
```

`integrations/aura/` が示すように、エージェントが外部サービスに委譲・決済を行う場合は、**信頼検証ゲート**（`before_settle()` 等）を明示的に挟む設計が推奨されます。

### ECC の設計方針との対応

| 普遍則 | ECC における実装 |
|---|---|
| 文脈を最小に保つ | `.mcp.json` を 6 サーバに絞る。`plugin.json` の `mcpServers: {}` で自動ロード抑止 |
| CLI で置換 | `/gh-pr` コマンド、`database-migrations` スキル等 |
| 秘密情報を埋めない | `mcp-servers.json` のプレースホルダ形式 |
| 追記のみのマージ | `merge-codex-config.js`、`cursor-project.js` |
| 最小権限 | `integrations/aura/` のフェイルクローズド設計 |

---

## 関連章

- [02 — アーキテクチャ概観](./02-architecture.md)
- [07 — フックとランタイム](./07-hooks-and-runtime.md)
- [08 — セッション・コンテキスト・トークン](./08-sessions-context-tokens.md)
- [10 — クロスハーネス移植性とインストーラ](./10-cross-harness-and-install.md)

## 参照ソース

- `/.mcp.json` — 標準バンドルMCP設定
- `/mcp-configs/mcp-servers.json` — 拡張MCPカタログ
- `/.claude-plugin/plugin.json` — プラグインマニフェスト（`mcpServers: {}`）
- `/.claude-plugin/PLUGIN_SCHEMA_NOTES.md` — バリデータ制約ドキュメント
- `/.codex/config.toml` — Codex TOML形式MCP設定
- `/.codex/AGENTS.md` — Codexマージ戦略の説明
- `/integrations/aura/README.md` — AURAアダプタ仕様
- `/the-longform-guide.md` — MCP置換トレードオフ（「Some MCPs are Replaceable」節）
- `/scripts/lib/install-targets/cursor-project.js` — CursorのMCPマージ実装
