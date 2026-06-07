# 第03章 コンポーネント体系と選択基準

この章で学ぶこと: Claude Code というプラットフォームの基礎（ハーネス・エージェントループ・設定の仕組み・権限モデル・トークン経済・モデル階層・プラグイン配布）を押さえたうえで、ECCが提供する6種類のコンポーネント（Skill / Agent / Command / Rule / Hook / MCP）の役割と違い、コンポーネント同士がどう連携するか、そして「ある要件をどのコンポーネントに実装すべきか」を判断するための基準を習得します。この章は **ガイド全体の「プラットフォーム基礎」のハブ** であり、各章（[04〜09章](#コンポーネント別の詳細章へのリンク)）からここを参照します。

---

## Claude Code プラットフォーム入門（初心者向け）

> この節は Claude Code を初めて使う方を対象にした基礎解説です。すでに Claude Code の仕組みに慣れている方は「[コンポーネント一覧](#コンポーネント一覧)」へ進んでください。

### Claude Code とは何か

**Claude Code** は Anthropic が提供する公式の **エージェント型コーディング環境** です。CLI・IDE拡張・デスクトップアプリ・Webの複数の形態で動作します。

通常の AI チャットと異なる最大の特徴は、Claude が「ツール」を使ってタスクを実際に実行できる点です。ファイルを読む・編集する・シェルコマンドを実行する・Web 検索をするといったツールを組み合わせて、自然言語の指示を **エンドツーエンドで完遂** します。

### 「ハーネス」と「エージェントループ」

**ハーネス（Harness）** とは、LLM（大規模言語モデル）にツール群・コンテキスト・制御ループを与えて、単なるテキスト生成以上のことができるようにした **実行環境** のことです。Claude Code はそのハーネスの一例です。

**エージェントループ** はその制御ループの核心部分です。

```
ユーザの指示
     │
     ▼
  LLM が応答を生成（ツール呼び出しを含む場合あり）
     │
     ├─ ツール呼び出しがある → ハーネスがツールを実行 → 結果を LLM へ返す
     │                                                        ↑ ここでループ
     └─ ツール呼び出しがない → ユーザへ最終回答を出力
```

このループが続く限り、Claude は複数ステップのタスクを自律的に完遂します。ECC はこのループを **より強力・安全・再利用可能** にするためのコンポーネント群を提供します。

### 設定の在りか: `~/.claude/` と プロジェクト `.claude/`

Claude Code の設定はスコープによって2か所に分かれます。

```
~/.claude/                    ← ユーザ全体の設定（どのプロジェクトでも有効）
  ├── settings.json           ← 権限・フック・環境変数・モデル等のグローバル設定
  ├── agents/                 ← グローバルなサブエージェント定義
  ├── skills/                 ← グローバルなスキル
  └── CLAUDE.md               ← ユーザ全体の常時指示

<リポジトリ直下>/
  .claude/                    ← プロジェクト固有の設定（このリポジトリだけに有効）
  ├── settings.json           ← プロジェクトの権限・フック設定
  ├── agents/                 ← プロジェクト固有のエージェント
  ├── skills/                 ← プロジェクト固有のスキル
  └── rules/                  ← プロジェクト固有のルール

CLAUDE.md                     ← プロジェクト指示。Claude 起動時に自動でコンテキストへ読み込まれる
```

**`CLAUDE.md`** は特別なファイルです。Claude Code がセッションを開始するたびに自動でコンテキストへ読み込まれ、アーキテクチャ・コーディング規約・スキル対応表などプロジェクト固有の指示を常に有効にします。

### 6コンポーネントを Claude Code がどう発見・起動するか

| コンポーネント | 配置場所（標準） | 起動トリガー |
|---|---|---|
| **Agent（サブエージェント）** | `.claude/agents/*.md` または `~/.claude/agents/` | Task ツールによる委譲（明示的な名前指定） |
| **Skill（スキル）** | `.claude/skills/<name>/SKILL.md` | コンテキスト自動マッチ、またはコマンド・エージェントから明示参照 |
| **Command（スラッシュコマンド）** | `.claude/commands/<name>.md` | ユーザが `/name` を入力 |
| **Rule（ルール）** | `.claude/rules/*.md` または `CLAUDE.md` 内 | セッション全体に常時有効（paths glob でスコープ制御可能） |
| **Hook（フック）** | `settings.json` の `hooks` キー | ライフサイクルイベント自動発火（PreToolUse / PostToolUse / Stop / SessionStart 等） |
| **MCP（Model Context Protocol）** | `.mcp.json` 等でサーバ接続を設定 | 設定済みサーバへのツール呼び出し時 |

各コンポーネントの詳細な配置ポリシー・フォーマットは各詳細章で解説します（[コンポーネント別の詳細章へのリンク](#コンポーネント別の詳細章へのリンク)）。

### 権限モデル

Claude Code はすべてのツール実行を **権限モデル** で制御します。ファイル書き込み・シェル実行・外部通信など、影響の大きい操作は実行前にユーザの許可を求めます。`settings.json` の `permissions` セクションで allow/deny ルールをあらかじめ設定しておくと、承認ダイアログを減らして作業を効率化できます。

### コンテキストウィンドウとトークン経済

Claude のコンテキストウィンドウ（一度に処理できるテキスト量）は **有限** です。会話が長くなると自動的に **圧縮（compaction）** や要約が実行されます。

- 長大なファイルの全文を何度もコンテキストに入れると費用と時間が増す
- スキルの **progressive disclosure**（必要なときだけ詳細を開く）設計はこのトークン経済を最適化するためのもの
- エージェントに委譲するとサブエージェントは独立したコンテキストで動き、結果の要約だけがメインへ返るためコンテキストを節約できる

### モデル階層

| モデル | 特性 | 典型的な用途 |
|---|---|---|
| **Opus** | 高精度・高コスト | 複雑な設計判断・難解なバグ・長期計画 |
| **Sonnet** | バランス型（日常のデフォルト） | コードレビュー・実装・リファクタリング |
| **Haiku** | 軽量・高速・低コスト | 単純なフォーマット・補完・監視タスク |

エージェント定義のフロントマターで `model: sonnet` のように指定することで、タスクの重さに応じたモデルを選べます。

### プラグイン配布

スキル・エージェント・コマンド・フック・MCP をまとめて **プラグイン** として配布できます。ECC 自体がプラグインとして Claude Code マーケットプレイス経由で複数プロジェクト・複数ハーネスへ展開される仕組みです。ユーザは `~/.claude/plugins/` 以下にインストールされたプラグインを透過的に利用します。

---

## コンポーネント一覧

ECCは以下の6種類のコンポーネントで構成されています。それぞれが明確に異なる責務を持ちます。

| コンポーネント | 目的 | 起動方法 | いつ使うか | 永続性 / ハーネス横断性 |
|---|---|---|---|---|
| **Skill** | ドメイン知識・ワークフローの構造化（パッシブな知識モジュール） | コンテキスト自動マッチ、または名前で明示参照 | フレームワーク規約・テスト手順・設計パターンなど「知識」を持ち運びたいとき | 高い。curated は ships され他プロジェクトへも移植可能 |
| **Agent** | 限定スコープの専門サブエージェント（タスク実行） | 明示的な委譲（`/command` 内や会話で名前指定） | コードレビュー・計画・E2Eテストなど専門性の高い作業を分離したいとき | セッション内。再利用はスキルで行う |
| **Command** | ユーザが起動するスラッシュ操作のエントリポイント | `/command-name` を入力 | ユーザがワンショットでワークフローを呼び出したいとき | スラッシュ操作のシム。エージェントやスキルへ委譲する窓口 |
| **Rule** | 常時適用のガイドライン（always-on制約） | `paths` glob にマッチしたファイルを操作するとき自動適用 | セキュリティ要件・コーディングスタイル・テスト方針など「常に守るべき規約」を強制したいとき | セッション全体に常時有効。glob でスコープ制限可能 |
| **Hook** | イベント駆動の自動化（PreToolUse / PostToolUse / Stop 等） | Claude Codeのツール呼び出しイベント発火時に自動実行 | フォーマット検査・品質ゲート・セッション永続化など副作用を自動化したいとき | セッション中に常駐。ハーネス全体のランタイムと密結合 |
| **MCP** | 外部サービスとのインタラクション境界（Model Context Protocol） | 設定済みMCPサーバへのツール呼び出し | Jira・GitHub・Supabase等の外部APIを継続的・インタラクティブに操作したいとき | 設定持続。ただしサーバ起動コストあり |

### Claude Code における各コンポーネントのプラットフォーム定義

上の表は ECC 的な視点でのまとめです。Claude Code プラットフォームとしての定義を以下に補足します。

**Skill（スキル）**
: Claude Code が「コンテキストに関連する知識モジュール」として参照する Markdown ファイル。実行コードではなくモデルに注入される構造化知識。フロントマターの `description` をモデルが読み、関連性があると判断したときにのみ本文を読み込む（progressive disclosure）。詳細 → [04-skills.md](./04-skills.md)

**Agent（サブエージェント）**
: Task ツールで委譲される専門サブエージェント。メインのコンテキストとは独立した専用ウィンドウで動作し、結果の要約のみをメインへ返す。コンテキスト隔離によってメインの汚染と費用増を防ぐ。詳細 → [05-agents.md](./05-agents.md)

**Command（スラッシュコマンド）**
: `/name` という形式でユーザが直接起動する操作のエントリポイント。コマンド自体はロジックを持たず、エージェントやスキルへ処理を委譲する「玄関」として機能する。詳細 → [06-commands-and-rules.md](./06-commands-and-rules.md)

**Rule（ルール）**
: セッション全体に常時適用される制約・ガイドライン。paths glob でスコープを絞ることができる。モデルへの「常に従うべき指示」として機能し、LLM の判断より優先される。詳細 → [06-commands-and-rules.md](./06-commands-and-rules.md)

**Hook（フック）**
: `settings.json` に登録されたライフサイクルイベントハンドラ。LLM が「やるかどうか判断する」のではなく、**ハーネスが決定論的に必ず実行する**。PreToolUse・PostToolUse・Stop・SessionStart 等のイベントで Node.js スクリプトを自動起動できる。詳細 → [07-hooks-and-runtime.md](./07-hooks-and-runtime.md)

**MCP（Model Context Protocol）**
: 外部ツールサーバに接続してツールを追加する Anthropic 標準プロトコル。GitHub・Slack・データベース等の外部サービスを Claude のツールとして透過的に利用可能にする。詳細 → [09-mcp-and-integrations.md](./09-mcp-and-integrations.md)

---

## コンポーネントの参照関係

コンポーネントは互いに参照・委譲し合います。主な依存の流れは以下のとおりです。

```
ユーザ
  └─ /command（スラッシュコマンド）
        ├─ Agent へ委譲（docs/COMMAND-AGENT-MAP.md 参照）
        │    例: /code-review → code-reviewer
        │       /plan        → planner
        │       /tdd         → tdd-guide
        └─ Skill を直接参照
             例: /learn  → continuous-learning スキル
                 /verify → verification-loop スキル

Agent
  └─ Skill の規約をプロンプトに注入（CLAUDE.md: "always pass conventions
       from the respective skill into the agent's prompt"）

Rule（paths glob で暗黙適用）
  └─ 常時アクティブ。エージェント・スキル双方に影響

Hook（イベント駆動）
  └─ run-with-flags.js ラッパー経由でスキルやスクリプトを実行可能

MCP
  └─ Agent や Command から外部サービスへのツールブリッジ
```

### CLAUDE.md ファイル→スキル対応表

CLAUDE.md の `## Skills` セクションには、ファイルパターンとスキルの対応が定義されています。

| ファイルパターン | 使用スキル | 備考 |
|---|---|---|
| `README.md` | `/readme` | READMEの編集時 |
| `.github/workflows/*.yml` | `/ci-workflow` | CI設定の変更時 |
| `*.tsx`, `*.jsx`, `components/**` | `react-patterns`, `react-testing` | React固有作業時は `/react-review` 等を呼び出す |

### コマンド→エージェントマップ

主要なコマンドとエージェントの関係（詳細は [docs/COMMAND-AGENT-MAP.md](../COMMAND-AGENT-MAP.md)）:

| コマンド | 主エージェント | スキル参照 |
|---|---|---|
| `/plan` | planner | — |
| `/tdd` | tdd-guide | — |
| `/code-review` | code-reviewer | — |
| `/orchestrate` | planner, tdd-guide, code-reviewer, security-reviewer, architect | マルチエージェント |
| `/learn` | — | continuous-learning |
| `/verify` | — | verification-loop |
| `/security-scan` | security-reviewer | security-scan |

---

## 「どのコンポーネントを使うか」決定木

要件を前にしたとき、以下の順で判断してください。

```
要件 or 機能を実装したい
│
├─ 常に守るべき制約・禁止事項・スタイル規約か？
│    → Rule（rules/<lang>/coding-style.md 等）
│
├─ 外部サービス（Jira / GitHub / DBなど）への継続的なアクセスが必要か？
│    → MCP（mcp-configs/ に設定を追加）
│
├─ ツール実行直前・直後・セッション終了時に副作用を自動実行したいか？
│    → Hook（hooks/hooks.json に PreToolUse / Stop 等を追加）
│
├─ ユーザが `/command` で呼び出すワンショットのエントリポイントか？
│    → Command（commands/<name>.md）
│         └─ 中身はエージェント or スキルへ委譲する構造にする
│
├─ 専門性の高い作業を独立したサブエージェントに分離したいか？
│    （コードレビュー・計画・テスト実行 等）
│    → Agent（agents/<name>.md）
│         └─ 対応スキルの規約をプロンプトに注入すること
│
└─ ドメイン知識・ワークフロー・ベストプラクティスを構造化して持ち運びたいか？
     → Skill（skills/<name>/SKILL.md）
          └─ 最も移植性が高い。まず Skill で表現できないか検討する
```

### 補足: Rule vs Skill

| 観点 | Rule | Skill |
|---|---|---|
| 適用タイミング | 常時（paths globでスコープ） | コンテキストマッチ時 or 明示参照時 |
| 内容の性質 | 禁止・制約・強制事項 | 知識・手順・パターン |
| 長さ | 短い（チェックリスト程度） | 詳細（200〜800行) |
| 使い分け | セキュリティ禁止事項・命名規約 | フレームワークパターン・TDDフロー |

---

## 実践フロー: 要件をコンポーネントに落とし込む

**例: 「Pythonファイルを編集するとき必ず型ヒントを使わせたい」**

1. 常時適用の制約 → **Rule** が適切
2. `rules/python/coding-style.md` に型ヒント義務を追記
3. paths glob で `**/*.py` を指定すれば自動適用

**例: 「FastAPIアプリを開発するときのベストプラクティスを提供したい」**

1. ドメイン知識・ワークフロー → **Skill** が適切
2. `skills/fastapi-patterns/SKILL.md` を作成
3. `When to Activate` に「FastAPIアプリを構築・レビューするとき」と明記

**例: 「コードレビューを専門エージェントに委譲したい」**

1. 専門タスクの分離 → **Agent** が適切
2. `agents/code-reviewer.md` に `tools` と `model` を定義
3. `commands/code-review.md` を作成してユーザが `/code-review` で呼べるようにする
4. エージェントのプロンプトにスキル（例: `coding-standards`）の規約を注入する

---

## コンポーネント別の詳細章へのリンク

| コンポーネント | 詳細章 |
|---|---|
| Skill | [04-skills.md](./04-skills.md) |
| Agent | [05-agents.md](./05-agents.md) |
| Command / Rule | [06-commands-and-rules.md](./06-commands-and-rules.md) |
| Hook | [07-hooks-and-runtime.md](./07-hooks-and-runtime.md) |
| MCP | [09-mcp-and-integrations.md](./09-mcp-and-integrations.md) |

---

## 設計思想: なぜこの6分類なのか

### 関心の分離（Separation of Concerns）

6つのコンポーネントは、AI エージェント開発における **異なる関心ごと** をそれぞれ担当します。

```
                    「何を知るか」
                    ┌──────────┐
                    │  Skill   │  構造化知識 / ドメインノウハウ
                    └──────────┘

「誰が実行するか」           「何に従うか」
┌──────────┐           ┌──────────┐
│  Agent   │           │  Rule   │  常時制約 / ガイドライン
│ Command  │           └──────────┘
└──────────┘

「いつ・何を自動化するか」    「どこと繋がるか」
┌──────────┐           ┌──────────┐
│  Hook    │           │  MCP    │  外部サービス接続
└──────────┘           └──────────┘
```

この分類は **「層に応じて適切な道具を選ぶ」** というベストプラクティスを体現しています。すべてを一つの巨大なプロンプトや単一スクリプトに詰め込むのではなく、関心ごとに分離することで、変更の局所化・再利用性・可読性が向上します。

### Claude Code が各コンポーネントに込めた設計意図

**Progressive Disclosure（スキル）**
スキルの本文は「必要なときだけ」コンテキストに読み込まれます。モデルはフロントマターの `description` だけを常時参照し、関連性を判断してから詳細を開きます。これによってコンテキストウィンドウの無駄遣いを防ぎ、トークン経済を最適化します。

**コンテキスト隔離（エージェント）**
サブエージェントはメインとは独立したコンテキストウィンドウで動作します。長大なコードベースを読み込んでレビューするような処理をサブエージェントに委譲すると、その詳細な中間処理はメインのコンテキストに残りません。返るのは要約だけです。これによってメインセッションを汚染せずに専門タスクを実行できます。

**決定論的実行（フック）**
フックは LLM の「判断」に依存しません。`settings.json` に登録された瞬間から、指定されたイベントが発火するたびにハーネスが **必ず** スクリプトを実行します。「フォーマットを忘れずに確認してね」という指示に頼るのではなく、PostToolUse フックで自動フォーマットを強制できます。品質ゲート・セキュリティチェック・ログ記録など「必ず実行しなければならない処理」はフックに任せるべきです。

**標準プロトコルによる拡張性（MCP）**
MCP は Anthropic が策定したオープンプロトコルです。任意の外部サービスをこのプロトコルに準拠したサーバとして実装すれば、Claude のネイティブツールと同等の扱いで利用できます。Claude Code とサービスの間の「接続の形」を標準化することで、ハーネスが変わっても同じサーバを使い回せます。

**常時有効な制約（ルール）**
LLM は指示を「忘れる」場合があります（コンテキスト圧縮・会話の長さ）。Rule ファイルは CLAUDE.md やルールファイルとして常時コンテキストに注入されるため、セキュリティ禁止事項や命名規約を「常に有効な制約」として維持できます。

### ECC はこの思想をどう踏襲・拡張したか

ECC は Claude Code の標準設計に加えて、以下の拡張を行っています。

| 拡張の観点 | ECC の実装 |
|---|---|
| スキルの移植性 | curated スキルは `skills/` に、生成・インポート済みは `~/.claude/skills/` に分離（[docs/SKILL-PLACEMENT-POLICY.md](../SKILL-PLACEMENT-POLICY.md)） |
| フックの安全な無効化 | `run-with-flags.js` ラッパーで `ECC_DISABLED_HOOKS` による実行時ゲーティングを実現 |
| クロスハーネス対応 | ワークフロー知識を `skills/` に集中させ、ハーネス固有のアダプタだけを分離（[01-philosophy.md](./01-philosophy.md) 参照） |
| エージェントとスキルの連携規約 | CLAUDE.md で「エージェントのプロンプトに必ず対応スキルの規約を注入する」と明文化 |
| プラグイン配布 | スキル・エージェント・コマンド・フック・MCP をワンセットとして複数ハーネスへ配布（[10-cross-harness-and-install.md](./10-cross-harness-and-install.md)） |

各コンポーネントの詳細な実装ガイドは以下の各章を参照してください。

- スキルの設計・フォーマット・配置ポリシー → [04-skills.md](./04-skills.md)
- エージェントの定義・委譲パターン → [05-agents.md](./05-agents.md)
- コマンドとルールの書き方 → [06-commands-and-rules.md](./06-commands-and-rules.md)
- フックの実装・安全なゲーティング → [07-hooks-and-runtime.md](./07-hooks-and-runtime.md)
- MCP の設定・サーバ選定 → [09-mcp-and-integrations.md](./09-mcp-and-integrations.md)
- クロスハーネス展開・インストール → [10-cross-harness-and-install.md](./10-cross-harness-and-install.md)

---

## 関連章

- [02-architecture.md](./02-architecture.md) — リポジトリ全体の構造
- [04-skills.md](./04-skills.md) — スキルの詳細
- [05-agents.md](./05-agents.md) — エージェントの詳細
- [06-commands-and-rules.md](./06-commands-and-rules.md) — コマンドとルールの詳細
- [07-hooks-and-runtime.md](./07-hooks-and-runtime.md) — フックとランタイムの詳細

## 参照ソース

- [`CLAUDE.md`](../../CLAUDE.md) — Architecture節、Skills表
- [`docs/COMMAND-AGENT-MAP.md`](../COMMAND-AGENT-MAP.md) — コマンド→エージェントマップ
- [`docs/SKILL-DEVELOPMENT-GUIDE.md`](../SKILL-DEVELOPMENT-GUIDE.md) — スキル vs エージェント vs コマンドの比較
- [`hooks/hooks.json`](../../hooks/hooks.json) — フック設定例
- [`mcp-configs/mcp-servers.json`](../../mcp-configs/mcp-servers.json) — MCP設定例
