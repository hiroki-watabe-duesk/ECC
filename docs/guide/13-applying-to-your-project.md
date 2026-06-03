# 第13章 自分のプロジェクトへの応用

## この章で学ぶこと

- 自分のリポジトリに最小限のハーネスを構築する方法
- コンポーネント設計の指針（単一責務・MECE・命名）
- 段階的導入ロードマップ（Day 1 / Week 1 / Month 1）
- 永続ロジックと固有ロジックの分離原則
- 自己改善ループ（観測 → 提案 → 検証 → 昇格 → ロールバック）
- ハーネスが正しく機能しているかの自己点検チェックリスト

---

## 13.1 Minimum Viable Harness（MVH）

ハーネスを構築する際の出発点は「最小限で動く」ことです。
以下の 5 要素を揃えることで、ECC の主要価値をすぐに享受できます。

### MVH の構成要素

| 要素 | ファイル | 役割 |
|------|---------|------|
| 基本指示 | `CLAUDE.md` | プロジェクト概要・実行コマンド・慣習 |
| エージェント指示 | `AGENTS.md` | 全エージェントへの共通指示 |
| 中核ルール | `rules/common/security.md` ほか | 常時適用のガイドライン |
| 中核エージェント | `agents/planner.md`, `agents/code-reviewer.md` | 計画・レビューの委譲先 |
| 安全フック | `hooks/hooks.json`（危険コマンドブロック） | 破壊的操作の防止 |

### 手順（新規プロジェクト）

1. **CLAUDE.md を作成する**

   ```markdown
   # CLAUDE.md

   ## Project Overview
   [プロジェクトの目的・技術スタックを簡潔に]

   ## Running Tests
   \`\`\`bash
   npm test
   \`\`\`

   ## Architecture
   [主要ディレクトリと役割]

   ## Key Commands
   [よく使うコマンド一覧]
   ```

2. **AGENTS.md を作成する**

   ```markdown
   # [Project Name] — Agent Instructions

   ## Core Principles
   1. Agent-First — 専門エージェントに委譲する
   2. Test-Driven — テストを先に書く
   3. Security-First — セキュリティを妥協しない

   ## Available Agents
   | Agent | Purpose | When to Use |
   |-------|---------|-------------|
   | planner | 実装計画 | 複雑な機能・リファクタリング |
   | code-reviewer | コードレビュー | コード変更後 |
   ```

3. **中核ルールをコピーする（または symlink）**

   ```bash
   # ECC からコピーする場合
   mkdir -p rules/common
   cp path/to/ecc/rules/common/security.md rules/common/
   cp path/to/ecc/rules/common/testing.md  rules/common/
   cp path/to/ecc/rules/common/git-workflow.md rules/common/
   ```

4. **中核エージェントを配置する**

   最低限 `planner.md` と `code-reviewer.md` を `agents/` に配置します。
   ECC の既存ファイルをそのままコピーするか、プロジェクト固有の指示を加えます。

5. **安全フックを設定する**

   `hooks/hooks.json` に最低限の危険コマンドブロックを追加します。

   ```json
   {
     "hooks": {
       "PreToolUse": [
         {
           "matcher": "Bash",
           "hooks": [
             {
               "type": "command",
               "command": "echo '[Hook] rm -rf / blocked' && exit 1",
               "match_pattern": "rm -rf /"
             }
           ],
           "description": "Block destructive rm commands"
         }
       ]
     }
   }
   ```

---

## 13.2 コンポーネント設計指針

### 単一責務の原則

各コンポーネントは「1 つのことを、それだけ」やる。

| 悪い例（広すぎる） | 良い例（絞られている） |
|------------------|--------------------|
| `backend-patterns` スキル（API + DB + 認証 + キャッシュ） | `postgres-patterns`, `api-design`, `auth-patterns` を別々に |
| `reviewer` エージェント（Python + TypeScript + Go すべて） | `python-reviewer`, `typescript-reviewer`, `go-reviewer` を別々に |
| `all-checks` フック（フォーマット + lint + セキュリティ） | `format-hook`, `lint-hook`, `security-hook` を分離 |

### MECE 設計

コンポーネントの集合全体で:
- **漏れなく**: 主要なユースケースがどこかのコンポーネントでカバーされている
- **重複なく**: 同じ責務を持つコンポーネントが複数存在しない

チェック方法:
1. ユースケース一覧を書き出す
2. 各ユースケースを担当するコンポーネントを割り当てる
3. 担当なし（漏れ）や複数担当（重複）を解消する

### 命名規約

```
<language/domain>-<topic/purpose>
例:
  python-security       ← 言語-トピック
  tdd-workflow          ← プロセス-目的
  block-dangerous-rm    ← 動詞-対象
  code-reviewer         ← 役割
```

- すべて lowercase-hyphen
- 略語は避ける（`ts` より `typescript`）
- 動詞は明確に（`check-`, `block-`, `format-`, `review-`）

---

## 13.3 段階的導入ロードマップ

### Day 1: 基盤設置

**目標**: Claude Code がプロジェクトを理解し、基本的なガイドラインを守る。

```
CLAUDE.md               ← プロジェクト概要と実行コマンド
AGENTS.md               ← 共通エージェント指示
rules/common/security.md ← セキュリティ必須事項
rules/common/testing.md  ← テスト方針
agents/code-reviewer.md  ← コードレビュー委譲
```

確認: Claude に `CLAUDE.md を読んでプロジェクトを説明して` と質問して、
正確な情報が返ってくるか確認する。

---

### Week 1: ワークフロー整備

**目標**: 主要な開発ワークフローを自動化する。

```
agents/planner.md       ← 実装計画
agents/tdd-guide.md     ← TDD ワークフロー
commands/plan.md        ← /plan コマンド
commands/code-review.md ← /code-review コマンド
hooks/hooks.json        ← 基本的な安全フック
```

追加するスキル（プロジェクトの技術スタックに合わせて選択）:
- `skills/tdd-workflow/` — TDD の手順
- `skills/security-review/` — セキュリティチェックリスト
- `skills/<language>-patterns/` — 言語パターン

確認: `/plan 新機能を追加する` でプランが生成されるか確認する。

---

### Month 1: 高度化と最適化

**目標**: 自己改善ループを回し、チーム全体に展開する。

```
# 追加コンポーネント
agents/<language>-reviewer.md     ← 言語特化レビュー
skills/<framework>-patterns/       ← フレームワークパターン
hooks/post-edit-format.js         ← 編集後フォーマット
rules/<language>/                 ← 言語別ルール

# 自己改善
/learn-eval                       ← セッションからパターン抽出
/evolve                           ← instinct から skill 昇格
/skill-create                     ← git 履歴から skill 生成
```

確認: `npm run harness:audit` で設定の品質スコアを確認する。

---

## 13.4 永続ロジックと固有ロジックの分離

### 基本原則

```
永続ロジック（共有層）          固有ロジック（エッジ）
─────────────────────────     ──────────────────────
共通パターン、ベストプラクティス   プロジェクト固有の制約
言語/フレームワーク知識          ビジネスドメインルール
セキュリティガイドライン          チーム固有の規約
テスト戦略                      特定サービスとの統合
```

### 適用例

**良い設計** — 永続ロジックは `rules/common/` や `skills/`、固有ロジックは `CLAUDE.md`:

```markdown
# CLAUDE.md（固有ロジック）

## Project-Specific Rules

- データベースは PostgreSQL 16 を使用（AWS RDS 上）
- マイグレーションは必ず `scripts/migrate.sh` 経由で実行する
- 本番デプロイ前に必ず `/code-review` を実行する

(以下は rules/common/ に委ねる)
- セキュリティ: rules/common/security.md を参照
- テスト方針: rules/common/testing.md を参照
```

**悪い設計** — 共通事項を CLAUDE.md に重複記述:

```markdown
# CLAUDE.md（アンチパターン）

## Security Rules
- APIキーをコードに書かない  ← rules/common/security.md の重複
- 全入力をバリデーションする  ← 重複
- パラメータ化クエリを使う    ← 重複
```

---

## 13.5 起動パターン

### パターン 1: Two-Instance Kickoff

複雑なタスクに対して、2 つの Claude セッションを使い分ける手法。

```
インスタンス A（足場づくり）      インスタンス B（深掘り）
────────────────────────────    ──────────────────────
/plan で実装計画を立てる         /docs で最新 API を調査する
既存コードを読んで把握する        外部ライブラリのベストプラクティスを調査
骨格コードを作成する             セキュリティ上の考慮事項を洗い出す
         ↓                                ↓
         ────────── 統合 ──────────
                   実装完了
```

### パターン 2: Groundwork（地ならし）

```bash
# セッション開始時に現状を把握させる
claude "CLAUDE.md を読んで、現在の実装状況を確認し、
        未完了タスクと次のアクションを提案してください"
```

### パターン 3: llms.txt 取り込み

外部ライブラリの最新ドキュメントをコンテキストに注入する:

```bash
# /docs コマンドで Context7 経由のドキュメント取得
/docs next.js app-router

# または直接 llms.txt を参照させる
claude "https://example.com/llms.txt を読んで、
        このプロジェクトへの適用方法を提案してください"
```

---

## 13.6 既存プロジェクトへの ECC 導入

ECC の `install.sh` を使うと、既存プロジェクトへのインストールが自動化されます。

### インストール手順

```bash
# ECC リポジトリを clone
git clone https://github.com/affaan-m/everything-claude-code.git
cd everything-claude-code

# プロファイルを選択してインストール
bash install.sh --target /path/to/your/project --profile developer

# または npm パッケージ経由
npx ecc typescript
```

### プロファイルの選択指針

| プロファイル | 含むモジュール | 適したプロジェクト |
|-------------|--------------|-----------------|
| `minimal` | ルール・エージェント・コマンド（フックなし） | 初めて導入する、リスク最小化 |
| `core` | minimal ＋ フックランタイム | 標準的な開発プロジェクト |
| `developer` | core ＋ フレームワーク・DB・オーケストレーション | アプリ開発チーム（推奨） |
| `security` | core ＋ セキュリティ特化 | セキュリティが重要なプロジェクト |
| `full` | すべてのモジュール | 包括的に利用したいチーム |

プロファイルの詳細は [10 章](./10-cross-harness-and-install.md) を参照してください。

---

## 13.7 自己改善ループ

ECC は「観測 → 提案 → 検証 → 昇格 → ロールバック」のループで自律的に改善します。
（`docs/ECC-2.0-REFERENCE-ARCHITECTURE.md` の Self-Improving Harness Loop 参照）

### ループの各フェーズ

```
┌─────────────────────────────────────────────┐
│ 1. 観測 (Observation)                       │
│    observe-runner.js がツール使用を記録      │
│    evaluate-session.js がセッションを分析    │
├─────────────────────────────────────────────┤
│ 2. 提案 (Proposal)                          │
│    /learn-eval でパターンを抽出             │
│    instinct として confidence 付きで保存     │
├─────────────────────────────────────────────┤
│ 3. 検証 (Verification)                      │
│    verifier が提案をシナリオで検証           │
│    blast radius（影響範囲）を評価           │
├─────────────────────────────────────────────┤
│ 4. 昇格 (Promotion)                         │
│    /evolve で instinct を Skill に変換       │
│    /promote でプロジェクトスコープを昇格    │
├─────────────────────────────────────────────┤
│ 5. ロールバック (Rollback)                  │
│    不適切な提案は reject して保留           │
│    元の状態に戻す（git で管理）             │
└─────────────────────────────────────────────┘
```

### 実践的な使い方

```bash
# セッション終了時にパターンを記録
/learn-eval

# 蓄積されたインスタイトを確認
/instinct-status

# インスタイトをスキルに昇格
/evolve

# プロジェクトスコープのインスタイトをグローバルへ
/promote

# git 履歴からスキルを自動生成
/skill-create
```

---

## 13.8 ハーネスエンジニアリング自己点検チェックリスト

以下のチェックリストで自分のハーネスを定期的に評価してください。

### 基盤

- [ ] `CLAUDE.md` にプロジェクト概要・実行コマンド・慣習が記述されている
- [ ] `AGENTS.md` に共通エージェント指示が記述されている
- [ ] `rules/common/security.md` が存在し、最低限のセキュリティ要件が定義されている
- [ ] `rules/common/testing.md` が存在し、テスト方針が定義されている

### エージェント

- [ ] `planner` エージェントが存在し、複雑なタスクを計画できる
- [ ] `code-reviewer` エージェントが存在し、コード変更後に呼ばれる
- [ ] 言語固有のレビュアーが主要言語をカバーしている
- [ ] 各エージェントの `description` が「いつ呼ぶか」を具体的に示している

### スキル

- [ ] 主要な技術スタックに対応するスキルが存在する
- [ ] `When to Activate` が具体的で、自動起動に使える
- [ ] スキルに重複がない（MECE）
- [ ] スキルが単一のドメインに絞られている（単一責務）

### フック

- [ ] 破壊的コマンド（`rm -rf /` 等）がブロックされている
- [ ] セッション開始時に前回のコンテキストがロードされる
- [ ] フックが非クリティカルエラーで `exit 1` していない

### コマンド

- [ ] `/plan`, `/code-review`, `/tdd` などの主要コマンドが機能する
- [ ] `/save-session`, `/resume-session` でセッションが永続化できる

### CI / 品質

- [ ] `node tests/run-all.js` がクリーンに通る
- [ ] `npm run catalog:sync` が最新の状態を反映している
- [ ] Markdown lint が通っている

### 自己改善

- [ ] `/learn-eval` でセッションからパターンを抽出できる
- [ ] `~/.claude/skills/learned/` にスキルが蓄積されている
- [ ] `/instinct-status` でインスタイトを確認できる

---

## 関連章

- [02 アーキテクチャ](./02-architecture.md)
- [10 クロスハーネスとインストール](./10-cross-harness-and-install.md)
- [12 コンポーネントの作成](./12-authoring-components.md)
- [14 リファレンス・用語集](./14-reference-glossary.md)

## 参照ソース

- `CLAUDE.md`
- `AGENTS.md`
- `docs/ECC-2.0-REFERENCE-ARCHITECTURE.md`
- `docs/architecture/cross-harness.md`
- `manifests/install-profiles.json`
- `scripts/hooks/session-start.js`（自己改善ループの実装）
- `scripts/hooks/observe-runner.js`（観測フェーズ）
