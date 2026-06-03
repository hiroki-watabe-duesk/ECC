# 第03章 コンポーネント体系と選択基準

この章で学ぶこと: ECCが提供する6種類のコンポーネント（Skill / Agent / Command / Rule / Hook / MCP）の役割と違い、コンポーネント同士がどう連携するか、そして「ある要件をどのコンポーネントに実装すべきか」を判断するための基準を習得します。

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
